# LORAS Furniture — Infrastructure

This repository contains all infrastructure-as-code for the LORAS Furniture website:

- `backend/Dockerfile` — multi-stage build for the Spring Boot backend
- `frontend/Dockerfile` — multi-stage build for the React/Vite frontend served by nginx
- `frontend/nginx.conf` — nginx config with SPA routing, API proxy, security headers, and cache policy
- `docker-compose.yml` — local development stack (Postgres + backend + frontend)
- `cloudbuild.yaml` — Cloud Build CI/CD pipeline (build → migrate → deploy)
- `backend/.env.example` — backend environment variable template
- `frontend/.env.example` — frontend environment variable template

---

## 1. Local Development Setup

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine + Compose plugin)
- Git

### Clone the application repositories

The `docker-compose.yml` expects the backend and frontend source code in `./backend` and `./frontend` relative to this repo:

```bash
# Clone all three repos side-by-side, then enter this one
git clone https://github.com/your-org/loras-backend  ./backend
git clone https://github.com/your-org/loras-frontend ./frontend
```

### Configure environment variables

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Edit `backend/.env` at minimum:
- `JWT_SECRET` — generate with `openssl rand -base64 64`
- `MAIL_USERNAME` / `MAIL_PASSWORD` — use [Mailtrap](https://mailtrap.io/) or leave blank to skip email delivery locally
- `GCS_BUCKET_NAME` — set to a dev GCS bucket if you want to test image uploads, otherwise any placeholder string works for non-upload features

The rest of the database variables already point to the local Docker Compose Postgres container and do not need to be changed.

### Start all services

```bash
docker compose up --build
```

| Service  | URL                           | Notes                              |
|----------|-------------------------------|------------------------------------|
| Frontend | http://localhost:5173         | React SPA served by nginx          |
| Backend  | http://localhost:8080/api/v1  | Spring Boot REST API               |
| Database | localhost:5432                | Postgres 15 (user: loras)          |

Flyway migrations run automatically when the backend starts. The seed data creates default categories and an admin user (`admin` / `loras2024`).

### First login

1. Open http://localhost:5173/admin/login
2. Log in with `admin` / `loras2024`
3. You will be immediately prompted to change the password before accessing the admin panel

### Stopping

```bash
docker compose down          # stop but keep the database volume
docker compose down -v       # stop and delete all data (clean slate)
```

---

## 2. GCP Prerequisites

The following GCP services must be provisioned before the first Cloud Build run.  All commands use `gcloud` — ensure you are authenticated (`gcloud auth login`) and targeting the correct project (`gcloud config set project PROJECT_ID`).

### 2.1 Enable required APIs

```bash
gcloud services enable \
  run.googleapis.com \
  sqladmin.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  storage.googleapis.com \
  iam.googleapis.com
```

### 2.2 Create the Artifact Registry repository

```bash
gcloud artifacts repositories create loras \
  --repository-format=docker \
  --location=europe-west1 \
  --description="LORAS Furniture Docker images"
```

### 2.3 Create the Cloud SQL (PostgreSQL 15) instance

```bash
# Create the instance (~5 minutes)
gcloud sql instances create loras-pg \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=europe-west1 \
  --no-assign-ip \
  --network=default

# Create the database
gcloud sql databases create loras --instance=loras-pg

# Create the database user
gcloud sql users create loras \
  --instance=loras-pg \
  --password=STRONG_PASSWORD_HERE
```

Note the instance connection name (`PROJECT_ID:europe-west1:loras-pg`) — you will need it for `_CLOUD_SQL_INSTANCE` in the Cloud Build trigger substitutions.

### 2.4 Create the GCS media bucket

```bash
# Create the bucket (choose a globally unique name)
gcloud storage buckets create gs://loras-media \
  --location=europe-west1 \
  --uniform-bucket-level-access

# Make objects publicly readable (portfolio images are public)
gcloud storage buckets add-iam-policy-binding gs://loras-media \
  --member=allUsers \
  --role=roles/storage.objectViewer

# Enable CORS so the browser can PUT directly from the signed URL flow
cat > /tmp/cors.json <<'EOF'
[
  {
    "origin": ["https://your-frontend-domain.run.app", "http://localhost:5173"],
    "method": ["GET", "PUT", "HEAD"],
    "responseHeader": ["Content-Type"],
    "maxAgeSeconds": 3600
  }
]
EOF
gcloud storage buckets update gs://loras-media --cors-file=/tmp/cors.json
```

### 2.5 Create the backend service account

```bash
# Create the service account
gcloud iam service-accounts create loras-backend \
  --display-name="LORAS Backend Cloud Run SA"

SA="loras-backend@${PROJECT_ID}.iam.gserviceaccount.com"

# Cloud SQL client (required for the Cloud SQL Auth Proxy connector)
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA}" \
  --role=roles/cloudsql.client

# GCS object admin (for signed URL generation and object deletion)
gcloud storage buckets add-iam-policy-binding gs://loras-media \
  --member="serviceAccount:${SA}" \
  --role=roles/storage.objectAdmin

# Secret Manager accessor (to read the secrets mounted at runtime)
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA}" \
  --role=roles/secretmanager.secretAccessor
```

### 2.6 Create Secret Manager secrets

```bash
# Database password
echo -n "STRONG_DB_PASSWORD" | \
  gcloud secrets create loras-db-password --data-file=-

# JWT secret (minimum 256-bit / 32 bytes of entropy, Base64-encoded)
openssl rand -base64 64 | tr -d '\n' | \
  gcloud secrets create loras-jwt-secret --data-file=-

# Gmail app password (Settings > Security > App passwords)
echo -n "GMAIL_APP_PASSWORD" | \
  gcloud secrets create loras-mail-password --data-file=-

# Gmail sender address
echo -n "sender@gmail.com" | \
  gcloud secrets create loras-mail-username --data-file=-
```

Grant the Cloud Build service account access to read secrets during the migration step:

```bash
BUILD_SA="${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${BUILD_SA}" \
  --role=roles/secretmanager.secretAccessor
```

Also grant Cloud Build permission to deploy Cloud Run services and act as the backend service account:

```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${BUILD_SA}" \
  --role=roles/run.admin

gcloud iam service-accounts add-iam-policy-binding \
  loras-backend@${PROJECT_ID}.iam.gserviceaccount.com \
  --member="serviceAccount:${BUILD_SA}" \
  --role=roles/iam.serviceAccountUser
```

### 2.7 Configure Cloud Build trigger

In the Cloud Build console (or via `gcloud builds triggers create`), point the trigger to this repository and set the following substitution variables:

| Variable             | Example value                                             |
|----------------------|-----------------------------------------------------------|
| `_REGION`            | `europe-west1`                                            |
| `_PROJECT_ID`        | `loras-prod`                                              |
| `_AR_REPO`           | `europe-west1-docker.pkg.dev/loras-prod/loras`            |
| `_CLOUD_SQL_INSTANCE`| `loras-prod:europe-west1:loras-pg`                        |
| `_DB_NAME`           | `loras`                                                   |
| `_DB_USER`           | `loras`                                                   |
| `_SERVICE_ACCOUNT`   | `loras-backend@loras-prod.iam.gserviceaccount.com`        |
| `_BACKEND_URL`       | *(fill in after first backend deploy)*                    |
| `_FRONTEND_URL`      | *(fill in after first frontend deploy)*                   |

`_BACKEND_URL` and `_FRONTEND_URL` are the Cloud Run service URLs.  On the very first deploy they are unknown — deploy once with placeholder values, copy the generated URLs from the Cloud Run console, then update the substitutions and redeploy.

---

## 3. First Deployment

```bash
cd /path/to/loras-infra

gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=\
_REGION=europe-west1,\
_PROJECT_ID=loras-prod,\
_AR_REPO=europe-west1-docker.pkg.dev/loras-prod/loras,\
_CLOUD_SQL_INSTANCE=loras-prod:europe-west1:loras-pg,\
_DB_NAME=loras,\
_DB_USER=loras,\
_SERVICE_ACCOUNT=loras-backend@loras-prod.iam.gserviceaccount.com,\
_BACKEND_URL=https://placeholder.run.app,\
_FRONTEND_URL=https://placeholder.run.app
```

After the first run:
1. Find the backend Cloud Run URL in the console: `https://loras-backend-<hash>-ew.a.run.app`
2. Find the frontend Cloud Run URL: `https://loras-frontend-<hash>-ew.a.run.app`
3. Update the trigger substitutions with the real URLs
4. Update `ALLOWED_ORIGINS` on the backend Cloud Run service to include the frontend URL
5. Trigger a second build so the frontend bundle is rebuilt with the correct `VITE_API_BASE_URL`

Subsequent deployments happen automatically when code is pushed to `main`.

---

## 4. Rotating the Admin Password

The seed migration sets the admin password to `loras2024` with `must_change_password=true`.  The UI enforces a change on first login.  If you need to reset the password manually:

1. Connect to Cloud SQL via Cloud Shell or the Auth Proxy:
   ```bash
   gcloud sql connect loras-pg --user=loras --database=loras
   ```

2. Generate a bcrypt hash (use Spring Security's `BCryptPasswordEncoder` strength 10, or a tool like [bcrypt-generator.com](https://bcrypt-generator.com)):

3. Update the row:
   ```sql
   UPDATE admin_users
   SET password_hash = '$2a$10$YOUR_NEW_HASH_HERE',
       must_change_password = true
   WHERE username = 'admin';
   ```

4. The next login will force a password-change screen.

---

## 5. Adding English Translations

The frontend ships with Armenian (`hy`) and Russian (`ru`) fully translated.  English (`en`) locale files exist with placeholder values (`"KEY": "TODO"`) and fall back to English strings automatically.

To complete the English translations:

1. Open each locale file in the frontend repository:
   ```
   public/locales/en/common.json
   public/locales/en/portfolio.json
   public/locales/en/contact.json
   public/locales/en/admin.json
   ```

2. Replace every `"TODO"` value with the appropriate English string.  The Armenian (`hy`) files are the reference for what each key means.

3. Commit, push, and Cloud Build will rebuild and redeploy automatically.

There is no code change required — i18next picks up the locale files at runtime via HTTP.

---

## 6. Optional: Adding Cloud CDN in Front of GCS

At MVP, GCS is accessed directly without a CDN.  For production traffic, Cloud CDN in front of the GCS bucket significantly reduces latency for image-heavy pages.

### Setup

```bash
# Create a backend bucket pointing at the GCS bucket
gcloud compute backend-buckets create loras-media-cdn \
  --gcs-bucket-name=loras-media \
  --enable-cdn

# Create a URL map
gcloud compute url-maps create loras-media-lb \
  --default-backend-bucket=loras-media-cdn

# Create an HTTPS proxy and forwarding rule
gcloud compute target-https-proxies create loras-media-proxy \
  --url-map=loras-media-lb \
  --ssl-certificates=CERT_NAME

gcloud compute forwarding-rules create loras-media-fwd \
  --load-balancing-scheme=EXTERNAL \
  --target-https-proxy=loras-media-proxy \
  --global \
  --ports=443
```

After setup:
- Use the CDN's IP/domain as the base URL for media assets instead of `https://storage.googleapis.com/loras-media/`
- Update the CSP `img-src` directive in `frontend/nginx.conf` to include the CDN domain
- Redeploy the frontend

### Cache invalidation

When a GCS object is deleted or replaced, Cloud CDN may serve stale content for up to the object's `Cache-Control` max-age.  Use UUID-based object paths (as implemented) to make deletions safe — old URLs simply return 404 from the CDN after expiry.  For immediate invalidation:

```bash
gcloud compute url-maps invalidate-cdn-cache loras-media-lb \
  --path="/images/*"
```

---

## Architecture reference

```
Browser
  │
  ├── Static assets (HTML/JS/CSS) ◄── nginx on Cloud Run (loras-frontend)
  │       /assets/*   Cache-Control: public, max-age=31536000, immutable
  │       /locales/*  Cache-Control: no-cache
  │       index.html  Cache-Control: no-cache
  │
  ├── /api/* ──────────────────────► Spring Boot on Cloud Run (loras-backend)
  │                                        │
  │                                        ├── Cloud SQL PostgreSQL 15
  │                                        │   (via Cloud SQL Auth Proxy connector)
  │                                        ├── GCS (signed URL generation only)
  │                                        └── SMTP / Gmail (outbound email)
  │
  └── Direct GCS upload (signed URL PUT — bypasses backend, no size limit)
            │
            └── gs://loras-media  (public read, admin write via signed URLs)
```

# ============================================================
# Stage 1: Builder
# ============================================================
FROM node:20-alpine AS builder

# VITE_API_BASE_URL is embedded into the JS bundle at build time.
# Pass it via: docker build --build-arg VITE_API_BASE_URL=https://your-backend.run.app
ARG VITE_API_BASE_URL
ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}

WORKDIR /app

# Copy dependency manifests first for better layer caching
COPY package.json package-lock.json* ./
RUN npm ci --prefer-offline

# Copy the rest of the source and build
COPY . .
RUN npm run build

# ============================================================
# Stage 2: Runner
# ============================================================
FROM nginx:alpine AS runner

# Remove default nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy nginx config as a template — the official nginx image's
# 20-envsubst-on-templates.sh will substitute ${BACKEND_URL} at
# container start time before nginx is spawned.
COPY nginx.conf /etc/nginx/templates/default.conf.template

# Copy Vite build output
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

# BACKEND_URL must be set at runtime, e.g.:
#   docker run -e BACKEND_URL=http://loras-backend:8080 loras-frontend
# In Cloud Run this is set via --set-env-vars in the deploy step.

# ============================================================
# Stage 1: Builder
# ============================================================
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Copy dependency manifest first for better layer caching
COPY pom.xml .
# Download dependencies in a separate layer (cached unless pom.xml changes)
RUN --mount=type=cache,target=/root/.m2 \
    mvn -q dependency:go-offline -f pom.xml 2>/dev/null || true

COPY src ./src

RUN --mount=type=cache,target=/root/.m2 \
    mvn -q package -DskipTests

# ============================================================
# Stage 2: Runner
# ============================================================
FROM eclipse-temurin:21-jre-alpine AS runner

# Create a non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy the fat JAR from the builder stage
COPY --from=builder /build/target/*.jar app.jar

# Set ownership
RUN chown appuser:appgroup app.jar

USER appuser

EXPOSE 8080

# Health check via Spring Boot Actuator
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]

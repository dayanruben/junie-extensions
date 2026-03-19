---
name: "docker-patterns"
description: "Docker and Docker Compose best practices: multi-stage builds, layer caching, security. Use when working with Dockerfile or docker-compose.yml."
---

# Docker Patterns Skill

Write efficient, secure Docker configurations.

## Scope and Boundaries

- Use this skill for Dockerfile optimization, Docker Compose, and container security.
- Use `kubernetes-patterns` for Kubernetes manifests and orchestration.
- Use `ci-cd-patterns` for building and pushing images in CI pipelines.
- Treat examples as starting points for JVM applications; adapt base images and build tools to the current project.

## When to Use
- Creating or reviewing `Dockerfile`
- Working with `docker-compose.yml`
- Optimizing image size or build time

---

## Multi-stage Builds (JVM)

```dockerfile
# ✅ Good — multi-stage for smaller final image
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY gradlew settings.gradle.kts build.gradle.kts ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon
COPY src ./src
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Layer Caching

Order instructions from least to most frequently changed.

```dockerfile
# ✅ Good — dependencies cached separately
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon
COPY src ./src          # only invalidates cache when source changes
RUN ./gradlew bootJar

# ❌ Bad — COPY . . invalidates cache on any file change
COPY . .
RUN ./gradlew bootJar
```

---

## Security

```dockerfile
# ✅ Good — run as non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# ✅ Good — use specific image tags
FROM eclipse-temurin:21.0.2_13-jre

# ❌ Bad — latest tag is unpredictable
FROM openjdk:latest
```

---

## Docker Compose

```yaml
# ✅ Good
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

---

## .dockerignore

Always create a `.dockerignore` to keep the build context small:

```
.git
.gradle
build/
*.md
.idea/
.env
```

---

## Health Checks in Dockerfile

```dockerfile
# ✅ Good — container-level health check (useful with Docker Compose)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

---

## JVM Tuning in Docker

```dockerfile
# ✅ Good — respect container memory limits
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

---

## Anti-Patterns

- Using `COPY . .` before dependency resolution — invalidates cache on every source change.
- Running containers as root — always add a non-root `USER`.
- Using `FROM openjdk:latest` — unpredictable, unpatched images. Pin a specific tag.
- Missing `.dockerignore` — sends `.git`, `build/`, and IDE files to the Docker daemon.
- Hardcoding secrets in `ENV` — use runtime environment variables or secret mounts (`--mount=type=secret`).

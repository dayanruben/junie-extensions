---
name: "ci-cd-patterns"
description: "CI/CD pipeline best practices for GitHub Actions. Use when working with pipeline configuration files."
---

# CI/CD Patterns Skill

Write efficient, reliable CI/CD pipelines.

## Scope and Boundaries

- Use this skill for GitHub Actions workflow configuration, caching, secrets, and Docker image publishing.
- Use `docker-patterns` for Dockerfile optimization and Docker Compose.
- Use `kubernetes-patterns` for deployment manifests and health checks.
- Treat examples as starting points; adapt runners, JDK versions, and caching strategies to the current project.

## When to Use
- Creating or reviewing GitHub Actions workflows
- Optimizing build times

---

## GitHub Actions — JVM Project

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: gradle

      - name: Build and test
        run: ./gradlew build --no-daemon

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: build/reports/tests/
```

---

## Caching Dependencies

```yaml
# ✅ Gradle cache
- uses: actions/setup-java@v4
  with:
    cache: gradle   # built-in Gradle cache

# ✅ Manual cache
- uses: actions/cache@v4
  with:
    path: ~/.gradle/caches
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle.kts') }}
    restore-keys: ${{ runner.os }}-gradle-
```

---

## Secrets Management

```yaml
# ✅ Good — use GitHub Secrets
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}

# ❌ Bad — hardcoded secrets
env:
  DATABASE_URL: "postgresql://user:password@host/db"
```

---

## Docker Build & Push

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: ${{ github.ref == 'refs/heads/main' }}
    tags: |
      myapp:latest
      myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## Matrix Builds

```yaml
# ✅ Test against multiple JDK versions
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: ['17', '21']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: 'temurin'
          cache: gradle
      - run: ./gradlew test --no-daemon
```

---

## Reusable Workflows

```yaml
# .github/workflows/reusable-build.yml
on:
  workflow_call:
    inputs:
      java-version:
        type: string
        default: '21'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ inputs.java-version }}
          distribution: 'temurin'
          cache: gradle
      - run: ./gradlew build --no-daemon
```

```yaml
# Caller workflow
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      java-version: '21'
```

---

## Anti-Patterns

- Running the full test suite on every push to every branch — use path filters or `pull_request` triggers.
- Hardcoding secrets in workflow files — always use `${{ secrets.* }}`.
- Not caching dependencies — Gradle/Maven downloads add 1-3 minutes per run.
- Using `latest` tags for actions — pin to major versions (`@v4`) for stability.
- Running deployment steps without environment protection rules — use GitHub Environments with required reviewers for production.

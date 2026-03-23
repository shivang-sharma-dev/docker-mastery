# Project: CI/CD Pipeline with Docker

**Level:** Intermediate
**Time:** 3–4 hours
**Covers:** Notes 08–10 (building, multi-stage, production)

---

## What You're Building

A complete automated pipeline that:
1. Tests your application in a Docker container
2. Builds a production-optimised image
3. Pushes it to a registry
4. Deploys it

---

## The Pipeline

```
git push
    ↓
GitHub Actions triggered
    ↓
Build test image → Run tests → Fail if tests fail
    ↓ (only if tests pass)
Build production image (multi-stage, minimal)
    ↓
Scan for vulnerabilities
    ↓
Push to Docker Hub (or ECR)
    ↓
Deploy: pull new image, restart container
```

---

## Requirements

**Dockerfile (multi-stage):**
```dockerfile
# Stage 1: test
FROM python:3.11-slim AS test
WORKDIR /app
COPY requirements*.txt ./
RUN pip install -r requirements.txt -r requirements-dev.txt
COPY . .
RUN pytest tests/ -v

# Stage 2: production
FROM python:3.11-slim AS production
# ... (full production Dockerfile from note 09)
```

**GitHub Actions workflow (`.github/workflows/deploy.yml`):**
- Trigger on push to `main`
- Build test stage — fail if tests fail
- Build production stage
- Scan with Trivy — fail on CRITICAL CVEs
- Push to Docker Hub with two tags: `latest` and the git commit SHA
- SSH to your server and pull the new image

**Production container must have:**
- Memory limit: 256MB
- CPU limit: 0.5
- Health check
- Restart policy: `unless-stopped`
- Log rotation configured

---

## Deliverables

1. `Dockerfile` with test and production stages
2. `.github/workflows/deploy.yml`
3. `docker-compose.prod.yml` for production deployment
4. Screenshot or log showing a successful pipeline run
5. `README.md` documenting the pipeline

---

## Stretch Goals

- Add a staging deployment step before production
- Add rollback: if health check fails after deployment, redeploy previous image
- Add notification (Slack/Discord) on success and failure

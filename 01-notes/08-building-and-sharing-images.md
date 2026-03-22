# 08 — Building and Sharing Images

> Building an image locally is half the story.
> The other half is getting it to production — tagging it
> meaningfully, pushing it to a registry, and pulling it
> reliably in your CI/CD pipeline.

---

## Image Tagging Strategy

A tag is a human-readable pointer to a specific image version.
How you tag matters because your deployment pipeline and rollback
strategy depend on it.

```bash
# Bad tagging — ambiguous and dangerous
docker build -t myapp .            # implies :latest
docker build -t myapp:latest .     # latest is a moving target

# Good tagging — precise and traceable
docker build -t myapp:v2.4.1 .                          # semantic version
docker build -t myapp:$(git rev-parse --short HEAD) .   # git commit hash
docker build -t myapp:2024.01.15 .                      # date-based

# Best — tag with both version AND environment
docker build \
  -t mycompany/myapp:v2.4.1 \
  -t mycompany/myapp:latest \
  .
```

### Tagging after the build
```bash
# Add a new tag to an existing local image
docker tag myapp:v2.4.1 mycompany/myapp:v2.4.1
docker tag myapp:v2.4.1 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v2.4.1
docker tag myapp:v2.4.1 myapp:latest    # update latest pointer
```

---

## Registries — Where Images Live

```
PUBLIC REGISTRIES
  Docker Hub (hub.docker.com)
    Free for public images.
    Rate-limited for pulls (100/6h anonymous, 200/6h authenticated).
    Namespace: username/imagename or organisation/imagename.

  GitHub Container Registry (ghcr.io)
    Integrated with GitHub Actions.
    Free for public repositories.
    ghcr.io/username/imagename

  Quay.io
    Red Hat's registry. Popular for enterprise.

PRIVATE REGISTRIES (for production use)
  AWS ECR (Elastic Container Registry)
    Best if your workloads run on AWS (ECS, EKS).
    IAM-based authentication — no passwords to manage.
    Automatic scanning for vulnerabilities.

  GCP Artifact Registry
    Best if your workloads run on GCP (GKE, Cloud Run).

  Azure Container Registry (ACR)
    Best for Azure workloads.

  Self-hosted: Harbor, Gitea, plain Docker Registry
    Full control. Higher operational burden.
```

---

## Docker Hub

```bash
# Login
docker login
# Username: yourusername
# Password: yourpassword or access token

# Push an image
# Image must be tagged as: username/imagename:tag
docker tag myapp:v2.4.1 yourusername/myapp:v2.4.1
docker push yourusername/myapp:v2.4.1
docker push yourusername/myapp:latest

# Pull
docker pull yourusername/myapp:v2.4.1

# Logout
docker logout
```

### Use access tokens, not passwords
Docker Hub supports access tokens with limited scope.
Generate one at https://hub.docker.com/settings/security
Use it instead of your password — it can be revoked independently.

---

## AWS ECR

```bash
# Authenticate (requires AWS CLI configured)
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create a repository (one-time)
aws ecr create-repository --repository-name myapp --region us-east-1

# Tag image for ECR
REGISTRY=123456789012.dkr.ecr.us-east-1.amazonaws.com
docker tag myapp:v2.4.1 $REGISTRY/myapp:v2.4.1

# Push
docker push $REGISTRY/myapp:v2.4.1

# Pull (on another machine, after auth)
docker pull $REGISTRY/myapp:v2.4.1
```

---

## Building in CI/CD

The standard pipeline: push code → CI builds image → push to registry → deploy.

```yaml
# GitHub Actions example
# .github/workflows/build-push.yml

name: Build and Push

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            yourusername/myapp:latest
            yourusername/myapp:${{ github.sha }}
          cache-from: type=registry,ref=yourusername/myapp:buildcache
          cache-to: type=registry,ref=yourusername/myapp:buildcache,mode=max
```

The `cache-from`/`cache-to` lines enable registry-based build cache.
Without this, every CI build starts from scratch — slow and expensive.
With this, unchanged layers are pulled from the registry instead of rebuilt.

---

## Image Scanning — Find Vulnerabilities Before Deploying

```bash
# Docker Scout (built into Docker Desktop and CLI)
docker scout cves myapp:v2.4.1
docker scout recommendations myapp:v2.4.1

# Trivy — open source, very thorough
# Install: https://aquasecurity.github.io/trivy
trivy image myapp:v2.4.1
trivy image --severity HIGH,CRITICAL myapp:v2.4.1
trivy image --exit-code 1 --severity CRITICAL myapp:v2.4.1
# exit-code 1 fails the CI build if critical vulns found

# Snyk (free tier available)
snyk container test myapp:v2.4.1
```

Integrate scanning into your CI pipeline so vulnerabilities are
caught before they reach production, not after.

---

## Cleaning Up

```bash
# Images accumulate fast. Clean up regularly.

# Remove a specific image
docker rmi myapp:old-version

# Remove dangling images (untagged layers from old builds)
docker image prune

# Remove all unused images (not referenced by any container)
docker image prune -a

# Nuclear option — remove EVERYTHING not currently running
docker system prune -a

# See what is using space
docker system df
docker system df -v
```

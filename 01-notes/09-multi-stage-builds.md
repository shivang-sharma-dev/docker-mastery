# 09 — Multi-Stage Builds

> A single-stage build includes everything needed to BUILD your app
> in the final image. That means compilers, build tools, test
> dependencies — all shipped to production. Multi-stage builds
> fix this by separating build-time tools from runtime artifacts.

---

## The Problem With Single-Stage Builds

```dockerfile
# Single-stage Python build
FROM python:3.11

RUN apt-get update && apt-get install -y \
    gcc g++ make libpq-dev      # build dependencies for psycopg2

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .
CMD ["python", "app.py"]

# Final image contains:
# Python runtime            ~50MB
# gcc, g++, make           ~150MB   ← never needed at runtime
# All build artifacts       ~20MB
# Your application          ~5MB
# Total:                   ~225MB
```

Those build tools are dead weight in production. They make the
image larger (slower to pull, more bandwidth, more storage costs)
and they expand the attack surface — a vulnerability in gcc
in a production image is a real risk.

---

## The Multi-Stage Solution

```dockerfile
# Stage 1: Build
FROM python:3.11 AS builder

RUN apt-get update && apt-get install -y gcc libpq-dev

COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Stage 2: Runtime — starts from a clean base
FROM python:3.11-slim

# Copy only the installed packages from the builder
COPY --from=builder /root/.local /root/.local

COPY . /app
WORKDIR /app

ENV PATH=/root/.local/bin:$PATH

CMD ["python", "app.py"]

# Final image contains:
# Python slim runtime       ~50MB
# Installed packages        ~20MB
# Your application          ~5MB
# Total:                    ~75MB   ← 3x smaller, no build tools
```

The key instruction is `COPY --from=builder`. It copies files from
a previous stage into the current stage. The builder stage is
discarded. Its layers never appear in the final image.

---

## A Go Binary — Extreme Reduction

Go compiles to a static binary. The runtime image can be `scratch`
(completely empty — no OS, no shell, nothing).

```dockerfile
# Stage 1: Compile
FROM golang:1.21 AS builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# Stage 2: Runtime — completely empty base
FROM scratch

COPY --from=builder /build/server /server

# If your app makes HTTPS requests, you need CA certificates
# scratch has none, so copy them from the builder
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080
CMD ["/server"]

# Final image size: ~8MB (just the binary + certs)
# vs golang:1.21 base: ~900MB
# Reduction: 99%
```

---

## Node.js — Separating Build and Runtime

```dockerfile
# Stage 1: Install all deps (including devDependencies) and build
FROM node:20 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci                         # installs dev + prod deps
COPY . .
RUN npm run build                  # compile TypeScript, bundle, etc.

# Stage 2: Production runtime — only prod deps
FROM node:20-slim AS runtime

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production       # production deps only

COPY --from=builder /app/dist ./dist   # built output from Stage 1

USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## Test Stage — Run Tests in CI, Not Production

```dockerfile
FROM python:3.11-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

# Test stage — has test deps, runs tests
FROM base AS test
RUN pip install --no-cache-dir pytest coverage
RUN pytest tests/ --coverage-report=term-missing

# Production stage — no test tools
FROM base AS production
RUN useradd -r -u 1001 appuser && chown -R appuser /app
USER appuser
CMD ["gunicorn", "app:create_app()", "-b", "0.0.0.0:8000"]
```

```bash
# In CI: build to test stage — fails pipeline if tests fail
docker build --target test -t myapp:test .

# For production: build to production stage
docker build --target production -t myapp:prod .
```

---

## Build Cache Optimisation in Multi-Stage Builds

Each stage has its own independent layer cache. The same rules apply:
put frequently-changing instructions at the bottom of each stage.

```dockerfile
FROM node:20 AS builder

WORKDIR /app

# Dependencies change less often than source code
COPY package*.json ./
RUN npm ci                          # cached unless package.json changes

COPY . .                            # changes on every commit
RUN npm run build                   # only re-runs when source changes

FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production        # independent cache from builder stage
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

---

## Using External Images as Stages

You can copy from any image, not just stages you defined:

```dockerfile
FROM python:3.11-slim

# Copy compiled binary from an official image
COPY --from=golang:1.21 /usr/local/go/bin/go /usr/local/bin/go

# Copy a specific version of a tool from its official image
COPY --from=aquasec/trivy:latest /usr/local/bin/trivy /usr/local/bin/trivy
COPY --from=hashicorp/terraform:1.6 /bin/terraform /usr/local/bin/terraform
```

This is a clean way to include tools without installing them
(and their dependencies) via apt — the binary is simply copied in.

---

## Image Size Comparison

These are real measurements showing what multi-stage builds achieve:

```
Python Flask app:
  Single stage (python:3.11)        →  950MB
  Single stage (python:3.11-slim)   →  200MB
  Multi-stage (slim + no build deps)→   85MB

Node.js Express app:
  Single stage (node:20)            →  980MB
  Single stage (node:20-slim)       →  200MB
  Multi-stage (slim + prod-only)    →  120MB

Go HTTP server:
  Single stage (golang:1.21)        →  920MB
  Multi-stage (scratch)             →    8MB

The 10x size reduction means:
  → 10x faster image pulls in CI/CD
  → 10x less storage cost in your registry
  → 10x smaller attack surface (fewer packages to have CVEs)
  → 10x faster container startup in auto-scaling scenarios
```

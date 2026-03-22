# 04 — Dockerfile

> A Dockerfile is the recipe for building an image.
> Writing a bad Dockerfile is easy. Writing one that builds fast,
> produces small images, and is secure takes deliberate practice.
> This note covers both.

---

## The Anatomy of a Dockerfile

```dockerfile
# Every Dockerfile starts with FROM — the base image
FROM python:3.11-slim

# LABEL adds metadata (optional but useful)
LABEL maintainer="alice@example.com"
LABEL version="1.0"

# ENV sets environment variables available during build AND at runtime
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    APP_HOME=/app

# RUN executes a command during the build
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# WORKDIR sets the working directory for subsequent instructions
WORKDIR $APP_HOME

# COPY copies files from host into the image
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# EXPOSE documents which port the app listens on (does NOT publish it)
EXPOSE 8000

# USER sets the user for subsequent RUN/CMD/ENTRYPOINT instructions
USER nobody

# CMD is the default command when the container starts
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Every Instruction, Explained

### FROM
```dockerfile
FROM python:3.11-slim
FROM ubuntu:22.04 AS builder    # named stage for multi-stage builds
FROM scratch                    # empty base — for compiled binaries
```
Every Dockerfile must start with FROM. This is the base layer
everything else is built on top of.

### RUN
```dockerfile
# Each RUN creates a new layer
RUN apt-get update
RUN apt-get install -y nginx    # BAD — two layers, apt cache not cleaned up

# Chain commands with && to keep them in one layer
RUN apt-get update \
    && apt-get install -y nginx curl \
    && rm -rf /var/lib/apt/lists/*   # clean up cache in SAME layer
```

Cleaning up in the same RUN instruction is critical.
If you install packages in one RUN and clean up in a separate RUN,
the cache files are committed in the first layer — they persist in
the image even though they are deleted in a later layer.

### COPY vs ADD
```dockerfile
COPY src/ /app/src/          # copy files/dirs from build context
COPY requirements.txt .      # copy to WORKDIR

ADD https://example.com/file.tar.gz /app/   # ADD can download URLs
ADD archive.tar.gz /app/                     # ADD auto-extracts archives

# Rule: use COPY unless you specifically need ADD's extra features
# ADD's auto-extraction and URL fetching are implicit and surprising
```

### CMD vs ENTRYPOINT
This is the most commonly misunderstood part of Dockerfiles.

```dockerfile
# CMD — the default command, easily overridden
CMD ["nginx", "-g", "daemon off;"]

# Override at runtime:
docker run myimage           # runs nginx
docker run myimage bash      # runs bash instead — CMD completely replaced


# ENTRYPOINT — the main executable, not easily overridden
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]    # default arguments to entrypoint

# docker run myimage          → runs: nginx -g "daemon off;"
# docker run myimage -c /cfg  → runs: nginx -c /cfg   (CMD replaced, ENTRYPOINT kept)
# docker run --entrypoint bash myimage  → override entrypoint explicitly


# Shell form vs exec form:
CMD nginx -g "daemon off;"          # shell form: runs via /bin/sh -c
CMD ["nginx", "-g", "daemon off;"]  # exec form: runs directly (preferred)

# Shell form problems:
# - PID 1 is /bin/sh, not your process
# - Signals (SIGTERM) go to shell, may not reach your app
# - Slower to start
# Always prefer exec form (JSON array syntax)
```

### ENV and ARG
```dockerfile
# ENV — available during build AND at runtime
ENV APP_ENV=production
ENV DATABASE_URL=postgresql://localhost/mydb    # don't put secrets here

# ARG — only available during build, not at runtime
ARG BUILD_VERSION=latest
ARG REGISTRY=docker.io

# Using them
RUN echo "Building version ${BUILD_VERSION}"
FROM ${REGISTRY}/python:3.11-slim

# Build with custom ARG:
docker build --build-arg BUILD_VERSION=2.4.1 .
```

### USER
```dockerfile
# Always run as non-root in production
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# Or use an existing non-root user
USER nobody
USER 1001     # by UID
```

### HEALTHCHECK
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Docker will check this every 30s
# After 3 failures the container is marked "unhealthy"
# Orchestrators (Kubernetes, Docker Swarm) use this to restart failed containers
```

---

## Writing a Dockerfile Well

### A bad Dockerfile
```dockerfile
FROM ubuntu:latest          # never use latest in production
RUN apt-get update
RUN apt-get install -y python3 python3-pip
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
CMD python3 app.py          # shell form — signals won't reach Python
```

Problems:
- `ubuntu:latest` — unpredictable, changes without warning
- Two separate RUN for apt — redundant layer, cache bloat
- `COPY . /app` before `pip install` — invalidates pip cache on any file change
- No cleanup of apt cache — larger image
- CMD in shell form — Ctrl+C won't work, graceful shutdown broken
- Running as root — security risk

### The same Dockerfile, done properly
```dockerfile
FROM python:3.11-slim

# Install system dependencies in one layer, clean up immediately
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Dependencies first — cached unless requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# App code last — changes most often
COPY . .

# Create a non-root user and switch to it
RUN useradd -r -u 1001 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Exec form — signals work correctly, proper PID 1
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## .dockerignore

Just like `.gitignore`, `.dockerignore` prevents files from being
sent to the Docker build context. This matters because:

1. **Build speed**: the build context is sent to the Docker daemon
   before the build starts. A large context (node_modules, .git, logs)
   makes every build slow even if nothing changed.

2. **Image security**: without `.dockerignore`, `COPY . .` might
   accidentally include `.env` files, SSH keys, or secrets.

```
# .dockerignore
.git
.gitignore
.env
.env.*
*.log
node_modules/
__pycache__/
.pytest_cache/
.coverage
dist/
build/
*.egg-info/
.DS_Store
Dockerfile
docker-compose.yml
README.md
```

---

## Building Images

```bash
# Build from Dockerfile in current directory
docker build -t myapp:v1.0 .

# Build from specific Dockerfile
docker build -f docker/Dockerfile.prod -t myapp:prod .

# Build with build args
docker build --build-arg APP_VERSION=2.4.1 -t myapp:2.4.1 .

# Build for a specific platform (cross-compile for ARM)
docker build --platform linux/arm64 -t myapp:arm .

# Build without cache (forces rebuild of all layers)
docker build --no-cache -t myapp:latest .

# See build output with more detail
docker build --progress=plain -t myapp .
```

---

## Practical: Node.js Dockerfile

```dockerfile
FROM node:20-slim

ENV NODE_ENV=production

WORKDIR /app

# Package files first — npm install cached unless package*.json changes
COPY package.json package-lock.json ./
RUN npm ci --only=production       # ci is stricter and faster than install

COPY . .

RUN useradd -r -u 1001 nodeuser && chown -R nodeuser:nodeuser /app
USER nodeuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s \
    CMD node healthcheck.js || exit 1

CMD ["node", "server.js"]
```

---

## Practical: Go Binary Dockerfile (Preview of Multi-Stage)

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /build
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o app .

# Stage 2: Run — just the binary, no Go toolchain
FROM scratch
COPY --from=builder /build/app /app
CMD ["/app"]
```

The final image contains only the Go binary. Nothing else.
No OS. No shell. ~10MB instead of ~900MB.
Full multi-stage builds are covered in note 09.

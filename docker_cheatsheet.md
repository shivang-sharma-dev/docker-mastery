# 🐳 Docker Daily Cheatsheet — All-in-One Reference

> **Pro Tip:** Add `--help` to any Docker command to see all available options.  
> Example: `docker run --help`

---

## 📌 Quick Reference Card

| Task | Command |
|---|---|
| List running containers | `docker ps` |
| List ALL containers | `docker ps -a` |
| List images | `docker images` |
| Pull an image | `docker pull <image>:<tag>` |
| Build an image | `docker build -t <name>:<tag> .` |
| Run a container | `docker run -d -p 8080:80 <image>` |
| Stop a container | `docker stop <name/id>` |
| Remove a container | `docker rm <name/id>` |
| Remove an image | `docker rmi <image>` |
| Shell into container | `docker exec -it <name> /bin/bash` |
| View logs | `docker logs -f <name>` |
| View resource usage | `docker stats` |

---

## 🖼️ 1. IMAGES — Build, Pull, Push, Manage

```bash
# Pull an image from Docker Hub (or any registry)
docker pull nginx                          # Latest tag
docker pull nginx:1.25-alpine             # Specific version
docker pull ubuntu:22.04
docker pull ghcr.io/user/image:tag        # GitHub Container Registry
docker pull <ecr-url>/image:tag           # AWS ECR

# List local images
docker images
docker images -a                           # Include intermediate layers
docker images --filter "dangling=true"    # Untagged/orphan images

# Tag an image (rename / add registry prefix)
docker tag <source-image>:<tag> <target-image>:<tag>
# Example:
docker tag notes-app:latest shivangsharma/notes-app-k8s:v1.0

# Push image to registry
docker push shivangsharma/notes-app-k8s:v1.0
docker push ghcr.io/shivang-sharma-dev/notes-app:latest

# Remove an image
docker rmi <image-name>:<tag>
docker rmi <image-id>

# Remove ALL unused images (dangling)
docker image prune

# Remove ALL unused images (including unreferenced ones)
docker image prune -a

# Inspect image metadata
docker inspect <image-name>
docker history <image-name>               # Layer-by-layer build history

# Save image to a tar file (for offline transfer)
docker save -o myimage.tar <image>:<tag>

# Load image from a tar file
docker load -i myimage.tar
```

---

## 🏗️ 2. BUILD — Dockerfile to Image

```bash
# Build from Dockerfile in current directory
docker build -t <name>:<tag> .
docker build -t myapp:latest .

# Build from specific Dockerfile path
docker build -t myapp:latest -f ./docker/Dockerfile.prod .

# Build with build args
docker build --build-arg ENV=production --build-arg PORT=8000 -t myapp:prod .

# No cache (fresh build — useful for troubleshooting stale layers)
docker build --no-cache -t myapp:latest .

# Multi-platform build (requires buildx)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .

# Build and show full output (no compressed logs)
docker build --progress=plain -t myapp:latest .

# Squash all layers into one (reduces image size)
docker build --squash -t myapp:slim .
```

---

## 🚀 3. RUN — Start Containers

```bash
# Basic run (foreground)
docker run <image>

# Run in detached/background mode
docker run -d <image>

# Run with port mapping  HOST:CONTAINER
docker run -d -p 8080:80 nginx             # http://localhost:8080
docker run -d -p 8000:8000 -p 5555:5555 myapp   # Multiple ports

# Run with a name (easier management)
docker run -d --name my-nginx nginx

# Run with environment variables
docker run -d \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  -e SECRET_KEY=mysecret \
  --name django-app myapp:latest

# Run with env file
docker run -d --env-file .env --name django-app myapp:latest

# Run with volume mount
docker run -d \
  -v /host/path:/container/path \
  --name myapp myimage

# Run with named volume
docker run -d \
  -v mydata:/var/lib/data \
  --name myapp myimage

# Run with read-only volume
docker run -d -v mydata:/data:ro --name myapp myimage

# Run interactively (for debugging)
docker run -it ubuntu:22.04 /bin/bash
docker run -it --rm ubuntu:22.04 /bin/bash   # --rm auto-removes on exit

# Run with network
docker run -d --network my-network --name myapp myimage

# Run with resource limits
docker run -d \
  --memory="512m" \
  --cpus="1.0" \
  --name myapp myimage

# Run with restart policy
docker run -d \
  --restart=always \              # Always restart
  --restart=unless-stopped \     # Restart unless manually stopped
  --restart=on-failure:3 \       # Restart on failure, max 3 times
  --name myapp myimage

# Run with hostname
docker run -d --hostname app-server --name myapp myimage
```

---

## 📋 4. CONTAINER LIFECYCLE — Start, Stop, Remove

```bash
# List containers
docker ps                                  # Running containers
docker ps -a                               # ALL containers (including stopped)
docker ps -q                               # Only container IDs
docker ps --filter "status=exited"        # Filter by status
docker ps --filter "name=django"          # Filter by name
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"  # Custom format

# Start / Stop / Restart
docker start <name/id>
docker stop <name/id>
docker restart <name/id>

# Stop with timeout (default 10s grace period)
docker stop -t 30 <name/id>

# Force kill (SIGKILL — no graceful shutdown)
docker kill <name/id>

# Pause / Unpause (freezes processes)
docker pause <name/id>
docker unpause <name/id>

# Remove containers
docker rm <name/id>
docker rm -f <name/id>                     # Force remove (even if running)
docker rm $(docker ps -aq)                 # Remove ALL stopped containers
docker container prune                     # Remove all stopped containers (with prompt)
docker container prune -f                  # Force (no prompt)

# Rename a container
docker rename <old-name> <new-name>
```

---

## 🔍 5. INSPECT & MONITOR

```bash
# View container details (IP, mounts, env, etc.)
docker inspect <name/id>

# Get container IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <name>

# Get environment variables
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' <name>

# Resource usage (CPU, Memory, Network, I/O)
docker stats                               # All running containers (live)
docker stats <name/id>                    # Specific container (live)
docker stats --no-stream                  # One-time snapshot

# View running processes inside container
docker top <name/id>

# View filesystem changes (modified files)
docker diff <name/id>

# Port mappings
docker port <name/id>
```

---

## 📄 6. LOGS

```bash
# View logs
docker logs <name/id>
docker logs <name/id> --tail=100          # Last 100 lines
docker logs <name/id> -f                  # Follow/stream logs (like tail -f)
docker logs <name/id> --since=1h          # Logs from last 1 hour
docker logs <name/id> --since="2026-05-01T00:00:00"   # Since timestamp
docker logs <name/id> -t                  # Add timestamps

# Combine options
docker logs <name> -f --tail=50 -t

# Pipe to grep
docker logs <name> 2>&1 | grep ERROR
docker logs <name> 2>&1 | grep -i "exception\|error\|fail"
```

---

## 💻 7. EXEC — Run Commands Inside Containers

```bash
# Interactive shell
docker exec -it <name> /bin/bash
docker exec -it <name> /bin/sh             # Alpine-based images
docker exec -it <name> sh

# Run a single command
docker exec <name> ls /app
docker exec <name> cat /etc/nginx/nginx.conf
docker exec <name> env                     # View environment variables
docker exec <name> python manage.py migrate

# Run as specific user
docker exec -it --user root <name> /bin/bash
docker exec -it --user 1000 <name> /bin/bash

# Run with environment variable
docker exec -e DEBUG=true -it <name> /bin/bash

# Set working directory
docker exec -it -w /app/src <name> /bin/bash
```

---

## 📁 8. VOLUMES — Persistent Storage

```bash
# List volumes
docker volume ls

# Create a named volume
docker volume create mydata

# Inspect a volume (find mount path)
docker volume inspect mydata

# Remove a volume
docker volume rm mydata

# Remove ALL unused volumes
docker volume prune
docker volume prune -f                     # Force (no prompt)

# Mount types:
# 1. Named Volume (managed by Docker — preferred for data)
docker run -v mydata:/var/lib/postgresql/data postgres

# 2. Bind Mount (maps host directory directly)
docker run -v /home/user/project:/app myimage
docker run -v $(pwd):/app myimage          # Current directory shorthand

# 3. tmpfs mount (in-memory, not persisted)
docker run --tmpfs /tmp myimage

# Copy files between host and container
docker cp <name>:/path/in/container ./local-path     # Container → Host
docker cp ./local-file <name>:/path/in/container     # Host → Container

# Example:
docker cp django-app:/app/logs/error.log ./error.log
```

---

## 🌐 9. NETWORKS

```bash
# List networks
docker network ls

# Inspect a network
docker network inspect <network-name>

# Create a network
docker network create my-network
docker network create --driver bridge my-bridge
docker network create --driver overlay my-overlay     # For Swarm

# Create with subnet
docker network create \
  --subnet=172.20.0.0/16 \
  --gateway=172.20.0.1 \
  my-custom-net

# Connect a running container to a network
docker network connect my-network <container-name>

# Disconnect a container from a network
docker network disconnect my-network <container-name>

# Remove a network
docker network rm my-network

# Remove all unused networks
docker network prune

# Run container in host network (shares host networking — no isolation)
docker run -d --network host nginx

# Run container with no network
docker run -d --network none myimage
```

---

## 🔐 10. REGISTRY — Login, Push, Pull

```bash
# Login to Docker Hub
docker login
docker login -u <username>

# Login to specific registry
docker login ghcr.io                             # GitHub Container Registry
docker login <aws-account>.dkr.ecr.<region>.amazonaws.com   # AWS ECR

# Logout
docker logout
docker logout ghcr.io

# AWS ECR login (one-liner)
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com

# Push workflow: tag → push
docker tag myapp:latest shivangsharma/notes-app-k8s:latest
docker push shivangsharma/notes-app-k8s:latest

# Pull private image (must be logged in)
docker pull shivangsharma/notes-app-k8s:latest
```

---

## 🧩 11. DOCKER COMPOSE — Multi-Container Apps

```bash
# Start services (detached)
docker compose up -d

# Start and rebuild images first
docker compose up -d --build

# Start specific service only
docker compose up -d <service-name>

# Stop services (keep containers)
docker compose stop

# Stop and remove containers
docker compose down

# Stop, remove containers + volumes (DANGER: deletes data!)
docker compose down -v

# Stop, remove containers + images
docker compose down --rmi all

# View running services
docker compose ps

# View logs
docker compose logs
docker compose logs -f                     # Follow all services
docker compose logs -f <service-name>     # Follow specific service
docker compose logs --tail=50 <service>

# Execute command in a service container
docker compose exec <service> /bin/bash
docker compose exec <service> python manage.py migrate
docker compose exec db psql -U postgres

# Run one-off command (creates new container)
docker compose run --rm <service> python manage.py createsuperuser

# Scale a service
docker compose up -d --scale web=3

# Rebuild a single service
docker compose build <service>
docker compose up -d --build <service>

# Pull updated images
docker compose pull

# View service config (after variable substitution)
docker compose config

# Validate docker-compose.yml
docker compose config --quiet && echo "Valid!"
```

---

## 🐳 12. DOCKERFILE REFERENCE

```dockerfile
# Best-practice Dockerfile for a Python/Django app

# Stage 1: Base
FROM python:3.11-slim AS base

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    libpq-dev gcc curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies (separate layer for caching)
COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt

# Copy application code
COPY . .

# Create non-root user (security best practice)
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1

# Default command
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "myproject.wsgi:application"]
```

```dockerfile
# Multi-stage build — smaller production image

FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM python:3.11-slim AS production
COPY --from=builder /install /usr/local
WORKDIR /app
COPY . .
EXPOSE 8000
CMD ["gunicorn", "myproject.wsgi:application"]
```

---

## 🧹 13. CLEANUP & PRUNE

```bash
# Remove stopped containers
docker container prune -f

# Remove unused images
docker image prune -f           # Dangling only
docker image prune -a -f        # All unused images

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# 🔥 Nuclear option: Remove EVERYTHING unused at once
docker system prune -f
docker system prune -a -f       # Includes all unused images (not just dangling)
docker system prune -a --volumes -f   # Also removes volumes ⚠️ DATA LOSS

# Check disk usage
docker system df
docker system df -v             # Verbose (per image/container/volume)
```

---

## 📊 14. USEFUL ONE-LINERS

```bash
# Stop ALL running containers
docker stop $(docker ps -q)

# Remove ALL containers (stopped + running)
docker rm -f $(docker ps -aq)

# Remove ALL images
docker rmi -f $(docker images -q)

# Get shell in a container by partial name
docker exec -it $(docker ps | grep django | awk '{print $1}') bash

# See which container is using a port
docker ps --filter "publish=8080"

# Show container IPs in a network
docker network inspect bridge -f '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'

# Run a temporary Alpine container for debugging network
docker run --rm -it --network my-network alpine sh

# Watch docker events in real time
docker events

# Follow container logs with timestamps until Ctrl+C
docker logs -f -t --tail=0 <name>

# Export container filesystem as tar
docker export <name> > container.tar

# Show all environment vars of a container
docker inspect <name> | python3 -m json.tool | grep -A 50 '"Env"'
```

---

## ⚡ 15. DOCKER ALIASES (Add to ~/.bashrc or ~/.zshrc)

```bash
alias d='docker'
alias dc='docker compose'
alias dps='docker ps'
alias dpsa='docker ps -a'
alias di='docker images'
alias drm='docker rm -f'
alias drmi='docker rmi'
alias dex='docker exec -it'
alias dl='docker logs -f'
alias dst='docker stats --no-stream'
alias dsp='docker system prune -f'
alias db='docker build -t'

# Docker Compose shortcuts
alias dcu='docker compose up -d'
alias dcub='docker compose up -d --build'
alias dcd='docker compose down'
alias dcl='docker compose logs -f'
alias dce='docker compose exec'
alias dcr='docker compose run --rm'

# Usage examples:
# dex django-app /bin/bash
# dl django-app
# dcu                          ← docker compose up -d
# dcl web                      ← docker compose logs -f web
```

---

## 🧪 16. DOCKER COMPOSE FILE TEMPLATE

```yaml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    image: shivangsharma/notes-app-k8s:latest
    container_name: django-web
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DEBUG=False
      - DB_HOST=db
      - DB_PORT=5432
    env_file:
      - .env
    volumes:
      - static_files:/app/static
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15-alpine
    container_name: postgres-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: notesdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_files:/usr/share/nginx/html/static:ro
    depends_on:
      - web
    networks:
      - app-network

volumes:
  pgdata:
  static_files:

networks:
  app-network:
    driver: bridge
```

---

## 🔥 Common Mistakes & Fixes

| ❌ Mistake | ✅ Fix |
|---|---|
| Container exits immediately | Use `-d` for detached mode, or check `docker logs` |
| `docker push` denied | Run `docker login` first, check image tag matches registry |
| Port already in use | Change host port: `-p 8081:80` or stop the conflicting process |
| Old cached layer in build | Use `--no-cache` flag |
| Volume data not persisted | Use named volumes, not bind mounts for DB data |
| Can't connect between containers | Put them on the same Docker network |
| Container can't reach internet | Check if `--network none` was set accidentally |
| Image too large | Use Alpine base, multi-stage builds, `.dockerignore` |
| Secrets in image layers | Use `--secret` or env files, never `COPY .env` in Dockerfile |
| `exec` fails on stopped container | Run `docker start <name>` first |

---

## 📝 .dockerignore Template

```
# Version control
.git
.gitignore

# Python
__pycache__/
*.py[cod]
*.pyo
*.egg-info/
.venv/
venv/
env/

# Environment files (NEVER include in image)
.env
.env.*
*.env

# Docker files
Dockerfile*
docker-compose*.yml

# Docs and tests
README.md
docs/
tests/

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Build artifacts
dist/
build/
*.tar.gz
```

---

## 🔗 Docker + Kubernetes Bridge

```bash
# Build image and deploy to k8s in one workflow:

# 1. Build & tag
docker build -t shivangsharma/notes-app-k8s:v2.0 .

# 2. Push to registry
docker push shivangsharma/notes-app-k8s:v2.0

# 3. Update k8s deployment
kubectl set image deployment/django-app \
  django=shivangsharma/notes-app-k8s:v2.0 \
  -n notes-app-k8s

# 4. Watch rollout
kubectl rollout status deployment/django-app -n notes-app-k8s

# Load local image into Minikube (skip registry push)
minikube image load myapp:latest

# Load into Kind cluster
kind load docker-image myapp:latest --name my-cluster
```

---

*Generated for daily Docker container management · Last updated: May 2026*

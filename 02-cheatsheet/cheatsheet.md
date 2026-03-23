# Docker Mastery — Cheatsheet

---

## Images

```bash
# Pull
docker pull nginx                      # latest
docker pull nginx:1.25.3               # specific version
docker pull nginx:alpine               # alpine variant

# List
docker images
docker images --filter "dangling=true" # untagged images

# Inspect
docker inspect nginx
docker history nginx                   # show layers and sizes

# Tag
docker tag myapp:v1 mycompany/myapp:v1
docker tag myapp:v1 myapp:latest

# Remove
docker rmi nginx
docker image prune                     # remove dangling
docker image prune -a                  # remove all unused

# Save / load (for air-gapped environments)
docker save nginx -o nginx.tar
docker load -i nginx.tar

# Build
docker build -t myapp:v1 .
docker build -f Dockerfile.prod -t myapp:prod .
docker build --no-cache -t myapp .
docker build --target prod -t myapp:prod .    # multi-stage target
docker build --build-arg VERSION=2.4.1 .
```

---

## Containers

```bash
# Run
docker run nginx                                 # foreground
docker run -d nginx                              # detached
docker run -d --name web nginx                   # named
docker run -d -p 8080:80 nginx                   # port map host:container
docker run -d -p 127.0.0.1:8080:80 nginx         # localhost only
docker run -it ubuntu bash                       # interactive shell
docker run --rm ubuntu cat /etc/os-release       # auto-remove on exit

# Environment
docker run -d -e APP_ENV=prod -e PORT=8000 myapp
docker run -d --env-file .env myapp

# Resources
docker run -d --memory=512m --cpus=1.5 myapp
docker run -d --restart=unless-stopped myapp
docker run -d --restart=on-failure:3 myapp

# Volumes
docker run -d -v mydata:/var/lib/data myapp      # named volume
docker run -d -v $(pwd):/app myapp               # bind mount
docker run -d -v /etc/config.yml:/app/config.yml:ro myapp  # read-only

# List
docker ps                        # running
docker ps -a                     # all
docker ps -q                     # IDs only

# Lifecycle
docker stop container            # SIGTERM → wait → SIGKILL
docker stop -t 30 container      # 30s grace period
docker start container
docker restart container
docker kill container             # immediate SIGKILL
docker rm container               # remove stopped container
docker rm -f container            # force remove running container
docker container prune            # remove all stopped

# Exec
docker exec -it container bash
docker exec -it -u root container bash
docker exec container cat /etc/nginx/nginx.conf
docker exec -d container /scripts/cleanup.sh

# Logs
docker logs container
docker logs -f container          # follow
docker logs --tail 50 container
docker logs -t container          # with timestamps
docker logs --since 1h container

# Inspect
docker inspect container
docker inspect -f '{{.State.Status}}' container
docker inspect -f '{{.NetworkSettings.IPAddress}}' container
docker stats container            # live resource usage
docker stats --no-stream container
docker top container              # processes inside

# Copy files
docker cp container:/etc/nginx/nginx.conf ./
docker cp ./config.yml container:/etc/app/

# Port info
docker port container
```

---

## Dockerfile

```dockerfile
FROM python:3.11-slim                          # base image
LABEL maintainer="you@example.com"
ENV PYTHONUNBUFFERED=1 APP_HOME=/app           # env vars
ARG BUILD_VERSION=latest                        # build-time only
WORKDIR /app                                    # set working dir
COPY requirements.txt .                         # copy files
RUN pip install --no-cache-dir -r requirements.txt  # run command
COPY . .
EXPOSE 8000                                     # document port
RUN useradd -r -u 1001 appuser && chown -R appuser /app
USER appuser                                    # non-root user
HEALTHCHECK --interval=30s --timeout=5s \
    CMD curl -f http://localhost:8000/health || exit 1
CMD ["python", "-m", "uvicorn", "main:app", \
     "--host", "0.0.0.0", "--port", "8000"]    # exec form
```

### Key rules
```
# Dependencies before source code (cache optimisation)
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .                          # changes often — put last

# Clean up in the SAME RUN layer
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# Exec form (not shell form) for CMD/ENTRYPOINT
CMD ["python", "app.py"]          # correct — signals work
CMD python app.py                 # wrong — shell is PID 1

# Always use non-root user
USER nobody
```

---

## Volumes

```bash
# Named volumes
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune                   # remove unused volumes

# Using volumes
docker run -d -v mydata:/var/lib/data myapp         # named
docker run -d -v $(pwd):/app myapp                  # bind mount
docker run -d -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx  # read-only

# Long syntax (more explicit)
docker run -d \
  --mount type=volume,source=mydata,target=/var/lib/data \
  myapp

docker run -d \
  --mount type=bind,source=$(pwd),target=/app \
  myapp

# Backup a volume
docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/mydata_backup.tar.gz -C /source .
```

---

## Networking

```bash
# Networks
docker network create mynet
docker network create --subnet 10.0.1.0/24 mynet
docker network ls
docker network inspect mynet
docker network connect mynet container
docker network disconnect mynet container
docker network rm mynet
docker network prune

# Run on a specific network
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name api myapp
# api can reach db as: http://db:5432

# Network modes
docker run -d --network host myapp        # host networking
docker run -d --network none myapp        # no networking

# DNS
docker run -d --dns 8.8.8.8 myapp
docker run -d --add-host db.internal:10.0.1.5 myapp
```

---

## Docker Compose

```bash
docker compose up -d                   # start all services
docker compose up -d --build           # build then start
docker compose down                    # stop and remove containers + networks
docker compose down -v                 # also remove volumes
docker compose stop                    # stop without removing

docker compose ps                      # list services
docker compose logs                    # all logs
docker compose logs -f api             # follow specific service
docker compose logs --tail 50 api

docker compose exec api bash           # shell in running service
docker compose run --rm api pytest     # one-off command

docker compose build api               # rebuild specific service
docker compose pull                    # pull latest images
docker compose restart api             # restart specific service

docker compose up -d --scale api=3    # run 3 instances
```

### docker-compose.yml quick reference

```yaml
services:
  api:
    image: myapp:v1               # use existing image
    build: .                      # or build from Dockerfile
    build:
      context: .
      dockerfile: Dockerfile.prod
      target: production          # multi-stage target
    ports:
      - "8000:8000"
    environment:
      APP_ENV: production
      DB_URL: ${DB_URL}           # from .env file
    env_file:
      - .env
    volumes:
      - ./config:/app/config:ro
      - appdata:/var/lib/app
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  appdata:
```

---

## Registries

```bash
# Docker Hub
docker login
docker push username/myapp:v1
docker pull username/myapp:v1
docker logout

# AWS ECR
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    123456789.dkr.ecr.us-east-1.amazonaws.com

docker tag myapp:v1 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

---

## System / Cleanup

```bash
docker system df                  # disk usage summary
docker system df -v               # verbose
docker system prune               # stopped containers + dangling images + unused networks
docker system prune -a            # also remove unused images
docker system prune -a --volumes  # also remove unused volumes

# Individual cleanup
docker container prune
docker image prune
docker image prune -a
docker volume prune
docker network prune
```

---

## Multi-Stage Build Reference

```dockerfile
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist   # copy from builder stage
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```bash
# Build to a specific stage
docker build --target builder -t myapp:builder .
docker build --target production -t myapp:prod .
```

---

## Production Checklist

```
Image:
  ✓ Pinned version tag (not :latest)
  ✓ Multi-stage build
  ✓ Non-root USER
  ✓ Minimal base (slim or alpine)
  ✓ .dockerignore in place
  ✓ Scanned for CVEs

Container:
  ✓ HEALTHCHECK defined
  ✓ Memory + CPU limits set
  ✓ restart: unless-stopped
  ✓ Log size limits
  ✓ No secrets in ENV or image

Application:
  ✓ Logs to stdout/stderr
  ✓ /health checks real dependencies
  ✓ Handles SIGTERM gracefully
  ✓ Listens on 0.0.0.0
  ✓ Config from environment variables
```

---

## Debugging Quick Reference

```bash
# Container exits immediately
docker logs container              # check what it printed
docker run -it --entrypoint bash image  # override entrypoint

# Can't connect to container
docker port container              # check port mappings
docker exec container ss -tulnp    # check what's listening
docker inspect container | jq '.[0].NetworkSettings'

# Containers can't reach each other
docker network inspect mynet       # check they're on same network
docker exec container ping other   # test connectivity

# Out of disk space
docker system df                   # find what's using space
docker system prune -a             # clean up unused
```

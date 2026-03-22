# 07 — Docker Compose

> Running a single container is a `docker run` command.
> Running a real application — app + database + cache + reverse proxy —
> means managing 4+ containers, their networks, their volumes, their
> environment variables, their startup order. Docker Compose does all
> of that with one file and one command.

---

## What Compose Does

```
WITHOUT Compose:

  docker network create appnet
  docker volume create pgdata
  docker run -d --name db --network appnet -v pgdata:/var/lib/postgresql/data \
    -e POSTGRES_DB=myapp -e POSTGRES_PASSWORD=secret postgres:16
  docker run -d --name redis --network appnet redis:7-alpine
  docker run -d --name api --network appnet -p 8000:8000 \
    -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
    -e REDIS_URL=redis://redis:6379 myapp:latest

  To stop everything:
  docker stop db redis api
  docker rm db redis api
  docker network rm appnet
  # (volumes left manually)

  Remember to do this every time. In the right order.


WITH Compose:

  docker compose up -d     # starts everything
  docker compose down      # stops and removes everything
  docker compose logs -f   # logs from all services
```

---

## The docker-compose.yml File

```yaml
# docker-compose.yml

services:

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

  api:
    build: .                          # build from Dockerfile in current dir
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:secret@db:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy    # wait until db healthcheck passes
      redis:
        condition: service_started
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/ssl/certs:ro
    depends_on:
      - api
    restart: unless-stopped

volumes:
  pgdata:                             # named volume managed by Docker
```

---

## Core Compose Commands

```bash
# Start all services (detached)
docker compose up -d

# Start and rebuild images first
docker compose up -d --build

# Stop all services (containers stopped, not removed)
docker compose stop

# Stop and remove containers + networks (volumes kept)
docker compose down

# Stop and remove containers + networks + volumes
docker compose down -v

# View logs from all services
docker compose logs
docker compose logs -f              # follow live
docker compose logs -f api          # specific service only
docker compose logs --tail 50 api

# List running services
docker compose ps

# Scale a service (run multiple instances)
docker compose up -d --scale api=3

# Rebuild a specific service's image
docker compose build api
docker compose build --no-cache api

# Execute a command in a running service
docker compose exec api bash
docker compose exec db psql -U postgres myapp

# Run a one-off command in a new container
docker compose run --rm api python manage.py migrate
docker compose run --rm api python manage.py createsuperuser

# Pull latest images for all services
docker compose pull

# Restart a specific service
docker compose restart api
```

---

## Environment Variables in Compose

```yaml
# Option 1: hardcoded (only for non-sensitive values)
environment:
  APP_ENV: production
  PORT: "8000"

# Option 2: from a .env file (Compose reads this automatically)
# .env file:
# DB_PASSWORD=secret
# API_KEY=abc123

environment:
  DB_PASSWORD: ${DB_PASSWORD}
  API_KEY: ${API_KEY}

# Option 3: env_file (load from a file into container environment)
env_file:
  - .env
  - .env.production

# Option 4: pass through from shell (no value = use shell's value)
environment:
  - DB_PASSWORD       # uses $DB_PASSWORD from your shell
```

**Never commit `.env` files to git. Add `.env` to `.gitignore`.**
Provide `.env.example` with placeholder values instead.

---

## depends_on — Service Startup Order

```yaml
# depends_on controls start ORDER, not readiness.
# The service starts after its dependencies start —
# not after they are actually ready to accept connections.

# This is wrong for databases:
depends_on:
  - db        # db container starts, but postgres may not be ready yet

# This is correct — wait for healthcheck:
depends_on:
  db:
    condition: service_healthy    # requires a healthcheck on db service
```

Even with `condition: service_healthy`, your app should have retry
logic for database connections. The health check gives you a good
starting point but network conditions and timing can still vary.

---

## Multiple Compose Files

```bash
# Override base config with an environment-specific file
# docker-compose.yml = base config (shared)
# docker-compose.prod.yml = production overrides

docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Common pattern:
# docker-compose.yml          — service definitions, volumes, networks
# docker-compose.override.yml — dev overrides (auto-loaded, don't commit)
# docker-compose.prod.yml     — production overrides (commit, use explicitly)
```

```yaml
# docker-compose.override.yml (development overrides, NOT committed)
services:
  api:
    build:
      context: .
      target: dev           # build to dev stage of multi-stage Dockerfile
    volumes:
      - .:/app              # bind mount source for hot reload
    environment:
      DEBUG: "true"
    command: uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

---

## A Real Full-Stack Example

```yaml
# docker-compose.yml — a complete web application stack

services:

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-myapp}
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "${DB_USER:-myapp}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD:-secret}
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
    restart: unless-stopped

  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql://${DB_USER:-myapp}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-myapp}
      REDIS_URL: redis://:${REDIS_PASSWORD:-secret}@redis:6379/0
      SECRET_KEY: ${SECRET_KEY:?SECRET_KEY is required}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    # No ports — traffic comes through nginx only

  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static_files:/var/www/static:ro
    depends_on:
      - api
    restart: unless-stopped

volumes:
  pgdata:
  static_files:
```

The `${VARIABLE:?message}` syntax causes Compose to fail with a
helpful error message if a required variable is not set. Use this
for secrets and anything that breaks without a specific value.

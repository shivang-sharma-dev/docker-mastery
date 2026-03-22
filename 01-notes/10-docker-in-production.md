# 10 — Docker in Production

> Everything up to this point works fine in development.
> Production adds requirements that development doesn't have:
> high availability, observability, security, resource governance,
> and graceful handling of failures. This note covers all of them.

---

## Health Checks — Teaching Orchestrators Your App's State

Without a health check, Docker only knows if the container process
is running. It does not know if your app is actually serving traffic.
A container can be running but completely broken — nginx started,
but your upstream app is not responding.

```dockerfile
# In your Dockerfile:
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

```
--interval=30s      how often to check (default: 30s)
--timeout=5s        how long to wait for a response (default: 30s)
--start-period=10s  grace period on startup — failures don't count against retries
--retries=3         how many consecutive failures before marking unhealthy (default: 3)
```

The health check command must exit 0 (healthy) or 1 (unhealthy).

```bash
# Check health status
docker ps                           # STATUS column shows "(healthy)" or "(unhealthy)"
docker inspect --format='{{json .State.Health}}' container_name | jq
```

### Your /health endpoint should check real dependencies

```python
# A health endpoint that actually means something
@app.get("/health")
def health():
    checks = {}

    try:
        db.execute("SELECT 1")
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = str(e)

    try:
        redis_client.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = str(e)

    healthy = all(v == "ok" for v in checks.values())
    return JSONResponse(
        status_code=200 if healthy else 503,
        content={"status": "healthy" if healthy else "degraded", "checks": checks}
    )
```

---

## Resource Limits — Protecting the Host

Without limits, a single container can consume all CPU and memory
on the host and starve other containers and the host OS itself.

```bash
docker run -d \
  --memory=512m \          # hard memory limit
  --memory-reservation=256m \  # soft limit (when host is under pressure)
  --cpus=1.5 \             # CPU quota (1.5 = 150% of one core)
  --cpu-shares=512 \       # relative weight (default 1024) for contention
  --pids-limit=100 \       # max number of processes inside container
  myapp

# In docker-compose.yml:
services:
  api:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

### What happens when limits are hit

```
Memory limit exceeded:
  → The container is OOM killed (SIGKILL)
  → docker logs shows no warning — it just dies
  → docker inspect shows: OOMKilled: true
  → Fix: increase limit, find and fix memory leak, or add swap

CPU limit exceeded:
  → The container is throttled — it slows down but is not killed
  → Requests get slower, latency increases
  → Fix: increase limit or optimise the hot path
```

---

## Logging in Production

Containers should write logs to stdout/stderr. The container
runtime captures them and routes them to your logging backend.

```bash
# View Docker's default log driver
docker info | grep "Logging Driver"

# Configure log driver per container
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Or send directly to a logging service
docker run -d \
  --log-driver awslogs \
  --log-opt awslogs-region=us-east-1 \
  --log-opt awslogs-group=/myapp/production \
  myapp
```

In `docker-compose.yml`:
```yaml
services:
  api:
    image: myapp
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

Without `max-size` and `max-file`, log files grow forever and
eventually fill your disk. This is a very common production failure.

---

## Secrets — Never Bake Credentials Into Images

```dockerfile
# WRONG — credentials visible in image history
ENV DB_PASSWORD=supersecret
RUN echo "password=supersecret" > /app/config.ini
```

```bash
# docker history myimage  → exposes the password to anyone who pulls the image
```

### Correct approaches

**Option 1: Environment variables at runtime (simple, adequate for most cases)**
```bash
docker run -d \
  -e DB_PASSWORD="$(cat /run/secrets/db_password)" \
  myapp
```

**Option 2: Docker secrets (for Swarm)**
```bash
echo "supersecret" | docker secret create db_password -
docker service create --secret db_password myapp
# Secret available at /run/secrets/db_password inside container
```

**Option 3: Mount a secrets file at runtime**
```bash
docker run -d \
  -v /etc/myapp/secrets:/run/secrets:ro \
  myapp
# App reads from /run/secrets/db_password
```

**Option 4: Use a secrets manager (AWS Secrets Manager, HashiCorp Vault)**
```python
# App fetches its own secrets at startup
import boto3
client = boto3.client("secretsmanager")
secret = client.get_secret_value(SecretId="myapp/production/db")
DB_PASSWORD = json.loads(secret["SecretString"])["password"]
```

---

## Graceful Shutdown

When a container is stopped, Docker sends SIGTERM to PID 1,
waits `--stop-timeout` seconds (default 10), then sends SIGKILL.

Your application should catch SIGTERM and shut down cleanly:
finish in-flight requests, close database connections, flush buffers.

```python
import signal, sys

def shutdown_handler(signum, frame):
    print("Shutting down gracefully...")
    # finish current request
    # close db connections
    # flush logs
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown_handler)
signal.signal(signal.SIGINT, shutdown_handler)
```

```bash
# Give container more time to shut down
docker stop --time 30 container_name

# In docker-compose.yml
services:
  api:
    stop_grace_period: 30s
```

If your app is PID 1 in the container, it receives SIGTERM directly.
If it is started via a shell script, the shell is PID 1 and may
not forward signals. Use `exec` in shell scripts to replace the
shell process with your app process:

```bash
#!/bin/sh
# Without exec: sh is PID 1, signals don't reach python
# python app.py

# With exec: python becomes PID 1, receives SIGTERM directly
exec python app.py
```

---

## Container Security Baseline

```dockerfile
# In your Dockerfile:

# 1. Run as non-root
RUN useradd -r -u 1001 appuser
USER appuser

# 2. Use a minimal base image
FROM python:3.11-slim      # not python:3.11 (full)

# 3. Don't install unnecessary packages
RUN apt-get install -y --no-install-recommends curl  # no extras
```

```bash
# At runtime:
docker run -d \
  --read-only \                  # container filesystem is read-only
  --tmpfs /tmp \                 # writable tmp in memory
  --security-opt no-new-privileges \  # process can't escalate privileges
  --cap-drop ALL \               # drop all Linux capabilities
  --cap-add NET_BIND_SERVICE \   # add back only what's needed
  myapp
```

---

## Production Readiness Checklist

```
Image:
  [ ] Uses a specific version tag (not :latest)
  [ ] Multi-stage build — no build tools in production image
  [ ] Non-root USER in Dockerfile
  [ ] Minimal base image (slim or alpine)
  [ ] .dockerignore excludes .env, .git, secrets
  [ ] Image scanned for CVEs (trivy or similar)

Container configuration:
  [ ] HEALTHCHECK defined
  [ ] Memory and CPU limits set
  [ ] Restart policy configured (unless-stopped or on-failure)
  [ ] Log driver configured with size limits
  [ ] No secrets in environment variables or image layers

Application:
  [ ] Logs to stdout/stderr (not to files inside container)
  [ ] /health endpoint checks real dependencies
  [ ] Graceful SIGTERM handling
  [ ] Listens on 0.0.0.0, not 127.0.0.1
  [ ] Reads config from environment variables
```

# Task 05 — Full Stack with Docker Compose

**Covers:** Note 07 (Docker Compose)
**Time:** 1.5 hours

---

## What You're Building

A full web application stack:
- PostgreSQL database
- Redis cache
- Python Flask API
- nginx reverse proxy

Everything defined in a single docker-compose.yml file.

---

## Setup

```
mystack/
├── docker-compose.yml
├── .env
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
└── nginx/
    └── nginx.conf
```

**api/app.py:**
```python
from flask import Flask, jsonify
import redis, psycopg2, os

app = Flask(__name__)

r = redis.Redis(host=os.getenv("REDIS_HOST", "redis"), port=6379)

def get_db():
    return psycopg2.connect(os.getenv("DATABASE_URL"))

@app.route("/health")
def health():
    try:
        r.ping()
        db = get_db(); db.close()
        return jsonify({"status": "healthy", "db": "ok", "redis": "ok"})
    except Exception as e:
        return jsonify({"status": "degraded", "error": str(e)}), 503

@app.route("/visits")
def visits():
    count = r.incr("visit_count")
    return jsonify({"visits": int(count)})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**api/requirements.txt:**
```
flask==3.0.0
redis==5.0.1
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

**api/Dockerfile:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
HEALTHCHECK --interval=10s --timeout=5s \
    CMD curl -f http://localhost:5000/health || exit 1
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:5000"]
```

**nginx/nginx.conf:**
```nginx
events {}
http {
    upstream api {
        server api:5000;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

**.env:**
```
POSTGRES_DB=myappdb
POSTGRES_USER=myapp
POSTGRES_PASSWORD=changeme
DATABASE_URL=postgresql://myapp:changeme@postgres:5432/myappdb
```

---

## Your Task

Write the `docker-compose.yml` that wires everything together.
It should:

1. Define a `postgres` service with the env vars from `.env` and a health check
2. Define a `redis` service with a health check
3. Define an `api` service that builds from `./api`, depends on both (waiting for healthy), and is not exposed directly
4. Define an `nginx` service exposed on port 80, mounting the nginx config
5. Put all services on a user-defined network
6. Use a named volume for postgres data

When it works:
```bash
docker compose up -d
curl localhost/health        # {"status": "healthy", "db": "ok", "redis": "ok"}
curl localhost/visits        # {"visits": 1}
curl localhost/visits        # {"visits": 2}
docker compose down -v
```

---

## Bonus Challenges

1. Add `--scale api=3` and verify nginx distributes traffic across instances
2. Kill one api container while the others are running — does nginx recover?
3. Add a `pgadmin` service for database management

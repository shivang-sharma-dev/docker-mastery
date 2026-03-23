# Project: Multi-Container Dev Environment

**Level:** Beginner
**Time:** 2–3 hours
**Covers:** Notes 05–07 (volumes, networking, compose)

---

## What You're Building

A complete local development environment with hot reload, a database,
and a database management UI — all started with one command.

---

## Stack

- Your web app (Python or Node — use the one from the previous project)
- PostgreSQL database
- Adminer (lightweight database GUI, runs in a container)

---

## What "Done" Looks Like

```bash
docker compose up

# Terminal shows logs from all three services

# In your browser:
# localhost:8000    → your web app
# localhost:8080    → Adminer (connect to postgres with your creds)

# Edit a source file → app reloads automatically (no restart)
# Ctrl+C → everything stops
# docker compose down → clean up
# docker compose up → data still in postgres
```

---

## Requirements

**docker-compose.yml must have:**
1. `app` service — bind mount your source code, run in dev mode with hot reload
2. `postgres` service — named volume for data, health check
3. `adminer` service — image: `adminer`, port 8080 exposed

**Networking:**
- All services on a user-defined network
- `adminer` and `app` can reach `postgres` by name
- Only `app` and `adminer` are exposed to your host

**Development experience:**
- Change a source file → app picks up the change without `docker compose restart`
- Database data persists between `docker compose down` and `docker compose up`
- `docker compose logs -f` shows logs from all services

---

## Bonus

Add a `redis` service and have your app cache something in it.
Show that the cache is populated on first request and hit on subsequent ones.

# Project: Production-Ready Application Stack

**Level:** Intermediate
**Time:** 4–5 hours
**Covers:** All 10 notes

---

## What You're Building

A fully production-ready deployment of a web application — the kind
of setup you'd actually run for a real product.

---

## The Stack

```
Internet
    ↓ HTTPS (port 443)
nginx (SSL termination, reverse proxy)
    ↓
API (3 instances, load balanced by nginx)
    ↓                    ↓
PostgreSQL           Redis
(persistent data)    (sessions, cache)
```

---

## Production Requirements

**Security:**
- [ ] All services run as non-root
- [ ] Databases not exposed to host (only accessible within Docker network)
- [ ] SSL certificate (use self-signed for this project, Let's Encrypt in real life)
- [ ] Secrets passed via environment, not baked into image
- [ ] API image passes Trivy scan (no CRITICAL CVEs)

**Reliability:**
- [ ] Health checks on all services
- [ ] `restart: unless-stopped` on all services
- [ ] `depends_on` with `condition: service_healthy` for proper startup order
- [ ] API handles SIGTERM gracefully (finish in-flight requests)

**Performance:**
- [ ] API image built with multi-stage — under 100MB
- [ ] nginx caches static assets
- [ ] Resource limits on all services

**Observability:**
- [ ] Log rotation configured (max 10MB, 3 files)
- [ ] Application logs to stdout/stderr
- [ ] `/health` endpoint checks database and Redis connectivity
- [ ] `/metrics` endpoint exposing basic stats (optional but impressive)

---

## The docker-compose.prod.yml

Write this from scratch. It should bring up the entire stack with:

```bash
docker compose -f docker-compose.prod.yml up -d
```

And the application should be reachable at `https://localhost`.

---

## Deliverables

1. `Dockerfile` (multi-stage, production-optimised)
2. `docker-compose.prod.yml`
3. `nginx/nginx.conf` (with SSL, reverse proxy, caching headers)
4. `.env.example` (documenting required environment variables)
5. `README.md` with:
   - Architecture diagram
   - How to generate SSL certs
   - How to deploy
   - How to update the API without downtime (rolling restart)

---

## Evaluation

You have built something production-ready when:
- A stranger could follow your README and have the stack running in 15 minutes
- Killing one API container doesn't cause downtime
- The database data survives `docker compose restart`
- `docker compose logs` shows something useful when debugging

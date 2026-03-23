# Task 04 — Networking

**Covers:** Note 06 (Networking)
**Time:** 45 minutes

---

## Part 1: Why User-Defined Networks Matter

```bash
# Start two containers on the DEFAULT bridge network
docker run -d --name web nginx:alpine
docker run -d --name db postgres:16-alpine -e POSTGRES_PASSWORD=secret

# Try to reach db from web by NAME -- this FAILS on default bridge
docker exec web ping db
# ping: bad address 'db'

# It works by IP but that's impractical
DB_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' db)
docker exec web ping -c 2 $DB_IP   # works by IP, not by name

docker rm -f web db
```

---

## Part 2: User-Defined Network — Name Resolution Works

```bash
# Create a user-defined network
docker network create appnet

# Run both on the same user-defined network
docker run -d --name web --network appnet nginx:alpine
docker run -d --name db  --network appnet postgres:16-alpine \
  -e POSTGRES_PASSWORD=secret

# Now name resolution works
docker exec web ping -c 2 db     # works by name

# Inspect to confirm both are on appnet
docker network inspect appnet | jq '.[0].Containers | to_entries | .[] | .value.Name'

docker rm -f web db
docker network rm appnet
```

---

## Part 3: Multi-Container App with Correct Networking

```bash
# Build a proper isolated network
docker network create myapp

# Backend database -- not exposed to host, only accessible on myapp network
docker run -d \
  --name postgres \
  --network myapp \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myappdb \
  postgres:16-alpine

# Redis cache -- same network, not exposed
docker run -d \
  --name redis \
  --network myapp \
  redis:7-alpine

# Verify both are reachable by name from within the network
docker run --rm --network myapp alpine \
  sh -c "ping -c 1 postgres && ping -c 1 redis && echo 'Both reachable'"

# Cleanup
docker rm -f postgres redis
docker network rm myapp
```

---

## Part 4: Port Mapping and Security

```bash
# Only expose to localhost (not 0.0.0.0)
docker run -d --name secure-nginx \
  -p 127.0.0.1:8080:80 \
  nginx:alpine

# Verify: accessible from this machine
curl localhost:8080

# From outside your machine (or another terminal simulating it)
# This WOULD fail because we only bound to 127.0.0.1

docker rm -f secure-nginx
```

**Discussion:** Why would you bind to `127.0.0.1` instead of `0.0.0.0`?
When is it appropriate?

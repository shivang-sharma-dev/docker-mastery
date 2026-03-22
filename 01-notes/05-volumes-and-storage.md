# 05 — Volumes and Storage

> Containers are ephemeral by design. When a container is removed,
> everything written inside it disappears. Volumes are how you
> make data survive that. Getting this wrong means losing your
> database or your user uploads.

---

## Why This Matters

```
Container filesystem (without volumes):

  docker run -d postgres
  → create database, insert 10,000 rows
  docker rm -f postgres_container
  → all data is gone. No recovery.


Container filesystem (with a volume):

  docker run -d -v pgdata:/var/lib/postgresql/data postgres
  → create database, insert 10,000 rows
  docker rm -f postgres_container
  → container gone, but pgdata volume still exists on disk

  docker run -d -v pgdata:/var/lib/postgresql/data postgres
  → new container, same data — all 10,000 rows still there
```

This is not just about databases. Configuration files, uploaded
files, certificates, log files — anything that needs to outlive
the container needs a volume or a bind mount.

---

## Three Storage Options

```
┌─────────────────────────────────────────────────────────────┐
│                     Container                               │
│  Writable layer (container filesystem)                      │
│  ├── Files created/modified here vanish on rm               │
│  └── NOT visible to other containers                        │
├─────────────────────────────────────────────────────────────┤
│  Named Volume                  Bind Mount                   │
│  /var/lib/docker/volumes/      Any host path                │
│  Managed by Docker             Managed by you               │
│  Host path is opaque           Host path is explicit        │
│  Portable                      Tied to host filesystem      │
│  ✓ Best for data               ✓ Best for dev (hot reload)  │
├─────────────────────────────────────────────────────────────┤
│  tmpfs mount                                                 │
│  In-memory only, never written to disk                       │
│  ✓ Best for secrets, temp data                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Named Volumes — For Persistent Data

Docker creates and manages these. You reference them by name.
The actual location on disk is `/var/lib/docker/volumes/volumename/`.

```bash
# Create a volume explicitly
docker volume create pgdata

# Or let Docker create it automatically on first use
docker run -d \
  --name postgres \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# List volumes
docker volume ls

# Inspect a volume (shows actual host path)
docker volume inspect pgdata

# Remove a volume (only when no container uses it)
docker volume rm pgdata

# Remove all unused volumes
docker volume prune
```

### Volume syntax
```bash
# Short syntax
-v volumename:/path/in/container

# Long syntax (more explicit, preferred in production)
--mount type=volume,source=pgdata,target=/var/lib/postgresql/data

# Read-only volume (container cannot write to it)
-v configs:/etc/myapp/config:ro
--mount type=volume,source=configs,target=/etc/myapp/config,readonly
```

---

## Bind Mounts — For Development

Bind mounts map a specific path on your host directly into
the container. Changes on either side are immediately visible
on the other side. This is what enables hot reloading in development.

```bash
# Mount current directory into container
docker run -d \
  -v $(pwd):/app \
  -p 3000:3000 \
  node:20 \
  npm run dev

# Now: edit any file in your editor → container sees it immediately

# Specific file mount (for config injection)
docker run -d \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -p 80:80 \
  nginx

# Long syntax
--mount type=bind,source=$(pwd),target=/app
```

### The problem with bind mounts in production

Bind mounts depend on the host path existing and having the right
permissions. If you run the container on a different machine, the
path might not exist. If you change the directory structure, the
mount breaks. They are tightly coupled to the host.

Named volumes have none of these problems. Docker manages them.
They work the same on every machine.

**Use bind mounts for:** development (hot reload), injecting config files
**Use named volumes for:** databases, persistent app data, anything in production

---

## tmpfs Mounts — For Secrets and Temp Data

tmpfs mounts exist only in memory and are never written to disk.
Perfect for sensitive data that should not persist.

```bash
docker run -d \
  --mount type=tmpfs,target=/run/secrets \
  myapp

# Data at /run/secrets is:
# - In memory only
# - Not written to disk
# - Gone when container stops
# - Not visible in docker inspect history
```

---

## Volume Patterns in Practice

### Database with a volume
```bash
docker run -d \
  --name db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_DB=myapp \
  -e POSTGRES_USER=myapp \
  -e POSTGRES_PASSWORD=secret \
  --restart unless-stopped \
  postgres:16
```

### Config injection with read-only bind mount
```bash
docker run -d \
  --name nginx \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/ssl:/etc/ssl/certs:ro \
  -p 80:80 -p 443:443 \
  nginx:alpine
```

### Development with hot reload
```bash
docker run -it \
  --name dev \
  -v $(pwd):/app \          # bind mount source
  -v /app/node_modules \    # anonymous volume to exclude node_modules from bind
  -p 3000:3000 \
  node:20 \
  npm run dev
```

The anonymous volume trick (`-v /app/node_modules`) prevents the
host's `node_modules` (or its absence) from overwriting the
container's `node_modules`. The container keeps its own.

---

## Backing Up and Restoring Volumes

```bash
# Backup: run a temporary container that reads the volume and tar's it
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/pgdata_backup.tar.gz -C /source .

# Restore: run a temporary container that extracts the backup into the volume
docker volume create pgdata_restored
docker run --rm \
  -v pgdata_restored:/target \
  -v $(pwd):/backup:ro \
  alpine \
  tar xzf /backup/pgdata_backup.tar.gz -C /target

# For PostgreSQL specifically, use pg_dump for consistent backups
docker exec postgres_container pg_dump -U myapp myapp > backup.sql
```

---

## Permission Issues — The Most Common Problem

When a container writes to a bind mount, the files are owned
by the UID the container process runs as. If that UID does not
match a user on the host, you see confusing ownership.

```bash
# Example: nginx runs as UID 101 inside the container
# It writes to a bind mount → files on host owned by UID 101
# You (UID 1000) cannot delete them without sudo

# Solutions:
# 1. Match UIDs — run container as your UID
docker run -d --user $(id -u):$(id -g) myapp

# 2. Set directory permissions to be group-writable
chmod 777 /path/to/mount      # broad but works
# or
chmod g+w /path/to/mount      # group writable
chgrp docker /path/to/mount   # owned by docker group

# 3. Use named volumes instead of bind mounts
# Docker manages permissions, you don't have to deal with them
```

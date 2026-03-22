# 03 — Containers

> Running a container is one command. Understanding what happens
> when you run it — and what to do when it breaks — takes longer.
> This note covers the full lifecycle and the debugging toolkit.

---

## The Container Lifecycle

```
              docker run
                  │
                  ▼
            ┌──────────┐
            │ Created  │  ← filesystem allocated, not yet running
            └────┬─────┘
                 │ (process starts)
                 ▼
            ┌──────────┐
            │ Running  │  ← process is active, ports bound
            └────┬─────┘
                 │ (process exits or docker stop)
                 ▼
            ┌──────────┐
            │ Stopped  │  ← process gone, filesystem still exists
            └────┬─────┘
                 │ docker start (or docker rm)
                 ▼           ▼
            ┌──────────┐  ┌──────────┐
            │ Running  │  │ Removed  │  ← filesystem gone
            └──────────┘  └──────────┘
```

A stopped container is not gone. Its filesystem still exists
on disk. You can restart it and your changes are still there.
Only `docker rm` permanently deletes it.

This surprises people who think containers are ephemeral.
They can be persistent — it depends on how you manage them.
In production, you should treat them as ephemeral by design
and use volumes for data that needs to survive.

---

## docker run — Dissected

`docker run` is the most-used and most flag-heavy Docker command.

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARGS]
```

```bash
# Run in background (detached)
docker run -d nginx

# Name the container
docker run -d --name web nginx

# Port mapping: host:container
docker run -d -p 8080:80 nginx
docker run -d -p 127.0.0.1:8080:80 nginx    # only bind on localhost
docker run -d -p 8080:80 -p 8443:443 nginx  # multiple ports

# Environment variables
docker run -d -e APP_ENV=production -e DB_HOST=10.0.1.5 myapp

# Volume mounts
docker run -d -v /data/myapp:/var/lib/myapp nginx    # bind mount
docker run -d -v mydata:/var/lib/myapp nginx          # named volume

# Resource limits
docker run -d --memory=512m --cpus=1.5 myapp
docker run -d --memory=512m --memory-swap=512m myapp  # disable swap

# Restart policy
docker run -d --restart=always nginx          # restart on crash or reboot
docker run -d --restart=on-failure:3 myapp    # restart on failure, max 3 times
docker run -d --restart=unless-stopped nginx  # restart unless manually stopped

# Remove automatically when it exits
docker run --rm ubuntu cat /etc/os-release

# Interactive + allocate a TTY (for shells)
docker run -it ubuntu bash
docker run -it --rm python:3.11 python3

# Run as specific user
docker run -d --user 1001:1001 myapp
docker run -d --user nobody myapp
```

### What `docker run -it` actually does

`-i` (interactive) keeps stdin open so you can type input.
`-t` (tty) allocates a pseudo-terminal so you get a proper shell
prompt with line editing, colors, and cursor movement.

Without `-t`: you can send input but there is no shell prompt and
output looks garbled. Without `-i`: the shell starts but immediately
exits because stdin is closed. You need both together for interactive shells.

---

## Managing Running Containers

```bash
# List containers
docker ps                    # running only
docker ps -a                 # all (including stopped)
docker ps -q                 # just IDs (useful for scripting)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Start / stop / restart
docker stop container_name           # sends SIGTERM, waits 10s, then SIGKILL
docker stop -t 30 container_name     # give it 30s to shut down gracefully
docker start container_name
docker restart container_name
docker kill container_name           # sends SIGKILL immediately (last resort)

# Remove
docker rm container_name             # must be stopped first
docker rm -f container_name          # force remove even if running
docker container prune               # remove all stopped containers

# Bulk operations
docker stop $(docker ps -q)          # stop all running containers
docker rm $(docker ps -aq)           # remove all containers
```

---

## Executing Commands in a Running Container

```bash
# Get a shell in a running container
docker exec -it container_name bash
docker exec -it container_name sh      # for Alpine (no bash)

# Run a one-off command
docker exec container_name cat /etc/nginx/nginx.conf
docker exec container_name ps aux
docker exec container_name env | grep DB_

# Run as root (even if container runs as non-root)
docker exec -it -u root container_name bash

# Run in background
docker exec -d container_name /scripts/cleanup.sh
```

### exec vs run

`docker run` creates a NEW container from an image.
`docker exec` runs a command in an EXISTING running container.

A very common mistake: `docker run myapp bash` — this starts a new
fresh container (not the one already running) and gives you a shell
in it. Your running container is unaffected.

---

## Logs

```bash
# See all logs
docker logs container_name

# Follow logs live (like tail -f)
docker logs -f container_name

# Last N lines
docker logs --tail 50 container_name

# With timestamps
docker logs -t container_name

# Since a time
docker logs --since 1h container_name      # last hour
docker logs --since 2024-01-15T10:30 container_name

# Combine
docker logs -f --tail 100 --since 1h container_name
```

### Why logs only work when apps write to stdout/stderr

Docker captures what the process writes to its stdout and stderr.
If your app writes logs to a file inside the container, `docker logs`
shows nothing — those logs are in the container's filesystem, not
in the Docker log driver.

The correct approach: configure your app to log to stdout/stderr.
In production, Docker routes those to whatever logging driver you
configure (journald, AWS CloudWatch, Datadog, etc.).

```dockerfile
# In many base images, this is already handled.
# nginx official image does this:
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log
```

---

## Inspecting Containers

```bash
# Full JSON metadata about a container
docker inspect container_name

# Extract specific fields with Go template syntax
docker inspect -f '{{.State.Status}}' container_name
docker inspect -f '{{.NetworkSettings.IPAddress}}' container_name
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Or pipe to jq
docker inspect container_name | jq '.[0].State'
docker inspect container_name | jq '.[0].HostConfig.PortBindings'

# Resource usage (live stats)
docker stats                          # all containers
docker stats container_name           # specific container
docker stats --no-stream container_name  # single snapshot, no live update

# Top — processes inside the container
docker top container_name
docker top container_name aux         # with options
```

---

## Copying Files

```bash
# Copy from container to host
docker cp container_name:/etc/nginx/nginx.conf ./nginx.conf

# Copy from host to container
docker cp ./nginx.conf container_name:/etc/nginx/nginx.conf

# Useful for: extracting logs, configs, or build artifacts
# Not a replacement for volumes — files copied this way don't persist
```

---

## Container Networking Basics (Preview)

```bash
# See which ports are exposed
docker port container_name

# Get the container's IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Containers on the same Docker network can reach each other by name
docker network create mynet
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name api myapp
# api container can reach db as: postgres://db:5432/mydb
```

Full networking details are in note 06. But the key point: containers
on the same user-defined network can reach each other by container name.

---

## Common Container Issues and What They Mean

```
Container exits immediately after docker run:
  Cause 1: The main process is designed to exit (e.g. alpine, ubuntu without a command)
  Cause 2: The main process crashed on startup
  Fix: docker run -it ubuntu bash  (give it something to do)
       docker logs container_name   (check what it printed before dying)

"Port already in use":
  Another process (or container) is already using that host port.
  Fix: docker ps -a | grep 8080    (find what's using it)
       lsof -i :8080               (find host process)
       Change the host port: -p 8081:80

"Permission denied" inside container:
  The container process is running as a non-root user
  and doesn't have permission to access a mounted file/directory.
  Fix: Check file ownership on host: ls -la /path/to/mount
       Match UID: --user $(id -u):$(id -g)
       Or fix permissions: chmod/chown the host directory

Container running but app not responding:
  App may be listening on 127.0.0.1 (loopback) instead of 0.0.0.0
  From outside the container, 127.0.0.1 is the container's loopback.
  Fix: Configure app to listen on 0.0.0.0 or check -p flag.
```

---

## The Debugging Workflow

When a container is not behaving correctly, follow this order:

```
1. Is it running?
   docker ps -a
   → not running: check exit code, check logs

2. What did it print before dying?
   docker logs container_name

3. Can I get inside it?
   docker exec -it container_name bash
   → if crashed: docker run -it --entrypoint bash image_name

4. Is the config correct?
   docker exec container_name cat /etc/app/config.yaml
   docker inspect container_name | jq '.[0].Config.Env'

5. Can it reach its dependencies?
   docker exec container_name curl http://db:5432
   docker exec container_name ping db

6. Is it consuming too many resources?
   docker stats container_name
```

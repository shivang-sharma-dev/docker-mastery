# Docker Interview Q&A

> These are the questions that actually come up in DevOps interviews.
> Answers focus on understanding, not memorisation.

---

## Fundamentals

**Q: What is the difference between a Docker image and a container?**

An image is a read-only template — a stack of filesystem layers with
metadata. It is the blueprint. A container is a running instance of
an image. It has the read-only layers from the image, plus a writable
layer on top for any changes made during the container's life.

The analogy: an image is a class definition, a container is an object
instance. You can run many containers from the same image. They all
share the same read-only layers on disk.

---

**Q: How is a container different from a virtual machine?**

A VM includes a full guest operating system kernel. Each VM is fully
isolated at the hardware level via a hypervisor. VMs are heavy —
gigabytes in size, minutes to start.

A container shares the host kernel. Isolation is provided by Linux
namespaces (process, network, filesystem, etc.) and resource limits
via cgroups. Containers are lightweight — megabytes in size,
milliseconds to start.

The trade-off: VMs offer stronger isolation. Containers are faster
and more efficient. In practice, containers run inside VMs in
production environments (EC2 instance running Docker containers).

---

**Q: What is a Docker layer and why does it matter?**

Each instruction in a Dockerfile creates a new filesystem layer.
A layer is a diff — only the changes from the previous layer.
Layers are content-addressed by SHA256 hash and cached.

It matters because:
- If a layer hasn't changed, Docker uses the cached version — builds are fast
- Layers are shared between images — `docker pull` only downloads layers you don't have
- Layer order in your Dockerfile determines build efficiency — frequently-changing content belongs at the bottom so rarely-changed layers stay cached

---

**Q: What happens when you run `docker run nginx`?**

1. Docker checks if the `nginx:latest` image exists locally
2. If not, pulls it from Docker Hub (layer by layer)
3. Creates a container (new PID/net/mnt/uts/ipc namespace)
4. Sets up a writable layer on top of the image
5. Creates a veth pair — one end in the container, one on the host bridge
6. Assigns the container an IP from the bridge subnet
7. Runs the command defined by the image's CMD instruction
8. Returns the container ID (if `-d`) or attaches to its stdout

---

## Dockerfile

**Q: What is the difference between CMD and ENTRYPOINT?**

Both define what runs when the container starts, but differently.

`CMD` is the default command and is completely replaced if you pass
a command to `docker run`. `ENTRYPOINT` is the fixed executable and
is not replaced — arguments to `docker run` become arguments to it.

In practice: use `ENTRYPOINT` for the main executable, `CMD` for
default arguments. Use `CMD` alone when you want the command to be
easily overridable.

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
# docker run myimage           → nginx -g "daemon off;"
# docker run myimage -c /cfg   → nginx -c /cfg
```

---

**Q: Why should you use exec form instead of shell form for CMD?**

Shell form runs your command through `/bin/sh -c`, making the shell
PID 1 in the container. Signals sent to the container (like SIGTERM
from `docker stop`) go to the shell, which may not forward them to
your application. This means graceful shutdown doesn't work.

Exec form runs your process directly as PID 1, receives signals
directly, and shuts down cleanly.

```dockerfile
CMD python app.py           # shell form — sh is PID 1, signals lost
CMD ["python", "app.py"]   # exec form — python is PID 1, signals work
```

---

**Q: Why clean up in the same RUN instruction?**

Docker commits a new layer after each RUN. If you install packages
in one RUN and delete cache in a separate RUN, the cache files are
committed to the first layer — they exist in the image history even
after deletion. Cleaning up in the same RUN ensures they never get
committed to any layer.

```dockerfile
# Cache files committed in layer 1, persist in image despite deletion in layer 2
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*    # too late — already in layer 1

# Nothing committed — all happens in one layer
RUN apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

---

## Volumes and Networking

**Q: What is the difference between a bind mount and a named volume?**

A bind mount maps a specific host path into the container. The exact
host path must exist and you manage it. Changes on either side are
immediately visible on the other. Great for development (hot reload),
brittle for production (depends on host filesystem layout).

A named volume is managed by Docker. You reference it by name, not
path. Docker stores it in `/var/lib/docker/volumes/`. It is portable —
works identically on any Docker host. Preferred for production data.

---

**Q: Why can't containers on the default bridge network reach each other by name?**

The default bridge network does not have an embedded DNS server.
Containers can reach each other by IP address only.

User-defined bridge networks have an embedded DNS server at
`127.0.0.11` that resolves container names. This is why you should
always create a user-defined network for multi-container applications.

Docker Compose does this automatically — every Compose project
gets its own user-defined network.

---

**Q: A container can't connect to the internet. What do you check?**

1. Is IP forwarding enabled on the host?
   `cat /proc/sys/net/ipv4/ip_forward` — must be 1
2. Are iptables MASQUERADE rules present?
   `sudo iptables -t nat -L POSTROUTING`
3. Is there a network configured on the container?
   `docker inspect container | jq '.[0].NetworkSettings'`
4. Did you start Docker with `--iptables=false` (disables Docker's NAT)?
5. Is there a corporate proxy blocking traffic?

---

## Production

**Q: How do you pass secrets to a Docker container without baking them into the image?**

Several approaches, in order of preference:

1. **Secrets manager at runtime** — app fetches its own secrets from
   AWS Secrets Manager or HashiCorp Vault on startup. Nothing in
   the container config, nothing in the image.

2. **Docker secrets** — for Swarm deployments, secrets are mounted
   as files at `/run/secrets/name`. Not accessible from image history.

3. **Environment variables at runtime** — pass via `-e` or env_file
   when running the container, not in the Dockerfile. Visible in
   `docker inspect` but not in the image.

Never use `ENV` in a Dockerfile for secrets — they appear in
`docker history` and anyone who can pull the image can read them.

---

**Q: What is a multi-stage build and why would you use one?**

A multi-stage build uses multiple `FROM` instructions in one Dockerfile.
Earlier stages are used to build/compile the application. The final
stage copies only the built artifacts — not the build tools.

The result is a production image that contains only what is needed
to run the app, not what was needed to build it. A Go app that builds
in a 900MB image runs from a 10MB image when compiled to a static
binary and copied to `FROM scratch`.

Benefits: smaller image (faster pulls, less storage), smaller attack
surface (fewer packages = fewer CVEs), cleaner separation of build
and runtime dependencies.

---

**Q: What does a Docker health check do and why does it matter?**

A health check runs a command inside the container on a schedule.
If the command exits 0, the container is healthy. If it exits
non-zero, it's unhealthy.

Without a health check, Docker only knows if the process is running —
not if it's actually working. A web server process can run fine while
the application is deadlocked and not serving requests.

Orchestrators (Kubernetes, Swarm) use health check status to route
traffic and restart unhealthy containers. Without it, broken containers
keep receiving traffic.

---

**Q: What is the difference between `docker stop` and `docker kill`?**

`docker stop` sends SIGTERM, waits for the container to shut down
gracefully (default 10 seconds), then sends SIGKILL if it's still running.
This allows the application to finish in-flight requests, flush buffers,
and close database connections.

`docker kill` sends SIGKILL immediately (or any signal you specify).
The process is terminated with no chance to clean up. Use as a last
resort when the container won't respond to stop.

---

**Q: You run `docker compose up` and the API container exits immediately. How do you debug it?**

```bash
# Step 1: See what it printed before dying
docker compose logs api

# Step 2: Check the exit code
docker compose ps     # look at the Exit column

# Step 3: Override the entrypoint to get a shell
docker compose run --rm --entrypoint sh api

# Step 4: Try running the CMD manually inside the shell
# This shows the exact error your app is hitting on startup

# Step 5: Check environment variables
docker compose run --rm api env | sort

# Common causes:
# - Database connection failing (db not ready yet)
# - Missing environment variable
# - Wrong file path or permission
# - Port already in use
```

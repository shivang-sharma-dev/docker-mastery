# 02 — Images

> Images are the foundation of everything in Docker.
> If you understand how layers work — truly understand them —
> your builds will be fast, your images will be small,
> and your debugging will be methodical.

---

## What an Image Actually Is

An image is not a file. It is not an archive.
It is a stack of read-only filesystem layers, each one
recording a diff from the layer below it.

```
nginx:latest image (viewed as layers):

  Layer 4  [sha256:a3b2...]  COPY ./app /usr/share/nginx/html    (your files, 50KB)
  Layer 3  [sha256:9f1c...]  RUN apt-get install curl             (curl binary, 8MB)
  Layer 2  [sha256:7e4a...]  RUN apt-get update                   (package lists, 25MB)
  Layer 1  [sha256:2c3d...]  FROM debian:bullseye-slim            (base OS, 80MB)

Total image size on disk: ~113MB
But Layer 1 is shared with every other image that uses debian:bullseye-slim.
```

Each layer is identified by a SHA256 hash of its content.
If the content changes, the hash changes — Docker knows immediately
that a layer is different.

This is not just a storage optimisation. It is the foundation of
the build cache, which is what makes Docker builds fast.

---

## The Layer Cache — Your Most Powerful Build Tool

When Docker builds an image, it checks each instruction against
its cache. If the instruction and all its inputs are identical
to a previous build — same base layer, same command, same files —
Docker reuses the cached layer. It does not re-run the command.

```dockerfile
# This Dockerfile has a critical ordering problem:

FROM python:3.11-slim      # Layer 1: base image
COPY . /app                # Layer 2: copy ALL files
RUN pip install -r requirements.txt  # Layer 3: install deps

# Problem: every time you change ANY file in your project
# (even a README), Layer 2 cache is invalidated.
# Layer 3 re-runs every time. pip install on every build.
# If your dependencies haven't changed, this is pure waste.

# ─────────────────────────────────────────────────────────

# Corrected version — dependencies cached separately:

FROM python:3.11-slim
COPY requirements.txt /app/          # Layer 2: just the deps file
RUN pip install -r /app/requirements.txt  # Layer 3: install (cached unless requirements.txt changes)
COPY . /app                          # Layer 4: copy all other files (changes every time)

# Now: if you only change app code, Layer 2 and 3 are cached.
# pip install only re-runs when requirements.txt changes.
# Builds go from 3 minutes to 5 seconds.
```

**The rule: put things that change less frequently near the top of your Dockerfile.**

---

## Image Naming and Tags

```
docker.io / library / nginx : 1.25.3
    │           │       │       │
    │           │       │       └── tag (version)
    │           │       └────────── image name
    │           └────────────────── namespace (library = official images)
    └────────────────────────────── registry (docker.io = Docker Hub)

When you write just "nginx", Docker expands it to:
  docker.io/library/nginx:latest

When you write "mycompany/myapp:v2.1", Docker expands to:
  docker.io/mycompany/myapp:v2.1

When you write "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v2.1"
Docker knows it's a private ECR registry (not Docker Hub).
```

### Tags are mutable pointers — not fixed versions

This is a subtle but important point. A tag like `nginx:latest`
is just a pointer. The image it points to can be updated at any
time by whoever controls the repository. If you pull `nginx:latest`
today and again in 6 months, you may get completely different images.

```bash
# The digest is the immutable identifier
docker pull nginx@sha256:a3b278abe39a3f8b80c9571e8c4abcdef123...
# This always refers to exactly this image, forever.

# For production, pin to a digest or a specific version tag
FROM nginx:1.25.3            # better than nginx:latest
FROM nginx@sha256:a3b278...  # best — truly immutable
```

---

## Working With Images

```bash
# Pull an image from a registry
docker pull nginx                     # latest tag
docker pull nginx:1.25.3              # specific version
docker pull nginx:alpine              # alpine variant (smaller)
docker pull ubuntu:22.04

# List local images
docker images
docker image ls
docker images --filter "dangling=true"    # untagged images (<none>)

# Inspect an image — see all its metadata and layers
docker inspect nginx
docker inspect nginx | jq '.[0].Config'     # just the config section
docker history nginx                         # show all layers and sizes

# Remove images
docker rmi nginx                      # remove by name
docker rmi nginx:1.25.3               # remove specific tag
docker image rm sha256:abc123...      # remove by digest
docker image prune                    # remove all dangling images
docker image prune -a                 # remove all unused images (careful)

# Search Docker Hub
docker search nginx
docker search --filter stars=50 python
```

---

## Image Variants — Choosing the Right Base

Every popular image has multiple variants. Choosing correctly
matters for image size, security, and compatibility.

```
nginx:latest         ~180MB  Debian-based, full toolset
nginx:alpine          ~23MB  Alpine Linux, minimal
nginx:slim            ~60MB  Debian slim, some tools removed

python:3.11           ~900MB Full Debian + Python
python:3.11-slim      ~130MB Debian slim + Python
python:3.11-alpine     ~50MB Alpine + Python
python:3.11-bullseye  ~900MB Explicit Debian 11 version

ubuntu:22.04           ~77MB Standard Ubuntu
ubuntu:22.04-minimal   ~30MB Fewer packages
```

### Alpine vs slim vs full — when to use which

**Alpine** — smallest, uses musl libc instead of glibc.
Most software compiled for Linux assumes glibc.
Some Python packages with C extensions won't build on Alpine without
extra steps (`apk add gcc musl-dev`). Good for simple apps,
occasionally painful for complex ones.

**slim** — Debian with non-essential packages removed.
Generally safe. Uses glibc. Most packages work without modification.
Good default choice.

**full** — Everything included. Use during development when you need
debugging tools. Should not go to production.

A practical rule: start with `slim` for production, move to `alpine`
if you need smaller images and are willing to test compatibility.

---

## Where Images Are Stored Locally

```bash
# Docker stores images here (default)
sudo ls /var/lib/docker/overlay2/

# Each layer is a directory
# Layers are shared between images that have the same content
# Two images based on ubuntu:22.04 share the ubuntu layer on disk

# Check disk usage
docker system df                  # summary
docker system df -v               # verbose, per-image

# Remove everything not currently in use (be careful)
docker system prune               # stopped containers + dangling images
docker system prune -a            # also removes unused images
docker system prune -a --volumes  # also removes unused volumes
```

---

## Practical: Understanding What's In an Image

```bash
# Dive — an excellent tool for exploring image layers
# Install: https://github.com/wagoodman/dive
dive nginx:latest

# Without dive: use docker history
docker history nginx:latest
# IMAGE          CREATED       CREATED BY                       SIZE
# a3b278...      2 weeks ago   CMD ["nginx" "-g" "daemon off;"] 0B
# <missing>      2 weeks ago   EXPOSE 80                        0B
# <missing>      2 weeks ago   COPY /etc/nginx /etc/nginx       27.3kB
# <missing>      2 weeks ago   RUN /bin/sh -c apt-get install…  73.4MB
# <missing>      2 weeks ago   ADD file:...                     80.4MB

# Save an image to a tar file (for air-gapped environments)
docker save nginx:latest -o nginx.tar
docker load -i nginx.tar

# Export a container's filesystem (different from save — no layers)
docker export container_name -o container.tar
docker import container.tar myimage:tag
```

---

## The Mental Model to Keep

```
Registry (Docker Hub, ECR, etc.)
  │
  │ docker pull
  ▼
Local image store (/var/lib/docker/)
  │  ┌──────────────────────────────────────────────────┐
  │  │  Layer 4 [writable — per container, deleted on rm]│
  │  │  Layer 3 [read-only, shared]                      │
  │  │  Layer 2 [read-only, shared]                      │
  │  │  Layer 1 [read-only, shared with other images]    │
  │  └──────────────────────────────────────────────────┘
  │
  │ docker run
  ▼
Running container (process with layers mounted via overlay2)
```

Every container running from the same image shares the same
read-only layers. Each container only has a private writable
layer on top. This is why 10 containers from the same image
do not use 10x the disk space.

When a container writes to a file that exists in a read-only layer,
copy-on-write kicks in — the file is copied up to the writable layer,
then modified. The original in the read-only layer is untouched.

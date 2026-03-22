# 01 — What is Docker and Why

---

## The Problem Docker Solved

Before Docker, deploying software looked like this:

```
Developer's laptop:
  Python 3.8, Flask 1.0, PostgreSQL 12, Ubuntu 18.04

Staging server:
  Python 3.6, Flask 0.12, PostgreSQL 10, CentOS 7

Production server:
  Python 3.9, Flask 2.0, PostgreSQL 14, Ubuntu 20.04

Result: The app works on the laptop. Breaks on staging.
        Fixed on staging. Breaks in production.
        "Works on my machine" is not a deployment strategy.
```

The root cause is a dependency problem. Your application
does not run in isolation — it depends on a specific Python version,
specific library versions, specific OS configuration, specific
environment variables. Every server has a slightly different
combination of these, and the differences cause failures.

Docker's answer: **ship the environment with the application.**

Instead of telling a server "install Python 3.8, then Flask 1.0,
then run my app", you say "here is a complete, self-contained box
with Python 3.8, Flask 1.0, and my app already inside it — just run it."

---

## Containers vs Virtual Machines

This comparison comes up in every Docker interview.
Understanding it deeply separates people who use Docker from
people who understand it.

```
VIRTUAL MACHINE
═══════════════════════════════════════════
┌─────────────────────────────────────────┐
│             Your Application            │
├─────────────────────────────────────────┤
│         Guest OS (Ubuntu 20.04)         │  ← full OS kernel
├─────────────────────────────────────────┤
│              Hypervisor                 │  ← VMware, VirtualBox, KVM
├─────────────────────────────────────────┤
│         Host OS (Ubuntu 22.04)          │
├─────────────────────────────────────────┤
│               Hardware                  │
└─────────────────────────────────────────┘

Each VM needs a full operating system.
A minimal Ubuntu VM is ~1GB. Boots in 30–60 seconds.
Strong isolation — each VM has its own kernel.


CONTAINER
═══════════════════════════════════════════
┌──────────┐ ┌──────────┐ ┌──────────────┐
│  App A   │ │  App B   │ │    App C     │
│ + libs   │ │ + libs   │ │   + libs     │
├──────────┴─┴──────────┴─┴──────────────┤
│              Docker Engine             │
├────────────────────────────────────────┤
│          Host OS (Linux kernel)        │  ← shared kernel
├────────────────────────────────────────┤
│                Hardware                │
└────────────────────────────────────────┘

Containers share the host kernel.
A container starts in milliseconds. Uses MBs, not GBs.
Weaker isolation — they share a kernel.
```

### The trade-off in one sentence

VMs give you stronger isolation (separate kernels) at the cost
of size and startup time. Containers give you fast, lightweight
isolation by sharing the host kernel, at the cost of a slightly
smaller isolation boundary.

For most workloads, container isolation is completely sufficient.
For multi-tenant environments where you cannot trust the workloads,
VMs (or a VM + container combination) is the right choice.

---

## What Docker Actually Is

Docker is not one thing. It is a set of tools:

```
docker CLI          What you type: docker run, docker build, docker ps
       │
       │  talks to
       ▼
Docker Daemon       Background service (dockerd)
(Docker Engine)     Manages images, containers, networks, volumes
       │
       │  uses
       ▼
containerd          Industry-standard container runtime
       │
       │  uses
       ▼
runc                Low-level container runtime
                    Calls kernel syscalls: clone(), unshare(), etc.
                    Creates namespaces and cgroups
```

When you run `docker run nginx`, your terminal talks to the Docker
daemon, which tells containerd to pull the image and start a
container, which tells runc to actually create the namespaces and
cgroups and start the nginx process.

The reason this matters: container runtimes are standardised.
Kubernetes uses containerd directly, without Docker. The containers
are identical — it is just a different management layer on top.

---

## The Three Core Concepts

Everything in Docker maps to one of three things:

```
IMAGE
  A read-only template. The recipe.
  Contains: OS files, your application, its dependencies, config.
  Built from a Dockerfile.
  Stored in a registry (Docker Hub, ECR, etc.)
  Analogy: a class definition in object-oriented programming.

CONTAINER
  A running instance of an image.
  Has its own isolated filesystem, network, processes.
  Can be started, stopped, restarted, deleted.
  Writable layer on top of the read-only image.
  Analogy: an object instance created from a class.

REGISTRY
  Where images are stored and shared.
  Docker Hub is the public default.
  ECR (AWS), GCR (Google), ACR (Azure) are common private ones.
  Analogy: GitHub for images.


The relationship:
  Dockerfile ──build──► Image ──run──► Container
                          │
                          └──push──► Registry ──pull──► Image (another machine)
```

---

## Under the Hood — What Makes a Container

If you did linux-for-devops note 15, you already know this.
If not, here is the essential version.

A container is a Linux process given three things by the kernel:

**1. Namespaces — isolation**
```
The container gets its own isolated view of:
  pid namespace     → its own PID 1, can't see host processes
  net namespace     → its own network interfaces, IP, routing
  mnt namespace     → its own filesystem tree (its own /)
  uts namespace     → its own hostname
  ipc namespace     → its own IPC resources
  user namespace    → can map UID 0 inside to non-root outside

Each namespace type creates a separate "room" the container
lives in. From inside, the container thinks it is the only
tenant on the machine. The host can see everything.
```

**2. cgroups — resource limits**
```
The kernel enforces:
  How much CPU this container can use
  How much memory it can consume
  How much disk I/O it can do
  How much network bandwidth it can use

Without cgroups, one runaway container could starve all others.
This is why --memory=512m and --cpus=1.5 work.
```

**3. Union filesystem — layered storage**
```
The image is made of read-only layers.
When the container runs, a thin writable layer is added on top.
All writes go to that top layer.
When the container is deleted, that writable layer is deleted too.
The image layers below are untouched and shared between containers.

This is why:
  Data written inside a container disappears when it is deleted
  10 containers running the same image don't use 10x the disk space
  Docker builds are fast (unchanged layers are cached)
```

---

## Why Docker Won

Before Docker (2013), there were other container technologies:
LXC, OpenVZ, Solaris Zones. They all used the same kernel features.
Docker did not invent containers.

What Docker added was:

1. **A great image format** — layered, cacheable, shareable
2. **Docker Hub** — a public registry that made sharing trivial
3. **A simple CLI** — `docker run` was genuinely easier than anything before
4. **The Dockerfile** — a reproducible way to build images

The combination of easy image building, easy sharing, and easy
running created a network effect. Everyone moved to Docker because
everyone else was on Docker.

---

## Installing Docker

```bash
# Ubuntu (official method)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo "deb [signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add yourself to docker group (no sudo needed)
sudo usermod -aG docker $USER
newgrp docker       # apply without logout

# Verify
docker run hello-world
```

---

## Your First Real Container

```bash
# Run nginx and expose it on port 8080
docker run -d -p 8080:80 --name my-nginx nginx

# Check it's running
docker ps
curl localhost:8080      # should return nginx HTML

# Look inside the running container
docker exec -it my-nginx bash
# Inside: cat /etc/nginx/nginx.conf
# Inside: ps aux           (only nginx processes)
# Inside: hostname         (container ID, not your machine)
exit

# Check logs
docker logs my-nginx
docker logs -f my-nginx   # follow live

# Stop and remove
docker stop my-nginx
docker rm my-nginx
```

That six-command sequence — run, exec, logs, stop, rm — is 80%
of your daily Docker container workflow. The next note goes much
deeper on all of it.

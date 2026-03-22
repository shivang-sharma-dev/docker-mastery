# Docker Mastery

> Learn Docker the way engineers actually use it — not every flag and
> subcommand, but the core model deeply enough that everything else
> makes sense when you need it.

---

## A Different Approach to Learning Docker

Most Docker tutorials give you a list of commands.
You memorise them, pass the interview, forget them a week later.

This repo does something different. Each note file builds a mental
model first — what is actually happening, why it works this way,
what breaks when you get it wrong. The commands follow naturally
from understanding.

By the end you will not just know how to run Docker.
You will know why every flag exists, what happens inside the
container when you run it, and how to debug it when it breaks.

---

## The Mental Model You're Building

```
THE SHIPPING CONTAINER ANALOGY
═══════════════════════════════════════════════════════════════════

  Before shipping containers (1950s):
    Every port, every ship, every truck had different dimensions.
    Loading cargo took days. Goods were damaged in transit.
    The same banana required 20 different handling methods
    depending on where it was going.

  After shipping containers:
    One standard box. Works on every ship, truck, and crane.
    Load it once, move it anywhere. Contents irrelevant to handler.

  Before Docker (pre-2013):
    "Works on my machine" — the most dreaded phrase in engineering.
    Your app needed specific OS, specific library versions,
    specific config. Deploying meant hours of environment setup.
    Moving between servers was painful. Scaling was manual.

  After Docker:
    Package your app and everything it needs into one image.
    Run it anywhere — laptop, staging, production — identically.
    Scale by running more containers. No "works on my machine".


WHAT A CONTAINER ACTUALLY IS
═══════════════════════════════════════════════════════════════════

  A container is NOT a virtual machine.
  A container IS a process with boundaries.

  Specifically: a Linux process with
    ├── Namespace isolation   → its own view of filesystem, network, PIDs
    ├── cgroup limits         → bounded CPU, memory, disk I/O
    └── Union filesystem      → layered, copy-on-write file system

  Your container and your host share the same Linux kernel.
  That is why containers start in milliseconds (no OS to boot)
  and why a kernel vulnerability can affect both.

  (If you completed linux-for-devops note 15, you already know
   this at the kernel level. This is that knowledge applied.)
```

---

## What This Repo Covers

The 85% of Docker that you will use every single day:

| # | Topic | Why It's in Here |
|---|---|---|
| 01 | What is Docker and Why | The mental model. Skip this and nothing else sticks. |
| 02 | Images | How images work, layers, caching — misunderstand this and your builds will be slow and broken |
| 03 | Containers | Running, inspecting, debugging containers — your daily workflow |
| 04 | Dockerfile | Writing Dockerfiles that are small, fast, and maintainable |
| 05 | Volumes and Storage | Data that survives container restarts — databases, logs, uploads |
| 06 | Networking | How containers find each other — the most common source of confusion |
| 07 | Docker Compose | Running multi-container apps as a single unit |
| 08 | Building and Sharing Images | Tagging, pushing, pulling from registries — the deployment pipeline |
| 09 | Multi-Stage Builds | Production images that are 10x smaller and more secure |
| 10 | Docker in Production | What changes when real traffic hits your containers |

What is deliberately **not** in here: `docker swarm` (use Kubernetes),
`docker checkpoint` (experimental), `docker plugin` (you won't need it),
and every obscure flag that has a use case once a year.

---

## Prerequisites

- Completed `linux-for-devops` (especially note 15 — Linux for Containers)
  or comfortable with Linux terminal, processes, and filesystems
- A Linux machine, macOS, or Windows with WSL2
- Docker installed — see https://docs.docker.com/get-docker/

---

## Repo Structure

```
docker-mastery/
│
├── 01-notes/          10 focused notes — mental models + practical depth
├── 02-cheatsheet/     every command you need, grouped by task
├── 03-hands-on-labs/  guided exercises that build on each other
├── 04-projects/       real things to build and put on your resume
├── 05-interview-prep/ what interviewers actually ask
└── 06-resources/      the best external material, curated
```

---

## Quick Start Verification

Once Docker is installed, verify your setup:

```bash
# Check Docker is running
docker --version
docker info

# Run your first container
docker run hello-world

# Run an interactive Ubuntu container
docker run -it ubuntu bash

# Inside the container — notice it's isolated
cat /etc/os-release
ps aux               # only sees its own processes
exit

# See that it's gone
docker ps            # no containers running
```

If `docker run hello-world` prints "Hello from Docker!" — you are ready.

---

## Progress Tracker

- [ ] 01 What is Docker and Why
- [ ] 02 Images
- [ ] 03 Containers
- [ ] 04 Dockerfile
- [ ] 05 Volumes and Storage
- [ ] 06 Networking
- [ ] 07 Docker Compose
- [ ] 08 Building and Sharing Images
- [ ] 09 Multi-Stage Builds
- [ ] 10 Docker in Production
- [ ] All lab tasks completed
- [ ] At least one project built
- [ ] Interview prep reviewed

---

*Part of the [DevOps Mastery](https://github.com/yourusername) learning series.*

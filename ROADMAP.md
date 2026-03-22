# Docker Mastery — Roadmap

> 3 weeks at 1–2 hours a day.
> Each phase has a clear "you are ready to move on" checkpoint.
> Do not skip phases — each one is load-bearing for the next.

---

## How This Roadmap Works

Docker has a steep learning curve not because it is complicated
but because people learn commands before they learn concepts.
Then when something breaks, they have no mental model to debug with.

This roadmap front-loads understanding deliberately.
Phase 1 is almost entirely concepts. That investment pays off
in Phase 2 and 3 when everything clicks instead of being memorised.

---

## The Full Arc

```
PHASE 1                   PHASE 2                  PHASE 3
Understanding             Building                 Production
─────────────             ────────                 ──────────
What Docker is      ───►  Dockerfile         ───►  Multi-stage builds
How images work           Volumes                  Docker in production
Running containers        Networking               Debugging
                          Docker Compose
                          Registries
```

---

## Phase 1 — Understanding (Week 1)

**Goal:** Build the mental model. Know what is happening inside
the container, not just how to make it run.

| Day | Note | Time | What you will understand |
|---|---|---|---|
| 1 | `01-what-is-docker-and-why.md` | 1.5 hr | Why Docker exists, VMs vs containers, the Linux kernel connection |
| 2 | `02-images.md` | 2 hr | Layers, caching, how `docker pull` actually works, image storage |
| 3 | `03-containers.md` | 2 hr | Container lifecycle, inspecting, debugging, exec |
| 4–5 | Lab tasks 01–03 | 2 hr | Hands-on with images and containers |

**Phase 1 checkpoint — before moving on:**
- You can explain what a Docker image layer is without looking it up
- You can run a container, exec into it, check its logs, and kill it
- You can explain why `docker run ubuntu` exits immediately
- You understand what happens to data when a container is removed

---

## Phase 2 — Building (Week 2)

**Goal:** Write Dockerfiles that actually work in production.
Understand volumes and networking deeply enough to compose multi-service apps.

| Day | Note | Time | What you will be able to do |
|---|---|---|---|
| 1 | `04-dockerfile.md` | 2.5 hr | Write a production Dockerfile from scratch |
| 2 | `05-volumes-and-storage.md` | 1.5 hr | Persist data, mount configs, understand bind vs volume |
| 3 | `06-networking.md` | 2 hr | Connect containers, understand bridge/host/none |
| 4 | `07-docker-compose.md` | 2 hr | Define a full stack in one YAML file |
| 5 | `08-building-and-sharing-images.md` | 1.5 hr | Tag, push, pull, manage registries |
| 6–7 | Lab tasks 04–07 | 3 hr | Build and run a real multi-container app |

**Phase 2 checkpoint — before moving on:**
- You can write a Dockerfile for a Python/Node app from scratch
- You can run a full stack (app + database + cache) with one `docker compose up`
- You can push an image to Docker Hub and pull it on another machine
- You know the difference between a bind mount and a named volume and when to use each

---

## Phase 3 — Production (Week 3)

**Goal:** The gap between "works on my machine" and "runs reliably in prod".

| Day | Note | Time | What you will be able to do |
|---|---|---|---|
| 1 | `09-multi-stage-builds.md` | 2 hr | Build images that are 5–10x smaller |
| 2–3 | `10-docker-in-production.md` | 2.5 hr | Health checks, resource limits, logging, secrets |
| 4–7 | Projects | 4–6 hr | Build something real for your portfolio |

**Phase 3 checkpoint:**
- Your production images use multi-stage builds
- Your containers have health checks and resource limits
- You know how to get logs out of containers in production
- You know how to pass secrets to containers without baking them in

---

## Topic Dependency Map

```
what-is-docker
      │
      ├──► images ──────────────────────────────► building-and-sharing
      │       │                                           │
      │       └──► dockerfile ──► multi-stage-builds     │
      │                │                                  │
      └──► containers  │                                  │
               │       └──► volumes-and-storage           │
               │       └──► networking ──► docker-compose ┘
               │
               └──► docker-in-production
```

---

## The Question That Tests Your Understanding

At the end of each phase, try to answer this question
at a deeper level than before:

> "I run `docker run -p 8080:80 nginx`.
>  What exactly happens — from the kernel up?"

**After Phase 1 you should say:**
Docker pulls the nginx image if not cached, creates a new container
(a process with namespace and cgroup isolation), starts it, and maps
port 8080 on the host to port 80 inside the container.

**After Phase 2 you should say:**
...the image is made of layers stored in the overlay2 filesystem.
The container gets a writable layer on top. Docker creates a veth
pair connecting the container's network namespace to the docker0
bridge. iptables rules on the host forward port 8080 to the
container's internal IP on port 80.

**After Phase 3 you should say:**
...in production I would also add a health check so orchestrators
know when nginx is actually ready, set memory/CPU limits via cgroup
configuration, ensure logs go to stdout so they are captured by the
container runtime, and use a non-root user in the Dockerfile.

---

## Time Estimates

| Your background | Time to complete |
|---|---|
| No Docker experience | 3–4 weeks |
| Used Docker commands but not deeply | 1.5–2 weeks |
| Know Docker, filling gaps | 3–5 days |
| Pre-interview review | 1 day (cheatsheet + interview prep) |

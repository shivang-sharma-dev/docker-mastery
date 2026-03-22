# Docker Mastery

A comprehensive, practical guide to learning Docker for everyday engineering tasks. 

---

## Overview

This repository is designed to help you understand how Docker actually works under the hood so that the commands and concepts feel natural. Instead of just memorizing flags, we focus on building a strong mental model of containers.

You'll learn:
- How containers differ from virtual machines
- The relationship between namespaces, cgroups, and union filesystems
- How to structure your images efficiently
- How to debug containers when things go wrong

---

## Core Concepts

### The Container Model
Unlike virtual machines that bundle an entire operating system, containers are isolated processes sharing the host's Linux kernel. They achieve isolation through:
- **Namespaces**: Provide isolated views of the filesystem, network, and process IDs.
- **cgroups**: Enforce limits on CPU, memory, and disk I/O.
- **Union Filesystem**: Enables a layered, copy-on-write file system for efficient image storage.

---

## Course Contents

| # | Topic | Description |
|---|---|---|
| 01 | Introduction | What Docker is and why we use it |
| 02 | Images | Image layers, caching, and building |
| 03 | Containers | Running, inspecting, and debugging |
| 04 | Dockerfile | Writing optimized Dockerfiles |
| 05 | Volumes & Storage | Managing persistent data |
| 06 | Networking | Container communication and networking models |
| 07 | Docker Compose | Managing multi-container applications |
| 08 | Registries | Tagging, pushing, and pulling images |
| 09 | Multi-Stage Builds | Optimizing images for production |
| 10 | Production Docker | Best practices for real-world deployments |

---

## Prerequisites

- Basic familiarity with Linux terminal and filesystems
- A machine running Linux, macOS, or Windows with WSL2
- [Docker installed](https://docs.docker.com/get-docker/)

---

## Repository Structure

```text
docker-mastery/
├── 01-notes/          # Core concepts and practical guides
├── 02-cheatsheet/     # Quick reference for common commands
├── 03-hands-on-labs/  # Guided exercises
├── 04-projects/       # Practical application projects
├── 05-interview-prep/ # Common Docker interview questions
└── 06-resources/      # Curated external learning material
```

---

## Verification

Once Docker is installed, verify your setup by running:

```bash
# Check Docker version
docker --version
docker info

# Run a test container
docker run hello-world

# Run an interactive Ubuntu container
docker run -it ubuntu bash

# Inside the container, you can check its isolated state:
cat /etc/os-release
ps aux
exit

# Ensure no hanging containers are left
docker ps
```

If the `hello-world` container successfully prints "Hello from Docker!", your environment is ready to go.

---

## Progress Tracker

- [ ] 01 Introduction
- [ ] 02 Images
- [ ] 03 Containers
- [ ] 04 Dockerfile
- [ ] 05 Volumes & Storage
- [ ] 06 Networking
- [ ] 07 Docker Compose
- [ ] 08 Registries
- [ ] 09 Multi-Stage Builds
- [ ] 10 Production Docker
- [ ] Complete all lab tasks
- [ ] Build at least one project
- [ ] Review interview prep

---

*Part of the [DevOps Mastery](https://github.com/yourusername) learning series.*

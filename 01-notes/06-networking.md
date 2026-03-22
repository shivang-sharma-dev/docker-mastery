# 06 — Networking

> Container networking is where most Docker confusion happens.
> "My containers can't talk to each other" is the most common
> Docker problem after "my container exits immediately".
> This note gives you the mental model to debug it every time.

---

## The Core Mental Model

When Docker is installed, it creates a virtual network interface
on your host called `docker0`. It is a software bridge — like
a virtual network switch that containers plug into.

```
HOST MACHINE
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  eth0 (your real NIC)          docker0 (virtual bridge)    │
│  IP: 192.168.1.10              IP: 172.17.0.1               │
│         │                            │                      │
│         │ (iptables NAT)             │                      │
│         └──────────────────────────┐ │                      │
│                                   ↕ │                      │
│                          ┌──────────┤                       │
│                          │  veth0   │ veth1                 │
│                          └────┬─────┘   │                   │
│                               │         │                   │
└───────────────────────────────│─────────│───────────────────┘
                                │         │
              ┌─────────────────┘         └─────────────────┐
              │                                             │
        ┌─────┴──────┐                             ┌────────┴──────┐
        │ Container A│                             │  Container B  │
        │ 172.17.0.2 │                             │  172.17.0.3   │
        └────────────┘                             └───────────────┘

Each container gets its own network namespace with:
  - A virtual ethernet interface (veth pair — one end in container, one on host)
  - An IP address from Docker's subnet (172.17.0.0/16 by default)
  - The docker0 bridge IP as its default gateway
```

Port mapping (`-p 8080:80`) adds an iptables rule:
"traffic arriving at host port 8080, forward to container IP:80"

---

## Network Drivers

Docker ships with several network drivers. These are the ones
that matter:

```
bridge     Default for standalone containers.
           Creates a virtual bridge (docker0 or custom).
           Containers on same bridge can communicate.
           Requires port mapping to expose to host.

host       Container uses the host's network stack directly.
           No isolation — container binds ports directly on host.
           Best performance, worst isolation.
           Linux only (not macOS/Windows).

none       Container has no network interfaces.
           Completely isolated — cannot talk to anything.
           Use for batch jobs that don't need networking.

overlay    For Docker Swarm multi-host networking.
           Containers on different hosts can communicate.
           Not needed unless using Swarm.
```

---

## The Default Bridge vs User-Defined Networks

This is the most important thing to understand about Docker networking.

```
DEFAULT bridge network (docker0):

  Containers CAN reach each other by IP address.
  Containers CANNOT reach each other by name.
  (name resolution does not work on the default bridge)

  docker run -d --name web nginx
  docker run -d --name db postgres
  docker exec web curl http://172.17.0.3:5432  # works (by IP)
  docker exec web curl http://db:5432           # fails (name resolution broken)


USER-DEFINED bridge network:

  Containers CAN reach each other by IP address.
  Containers CAN reach each other by container name.
  (Docker provides automatic DNS resolution)

  docker network create mynet
  docker run -d --name web --network mynet nginx
  docker run -d --name db  --network mynet postgres
  docker exec web curl http://db:5432           # works!
```

**Always use user-defined networks for multi-container applications.**
The default bridge is a legacy leftover. In Docker Compose, every app
gets its own user-defined network automatically.

---

## Managing Networks

```bash
# List networks
docker network ls
# NETWORK ID   NAME      DRIVER    SCOPE
# abc123       bridge    bridge    local    ← the default docker0 bridge
# def456       host      host      local
# ghi789       none      null      local

# Create a user-defined network
docker network create mynet
docker network create --driver bridge --subnet 10.0.1.0/24 mynet

# Connect a running container to a network
docker network connect mynet container_name

# Disconnect
docker network disconnect mynet container_name

# Inspect a network (shows which containers are connected, their IPs)
docker network inspect mynet
docker network inspect mynet | jq '.[0].Containers'

# Remove
docker network rm mynet
docker network prune    # remove all unused networks
```

---

## Port Mapping in Depth

```bash
# Map host port to container port
-p 8080:80           # host:container — all interfaces
-p 127.0.0.1:8080:80 # only bind on localhost (more secure)
-p 8080:80/tcp       # explicit TCP
-p 514:514/udp       # UDP

# Expose a random host port
-p 80                # Docker picks an available host port
docker port container_name 80   # find out which port was assigned

# Multiple ports
-p 8080:80 -p 8443:443

# Publishing all exposed ports (from EXPOSE instructions in Dockerfile)
-P    # capital P — assigns random host ports to all exposed ports
```

---

## How Containers Talk to the Internet

Containers can reach the internet through NAT, same as a machine
behind a home router.

```
Container (172.17.0.2)
  ↓ default gateway → docker0 (172.17.0.1)
  ↓ iptables MASQUERADE rule on host
  ↓ appears to come from host's eth0 IP (192.168.1.10)
  → Internet

Container can pull from npm, pip, apt, curl external APIs.
Container IP is never visible outside the host.
```

---

## DNS Inside Containers

```bash
# Default DNS — uses host's resolv.conf via Docker's embedded DNS
docker exec container_name cat /etc/resolv.conf
# nameserver 127.0.0.11   ← Docker's embedded DNS server
# options ndots:0

# The 127.0.0.11 DNS server:
# - Resolves container names on user-defined networks
# - Forwards external queries to host's configured DNS

# Custom DNS server
docker run -d --dns 8.8.8.8 myapp

# Add /etc/hosts entries
docker run -d --add-host db.internal:10.0.1.5 myapp
```

---

## Practical: Multi-Container Networking

```bash
# Create a network
docker network create appnet

# Database — not exposed to host, only accessible within appnet
docker run -d \
  --name db \
  --network appnet \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# App — accessible to host on port 8080, connects to db by name
docker run -d \
  --name api \
  --network appnet \
  -p 8080:8000 \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
  myapp:latest

# Test: can api reach db?
docker exec api curl http://db:5432
# or
docker exec api python3 -c "import psycopg2; psycopg2.connect('host=db')"
```

---

## Common Networking Problems

```
"Connection refused" when container tries to connect to another container:
  Check: are both on the same network?
    docker inspect container_a | jq '.[0].NetworkSettings.Networks'
    docker inspect container_b | jq '.[0].NetworkSettings.Networks'
  Fix: docker network connect mynet container_a

"Name resolution failed" for container names:
  You're using the default bridge network.
  Fix: create a user-defined network and connect both containers to it.

"Connection refused" from host to container port:
  Check: is the port mapping correct?
    docker port container_name
  Check: is the app listening on 0.0.0.0, not 127.0.0.1?
    docker exec container_name ss -tulnp

"Can't reach internet from container":
  Check: is ip_forward enabled on the host?
    cat /proc/sys/net/ipv4/ip_forward    # should be 1
  Fix: sudo sysctl -w net.ipv4.ip_forward=1
```

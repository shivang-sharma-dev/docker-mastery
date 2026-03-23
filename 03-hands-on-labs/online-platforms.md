# Online Practice Platforms

> Practice Docker in your browser — no installation needed.

---

## Free Platforms

| Platform | Link | What you get | Best for |
|---|---|---|---|
| Play With Docker | https://labs.play-with-docker.com | Full Linux VM with Docker installed, 4-hour session | Any Docker practice |
| KillerCoda | https://killercoda.com/learn | Guided Docker scenarios with step-by-step instructions | Structured learning |
| Docker Labs | https://training.play-with-docker.com | Official Docker tutorials with terminals | Beginners |
| Ivan Velichko's scenarios | https://killercoda.com/iximiuz | Deep dive Docker + containers | Intermediate |

---

## Recommended Practice Order

### Week 1 — Basics (Play With Docker)
1. Pull and run nginx, check it works with curl
2. Run ubuntu interactively, explore the filesystem
3. Run postgres, connect to it with psql
4. Run multiple containers and connect them on a user-defined network

### Week 2 — Building (KillerCoda)
1. Write a Dockerfile for a simple Python Flask app
2. Build the image, run it, verify it works
3. Optimise the Dockerfile for layer caching
4. Add a health check and test it

### Week 3 — Compose (Play With Docker)
1. Write a docker-compose.yml for app + postgres + redis
2. Bring it up, test connectivity between services
3. Add depends_on with health checks
4. Practice bringing it up, modifying, and down

---

## Play With Docker — Getting Started

1. Go to https://labs.play-with-docker.com
2. Log in with your Docker Hub account (free)
3. Click "+ ADD NEW INSTANCE"
4. You get a full Linux terminal with Docker pre-installed

```bash
# Verify
docker --version
docker info

# Start practising
docker run -d -p 8080:80 nginx
curl localhost:8080
```

Sessions last 4 hours. Use it to test ideas without touching your machine.

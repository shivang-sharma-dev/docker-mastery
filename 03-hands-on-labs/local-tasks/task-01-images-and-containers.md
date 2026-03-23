# Task 01 — Images and Containers

**Covers:** Notes 01–03
**Time:** 45 minutes

---

## What You'll Practice

Running containers, exploring them from inside, understanding
what happens when they exit, and working with images.

---

## Tasks

### Part 1: Your First Containers

```bash
# 1. Pull nginx without running it
docker pull nginx:alpine

# 2. Check the image is there and note its size
docker images nginx

# 3. Run it in the background on port 8080
docker run -d --name my-nginx -p 8080:80 nginx:alpine

# 4. Verify it's serving traffic
curl localhost:8080 | head -5

# 5. Check what's running
docker ps
```

**Question:** Why does the nginx image have multiple tags but they're
different sizes? (`nginx:latest` vs `nginx:alpine`)

---

### Part 2: Inside a Container

```bash
# 6. Get a shell inside the running nginx container
docker exec -it my-nginx sh     # alpine has sh, not bash

# Inside the container:
# a. What OS is this?
cat /etc/os-release

# b. What processes are running?
ps aux

# c. What files are in the nginx config?
cat /etc/nginx/nginx.conf

# d. What's different about the filesystem vs your host?
ls /

# Exit
exit
```

**Question:** Why does `ps aux` inside the container show very few
processes compared to running it on your host?

---

### Part 3: Logs

```bash
# 7. Make some requests
curl localhost:8080
curl localhost:8080/nonexistent

# 8. Check the logs
docker logs my-nginx

# 9. Follow logs in one terminal while making requests in another
docker logs -f my-nginx
# (in another terminal) curl localhost:8080
# Watch the log update
```

---

### Part 4: Container Lifecycle

```bash
# 10. Stop the container
docker stop my-nginx

# 11. It's stopped — but not gone
docker ps           # not here
docker ps -a        # but here, with status "Exited"

# 12. Restart it — logs and any files written inside should still be there
docker start my-nginx
curl localhost:8080   # still works

# 13. Remove it (must stop first)
docker stop my-nginx
docker rm my-nginx
docker ps -a          # now it's gone
```

---

### Part 5: Ephemeral Containers

```bash
# 14. Run a container that does one thing and removes itself
docker run --rm ubuntu cat /etc/os-release

# 15. Try to find it afterward
docker ps -a     # it's not there -- --rm cleaned it up

# 16. Run an interactive container temporarily
docker run -it --rm python:3.11-slim python3
>>> import sys; print(sys.version)
>>> exit()
# Container is gone as soon as you exit
```

---

## Bonus Challenge

Run three different web servers simultaneously on different ports:
- nginx on port 8080
- httpd (Apache) on port 8081
- nginx:alpine on port 8082

Verify all three respond, then stop and remove all three with
a single command using `docker ps -q`.

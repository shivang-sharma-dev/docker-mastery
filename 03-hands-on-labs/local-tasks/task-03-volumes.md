# Task 03 — Volumes and Persistent Data

**Covers:** Note 05 (Volumes and Storage)
**Time:** 45 minutes

---

## Part 1: The Problem — Data Disappears

```bash
# Run postgres, create some data
docker run -d \
  --name postgres-no-volume \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=testdb \
  postgres:16-alpine

# Wait for it to be ready
sleep 5

# Create a table and insert data
docker exec -it postgres-no-volume psql -U postgres -d testdb -c "
  CREATE TABLE users (id SERIAL, name TEXT);
  INSERT INTO users (name) VALUES ('alice'), ('bob');
  SELECT * FROM users;
"

# Remove the container
docker rm -f postgres-no-volume

# Start a new postgres container (no volume)
docker run -d \
  --name postgres-no-volume \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=testdb \
  postgres:16-alpine

sleep 5

# Data is gone
docker exec -it postgres-no-volume psql -U postgres -d testdb -c "SELECT * FROM users;"
# ERROR: relation "users" does not exist
docker rm -f postgres-no-volume
```

---

## Part 2: Named Volumes — Data Survives

```bash
# Create a named volume
docker volume create pgdata

# Run postgres with the volume
docker run -d \
  --name postgres-with-volume \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=testdb \
  postgres:16-alpine

sleep 5

# Create data
docker exec -it postgres-with-volume psql -U postgres -d testdb -c "
  CREATE TABLE users (id SERIAL, name TEXT);
  INSERT INTO users (name) VALUES ('alice'), ('bob');
"

# Destroy the container
docker rm -f postgres-with-volume

# Verify volume still exists
docker volume ls | grep pgdata

# New container, same volume — data is still there
docker run -d \
  --name postgres-restored \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

sleep 5

docker exec -it postgres-restored psql -U postgres -d testdb -c "SELECT * FROM users;"
# alice and bob are still here

docker rm -f postgres-restored
```

---

## Part 3: Bind Mounts for Development

```bash
mkdir -p /tmp/nginx-test/html
echo "<h1>Hello from bind mount</h1>" > /tmp/nginx-test/html/index.html

# Mount the html directory into nginx
docker run -d \
  --name nginx-dev \
  -p 8080:80 \
  -v /tmp/nginx-test/html:/usr/share/nginx/html:ro \
  nginx:alpine

# Verify it works
curl localhost:8080

# Now edit the file on your HOST — no container restart needed
echo "<h1>Updated without restart!</h1>" > /tmp/nginx-test/html/index.html
curl localhost:8080  # see the change immediately

docker rm -f nginx-dev
```

---

## Part 4: Backup a Volume

```bash
# Backup the pgdata volume
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/pgdata_backup.tar.gz -C /source .

ls -lh pgdata_backup.tar.gz   # backup file on your host

# Clean up
docker volume rm pgdata
```

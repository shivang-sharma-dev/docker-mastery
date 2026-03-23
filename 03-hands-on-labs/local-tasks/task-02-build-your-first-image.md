# Task 02 — Build Your First Image

**Covers:** Notes 04 (Dockerfile)
**Time:** 1 hour

---

## What You'll Build

A containerised Python Flask web application, starting from
a bad Dockerfile and iterating to a good one.

---

## Setup

Create this directory structure:

```
myapp/
├── app.py
├── requirements.txt
└── Dockerfile
```

**app.py:**
```python
from flask import Flask, jsonify
import os, socket

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({
        "message": "Hello from Docker!",
        "hostname": socket.gethostname(),
        "environment": os.getenv("APP_ENV", "development")
    })

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

**requirements.txt:**
```
flask==3.0.0
gunicorn==21.2.0
```

---

## Part 1: The Bad Dockerfile

Write this Dockerfile first:

```dockerfile
FROM python:3.11
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
CMD python app.py
```

```bash
# Build it
docker build -t myapp:bad .

# Check the size — it's big
docker images myapp:bad

# Run it
docker run -d --name myapp-bad -p 8000:8000 myapp:bad
curl localhost:8000
curl localhost:8000/health

# Clean up
docker stop myapp-bad && docker rm myapp-bad
```

Note the image size. Write it down.

---

## Part 2: Fix It Step by Step

Now rewrite the Dockerfile fixing each problem:

1. Change `FROM python:3.11` to `FROM python:3.11-slim`
2. Fix the layer caching order (requirements.txt before source code)
3. Change CMD to exec form
4. Add a non-root user
5. Add a HEALTHCHECK
6. Add a `.dockerignore` file

Build and compare sizes after each change.

---

## Part 3: Test the Improvements

```bash
# Build the improved version
docker build -t myapp:good .

# Compare sizes
docker images myapp

# Run it
docker run -d --name myapp-good \
  -p 8000:8000 \
  -e APP_ENV=production \
  myapp:good

# Test the app
curl localhost:8000
curl localhost:8000/health

# Verify it's running as non-root
docker exec myapp-good whoami    # should NOT be root

# Check health status
docker inspect --format='{{json .State.Health.Status}}' myapp-good
```

---

## Part 4: Layer Cache in Action

```bash
# Build once (populates cache)
docker build -t myapp:cached .

# Change only app.py (not requirements.txt)
echo "# a comment" >> app.py

# Build again -- notice which steps are CACHED
docker build -t myapp:cached .
# Step 1/N: CACHED
# Step 2/N: CACHED
# ...
# See how pip install is cached even though app.py changed?

# Now change requirements.txt
echo "requests==2.31.0" >> requirements.txt

# Build again -- pip install reruns, app copy is also invalidated
docker build -t myapp:cached .
```

**Question:** Which line in your Dockerfile caused pip install to rerun?
Why does changing requirements.txt invalidate everything after it?

---

## Bonus

Add a second endpoint `/env` that lists all environment variables
(excluding any with "SECRET" or "PASSWORD" in the name).
Pass two environment variables when running the container and
verify they appear in the output.

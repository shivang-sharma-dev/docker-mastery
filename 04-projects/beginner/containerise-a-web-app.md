# Project: Containerise a Web Application

**Level:** Beginner
**Time:** 2–3 hours
**Covers:** Notes 01–05 (images, containers, Dockerfile, volumes)

---

## What You're Building

Take a simple web application, write a production-quality Dockerfile
for it, and make it run identically in development and production.

---

## The Application

Choose one of these or use your own:

**Option A — Python Flask:**
```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "<h1>My Containerised App</h1>"

@app.route("/health")
def health():
    return {"status": "ok"}, 200
```

**Option B — Node.js Express:**
```javascript
// server.js
const express = require("express")
const app = express()
app.get("/", (req, res) => res.send("<h1>My Containerised App</h1>"))
app.get("/health", (req, res) => res.json({ status: "ok" }))
app.listen(3000, "0.0.0.0")
```

---

## Requirements

Your submission must:

1. **Dockerfile** — follows all best practices from note 04:
   - Pinned base image version
   - Dependencies cached before source code
   - Non-root user
   - Health check
   - Exec form CMD
   - .dockerignore present

2. **Image size** — under 150MB for Python, under 200MB for Node

3. **Runs correctly:**
   ```bash
   docker build -t myapp:v1 .
   docker run -d -p 8080:8000 myapp:v1
   curl localhost:8080/health   # returns 200
   ```

4. **Config via environment variables** — app reads at least one
   config value from an env var (e.g. port, log level, app name)

5. **Persistent logs** — mount a volume at `/app/logs` and write
   access logs there so they survive container restarts

---

## Deliverables

```
myapp/
├── Dockerfile
├── .dockerignore
├── requirements.txt  (or package.json)
├── app.py            (or server.js)
└── README.md         (document how to build and run)
```

---

## Evaluation Criteria

- Does it build without errors?
- Is the image reasonably small?
- Does the health check pass?
- Can you read logs from the mounted volume?
- Would you be comfortable running this in production?

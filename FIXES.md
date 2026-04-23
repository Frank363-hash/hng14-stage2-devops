# FIXES.md — Bug Fixes & Remediation Log

**Project:** HNG14 Stage 2 DevOps — Job Processing System  
**Engineer:** Frank363-hash  
**Date:** April 2026  

---

## Overview

This document details every bug identified in the starter codebase, the root cause
of each issue, and the exact remediation applied. Bugs were discovered through manual
code review, local testing, container runtime analysis, and CI/CD pipeline feedback.

---

## FIX 001 — Hardcoded Redis Host in API

**File:** `api/main.py`  
**Line:** 8  
**Severity:** Critical  

**Problem:**  
The Redis client was initialized with `host="localhost"`, which resolves correctly
on a developer's local machine but fails entirely inside a Docker container. In a
containerized environment, each service runs in its own isolated network namespace.
The term `localhost` inside the API container refers to the API container itself —
not the Redis container. This means every attempt to connect to Redis would result
in a connection refused error, making the entire API non-functional in production.

**Fix Applied:**  
```python
# Before
r = redis.Redis(host="localhost", port=6379)

# After
r = redis.Redis(
    host=os.getenv("REDIS_HOST", "redis"),
    port=int(os.getenv("REDIS_PORT", 6379)),
    password=os.getenv("REDIS_PASSWORD", None),
    decode_responses=True
)
```
The host is now read from the `REDIS_HOST` environment variable, defaulting to
`"redis"` — the Docker Compose service name which Docker's internal DNS resolves
automatically to the correct container IP.

---

## FIX 002 — Redis Password Never Used in API

**File:** `api/main.py`  
**Line:** 8  
**Severity:** Critical  

**Problem:**  
The `.env` file contained `REDIS_PASSWORD=supersecretpassword123`, and Redis was
configured to require authentication. However, the Redis client in `main.py` was
initialized without a `password` parameter. This means the API would connect to
Redis without authenticating, causing every Redis operation to fail with an
`NOAUTH Authentication required` error at runtime.

**Fix Applied:**  
Added `password=os.getenv("REDIS_PASSWORD", None)` to the Redis constructor so
the password is read from the environment and passed during connection.

---

## FIX 003 — Redis Byte Responses Not Decoded in API

**File:** `api/main.py`  
**Line:** 8, 22  
**Severity:** Medium  

**Problem:**  
Without `decode_responses=True`, the Redis client returns raw byte strings
(e.g., `b"queued"` instead of `"queued"`). The original code attempted to handle
this with a manual `.decode()` call on line 22, but this approach is fragile —
any additional Redis reads added later would silently return bytes and cause
unexpected type errors.

**Fix Applied:**  
Added `decode_responses=True` to the Redis constructor. This ensures all responses
are automatically decoded to Python strings globally, eliminating the need for
manual `.decode()` calls anywhere in the codebase.

---

## FIX 004 — Incorrect HTTP Status Code on Missing Job

**File:** `api/main.py`  
**Line:** 22  
**Severity:** Critical  

**Problem:**  
When a job ID was not found in Redis, the endpoint returned:
```python
return {"error": "not found"}
```
This sends an HTTP 200 OK response with an error message in the body. HTTP 200
means "success" — returning it for a missing resource violates REST conventions
and breaks client-side error handling. The frontend's polling loop checks only
`data.status !== 'completed'` and has no way to detect a 200 with an error body,
causing the poll to run indefinitely for non-existent jobs.

**Fix Applied:**  
```python
raise HTTPException(status_code=404, detail="Job not found")
```
FastAPI's `HTTPException` sends a proper HTTP 404 Not Found response, which is
the correct semantic for a missing resource and allows clients to handle errors
appropriately.

---

## FIX 005 — Missing Health Endpoint in API

**File:** `api/main.py`  
**Severity:** Critical  

**Problem:**  
No `/health` endpoint existed in the API. Docker's `HEALTHCHECK` instruction and
the `docker-compose.yml` `depends_on: condition: service_healthy` both require a
reliable health endpoint to determine whether the service is ready to accept
traffic. Without it, Docker cannot determine container health and dependent
services cannot start safely.

**Fix Applied:**  
```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

---

## FIX 006 — No Uvicorn Run Block — Server Would Not Start

**File:** `api/main.py`  
**Severity:** Critical  

**Problem:**  
The FastAPI application defined routes but had no mechanism to start the ASGI
server. Inside a Docker container, there is no interactive terminal to manually
run `uvicorn main:app`. Without an explicit start command, the container would
exit immediately on startup with no error message, making this a particularly
difficult bug to diagnose.

**Fix Applied:**  
Added a `CMD` instruction in the Dockerfile:
```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
And a local convenience block in `main.py`:
```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000)
```

---

## FIX 007 — API Not Binding to 0.0.0.0

**File:** `api/main.py`, `api/Dockerfile`  
**Severity:** Critical  

**Problem:**  
Without explicitly setting `host="0.0.0.0"`, uvicorn defaults to binding on
`127.0.0.1` (loopback only). Inside Docker, this means the API only accepts
connections from within its own container. Any request from the frontend container
or the CI integration test would be refused, making the entire application
non-functional in a containerized environment.

**Fix Applied:**  
Set `--host 0.0.0.0` in both the Dockerfile CMD and the local run block to
ensure the server listens on all network interfaces.

---

## FIX 008 — Hardcoded Redis Host in Worker

**File:** `worker/worker.py`  
**Line:** 5  
**Severity:** Critical  

**Problem:**  
Identical to FIX 001. The worker used `host="localhost"` for its Redis connection,
which fails inside Docker for the same reasons. The worker would start, immediately
fail to connect to Redis, and crash — silently dropping all jobs from the queue.

**Fix Applied:**  
```python
r = redis.Redis(
    host=os.getenv("REDIS_HOST", "redis"),
    port=int(os.getenv("REDIS_PORT", 6379)),
    password=os.getenv("REDIS_PASSWORD", None),
    decode_responses=True
)
```

---

## FIX 009 — No Error Handling in Job Processor

**File:** `worker/worker.py`  
**Line:** 11  
**Severity:** Critical  

**Problem:**  
The `process_job` function had no exception handling. If any error occurred during
processing — a Redis timeout, a network interruption, or any unexpected exception
— the function would crash without updating the job status. The job would remain
in `queued` state indefinitely, the frontend would poll forever, and there would
be no log output indicating anything had gone wrong. This is silent data loss.

**Fix Applied:**  
```python
def process_job(job_id):
    try:
        time.sleep(2)
        r.hset(f"job:{job_id}", "status", "completed")
    except Exception as e:
        print(f"Error processing job {job_id}: {e}")
        try:
            r.hset(f"job:{job_id}", "status", "failed")
        except Exception:
            pass
```
Failures are now logged and the job is marked as `failed` so clients receive
a deterministic final state.

---

## FIX 010 — No Graceful Shutdown in Worker

**File:** `worker/worker.py`  
**Line:** 4  
**Severity:** Medium  

**Problem:**  
`import signal` was present but never used. When Docker stops a container, it
sends `SIGTERM` to the main process. If the process does not handle `SIGTERM`,
Docker waits 10 seconds and then sends `SIGKILL`, force-terminating the process.
If the worker is mid-job when this happens, the job is lost and its status is
never updated.

**Fix Applied:**  
```python
def handle_shutdown(signum, frame):
    print("Worker shutting down gracefully...")
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_shutdown)
signal.signal(signal.SIGINT, handle_shutdown)
```

---

## FIX 011 — Hardcoded API URL in Frontend

**File:** `frontend/app.js`  
**Line:** 5  
**Severity:** Critical  

**Problem:**  
The frontend used `const API_URL = "http://localhost:8000"`. Inside Docker, the
frontend container cannot reach `localhost:8000` because the API runs in a
separate container. Every job submission and status check would fail with a
connection refused error, making the frontend completely non-functional in any
containerized or production environment.

**Fix Applied:**  
```javascript
const API_URL = process.env.API_URL || "http://api:8000";
```
The URL is now read from the `API_URL` environment variable, set in
`docker-compose.yml` as `http://api:8000`, using Docker's internal DNS.

---

## FIX 012 — Missing Health Endpoint in Frontend

**File:** `frontend/app.js`  
**Severity:** Critical  

**Problem:**  
No `/health` endpoint existed in the Express server, preventing Docker from
performing health checks on the frontend container and blocking the
`depends_on: condition: service_healthy` chain in docker-compose.

**Fix Applied:**  
```javascript
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});
```

---

## FIX 013 — Unpinned Dependency Versions

**File:** `api/requirements.txt`, `worker/requirements.txt`  
**Severity:** Medium  

**Problem:**  
All Python dependencies were listed without version pins (e.g., `fastapi` instead
of `fastapi==0.111.0`). Unpinned dependencies mean that two builds on different
days may install different versions, introducing breaking changes silently. This
violates reproducibility — a core requirement of production systems.

**Fix Applied:**
fastapi==0.111.0
uvicorn==0.29.0
redis==5.0.4

---

## FIX 014 — Credentials Committed to Public Repository

**File:** `api/.env`  
**Severity:** Critical  

**Problem:**  
The file `api/.env` containing `REDIS_PASSWORD=supersecretpassword123` was
committed directly into the public repository. Any person who clones or forks
this repository has immediate access to the credential. In a real system, this
would require immediate credential rotation and a full security incident review.

**Fix Applied:**  
- Added `.gitignore` with `.env` pattern to prevent future commits
- Removed `api/.env` from git tracking with `git rm --cached api/.env`
- Created `.env.example` at the repository root with placeholder values only
- All secrets are now passed exclusively via environment variables at runtime

---

## FIX 015 — Base Images Contained Critical CVEs

**File:** `api/Dockerfile`, `worker/Dockerfile`, `frontend/Dockerfile`  
**Severity:** High  

**Problem:**  
The initial Dockerfiles used `python:3.11-slim` and `node:20-slim` as base images.
Trivy security scanning identified CRITICAL severity CVEs in these images. Using
vulnerable base images exposes the production environment to known exploits.

**Fix Applied:**  
Switched all base images to their `alpine` variants:
- `python:3.11-slim` → `python:3.11-alpine`
- `node:20-slim` → `node:20-alpine`

Alpine Linux is a minimal security-focused distribution with a significantly
smaller attack surface and fewer pre-installed packages that could contain
vulnerabilities.

---

## FIX 016 — Flake8 PEP8 Violations

**File:** `api/main.py`, `worker/worker.py`  
**Severity:** Low  

**Problem:**  
Both Python files violated PEP8 style conventions enforced by flake8:
- `E302`: Expected 2 blank lines before function definitions, found 1
- `E305`: Expected 2 blank lines after last function definition
- `W292`: No newline at end of file

These caused the lint stage of the CI/CD pipeline to fail, blocking all
subsequent stages.

**Fix Applied:**  
Added required blank lines between all top-level function definitions and
ensured both files terminate with a newline character as required by POSIX.

---

## Summary Table

| ID | File | Issue | Severity |
|----|------|-------|----------|
| 001 | `api/main.py:8` | Hardcoded Redis host | Critical |
| 002 | `api/main.py:8` | Redis password unused | Critical |
| 003 | `api/main.py:8` | Byte responses not decoded | Medium |
| 004 | `api/main.py:22` | Wrong HTTP status on 404 | Critical |
| 005 | `api/main.py` | Missing /health endpoint | Critical |
| 006 | `api/main.py` | No uvicorn run block | Critical |
| 007 | `api/main.py` | Not binding to 0.0.0.0 | Critical |
| 008 | `worker/worker.py:5` | Hardcoded Redis host | Critical |
| 009 | `worker/worker.py:11` | No error handling in processor | Critical |
| 010 | `worker/worker.py:4` | No graceful shutdown | Medium |
| 011 | `frontend/app.js:5` | Hardcoded API URL | Critical |
| 012 | `frontend/app.js` | Missing /health endpoint | Critical |
| 013 | `requirements.txt` | Unpinned dependencies | Medium |
| 014 | `api/.env` | Credentials in git history | Critical |
| 015 | `*/Dockerfile` | Base images with Critical CVEs | High |
| 016 | `*.py` | PEP8 violations (E302, W292) | Low |

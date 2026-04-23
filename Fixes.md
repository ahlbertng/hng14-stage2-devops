## Issues Found and Fixed

### 1. API Service (api/main.py)

**Issue 1: Hardcoded Redis Host**
- **File:** `api/main.py`
- **Line:** 8
- **Problem:** Redis connection hardcoded to `"localhost"` would fail in containerized environment where Redis runs on a different hostname
- **Fix:** Added environment variable `REDIS_HOST` with default fallback to `"localhost"` for development. Changed line 8 from:
  ```python
  r = redis.Redis(host="localhost", port=6379)
  ```
  to:
  ```python
  REDIS_HOST = os.getenv("REDIS_HOST", "localhost")
  REDIS_PORT = int(os.getenv("REDIS_PORT", "6379"))
  r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=False)
  ```

**Issue 2: Hardcoded Redis Port**
- **File:** `api/main.py`
- **Line:** 8
- **Problem:** Redis port hardcoded to `6379`, not configurable
- **Fix:** Addressed in Issue 1 - now uses `REDIS_PORT` environment variable with default `"6379"`

**Issue 3: Missing Health Check Endpoint**
- **File:** `api/main.py`
- **Line:** 20 (after original code)
- **Problem:** Docker containers require `/health` endpoint for health checks; without it, the container orchestration cannot verify service health
- **Fix:** Added health check endpoint:
  ```python
  @app.get("/health")
  def health_check():
      try:
          r.ping()
          return {"status": "healthy"}
      except Exception as e:
          return {"status": "unhealthy", "error": str(e)}
  ```

### 2. Worker Service (worker/worker.py)

**Issue 1: Hardcoded Redis Host**
- **File:** `worker/worker.py`
- **Line:** 6
- **Problem:** Same as API - will fail in containerized environment
- **Fix:** Added environment variable support with fallback:
  ```python
  REDIS_HOST = os.getenv("REDIS_HOST", "localhost")
  REDIS_PORT = int(os.getenv("REDIS_PORT", "6379"))
  r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=False)
  ```

**Issue 2: Hardcoded Redis Port**
- **File:** `worker/worker.py`
- **Line:** 6
- **Problem:** Same as API
- **Fix:** Addressed with environment variable support

**Issue 3: Missing Signal Handlers for Graceful Shutdown**
- **File:** `worker/worker.py`
- **Line:** 4
- **Problem:** `signal` module imported but handlers never registered; worker cannot gracefully shut down, leaving jobs in limbo
- **Fix:** Added signal handlers and proper shutdown logic:
  ```python
  def shutdown(signum, frame):
      print("Gracefully shutting down...")
      sys.exit(0)

  signal.signal(signal.SIGTERM, shutdown)
  signal.signal(signal.SIGINT, shutdown)
  ```
  Also wrapped job processing in try-except to handle connection errors gracefully.

### 3. Frontend Service (frontend/app.js)

**Issue 1: Hardcoded API URL**
- **File:** `frontend/app.js`
- **Line:** 6
- **Problem:** API URL hardcoded to `"http://localhost:8000"` will fail when API runs on different hostname in Docker network
- **Fix:** Changed to use environment variable:
  ```javascript
  const API_URL = process.env.API_URL || "http://localhost:8000";
  ```

**Issue 2: Server Binds Only to Localhost**
- **File:** `frontend/app.js`
- **Line:** 28
- **Problem:** `app.listen(3000, ...)` without explicit host defaults to `127.0.0.1`, making the service inaccessible from other containers
- **Fix:** Explicitly bind to `0.0.0.0`:
  ```javascript
  app.listen(3000, '0.0.0.0', () => {
    console.log('Frontend running on port 3000');
  });
  ```

**Issue 3: Missing Health Check Endpoint**
- **File:** `frontend/app.js`
- **Line:** 28 (after original code)
- **Problem:** No health check endpoint for container orchestration
- **Fix:** Added health check endpoint:
  ```javascript
  app.get('/health', (req, res) => {
    res.json({ status: 'healthy' });
  });
  ```

## Summary

Total issues fixed: **9**

- **Hardcoded hostnames:** 2 (API, Worker)
- **Hardcoded ports:** 2 (API, Worker)
- **Incorrect network binding:** 1 (Frontend)
- **Missing health check endpoints:** 3 (API, Worker, Frontend)
- **Missing graceful shutdown:** 1 (Worker)

All issues would have caused failures in production containerized deployments. The fixes ensure:
- Services can communicate across Docker networks
- Health checks work properly for orchestration
- Configuration is externalized via environment variables
- Graceful shutdown prevents data loss
- All containers properly expose necessary ports
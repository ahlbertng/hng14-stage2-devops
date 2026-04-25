# HNG14 Stage 2 DevOps - Job Processing Application

A containerized, production-ready job processing application with FastAPI backend, Redis queue, worker service, and Express.js frontend.

## Architecture

```
┌─────────────────┐
│   Frontend      │
│  (Express.js)   │
│   Port 3000     │
└────────┬────────┘
         │
         │ (HTTP)
         │
┌────────▼────────┐         ┌──────────────┐
│   API Service   │◄────────┤    Redis     │
│   (FastAPI)     │         │   (Queue)    │
│   Port 8000     │         │   Internal   │
└────────┬────────┘         └──────┬───────┘
         │                         │
         │ (Redis Protocol)        │
         │                         │
┌────────▼────────────────────────▼────────┐
│          Worker Service                   │
│  (Job Processing from Redis Queue)        │
└──────────────────────────────────────────┘

All services communicate over an internal Docker network.
Only the frontend is exposed on the host machine (port 3000).
Redis is not exposed to the host.
```

## Prerequisites

- **Docker**: Version 20.10+ ([Install Docker](https://docs.docker.com/get-docker/))
- **Docker Compose**: Version 1.29+ (usually included with Docker Desktop)
- **Git**: For cloning the repository
- **curl** or **wget**: For testing endpoints (optional)

### System Requirements

- **CPU**: 2+ cores recommended
- **RAM**: 2GB minimum, 4GB recommended
- **Disk Space**: 2GB for container images

## Quick Start

### 1. Clone and Setup

```bash
git clone <repository-url>
cd hng14-stage2-devops
```

### 2. Configure Environment

```bash
# Copy the example environment file
cp .env.example .env

# (Optional) Edit .env if using non-default settings
# Default values are already suitable for local development/testing
nano .env
```

### 3. Build and Start Services

```bash
# Build all container images and start services
docker-compose up -d

# Watch the startup progress (services wait for dependencies)
docker-compose logs -f
```

### 4. Verify Services Are Running

```bash
# Check service status
docker-compose ps

# Check individual service health
docker-compose exec api curl http://localhost:8000/health
docker-compose exec redis redis-cli ping
docker-compose exec frontend wget -q http://localhost:3000/health -O -
```

### 5. Access the Application

Open your browser to: **http://localhost:3000**

You should see the "Job Processor Dashboard" with a "Submit New Job" button.

## Usage

### Submit a Job (Via UI)

1. Open http://localhost:3000
2. Click "Submit New Job"
3. A job ID will appear and the status will update from "queued" → "processing" → "completed"

### Submit a Job (Via API)

```bash
# Create a job
curl -X POST http://localhost:8000/jobs
# Returns: {"job_id": "uuid-here"}

# Check job status
curl http://localhost:8000/jobs/uuid-here
# Returns: {"job_id": "uuid-here", "status": "completed"}
```

### Worker Processing

The worker service:
- Continuously monitors the Redis queue
- Pulls jobs from the queue (FIFO)
- Processes each job (simulates 2-second work)
- Updates job status to "completed"
- Handles graceful shutdowns (SIGTERM, SIGINT)

## Service Details

### Frontend Service
- **Image**: Node.js 18 Alpine
- **Port**: 3000 (host:container)
- **Health Check**: HTTP GET `/health`
- **Environment Variables**:
  - `API_URL`: Backend API endpoint (default: `http://api:8000`)
- **Resources**: 256MB RAM, 0.5 CPU limit

### API Service
- **Image**: Python 3.11 Slim
- **Port**: 8000 (internal only)
- **Health Check**: HTTP GET `/health` (pings Redis)
- **Environment Variables**:
  - `REDIS_HOST`: Redis hostname (default: `redis`)
  - `REDIS_PORT`: Redis port (default: `6379`)
- **Resources**: 512MB RAM, 1 CPU limit
- **Endpoints**:
  - `POST /jobs` - Create a new job
  - `GET /jobs/{job_id}` - Get job status
  - `GET /health` - Health check

### Worker Service
- **Image**: Python 3.11 Slim
- **No exposed ports** (internal communication only)
- **Health Check**: Redis connectivity check
- **Environment Variables**:
  - `REDIS_HOST`: Redis hostname (default: `redis`)
  - `REDIS_PORT`: Redis port (default: `6379`)
- **Resources**: 512MB RAM, 1 CPU limit
- **Features**:
  - Graceful shutdown handling
  - Automatic reconnection on failures

### Redis Service
- **Image**: Redis 7 Alpine
- **Port**: 6379 (internal only, not exposed)
- **Persistence**: Enabled with AOF (Append-Only File)
- **Health Check**: `redis-cli ping`
- **Resources**: 256MB RAM, 0.5 CPU limit
- **Data Volume**: `redis_data` (survives container restarts)

## Stopping and Starting

### Stop Services
```bash
# Gracefully stop all services (data persists)
docker-compose stop

# Restart services (picks up .env changes)
docker-compose up -d
```

### Remove Everything
```bash
# Stop and remove containers
docker-compose down

# Remove images too
docker-compose down --rmi all

# Remove volumes (WARNING: deletes Redis data)
docker-compose down -v
```

## Logs and Debugging

```bash
# View all service logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f api
docker-compose logs -f worker
docker-compose logs -f frontend
docker-compose logs -f redis

# View last 100 lines
docker-compose logs --tail=100 api

# Show logs since 5 minutes ago
docker-compose logs --since 5m api
```

### Check Service Health Directly

```bash
# API health
docker-compose exec api curl http://localhost:8000/health

# Redis health
docker-compose exec redis redis-cli ping

# Frontend health
docker-compose exec frontend wget -q http://localhost:3000/health -O -

# Worker health (checks Redis)
docker-compose exec worker python healthcheck.py
```

## Network Architecture

All services communicate over an internal Docker bridge network named `internal`:
- Services can reach each other by hostname (e.g., `http://api:8000`)
- **Redis** is not accessible from the host machine (port 6379 is not published)
- **API** is not directly accessible from the host (only through frontend)
- **Frontend** is the only service exposed to the host (port 3000)

```
Host Machine (localhost:3000)
         │
         ▼
    ┌─────────────┐
    │  Frontend   │
    │  :3000      │
    └──────┬──────┘
           │
    Internal Network (internal)
    ────────────────────────────
    │          │          │
    ▼          ▼          ▼
┌─────┐   ┌────────┐  ┌───────┐
│Redis│   │  API   │  │Worker │
│:6379│   │:8000   │  │       │
└─────┘   └────────┘  └───────┘
```

## Environment Configuration

See `.env.example` for all available environment variables. Copy to `.env` and modify as needed.

### Key Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `REDIS_HOST` | `redis` | Redis service hostname |
| `REDIS_PORT` | `6379` | Redis service port |
| `API_URL` | `http://api:8000` | Frontend's API endpoint |

## Troubleshooting

### Services Won't Start

```bash
# Check for port conflicts
lsof -i :3000  # Check if port 3000 is in use

# Check Docker daemon is running
docker ps

# View detailed error logs
docker-compose logs api
```

### Can't Connect to API from Frontend

- Ensure API service passed health check: `docker-compose ps`
- Check network connectivity: `docker-compose exec frontend ping api`
- Verify `API_URL` environment variable is correct

### Redis Connection Errors

```bash
# Verify Redis is running and healthy
docker-compose exec redis redis-cli ping
# Should return: PONG

# Check Redis port
docker-compose exec redis redis-cli info server
```

### Jobs Not Processing

```bash
# Check worker logs
docker-compose logs worker

# Verify worker can reach Redis
docker-compose exec worker python healthcheck.py

# Check Redis queue
docker-compose exec redis redis-cli llen job
```

## Production Deployment

### Building Custom Images

```bash
# Build specific service
docker build -t my-app/api:v1 ./api
docker build -t my-app/worker:v1 ./worker
docker build -t my-app/frontend:v1 ./frontend

# Push to registry (example with Docker Hub)
docker tag my-app/api:v1 username/api:v1
docker push username/api:v1
```

### Rolling Updates

```bash
# Update service image
docker-compose pull
docker-compose up -d

# The compose file handles rolling updates:
# 1. New container starts
# 2. Health checks verify it's ready
# 3. Traffic switches to new container
# 4. Old container stops
```

## Security Considerations

- ✅ All containers run as non-root user (UID 1000)
- ✅ No secrets or credentials in images
- ✅ No hardcoded sensitive data
- ✅ All configuration via environment variables
- ✅ Health checks prevent unhealthy container traffic
- ✅ Resource limits prevent denial-of-service
- ⚠️ Redis should have password protection in production
- ⚠️ API should have authentication/authorization
- ⚠️ Use TLS/HTTPS for production deployments

## CI/CD Pipeline

The GitHub Actions workflow (`/.github/workflows/ci-cd.yml`) includes:

1. **Lint** - Python (flake8), JavaScript (eslint), Dockerfiles (hadolint)
2. **Test** - Unit tests with coverage reporting
3. **Build** - Build all images with SHA and latest tags
4. **Security Scan** - Trivy vulnerability scanning
5. **Integration Test** - Full stack smoke test
6. **Deploy** - Rolling update to production

Runs on push to `main` and pull requests.

## API Documentation

### Create Job
```
POST /jobs
Response: {"job_id": "string"}
Status codes: 200 (success), 500 (server error)
```

### Get Job Status
```
GET /jobs/{job_id}
Response: {"job_id": "string", "status": "string"}
Status codes: 200 (success), 404 (not found), 500 (server error)
Possible statuses: queued, processing, completed, failed
```

### Health Check
```
GET /health
Response: {"status": "healthy"} or {"status": "unhealthy", "error": "string"}
Status codes: 200 (success)
```

## Development

### Local Testing Without Docker

```bash
# Install dependencies
pip install -r api/requirements.txt
pip install -r worker/requirements.txt
npm install --prefix frontend

# Start Redis (separate terminal)
docker run -p 6379:6379 redis:7-alpine

# Start API (separate terminal)
cd api && uvicorn main:app --reload

# Start Worker (separate terminal)
cd worker && python worker.py

# Start Frontend (separate terminal)
cd frontend && npm start
```

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Express.js Documentation](https://expressjs.com/)
- [Redis Documentation](https://redis.io/documentation)

## License

MIT

## Support

For issues or questions, please check the logs and review the [Troubleshooting](#troubleshooting) section.


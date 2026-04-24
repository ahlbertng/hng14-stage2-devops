# Stage 1: Build
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
# hadolint ignore=DL3008
RUN apt-get update && apt-get install -y --no-install-recommends gcc && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim AS runtime

WORKDIR /app

# Create non-root user
RUN useradd -m -u 1000 appuser

# Copy Python packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY worker.py .

# Create a health check script
RUN printf '%s\n' \
    'import redis' \
    'import os' \
    'import sys' \
    '' \
    'try:' \
    '    host = os.getenv("REDIS_HOST", "redis")' \
    '    port = int(os.getenv("REDIS_PORT", "6379"))' \
    '    r = redis.Redis(host=host, port=port, socket_connect_timeout=2)' \
    '    r.ping()' \
    '    sys.exit(0)' \
    'except Exception as e:' \
    '    print(f"Health check failed: {e}")' \
    '    sys.exit(1)' \
    > healthcheck.py

# Set environment variables
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Switch to non-root user
USER appuser

# Health check - verify Redis connectivity
HEALTHCHECK --interval=10s --timeout=5s --start-period=10s --retries=3 \
    CMD python healthcheck.py

# Start application
CMD ["python", "worker.py"]

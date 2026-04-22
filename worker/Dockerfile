FROM python:3.11-slim AS builder

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ─── STAGE 2: PRODUCTION ──────────────────────────────────────
FROM python:3.11-slim

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

COPY --from=builder /install /usr/local

COPY worker.py .

RUN chown -R appuser:appgroup /app

USER appuser

# Worker has no HTTP port to expose
# Health check — verify Redis connection is alive
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
    CMD python -c "import redis, os; r = redis.Redis(host=os.getenv('REDIS_HOST','redis'), password=os.getenv('REDIS_PASSWORD', None)); r.ping()"

CMD ["python", "worker.py"]
# ─── STAGE 1: BUILD ────────────────────────────────────────
FROM python:3.11-slim AS builder

# Set working directory inside the build stage
WORKDIR /app

# Copy only requirements first (Docker layer caching)
COPY requirements.txt .

# Install dependencies into a separate folder
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ─── STAGE 2: PRODUCTION ──────────────────────────────────────
FROM python:3.11-slim

# Create non-root user and group
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Set working directory
WORKDIR /app

# Copy installed packages from builder stage
COPY --from=builder /install /usr/local

# Copy application code
COPY main.py .

# Give ownership to non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose the port the API runs on
EXPOSE 8000

# Health check — hits our /health endpoint every 30s
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

# Start the server
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
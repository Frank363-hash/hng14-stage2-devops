FROM node:20-slim AS builder

WORKDIR /app

# Copy package files first (layer caching)
COPY package.json .

# Install ALL dependencies including devDependencies
RUN npm install --omit=dev

# ─── STAGE 2: PRODUCTION ──────────────────────────────────────
FROM node:20-slim

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

# Copy only production node_modules from builder
COPY --from=builder /app/node_modules ./node_modules

# Copy application files
COPY app.js .
COPY views/ ./views/

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["node", "app.js"]
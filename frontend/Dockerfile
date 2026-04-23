# ─── STAGE 1: BUILD ───────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json .

RUN npm install --omit=dev

# ─── STAGE 2: PRODUCTION ──────────────────────────────────────
FROM node:20-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules

COPY app.js .
COPY views/ ./views/

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["node", "app.js"]

# docker/frontend/Dockerfile
# OC-Web Frontend
#
# Multi-stage build for the oc-web React application.
# Connects to OpenCode backend for API/SSE communication.
#
# Build stages:
# 1. deps - Install dependencies
# 2. builder - Build the application
# 3. runtime - Production runtime

# ====================
# Stage 1: Dependencies
# ====================
FROM oven/bun:1-alpine AS deps

WORKDIR /app

# Copy package files for dependency installation
COPY package.json bun.lock* ./

# Install dependencies
# Note: Use --frozen-lockfile in CI/production when lockfile is stable
RUN bun install

# ====================
# Stage 2: Builder
# ====================
FROM oven/bun:1-alpine AS builder

WORKDIR /app

# Copy dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy source code
COPY . .

# Set build-time environment variables
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

# Build the application
RUN bun run build

# ====================
# Stage 3: Runtime
# ====================
FROM oven/bun:1-alpine AS runtime

WORKDIR /app

# Install curl for health checks
RUN apk add --no-cache curl

# Copy built assets and server files
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
COPY --from=builder /app/server.ts ./

# Copy required library files for runtime
COPY --from=builder /app/src/lib/opencode-config.ts ./src/lib/
COPY --from=builder /app/packages/opencode-web/sse-proxy.ts ./packages/opencode-web/

# Create non-root user for security
RUN addgroup -g 1001 -S ocweb && \
    adduser -S ocweb -u 1001 -G ocweb && \
    chown -R ocweb:ocweb /app

# Set runtime environment variables
ENV NODE_ENV=production
ENV OPENCODE_WEB_HOST=0.0.0.0
ENV OPENCODE_WEB_PORT=3000

# Health check - verify frontend is responding
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/ || exit 1

# Frontend port
EXPOSE 3000

# Run as non-root user
USER ocweb

# Start the production server
CMD ["bun", "server.ts"]

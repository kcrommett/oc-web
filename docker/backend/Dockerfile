# docker/backend/Dockerfile
# OpenCode Backend Server
#
# Runs the OpenCode server that provides the REST API and manages
# AI coding sessions. Connects to LLM proxy for AI capabilities.
#
# Key features:
# - Alpine Linux for minimal footprint
# - Bun runtime for fast JavaScript execution
# - OpenCode CLI installed globally
# - Mounts workspace, git config, and SSH keys from host

FROM alpine:3.19

# Install system dependencies
RUN apk add --no-cache \
    curl \
    git \
    openssh-client \
    bash \
    ca-certificates \
    unzip \
    # Required for some npm packages with native bindings
    libc6-compat

# Install Bun
ENV BUN_INSTALL=/usr/local
RUN curl -fsSL https://bun.sh/install | bash

# Verify Bun installation
RUN bun --version

# Install OpenCode globally
RUN bun install -g opencode

# Create necessary directories
RUN mkdir -p /workspace /root/.config/opencode

# Set working directory to workspace
WORKDIR /workspace

# Health check - verify OpenCode server is responding
# The /config endpoint returns server configuration
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:4096/config || exit 1

# OpenCode server default port
EXPOSE 4096

# Start OpenCode server
# --hostname 0.0.0.0 allows connections from other containers
CMD ["opencode", "serve", "--port", "4096", "--hostname", "0.0.0.0"]

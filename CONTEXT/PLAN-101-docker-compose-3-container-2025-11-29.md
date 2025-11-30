# Docker Compose 3-Container Architecture Plan (GH-101)

## Overview

This plan supersedes `PLAN-docker-compose-support-2025-11-03.md` (2-container architecture) with a more secure 3-container setup that isolates LLM provider credentials in a dedicated proxy container.

### Decision Context
- **Date**: 2025-11-29
- **Issue**: [GitHub Issue #101](https://github.com/sst/opencode-web/issues/101)
- **Architecture Change**: Moving from 2-container (frontend + backend) to 3-container (frontend + backend + LLM proxy) for improved security isolation

### Key Security Improvement
The primary motivation for the 3-container architecture is **credential isolation**: API keys for LLM providers (Anthropic, OpenAI, etc.) exist ONLY in the LLM proxy container. The OpenCode backend connects to the proxy as an unauthenticated local endpoint.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              HOST MACHINE                                        │
│                                                                                  │
│  ┌────────────────┐    ┌──────────────────────────────────────────────────────┐ │
│  │   Browser      │    │           Docker Network (opencode_internal)         │ │
│  │                │    │                                                      │ │
│  │  localhost:3000├────┤──▶ ┌──────────────────┐                              │ │
│  │                │    │    │   FRONTEND       │                              │ │
│  └────────────────┘    │    │   (oc-web)       │                              │ │
│                        │    │                  │                              │ │
│                        │    │  Port: 3000      │                              │ │
│                        │    │  Bun + Vite      │                              │ │
│                        │    │                  │                              │ │
│                        │    └────────┬─────────┘                              │ │
│                        │             │                                        │ │
│                        │             │ OPENCODE_SERVER_URL=http://backend:4096│ │
│                        │             ▼                                        │ │
│                        │    ┌──────────────────┐                              │ │
│  ┌─────────────────┐   │    │   BACKEND        │                              │ │
│  │ Volume Mounts   │   │    │   (opencode)     │                              │ │
│  │                 │   │    │                  │                              │ │
│  │ ~/.gitconfig ───┼───┼───▶│  Port: 4096      │                              │ │
│  │ ~/.ssh (ro) ────┼───┼───▶│  Alpine + Bun    │                              │ │
│  │ ~/.config/      │   │    │  + OpenCode      │                              │ │
│  │   opencode ─────┼───┼───▶│                  │                              │ │
│  │ ~/workspace ────┼───┼───▶│  /workspace      │                              │ │
│  │                 │   │    │                  │                              │ │
│  └─────────────────┘   │    └────────┬─────────┘                              │ │
│                        │             │                                        │ │
│                        │             │ Provider API: http://llm-proxy:4000/v1 │ │
│                        │             │ (NO API KEY REQUIRED)                  │ │
│                        │             ▼                                        │ │
│                        │    ┌──────────────────┐      ┌─────────────────────┐ │ │
│                        │    │   LLM-PROXY      │      │ External LLM APIs   │ │ │
│                        │    │   (LiteLLM)      │──────│                     │ │ │
│                        │    │                  │      │ - api.anthropic.com │ │ │
│       NOT EXPOSED      │    │  Port: 4000      │      │ - api.openai.com    │ │ │
│       TO HOST          │    │  (internal only) │      │ - etc.              │ │ │
│                        │    │                  │      └─────────────────────┘ │ │
│                        │    │  ANTHROPIC_API_KEY                              │ │
│                        │    │  OPENAI_API_KEY                                 │ │
│                        │    │  (credentials here ONLY)                        │ │
│                        │    └──────────────────┘                              │ │
│                        │                                                      │ │
│                        └──────────────────────────────────────────────────────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Network Security Model

| Container | Exposed to Host | Network Access |
|-----------|-----------------|----------------|
| `frontend` | Yes (port 3000) | Can reach `backend` |
| `backend` | No (optional debug) | Can reach `llm-proxy`, external for tools |
| `llm-proxy` | **No** | Can reach external LLM APIs only |

---

## File Structure

```
docker/
├── backend/
│   └── Dockerfile              # Alpine + Bun + OpenCode server
├── frontend/
│   └── Dockerfile              # Bun + oc-web build & serve
├── llm-proxy/
│   ├── Dockerfile              # LiteLLM proxy
│   └── config.yaml             # LiteLLM configuration template
├── docker-compose.yml          # Production configuration
├── docker-compose.dev.yml      # Development overrides (hot reload)
├── .env.example                # Environment variable template
└── README.md                   # Usage documentation
```

---

## Technical Specifications

### Container 1: Backend (OpenCode Server)

**Base Image**: Alpine Linux 3.19 (minimal footprint)

**Purpose**: Run the OpenCode server that provides the REST API and manages AI coding sessions.

**Key Configuration**:
- Connects to LLM proxy at `http://llm-proxy:4000/v1` (no auth required)
- OpenCode configured with a custom provider pointing to the proxy
- Mounts workspace, git config, and SSH keys from host

**Environment Variables**:
| Variable | Description | Default |
|----------|-------------|---------|
| `OPENCODE_API` | LLM API endpoint (internal proxy) | `http://llm-proxy:4000/v1` |
| `WORKSPACE_PATH` | Container path for mounted workspace | `/workspace` |

**Volume Mounts**:
| Host Path | Container Path | Mode | Purpose |
|-----------|----------------|------|---------|
| `${WORKSPACE_PATH:-~/workspace}` | `/workspace` | rw | Repository storage |
| `~/.gitconfig` | `/root/.gitconfig` | ro | Git configuration |
| `~/.ssh` | `/root/.ssh` | ro | SSH keys for private repos |
| `~/.config/opencode` | `/root/.config/opencode` | rw | OpenCode config & sessions |

### Container 2: Frontend (oc-web)

**Base Image**: `oven/bun:1.3-alpine`

**Purpose**: Build and serve the oc-web React application.

**Key Configuration**:
- Multi-stage build: deps → build → runtime
- Connects to backend at `http://backend:4096`
- Only container exposed to host network

**Environment Variables**:
| Variable | Description | Default |
|----------|-------------|---------|
| `OPENCODE_SERVER_URL` | Backend server URL (internal) | `http://backend:4096` |
| `OPENCODE_WEB_HOST` | Listen address | `0.0.0.0` |
| `OPENCODE_WEB_PORT` | Listen port | `3000` |
| `NODE_ENV` | Environment mode | `production` |

### Container 3: LLM Proxy (LiteLLM)

**Base Image**: `ghcr.io/berriai/litellm:main-latest`

**Purpose**: Proxy LLM API calls, handling authentication with providers so backend doesn't need credentials.

**Key Configuration**:
- NOT exposed to host network
- Contains ALL provider API keys
- Provides OpenAI-compatible API at port 4000

**Environment Variables** (stored in `.env`, NOT committed):
| Variable | Description | Required |
|----------|-------------|----------|
| `ANTHROPIC_API_KEY` | Anthropic API key | If using Claude |
| `OPENAI_API_KEY` | OpenAI API key | If using GPT |
| `GEMINI_API_KEY` | Google AI API key | If using Gemini |
| `LITELLM_MASTER_KEY` | Internal auth key | Yes |

---

## Detailed Implementation Tasks

### Phase 1: Directory Structure and Base Files

- [ ] **1.1** Create docker directory structure
  ```bash
  mkdir -p docker/{backend,frontend,llm-proxy}
  ```

- [ ] **1.2** Create `.dockerignore` file
  ```
  # docker/.dockerignore
  node_modules/
  dist/
  .git/
  CONTEXT/
  *.log
  .env
  .env.local
  ```

- [ ] **1.3** Create `.env.example` with all required variables

### Phase 2: LLM Proxy Container

- [ ] **2.1** Create `docker/llm-proxy/Dockerfile`
  ```dockerfile
  # docker/llm-proxy/Dockerfile
  FROM ghcr.io/berriai/litellm:main-latest

  # Copy configuration
  COPY config.yaml /app/config.yaml

  # Health check
  HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:4000/health || exit 1

  EXPOSE 4000

  CMD ["--config", "/app/config.yaml", "--port", "4000", "--host", "0.0.0.0"]
  ```

- [ ] **2.2** Create `docker/llm-proxy/config.yaml`
  ```yaml
  # docker/llm-proxy/config.yaml
  # LiteLLM Proxy Configuration for OpenCode
  # 
  # This proxy handles authentication with LLM providers so the OpenCode
  # backend can make requests without storing API keys.

  model_list:
    # Anthropic Claude models
    - model_name: claude-sonnet-4-20250514
      litellm_params:
        model: anthropic/claude-sonnet-4-20250514
        api_key: os.environ/ANTHROPIC_API_KEY
    
    - model_name: claude-3-5-sonnet-20241022
      litellm_params:
        model: anthropic/claude-3-5-sonnet-20241022
        api_key: os.environ/ANTHROPIC_API_KEY
    
    - model_name: claude-3-5-haiku-20241022
      litellm_params:
        model: anthropic/claude-3-5-haiku-20241022
        api_key: os.environ/ANTHROPIC_API_KEY

    # OpenAI models
    - model_name: gpt-4o
      litellm_params:
        model: openai/gpt-4o
        api_key: os.environ/OPENAI_API_KEY
    
    - model_name: gpt-4o-mini
      litellm_params:
        model: openai/gpt-4o-mini
        api_key: os.environ/OPENAI_API_KEY
    
    - model_name: o1
      litellm_params:
        model: openai/o1
        api_key: os.environ/OPENAI_API_KEY
    
    - model_name: o1-mini
      litellm_params:
        model: openai/o1-mini
        api_key: os.environ/OPENAI_API_KEY

    # Google Gemini models
    - model_name: gemini-2.0-flash
      litellm_params:
        model: gemini/gemini-2.0-flash
        api_key: os.environ/GEMINI_API_KEY
    
    - model_name: gemini-1.5-pro
      litellm_params:
        model: gemini/gemini-1.5-pro
        api_key: os.environ/GEMINI_API_KEY

  litellm_settings:
    drop_params: true
    set_verbose: false

  general_settings:
    master_key: os.environ/LITELLM_MASTER_KEY
  ```

- [ ] **2.3** Validate LiteLLM config with health endpoint

### Phase 3: Backend Container

- [ ] **3.1** Create `docker/backend/Dockerfile`
  ```dockerfile
  # docker/backend/Dockerfile
  # OpenCode Backend Server
  #
  # Runs the OpenCode server connected to LLM proxy for AI capabilities.
  # Mounts workspace and git config from host.

  FROM alpine:3.19

  # Install system dependencies
  RUN apk add --no-cache \
      curl \
      git \
      openssh-client \
      bash \
      ca-certificates \
      unzip

  # Install Bun
  ENV BUN_INSTALL=/usr/local
  RUN curl -fsSL https://bun.sh/install | bash

  # Install OpenCode globally
  RUN bun install -g opencode

  # Create workspace directory
  RUN mkdir -p /workspace /root/.config/opencode

  # Set working directory
  WORKDIR /workspace

  # Health check - verify OpenCode server is responding
  HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:4096/config || exit 1

  EXPOSE 4096

  # Start OpenCode server in headless mode
  # The server listens on port 4096 by default
  CMD ["opencode", "serve", "--port", "4096", "--hostname", "0.0.0.0"]
  ```

- [ ] **3.2** Create OpenCode configuration for proxy integration
  
  The backend needs an `opencode.json` config that points to the LLM proxy. This will be mounted or generated:
  
  ```json
  {
    "$schema": "https://opencode.ai/config.json",
    "provider": {
      "litellm-proxy": {
        "api": "http://llm-proxy:4000/v1",
        "models": {
          "claude-sonnet-4-20250514": {
            "name": "Claude Sonnet 4",
            "limit": { "context": 200000, "output": 8192 }
          },
          "claude-3-5-sonnet-20241022": {
            "name": "Claude 3.5 Sonnet",
            "limit": { "context": 200000, "output": 8192 }
          },
          "gpt-4o": {
            "name": "GPT-4o",
            "limit": { "context": 128000, "output": 16384 }
          }
        },
        "options": {
          "timeout": 300000
        }
      }
    },
    "model": "litellm-proxy/claude-sonnet-4-20250514"
  }
  ```

- [ ] **3.3** Test backend container can reach LLM proxy

### Phase 4: Frontend Container

- [ ] **4.1** Create `docker/frontend/Dockerfile`
  ```dockerfile
  # docker/frontend/Dockerfile
  # OC-Web Frontend
  #
  # Multi-stage build for the oc-web React application.
  # Connects to OpenCode backend for API/SSE communication.

  # ====================
  # Stage 1: Dependencies
  # ====================
  FROM oven/bun:1.3-alpine AS deps

  WORKDIR /app

  # Copy package files
  COPY package.json bun.lock* ./

  # Install dependencies
  RUN bun install --frozen-lockfile

  # ====================
  # Stage 2: Builder
  # ====================
  FROM oven/bun:1.3-alpine AS builder

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
  FROM oven/bun:1.3-alpine AS runtime

  WORKDIR /app

  # Copy built assets and server
  COPY --from=builder /app/dist ./dist
  COPY --from=builder /app/node_modules ./node_modules
  COPY --from=builder /app/package.json ./
  COPY --from=builder /app/server.ts ./
  COPY --from=builder /app/src/lib/opencode-config.ts ./src/lib/
  COPY --from=builder /app/packages/opencode-web/sse-proxy.ts ./packages/opencode-web/

  # Create non-root user for security
  RUN addgroup -g 1001 -S nodejs && \
      adduser -S nextjs -u 1001

  # Set runtime environment variables
  ENV NODE_ENV=production
  ENV OPENCODE_WEB_HOST=0.0.0.0
  ENV OPENCODE_WEB_PORT=3000

  # Health check
  HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/ || exit 1

  EXPOSE 3000

  # Run as non-root user
  USER nextjs

  CMD ["bun", "server.ts"]
  ```

- [ ] **4.2** Verify frontend can proxy to backend container

### Phase 5: Docker Compose Configuration

- [ ] **5.1** Create `docker/docker-compose.yml` (production)
  ```yaml
  # docker/docker-compose.yml
  # Production Docker Compose for oc-web 3-container architecture
  #
  # Usage:
  #   cp .env.example .env
  #   # Edit .env with your API keys
  #   docker compose up -d
  #
  # Architecture:
  #   - frontend: oc-web UI (exposed on port 3000)
  #   - backend: OpenCode server (internal)
  #   - llm-proxy: LiteLLM proxy for LLM API calls (internal, holds credentials)

  name: opencode-web

  services:
    # ===================
    # LLM Proxy Container
    # ===================
    # Handles authentication with LLM providers
    # NOT exposed to host - only accessible within Docker network
    llm-proxy:
      build:
        context: ./llm-proxy
        dockerfile: Dockerfile
      container_name: opencode-llm-proxy
      restart: unless-stopped
      environment:
        # Provider API keys - set in .env file
        - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY:-}
        - OPENAI_API_KEY=${OPENAI_API_KEY:-}
        - GEMINI_API_KEY=${GEMINI_API_KEY:-}
        - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY:-sk-opencode-internal}
      networks:
        - opencode_internal
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
        interval: 30s
        timeout: 10s
        retries: 3
        start_period: 10s
      # NOT exposed to host - security isolation

    # ===================
    # Backend Container
    # ===================
    # OpenCode server - connects to LLM proxy without credentials
    backend:
      build:
        context: ./backend
        dockerfile: Dockerfile
      container_name: opencode-backend
      restart: unless-stopped
      depends_on:
        llm-proxy:
          condition: service_healthy
      environment:
        # Point to LLM proxy (no API key needed)
        - OPENCODE_API=http://llm-proxy:4000/v1
      volumes:
        # Workspace for repositories
        - ${WORKSPACE_PATH:-~/workspace}:/workspace:rw
        # Git configuration (read-only)
        - ${HOME}/.gitconfig:/root/.gitconfig:ro
        # SSH keys for private repo access (read-only)
        - ${HOME}/.ssh:/root/.ssh:ro
        # OpenCode configuration and sessions
        - ${HOME}/.config/opencode:/root/.config/opencode:rw
      networks:
        - opencode_internal
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:4096/config"]
        interval: 30s
        timeout: 10s
        retries: 3
        start_period: 15s
      # NOT exposed to host by default (add ports for debugging if needed)

    # ===================
    # Frontend Container
    # ===================
    # oc-web UI - only container exposed to host
    frontend:
      build:
        context: ..
        dockerfile: docker/frontend/Dockerfile
      container_name: opencode-frontend
      restart: unless-stopped
      depends_on:
        backend:
          condition: service_healthy
      environment:
        - OPENCODE_SERVER_URL=http://backend:4096
        - OPENCODE_WEB_HOST=0.0.0.0
        - OPENCODE_WEB_PORT=3000
        - NODE_ENV=production
      ports:
        # Only exposed port - web UI
        - "${FRONTEND_PORT:-3000}:3000"
      networks:
        - opencode_internal
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:3000/"]
        interval: 30s
        timeout: 10s
        retries: 3
        start_period: 10s

  networks:
    opencode_internal:
      driver: bridge
      # Internal network - containers can communicate via service names
  ```

- [ ] **5.2** Create `docker/docker-compose.dev.yml` (development overrides)
  ```yaml
  # docker/docker-compose.dev.yml
  # Development overrides for hot reload and debugging
  #
  # Usage:
  #   docker compose -f docker-compose.yml -f docker-compose.dev.yml up
  #
  # Changes from production:
  #   - Frontend runs in dev mode with hot reload
  #   - Source code mounted for live editing
  #   - Backend port exposed for debugging

  services:
    frontend:
      build:
        context: ..
        dockerfile: docker/frontend/Dockerfile
        target: deps  # Stop at deps stage for dev
      environment:
        - NODE_ENV=development
        - OPENCODE_SERVER_URL=http://backend:4096
        - OPENCODE_WEB_HOST=0.0.0.0
        - OPENCODE_WEB_PORT=3000
      volumes:
        # Mount source for hot reload
        - ..:/app:cached
        - /app/node_modules  # Exclude node_modules
      command: ["bun", "run", "dev", "--host", "0.0.0.0"]
      healthcheck:
        # Dev server may take longer to start
        start_period: 30s

    backend:
      # Optionally expose backend for debugging
      ports:
        - "${BACKEND_DEBUG_PORT:-4096}:4096"

    llm-proxy:
      environment:
        # Enable verbose logging in dev
        - LITELLM_LOG=DEBUG
  ```

- [ ] **5.3** Create `docker/.env.example`
  ```bash
  # docker/.env.example
  # OpenCode Web Docker Configuration
  # 
  # Copy this file to .env and fill in your values:
  #   cp .env.example .env
  #
  # IMPORTANT: Never commit .env to version control!

  # ====================
  # LLM Provider API Keys
  # ====================
  # Set the API keys for providers you want to use.
  # These are stored ONLY in the llm-proxy container.

  # Anthropic (Claude models)
  ANTHROPIC_API_KEY=

  # OpenAI (GPT models)
  OPENAI_API_KEY=

  # Google (Gemini models)
  GEMINI_API_KEY=

  # Internal key for LiteLLM proxy (generate a random string)
  LITELLM_MASTER_KEY=sk-opencode-internal-changeme

  # ====================
  # Volume Mounts
  # ====================

  # Path to your workspace/repositories on the host
  # This directory will be mounted into the backend container
  WORKSPACE_PATH=~/workspace

  # ====================
  # Port Configuration
  # ====================

  # Frontend port (exposed to host)
  FRONTEND_PORT=3000

  # Backend debug port (only in dev mode)
  BACKEND_DEBUG_PORT=4096

  # ====================
  # Optional Settings
  # ====================

  # Set to 'development' for dev mode
  # NODE_ENV=production
  ```

### Phase 6: Documentation

- [ ] **6.1** Create `docker/README.md`
  ```markdown
  # Docker Compose Setup for OpenCode Web

  This directory contains Docker configuration for running oc-web with 
  OpenCode in a containerized environment.

  ## Architecture

  The setup uses three containers:

  1. **Frontend** (oc-web) - Web UI, exposed on port 3000
  2. **Backend** (OpenCode) - AI coding server, internal only
  3. **LLM Proxy** (LiteLLM) - Handles LLM API authentication, internal only

  ### Security Model

  API keys for LLM providers are stored ONLY in the LLM proxy container.
  The backend connects to the proxy as an unauthenticated local endpoint,
  so credentials never touch the workspace or user configuration.

  ## Quick Start

  ### Prerequisites

  - Docker Engine 24+
  - Docker Compose v2
  - At least one LLM provider API key

  ### Setup

  1. Copy environment file:
     ```bash
     cd docker
     cp .env.example .env
     ```

  2. Edit `.env` with your API keys:
     ```bash
     ANTHROPIC_API_KEY=sk-ant-...
     # or
     OPENAI_API_KEY=sk-...
     ```

  3. Start the stack:
     ```bash
     docker compose up -d
     ```

  4. Open http://localhost:3000

  ## Development Mode

  For hot reload during development:

  ```bash
  docker compose -f docker-compose.yml -f docker-compose.dev.yml up
  ```

  ## Configuration

  ### Volume Mounts

  | Host Path | Container | Purpose |
  |-----------|-----------|---------|
  | `~/workspace` | `/workspace` | Your code repositories |
  | `~/.gitconfig` | `/root/.gitconfig` | Git configuration |
  | `~/.ssh` | `/root/.ssh` | SSH keys (read-only) |
  | `~/.config/opencode` | `/root/.config/opencode` | OpenCode settings |

  ### Ports

  | Port | Service | Exposed |
  |------|---------|---------|
  | 3000 | Frontend | Yes |
  | 4096 | Backend | No (dev: optional) |
  | 4000 | LLM Proxy | No |

  ## Troubleshooting

  ### View logs
  ```bash
  docker compose logs -f
  docker compose logs -f backend  # specific service
  ```

  ### Rebuild after changes
  ```bash
  docker compose build --no-cache
  docker compose up -d
  ```

  ### Check container health
  ```bash
  docker compose ps
  ```

  ### Reset everything
  ```bash
  docker compose down -v
  docker compose up -d --build
  ```
  ```

### Phase 7: Integration and Testing

- [ ] **7.1** Test LLM proxy health endpoint
  ```bash
  docker compose up llm-proxy -d
  curl http://localhost:4000/health  # Should fail (not exposed)
  docker compose exec llm-proxy curl http://localhost:4000/health  # Should work
  ```

- [ ] **7.2** Test backend can reach proxy
  ```bash
  docker compose up llm-proxy backend -d
  docker compose exec backend curl http://llm-proxy:4000/health
  ```

- [ ] **7.3** Test frontend can reach backend
  ```bash
  docker compose up -d
  docker compose exec frontend curl http://backend:4096/config
  ```

- [ ] **7.4** End-to-end test
  ```bash
  docker compose up -d
  curl http://localhost:3000  # Should return oc-web UI
  # Open browser, create session, verify LLM responses work
  ```

- [ ] **7.5** Test volume mounts
  ```bash
  # Create test file on host
  touch ~/workspace/test-file.txt
  # Verify visible in container
  docker compose exec backend ls /workspace/test-file.txt
  ```

- [ ] **7.6** Test dev mode hot reload
  ```bash
  docker compose -f docker-compose.yml -f docker-compose.dev.yml up frontend -d
  # Edit src/app/index.tsx
  # Verify changes reflect in browser without restart
  ```

### Phase 8: Quality Assurance

- [ ] **8.1** Run lint inside frontend container
  ```bash
  docker compose exec frontend bun run lint
  ```

- [ ] **8.2** Run type check inside frontend container
  ```bash
  docker compose exec frontend bun x tsc --noEmit
  ```

- [ ] **8.3** Verify no credentials leak to backend
  ```bash
  docker compose exec backend env | grep -i api_key  # Should be empty
  docker compose exec backend env | grep -i anthropic  # Should be empty
  ```

- [ ] **8.4** Security audit
  - [ ] Verify llm-proxy port not accessible from host
  - [ ] Verify .env is in .gitignore
  - [ ] Verify sensitive volume mounts are read-only where appropriate

---

## Code References

### Internal Files (oc-web)

| File | Relevance |
|------|-----------|
| `package.json` | Build scripts: `bun run build`, `bun run dev` |
| `server.ts` | Production server entry point, SSE proxy |
| `src/lib/opencode-config.ts` | Environment variable handling for `OPENCODE_SERVER_URL` |
| `src/lib/opencode-http-api.ts` | All backend API endpoints that must be reachable |
| `packages/opencode-web/sse-proxy.ts` | SSE proxy implementation |
| `vite.config.ts` | Vite build configuration |
| `.env.example` | Existing env var documentation |

### External References

| Resource | URL | Usage |
|----------|-----|-------|
| OpenCode Repository | https://github.com/sst/opencode | Backend server source |
| OpenCode Docs | https://opencode.ai/docs | Server configuration |
| LiteLLM Docker | https://docs.litellm.ai/docs/proxy/docker_quick_start | Proxy setup |
| LiteLLM Config | https://docs.litellm.ai/docs/proxy/configs | Config.yaml reference |
| Bun Docker | https://hub.docker.com/r/oven/bun | Base image for frontend |
| Alpine Linux | https://hub.docker.com/_/alpine | Base image for backend |

---

## Environment Variable Reference

### Frontend Container

| Variable | Source | Default | Description |
|----------|--------|---------|-------------|
| `OPENCODE_SERVER_URL` | `opencode-config.ts` | `http://backend:4096` | Backend server URL |
| `OPENCODE_WEB_HOST` | `opencode-config.ts` | `0.0.0.0` | Listen host |
| `OPENCODE_WEB_PORT` | `opencode-config.ts` | `3000` | Listen port |
| `NODE_ENV` | Bun runtime | `production` | Environment mode |

### Backend Container

| Variable | Source | Default | Description |
|----------|--------|---------|-------------|
| `OPENCODE_API` | OpenCode | `http://llm-proxy:4000/v1` | LLM API endpoint |

### LLM Proxy Container

| Variable | Source | Required | Description |
|----------|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | LiteLLM | If using Claude | Anthropic API key |
| `OPENAI_API_KEY` | LiteLLM | If using GPT | OpenAI API key |
| `GEMINI_API_KEY` | LiteLLM | If using Gemini | Google AI API key |
| `LITELLM_MASTER_KEY` | LiteLLM | Yes | Internal auth key |

---

## Validation Checklist

### Functional Requirements

- [ ] Frontend accessible at http://localhost:3000
- [ ] Can create new session in UI
- [ ] LLM responses stream correctly (SSE)
- [ ] File operations work in mounted workspace
- [ ] Git operations work (using mounted .gitconfig and .ssh)
- [ ] OpenCode config persists across restarts

### Security Requirements

- [ ] LLM proxy port (4000) NOT accessible from host
- [ ] Backend port (4096) NOT accessible from host (except dev mode)
- [ ] API keys only exist in llm-proxy container
- [ ] `.env` file is in `.gitignore`
- [ ] SSH mount is read-only

### Performance Requirements

- [ ] Frontend cold start < 10 seconds
- [ ] Backend cold start < 15 seconds
- [ ] SSE connections stable for extended sessions
- [ ] Hot reload works in dev mode (< 2 second refresh)

### Documentation Requirements

- [ ] README covers quick start
- [ ] All environment variables documented
- [ ] Troubleshooting section exists
- [ ] Security model explained

---

## Milestone Summary

| Phase | Description | Dependencies | Estimated Effort |
|-------|-------------|--------------|------------------|
| 1 | Directory structure | None | 30 min |
| 2 | LLM Proxy container | Phase 1 | 2 hours |
| 3 | Backend container | Phase 2 | 3 hours |
| 4 | Frontend container | Phase 3 | 2 hours |
| 5 | Docker Compose | Phase 2-4 | 2 hours |
| 6 | Documentation | Phase 5 | 1 hour |
| 7 | Integration testing | Phase 5 | 2 hours |
| 8 | QA and security audit | Phase 7 | 1 hour |

**Total Estimated Effort**: ~13.5 hours

---

## Open Questions / Future Considerations

1. **GPU Support**: Should we add NVIDIA container toolkit support for local LLM inference?

2. **Multi-workspace**: Support for multiple workspace mounts with project switching?

3. **Database for LiteLLM**: Add PostgreSQL container for spend tracking and key management?

4. **TLS/HTTPS**: Should we include Caddy/nginx for HTTPS termination?

5. **Windows/macOS Volume Performance**: May need to document volume caching options (`cached`, `delegated`) for better performance.

6. **Pre-built Images**: Should we publish pre-built images to GitHub Container Registry for faster startup?

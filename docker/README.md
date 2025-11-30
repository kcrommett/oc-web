# Docker Compose Setup for OpenCode Web

This directory contains Docker configuration for running oc-web with OpenCode in a containerized environment with secure credential isolation.

## Architecture

The setup uses three containers for security isolation:

```
┌─────────────────────────────────────────────────────────────────┐
│                         HOST MACHINE                            │
│                                                                 │
│   Browser ──────► Frontend (port 3000)                          │
│                        │                                        │
│                        ▼                                        │
│   ┌─────────────── Docker Network ───────────────────────┐      │
│   │                                                      │      │
│   │   Frontend ──────► Backend ──────► LLM Proxy ──────► │ ──► LLM APIs
│   │   (oc-web)        (OpenCode)      (LiteLLM)         │      │
│   │   Port 3000       Port 4096       Port 4000         │      │
│   │   EXPOSED         internal        internal          │      │
│   │                                                      │      │
│   └──────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Container | Purpose | Exposed |
|-----------|---------|---------|
| **Frontend** (oc-web) | Web UI | Yes (port 3000) |
| **Backend** (OpenCode) | AI coding server | No |
| **LLM Proxy** (LiteLLM) | LLM API authentication | No |

### Security Model

**API keys for LLM providers are stored ONLY in the LLM proxy container.**

The backend connects to the proxy as an unauthenticated local endpoint, so credentials never touch the workspace, user configuration, or any exposed service.

## Quick Start

### Prerequisites

- Docker Engine 24+
- Docker Compose v2
- At least one LLM provider API key (Anthropic, OpenAI, or Google)

### Setup

1. **Navigate to the docker directory:**
   ```bash
   cd docker
   ```

2. **Copy the environment template:**
   ```bash
   cp .env.example .env
   ```

3. **Edit `.env` with your API keys:**
   ```bash
   # Required: At least one of these
   ANTHROPIC_API_KEY=sk-ant-...
   OPENAI_API_KEY=sk-...
   GEMINI_API_KEY=...

   # Recommended: Change the internal key
   LITELLM_MASTER_KEY=your-random-string-here
   ```

4. **Start the stack:**
   ```bash
   docker compose up -d
   ```

5. **Open http://localhost:3000** in your browser

### Verify Health

```bash
# Check all containers are healthy
docker compose ps

# View logs
docker compose logs -f

# Check specific service
docker compose logs -f backend
```

## Development Mode

For development with hot reload:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

This enables:
- Frontend hot reload (edit files and see changes immediately)
- Backend port exposed for direct API debugging
- Verbose logging in LLM proxy

## Configuration

### Volume Mounts

| Host Path | Container Path | Mode | Purpose |
|-----------|----------------|------|---------|
| `~/workspace` | `/workspace` | rw | Your code repositories |
| `~/.gitconfig` | `/root/.gitconfig` | ro | Git configuration |
| `~/.ssh` | `/root/.ssh` | ro | SSH keys for private repos |
| `~/.config/opencode` | `/root/.config/opencode` | rw | OpenCode settings & sessions |

### Environment Variables

Set these in your `.env` file:

| Variable | Description | Required |
|----------|-------------|----------|
| `ANTHROPIC_API_KEY` | Anthropic API key | If using Claude |
| `OPENAI_API_KEY` | OpenAI API key | If using GPT |
| `GEMINI_API_KEY` | Google AI API key | If using Gemini |
| `LITELLM_MASTER_KEY` | Internal proxy auth key | Recommended |
| `WORKSPACE_PATH` | Host workspace path | Optional (default: ~/workspace) |
| `FRONTEND_PORT` | Host port for UI | Optional (default: 3000) |
| `BACKEND_DEBUG_PORT` | Backend debug port (dev only) | Optional (default: 4096) |

### Ports

| Port | Service | Exposed to Host |
|------|---------|-----------------|
| 3000 | Frontend | Yes |
| 4096 | Backend | No (dev: optional) |
| 4000 | LLM Proxy | **Never** |

## Commands Reference

```bash
# Start all containers
docker compose up -d

# Stop all containers
docker compose down

# View logs (all services)
docker compose logs -f

# View logs (specific service)
docker compose logs -f backend

# Rebuild after code changes
docker compose build --no-cache
docker compose up -d

# Check container health
docker compose ps

# Execute command in container
docker compose exec backend sh
docker compose exec frontend sh

# Reset everything (including volumes)
docker compose down -v
docker compose up -d --build
```

## Troubleshooting

### Container won't start

1. Check logs:
   ```bash
   docker compose logs llm-proxy
   docker compose logs backend
   docker compose logs frontend
   ```

2. Verify health checks:
   ```bash
   docker compose ps
   ```

3. Test internal connectivity:
   ```bash
   # From backend, test LLM proxy
   docker compose exec backend curl http://llm-proxy:4000/health

   # From frontend, test backend
   docker compose exec frontend curl http://backend:4096/config
   ```

### LLM requests failing

1. Verify API keys are set:
   ```bash
   docker compose exec llm-proxy env | grep API_KEY
   ```

2. Check LiteLLM logs:
   ```bash
   docker compose logs llm-proxy
   ```

3. Test proxy directly (from backend container):
   ```bash
   docker compose exec backend curl http://llm-proxy:4000/v1/models
   ```

### Volume mount issues

1. Ensure host directories exist:
   ```bash
   mkdir -p ~/workspace ~/.config/opencode
   ```

2. Check permissions:
   ```bash
   ls -la ~/.gitconfig ~/.ssh ~/.config/opencode
   ```

### Slow startup on macOS/Windows

Docker Desktop volume mounts can be slow. The `:cached` flag is already set for development. For better performance:

1. Use Docker Desktop's file sharing settings
2. Consider using named volumes for node_modules
3. For intensive workloads, consider WSL2 on Windows or native Linux

## Security Considerations

1. **Never commit `.env`** - It contains your API keys
2. **SSH keys are read-only** - Container cannot modify your keys
3. **LLM proxy is isolated** - Cannot be accessed from host
4. **Use strong `LITELLM_MASTER_KEY`** - Prevents unauthorized proxy access

## Customization

### Adding More LLM Models

Edit `llm-proxy/config.yaml` to add models:

```yaml
model_list:
  - model_name: my-custom-model
    litellm_params:
      model: provider/model-name
      api_key: os.environ/MY_API_KEY
```

### Using a Different Workspace

Set `WORKSPACE_PATH` in `.env`:

```bash
WORKSPACE_PATH=/path/to/your/projects
```

### Changing Ports

```bash
FRONTEND_PORT=8080  # Access at http://localhost:8080
```

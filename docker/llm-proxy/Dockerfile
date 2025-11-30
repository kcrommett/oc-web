# docker/llm-proxy/Dockerfile
# LiteLLM Proxy Container
#
# Handles authentication with LLM providers so the OpenCode backend
# can make requests without storing API keys.
#
# Security: This container holds all provider credentials.
# It is NOT exposed to the host network - only accessible within
# the Docker internal network.

FROM ghcr.io/berriai/litellm:main-latest

# Copy LiteLLM configuration
COPY config.yaml /app/config.yaml

# Health check - verify LiteLLM is responding
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:4000/health || exit 1

# LiteLLM default port
EXPOSE 4000

# Start LiteLLM proxy with config
CMD ["--config", "/app/config.yaml", "--port", "4000", "--host", "0.0.0.0"]

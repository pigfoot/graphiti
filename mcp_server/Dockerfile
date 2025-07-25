# syntax=docker/dockerfile:1
###############################
# Builder stage
###############################
FROM docker.io/python:3.13-slim AS builder

# Ensure apt-get doesn't open a menu on you.
ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies first
RUN <<EOT
apt update -qy
DEBIAN_FRONTEND=noninteractive
apt install -qyy \
    --no-install-recommends --no-install-suggests \
    tini curl \
    ca-certificates
rm -rf /var/lib/apt/lists/*
EOT

# Install uv using official distroless image (best practice)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Configure uv environment variables
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never

# Set working directory
WORKDIR /app

RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev --no-editable

# Then, add the rest of the project source code and install it
COPY graphiti_mcp_server.py ./

# Installing separately from its dependencies allows optimal layer caching
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-dev --no-editable

###############################
# Runtime stage
###############################
FROM docker.io/python:3.13-slim

# Create non-root user and set up directory
RUN <<EOT
groupadd -r app
useradd -r -d /app -g app -N app
EOT

# Copy tini and the application from the builder
COPY --from=builder /usr/bin/tini-static /usr/bin/tini
COPY --from=builder --chown=app:app /app /app

# Setup virtual environment and place executables in the environment at the front of the path
ENV PYTHONUNBUFFERED=1 \
    VIRTUAL_ENV=/app/.venv \
    MCP_SERVER_HOST="0.0.0.0" \
    PATH="/app/.venv/bin:$PATH"

# Switch to non-root user early & Set working directory
USER app
WORKDIR /app

ENTRYPOINT ["tini", "--"]
CMD ["python", "graphiti_mcp_server.py"]

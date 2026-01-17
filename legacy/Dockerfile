# Enhanced production Dockerfile with Python/uvx support for MCP servers
FROM node:24.13.0-alpine

# Install system dependencies for Python servers
RUN apk update && apk add --no-cache \
    python3 \
    py3-pip \
    git \
    curl \
    bash

# Install uv for Python MCP servers (uv includes uvx functionality)
RUN pip3 install --break-system-packages uv

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S mcp-hub -u 1001 -G nodejs

# Set working directory
WORKDIR /app

# Copy the built CLI file (pre-built locally)
COPY --chown=mcp-hub:nodejs dist/cli.js ./cli.js

# Copy source files needed for REST API server
COPY --chown=mcp-hub:nodejs src/ ./src/

# Copy package.json and package-lock.json for dependencies
COPY --chown=mcp-hub:nodejs package.json ./
COPY --chown=mcp-hub:nodejs package-lock.json ./

# Install all dependencies needed for both CLI and REST API server
RUN npm ci

# Create directories for configurations and data
RUN mkdir -p /app/config /app/config/credentials /app/data /app/data/google-workspace /app/logs && \
    chown -R mcp-hub:nodejs /app/config /app/data /app/logs && \
    chmod 755 /app/logs

# Copy the MCP servers configuration
COPY --chown=mcp-hub:nodejs mcp-servers.json /app/config/mcp-servers.json

# Copy scripts directory
RUN mkdir -p /app/scripts
COPY --chown=mcp-hub:nodejs docker/scripts/startup.sh /app/scripts/startup.sh
COPY --chown=mcp-hub:nodejs docker/scripts/google-workspace-auth.sh /app/scripts/google-workspace-auth.sh
COPY --chown=mcp-hub:nodejs docker/scripts/wix-auth.sh /app/scripts/wix-auth.sh
RUN chmod +x /app/scripts/*.sh

# Switch to non-root user and set up Python paths
USER mcp-hub

# Create log files with proper permissions
RUN touch /app/logs/wix-auth.log /app/logs/google-workspace-auth.log || true

# Ensure uv is in PATH for the mcp-hub user
ENV PATH="/home/mcp-hub/.local/bin:$PATH"

# Create local bin directory and uvx alias
RUN mkdir -p /home/mcp-hub/.local/bin && \
    echo '#!/bin/bash\nexec uv tool run "$@"' > /home/mcp-hub/.local/bin/uvx && \
    chmod +x /home/mcp-hub/.local/bin/uvx

# Pre-install Python MCP servers to speed up container startup
RUN uv tool install mcp-server-fetch && \
    uv tool install mcp-server-git && \
    uv tool install mcp-server-time && \
    uv tool install mcp-server-sqlite

# Install AWS MCP servers individually (some may fail due to dependencies)
RUN uv tool install awslabs.aws-knowledge-mcp-server || echo "aws-knowledge-mcp-server failed to install"
RUN uv tool install awslabs.dynamodb-mcp-server || echo "dynamodb-mcp-server failed to install"
RUN uv tool install awslabs.lambda-tool-mcp-server || echo "lambda-tool-mcp-server failed to install"
RUN uv tool install awslabs.cdk-mcp-server || echo "cdk-mcp-server failed to install"
# Skip aws-api-mcp-server due to PyTorch dependency issues on Alpine
# RUN uv tool install awslabs.aws-api-mcp-server || echo "aws-api-mcp-server failed to install"

# Install Google Workspace MCP server
RUN uv tool install workspace-mcp || echo "workspace-mcp failed to install"

# Expose ports (37373 for MCP protocol, 8001 for REST API)
EXPOSE 37373 8001

# Enhanced health check that tests both MCP and REST API endpoints
HEALTHCHECK --interval=30s --timeout=15s --start-period=45s --retries=3 \
    CMD node -e "const http = require('http'); \
    const checkHealth = (port, path = '/api/health') => new Promise((resolve, reject) => { \
        const req = http.get({ host: 'localhost', port, path, timeout: 5000 }, (res) => resolve(res.statusCode === 200)); \
        req.on('error', reject); req.setTimeout(5000, reject); \
    }); \
    Promise.all([checkHealth(8001), checkHealth(process.env.PORT || 37373)]) \
        .then(() => process.exit(0)) \
        .catch(() => process.exit(1));" || exit 1

# Set default environment variables
ENV NODE_ENV=production
ENV PORT=37373

# Default command - use enhanced startup script
CMD ["/app/scripts/startup.sh"]

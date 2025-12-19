# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is an n8n automation platform deployment using Docker Compose with **Task Runners in External Mode**. The stack consists of n8n Main (web UI, webhook handler, and task broker) and 4 specialized Task Runner containers: 2 for JavaScript and 2 for Python execution in isolated sandboxes.

## Architecture

### Task Runners Design
- **n8n Main**: Handles web UI, webhooks, and operates the task broker on port 5679
- **JavaScript Runners (2x)**: Specialized sidecar containers for JavaScript code execution
- **Python Runners (2x)**: Specialized sidecar containers for Python code execution
- **Image**: All runners use `n8nio/runners` image with different launchers
- **Authentication**: Task runners authenticate with Main using `N8N_RUNNERS_AUTH_TOKEN`
- **Shared Storage**: Main and all runners share `n8n_storage` volume for workflow data
- **Concurrency**: Each runner handles up to 5 concurrent tasks, with 15s auto-shutdown timeout

### External Dependencies
- **PostgreSQL**: Runs in a separate docker-compose file (postgres.yaml) on `backend_net`
- **Traefik**: Reverse proxy on `proxy_net` (external) providing SSL/TLS termination
- Services auto-restart until PostgreSQL connection is established

### Network Architecture
- `backend_net`: Internal network for PostgreSQL, n8n Main, and task runners
- `proxy_net`: External network for Traefik reverse proxy

### Task Runner Features
- **Sandboxed Execution**: Code runs in isolated environments for security
- **JavaScript Support**: Primary runtime for n8n code execution
- **Health Checks**: Each runner exposes `/healthz` endpoint on port 5681
- **Dynamic Scaling**: Can add/remove runners without downtime

## Common Commands

### Starting Services
```bash
# Create required networks (one-time setup)
docker network create backend_net
docker network create proxy_net

# Start PostgreSQL (must be running first)
docker compose -f postgres.yaml up -d

# Start n8n stack
docker compose up -d
```

### Managing Services
```bash
# View logs
docker compose logs -f
docker compose logs -f main
docker compose logs -f runner-js-1
docker compose logs -f runner-js-2
docker compose logs -f runner-py-1
docker compose logs -f runner-py-2

# Stop services
docker compose down

# Restart specific service
docker compose restart main
docker compose restart runner-js-1
docker compose restart runner-py-1

# View service status
docker compose ps
```

### Troubleshooting
```bash
# Check healthchecks
docker inspect n8n-main | grep -A 10 Health
docker inspect n8n-runner-js-1 | grep -A 10 Health
docker inspect n8n-runner-py-1 | grep -A 10 Health

# View runner status
docker compose logs runner-js-1 | grep -i "task-runner"
docker compose logs runner-py-1 | grep -i "task-runner"

# Check task broker connectivity
docker exec n8n-main wget --spider http://localhost:5679/healthz

# View runner healthcheck
docker exec n8n-runner-js-1 wget --spider http://localhost:5681/healthz
docker exec n8n-runner-py-1 wget --spider http://localhost:5681/healthz
```

### Scaling Task Runners
```bash
# Add new runners (edit docker-compose.yml first)
docker compose up -d runner-js-3   # Third JavaScript runner
docker compose up -d runner-py-3   # Third Python runner

# Remove a runner
docker compose stop runner-py-2
docker compose rm runner-py-2
```

## Configuration

### n8n Main Configuration
- **Version**: n8nio/n8n:2.0.1
- **Task Runners Mode**: External mode enabled
- **Task Broker**: Listens on 0.0.0.0:5679 for runner connections
- **Security**: Environment variable access blocked in nodes, bare git repos disabled
- **Domain**: app.n8n.luismaroto.com (configured in Traefik labels)
- **Execution Pruning**: Auto-delete executions older than 7 days (168 hours)

### Task Runner Configuration
- **Version**: n8nio/runners:2.0.1 (must match n8n version)
- **Runtimes**:
  - JavaScript runners: Use `task-runner-launcher javascript`
  - Python runners: Use `task-runner-launcher python`
- **Max Concurrency**: 5 tasks per runner
- **Auto Shutdown**: 15 seconds after idle
- **Broker URI**: Connects to http://main:5679
- **Health Check**: Exposed on port 5681

### Traefik Integration
- Rate limiting: 100 req/min average, 50 burst
- Security headers: HSTS, XSS protection, X-Frame-Options SAMEORIGIN, CSP frame-ancestors
- Automatic HTTPS redirect
- Let's Encrypt certificate resolver
- Healthcheck path: `/healthz`

## Development Notes

### Service Dependencies
- Services use `restart: unless-stopped` for automatic recovery
- Healthchecks enforce startup order: Main → Task Runners
- PostgreSQL dependency cannot be declared (separate compose file) - services retry connection until successful
- Task runners depend on Main being healthy before starting

### Storage
- `n8n_storage`: Shared between Main and all task runners for workflows, credentials, and settings
- All runners need access to the same storage volume to execute workflows correctly

### Accessing Host Services
- Both Main and Task Runner containers use `extra_hosts` to access `host.docker.internal`
- Useful for workflows that need to communicate with services on the Docker host

### Important Considerations
- **Version Matching**: n8nio/n8n and n8nio/runners images must have matching versions
- **Token Security**: Generate a strong `N8N_RUNNERS_AUTH_TOKEN` using `openssl rand -base64 32`
- **Entrypoint Override**: Task runners use custom entrypoint to bypass default n8n wrapper
- **Minimum Version**: Task runners require n8n >= 2.0 (separate runners image introduced in v2.0)

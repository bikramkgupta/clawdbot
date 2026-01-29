# Container Deployment

This document explains how to deploy Moltbot in container environments like Docker, Kubernetes, and DigitalOcean App Platform.

## Deployment Models

Moltbot supports three deployment models:

### 1. Single Container (Simple)

All components in one container - suitable for personal use or small teams.

```
┌─────────────────────────────────────────────────────────────┐
│                    Single Container                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Gateway + Agent Executor + Channels + Browser Control  ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Ports: 18789 (WebSocket/HTTP)                              │
│  Volumes: ~/.moltbot (config/sessions)                      │
└─────────────────────────────────────────────────────────────┘
```

### 2. Gateway + Browser Separated

Gateway and browser automation in separate containers - better isolation.

```
┌───────────────────────────────┐     ┌───────────────────────┐
│      Gateway Container        │     │   Browser Container   │
│  ┌─────────────────────────┐ │     │  ┌─────────────────┐  │
│  │ Gateway + Agent + Cron  │ │◄───►│  │ Chromium + VNC  │  │
│  └─────────────────────────┘ │ CDP │  └─────────────────┘  │
│                               │     │                       │
│  Port: 18789                  │     │  Ports: 9222 (CDP)    │
│                               │     │         5900 (VNC)    │
└───────────────────────────────┘     │         6080 (noVNC)  │
                                      └───────────────────────┘
```

### 3. Distributed (Production)

Multiple gateway instances with shared state - horizontal scaling.

```
                    ┌─────────────────────┐
                    │   Load Balancer     │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Gateway #1    │  │   Gateway #2    │  │   Gateway #3    │
│   (Agent: A,B)  │  │   (Agent: C,D)  │  │   (Agent: A,C)  │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   Shared Storage   │
                    │   (Sessions/State) │
                    └───────────────────┘
```

## Docker Configuration

### Main Dockerfile

The production image is based on Node 22:

```dockerfile
# Dockerfile (simplified)
FROM node:22-bookworm

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    git curl jq socat \
    && rm -rf /var/lib/apt/lists/*

# Install pnpm and bun
RUN npm install -g pnpm@latest && \
    curl -fsSL https://bun.sh/install | bash

WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

# Default command runs gateway
CMD ["node", "dist/entry.js", "gateway", "run"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  moltbot-gateway:
    image: moltbot:latest
    build: .
    ports:
      - "18789:18789"    # WebSocket + HTTP
      - "18790:18790"    # Bridge port
    environment:
      - HOME=/home/node
      - NODE_ENV=production
      - CLAWDBOT_GATEWAY_TOKEN=${CLAWDBOT_GATEWAY_TOKEN}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - ${HOME}/.moltbot:/home/node/.moltbot:rw
      - workspace:/workspace
    command: >
      gateway run
      --bind lan
      --port 18789
    restart: unless-stopped

  moltbot-browser:
    image: moltbot:sandbox-browser
    build:
      context: .
      dockerfile: Dockerfile.sandbox-browser
    ports:
      - "9222:9222"      # Chrome DevTools Protocol
      - "5900:5900"      # VNC
      - "6080:6080"      # noVNC (web)
    environment:
      - DISPLAY=:1
      - CDP_PORT=9222
      - VNC_PORT=5900
      - NOVNC_PORT=6080
      - CLAWDBOT_BROWSER_HEADLESS=0
      - CLAWDBOT_BROWSER_ENABLE_NOVNC=1
    shm_size: 2gb        # Required for Chrome
    restart: unless-stopped

volumes:
  workspace:
```

## Environment Variables

### Core Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `CLAWDBOT_STATE_DIR` | Config/state directory | `~/.moltbot` |
| `CLAWDBOT_CONFIG_PATH` | Config file path | `~/.moltbot/config.json5` |
| `CLAWDBOT_GATEWAY_PORT` | Gateway port | `18789` |
| `CLAWDBOT_GATEWAY_TOKEN` | Auth token | (required for LAN bind) |
| `CLAWDBOT_GATEWAY_PASSWORD` | Auth password | (alternative to token) |

### Component Toggles

| Variable | Description | Default |
|----------|-------------|---------|
| `CLAWDBOT_SKIP_CHANNELS` | Disable channel init | `0` |
| `CLAWDBOT_SKIP_BROWSER_CONTROL_SERVER` | Disable browser server | `0` |
| `CLAWDBOT_SKIP_GMAIL_WATCHER` | Disable Gmail watcher | `0` |
| `CLAWDBOT_SKIP_CRON` | Disable cron service | `0` |
| `CLAWDBOT_SKIP_CANVAS_HOST` | Disable canvas host | `0` |
| `CLAWDBOT_ALLOW_MULTI_GATEWAY` | Allow multiple gateways | `0` |

### Browser Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `CLAWDBOT_BROWSER_CDP_PORT` | CDP port | `9222` |
| `CLAWDBOT_BROWSER_VNC_PORT` | VNC port | `5900` |
| `CLAWDBOT_BROWSER_NOVNC_PORT` | noVNC port | `6080` |
| `CLAWDBOT_BROWSER_HEADLESS` | Headless mode | `1` |
| `CLAWDBOT_BROWSER_ENABLE_NOVNC` | Enable noVNC | `0` |

### AI Provider Keys

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Anthropic (Claude) API key |
| `OPENAI_API_KEY` | OpenAI API key |
| `GOOGLE_AI_API_KEY` | Google AI (Gemini) API key |

## Gateway Binding Modes

The `--bind` option controls network interface binding:

| Mode | Bind Address | Use Case |
|------|--------------|----------|
| `loopback` | `127.0.0.1` | Local-only (default, safe) |
| `lan` | `0.0.0.0` | All interfaces (requires auth) |
| `tailnet` | Tailscale IP | VPN-only access |
| `auto` | Loopback → 0.0.0.0 | Auto-fallback |
| `custom` | Custom IP | Specific interface |

**Security Note:** Binding beyond loopback requires authentication:

```bash
# Generate secure token
export CLAWDBOT_GATEWAY_TOKEN=$(openssl rand -hex 32)

# Run with LAN binding
moltbot gateway run --bind lan --port 18789
```

## DigitalOcean App Platform

### App Specification

```yaml
# .do/app.yaml
name: moltbot
region: nyc
services:
  - name: gateway
    dockerfile_path: Dockerfile
    source_dir: /
    http_port: 18789
    instance_count: 1
    instance_size_slug: professional-xs
    envs:
      - key: NODE_ENV
        value: production
      - key: CLAWDBOT_GATEWAY_TOKEN
        type: SECRET
      - key: ANTHROPIC_API_KEY
        type: SECRET
    health_check:
      http_path: /health
      initial_delay_seconds: 30
      period_seconds: 10
    routes:
      - path: /
```

### Volume Mounts (Persistent Storage)

For persistent sessions and config:

```yaml
services:
  - name: gateway
    # ...
    internal_ports:
      - 18789
    volumes:
      - name: moltbot-data
        size_gib: 5
        mount_path: /home/node/.moltbot
```

## Kubernetes Deployment

### Deployment Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: moltbot-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moltbot-gateway
  template:
    metadata:
      labels:
        app: moltbot-gateway
    spec:
      containers:
        - name: gateway
          image: moltbot:latest
          ports:
            - containerPort: 18789
          env:
            - name: NODE_ENV
              value: production
            - name: CLAWDBOT_GATEWAY_TOKEN
              valueFrom:
                secretKeyRef:
                  name: moltbot-secrets
                  key: gateway-token
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: moltbot-secrets
                  key: anthropic-api-key
          volumeMounts:
            - name: moltbot-data
              mountPath: /home/node/.moltbot
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
      volumes:
        - name: moltbot-data
          persistentVolumeClaim:
            claimName: moltbot-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: moltbot-gateway
spec:
  selector:
    app: moltbot-gateway
  ports:
    - port: 18789
      targetPort: 18789
  type: ClusterIP
```

### Horizontal Scaling

For multiple gateway instances:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: moltbot-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: moltbot-gateway
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Note:** Multiple gateway instances require shared session storage (see [Multi-Tenancy](./multi-tenancy.md)).

## Agent-Specific Containers

You can run different agents in separate containers:

```yaml
# docker-compose.multi-agent.yml
version: '3.8'

services:
  gateway-main:
    image: moltbot:latest
    environment:
      - CLAWDBOT_CONFIG_PATH=/config/main-agent.json5
    volumes:
      - ./configs/main-agent.json5:/config/main-agent.json5:ro
      - main-data:/home/node/.moltbot
    ports:
      - "18789:18789"

  gateway-support:
    image: moltbot:latest
    environment:
      - CLAWDBOT_CONFIG_PATH=/config/support-agent.json5
    volumes:
      - ./configs/support-agent.json5:/config/support-agent.json5:ro
      - support-data:/home/node/.moltbot
    ports:
      - "18790:18789"

  gateway-coder:
    image: moltbot:latest
    environment:
      - CLAWDBOT_CONFIG_PATH=/config/coder-agent.json5
    volumes:
      - ./configs/coder-agent.json5:/config/coder-agent.json5:ro
      - coder-data:/home/node/.moltbot
    ports:
      - "18791:18789"

volumes:
  main-data:
  support-data:
  coder-data:
```

## Sandbox Containers

Agents can execute code in isolated sandbox containers:

```yaml
# Sandbox settings in config.json5
{
  "sandbox": {
    "mode": "all",              // "off", "non-main", "all"
    "scope": "session",         // "session", "agent", "shared"
    "docker": {
      "image": "node:22-bookworm",
      "containerPrefix": "moltbot-sandbox",
      "network": "none",        // Network isolation
      "readOnlyRoot": true,
      "memory": "512m",
      "cpus": 0.5,
      "pidsLimit": 100
    }
  }
}
```

## Browser Container

For browser automation tasks:

```dockerfile
# Dockerfile.sandbox-browser
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    chromium \
    xvfb \
    x11vnc \
    novnc \
    websockify \
    socat \
    && rm -rf /var/lib/apt/lists/*

COPY scripts/sandbox-browser-entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 9222 5900 6080

ENTRYPOINT ["/entrypoint.sh"]
```

### Connecting Gateway to Browser Container

```json5
// config.json5
{
  "browser": {
    "enabled": true,
    "cdpHost": "moltbot-browser",  // Docker service name
    "cdpProtocol": "http",
    "controlPort": 9222
  }
}
```

## Health Checks

The gateway exposes a health endpoint:

```bash
# Health check
curl http://localhost:18789/health

# Response
{"status": "ok", "uptime": 12345}
```

For Kubernetes/Docker health probes:

```yaml
# Kubernetes
livenessProbe:
  httpGet:
    path: /health
    port: 18789
  initialDelaySeconds: 30
  periodSeconds: 10

# Docker Compose
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:18789/health"]
  interval: 30s
  timeout: 10s
  retries: 3
```

## Scaling Considerations

### Stateless Gateway

The gateway is mostly stateless - sessions are stored on disk. For horizontal scaling:

1. **Shared Storage**: Mount shared volume for `~/.moltbot`
2. **Session Affinity**: Use sticky sessions for WebSocket connections
3. **Agent Routing**: Route specific agents to specific instances

### Per-Agent Scaling

Run dedicated containers per agent type:

```
┌──────────────────────────────────────────────────────────────────┐
│                      Load Balancer                                │
│                    (Route by agent ID)                           │
└──────────────────────────────┬───────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Gateway Pool   │  │  Gateway Pool   │  │  Gateway Pool   │
│  Agent: main    │  │  Agent: support │  │  Agent: coder   │
│  Replicas: 2    │  │  Replicas: 3    │  │  Replicas: 1    │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

## Next Steps

- [Multi-Tenancy](./multi-tenancy.md) - Tenant isolation patterns
- [Skills and Tools](./skills-and-tools.md) - Tool configuration
- [Browser Integration](./browser-integration.md) - Headless browser setup

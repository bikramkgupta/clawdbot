# Browser Integration

This document explains how to configure and use browser automation in Moltbot, including headless browser containers and integration with services like Browserless.

## Browser Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       BROWSER INTEGRATION OPTIONS                            │
└─────────────────────────────────────────────────────────────────────────────┘

  Option 1: Embedded Browser (Default)
  ─────────────────────────────────────
  ┌─────────────────────────────────────┐
  │           Gateway Process           │
  │  ┌─────────────────────────────┐   │
  │  │   Browser Control Server    │   │
  │  │   (Puppeteer + Chromium)    │   │
  │  └─────────────────────────────┘   │
  └─────────────────────────────────────┘

  Option 2: Separate Browser Container
  ─────────────────────────────────────
  ┌───────────────────┐     ┌───────────────────────┐
  │     Gateway       │     │   Browser Container   │
  │                   │◄───►│   (Chromium + CDP)    │
  │                   │ CDP │                       │
  └───────────────────┘     └───────────────────────┘

  Option 3: Remote Browser Service (Browserless)
  ──────────────────────────────────────────────
  ┌───────────────────┐     ┌───────────────────────┐
  │     Gateway       │     │     Browserless       │
  │                   │◄───►│   (Cloud Service)     │
  │                   │ CDP │                       │
  └───────────────────┘     └───────────────────────┘
```

## Configuration Options

### Browser Configuration Schema

```json5
{
  "browser": {
    "enabled": true,              // Enable browser tools
    "defaultProfile": "clawd",    // Default browser profile
    "controlPort": 9222,          // CDP port for control server

    "profiles": {
      "clawd": {
        // Embedded browser profile (default)
        "headless": true,
        "noSandbox": true         // Required for containers
      },
      "remote": {
        // Remote browser profile
        "cdpHost": "browser-service.example.com",
        "cdpProtocol": "https",
        "attachOnly": true        // Don't start local browser
      }
    }
  }
}
```

### Full Configuration Options

```typescript
type BrowserConfig = {
  enabled: boolean;              // Enable browser tools
  evaluateEnabled: boolean;      // Enable JS evaluation
  controlPort: number;           // Control server port (default: 9222)
  cdpProtocol: "http" | "https"; // CDP protocol
  cdpHost: string;               // CDP host address
  remoteCdpTimeoutMs: number;    // Remote CDP timeout (default: 30000)
  executablePath?: string;       // Custom Chrome binary path
  headless: boolean;             // Headless mode (default: true)
  noSandbox: boolean;            // Disable Chrome sandbox
  attachOnly: boolean;           // Don't start Chrome, attach only
  defaultProfile: string;        // Default profile name

  profiles: Record<string, {
    cdpHost?: string;
    cdpProtocol?: "http" | "https";
    executablePath?: string;
    headless?: boolean;
    noSandbox?: boolean;
    attachOnly?: boolean;
    color?: string;              // Profile color in UI
  }>;
};
```

## Option 1: Embedded Browser

The simplest option - browser runs inside the gateway process.

### Configuration

```json5
{
  "browser": {
    "enabled": true,
    "headless": true,
    "noSandbox": true    // Required for Docker/containers
  }
}
```

### Docker Requirements

```dockerfile
# Dockerfile additions for embedded browser
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libcups2 \
    libdrm2 \
    libgbm1 \
    libnspr4 \
    libnss3 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    xdg-utils
```

## Option 2: Separate Browser Container

Better isolation - browser runs in a dedicated container.

### Browser Container

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

ENV DISPLAY=:1
ENV CDP_PORT=9222
ENV VNC_PORT=5900
ENV NOVNC_PORT=6080

COPY scripts/sandbox-browser-entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 9222 5900 6080

ENTRYPOINT ["/entrypoint.sh"]
```

### Docker Compose

```yaml
version: '3.8'

services:
  gateway:
    image: moltbot:latest
    ports:
      - "18789:18789"
    environment:
      - CLAWDBOT_BROWSER_CDP_HOST=browser
      - CLAWDBOT_BROWSER_CDP_PORT=9222
    depends_on:
      - browser

  browser:
    image: moltbot:sandbox-browser
    ports:
      - "9222:9222"      # CDP
      - "5900:5900"      # VNC (optional)
      - "6080:6080"      # noVNC (optional)
    environment:
      - CLAWDBOT_BROWSER_HEADLESS=1
    shm_size: 2gb        # Required for Chrome
```

### Gateway Configuration

```json5
{
  "browser": {
    "enabled": true,
    "cdpHost": "browser",     // Docker service name
    "cdpProtocol": "http",
    "controlPort": 9222,
    "attachOnly": true        // Don't start local browser
  }
}
```

## Option 3: Browserless Integration

Use a managed browser service like Browserless.io.

### Browserless Configuration

```json5
{
  "browser": {
    "enabled": true,
    "profiles": {
      "browserless": {
        "cdpHost": "chrome.browserless.io",
        "cdpProtocol": "wss",
        "attachOnly": true
      }
    },
    "defaultProfile": "browserless"
  }
}
```

### With Authentication

```json5
{
  "browser": {
    "enabled": true,
    "profiles": {
      "browserless": {
        "cdpHost": "chrome.browserless.io",
        "cdpProtocol": "wss",
        "attachOnly": true,
        // Auth via CDP URL query params
        "cdpUrl": "wss://chrome.browserless.io?token=${BROWSERLESS_TOKEN}"
      }
    }
  }
}
```

### Environment Variables

```bash
export BROWSERLESS_TOKEN="your-api-token"
export CLAWDBOT_LIVE_BROWSER_CDP_URL="wss://chrome.browserless.io?token=${BROWSERLESS_TOKEN}"
```

## Sandbox Browser Configuration

For agent sandboxing with browser access:

```json5
{
  "sandbox": {
    "browser": {
      "enabled": true,
      "image": "moltbot:sandbox-browser",
      "containerPrefix": "moltbot-browser",
      "cdpPort": 9222,
      "vncPort": 5900,
      "noVncPort": 6080,
      "headless": true,
      "enableNoVnc": false,      // Enable web VNC viewer
      "allowHostControl": false, // Use host browser server
      "autoStart": true,         // Auto-start container
      "autoStartTimeoutMs": 30000
    }
  }
}
```

## Browser Tool Usage

### Agent Access to Browser

```json5
{
  "agents": {
    "entries": [
      {
        "id": "web-automator",
        "tools": {
          "allow": ["browser", "web.fetch"]
        }
      }
    ]
  }
}
```

### Browser Tool Capabilities

The `browser` tool allows agents to:
- Navigate to URLs
- Take screenshots
- Execute JavaScript
- Click elements
- Fill forms
- Extract content
- Manage tabs/windows

### Example Agent Interaction

```
User: Go to example.com and take a screenshot

Agent: I'll use the browser tool to navigate and capture a screenshot.

[Tool: browser]
{
  "action": "navigate",
  "url": "https://example.com"
}

[Tool: browser]
{
  "action": "screenshot",
  "fullPage": true
}

Agent: I've navigated to example.com and captured a screenshot. [Image attached]
```

## Multi-Container Browser Setup

### High-Availability Configuration

```yaml
# docker-compose.browser-ha.yml
version: '3.8'

services:
  gateway:
    image: moltbot:latest
    ports:
      - "18789:18789"
    deploy:
      replicas: 2

  browser-pool:
    image: moltbot:sandbox-browser
    deploy:
      replicas: 3
    shm_size: 2gb
    networks:
      - browser-net

  browser-lb:
    image: haproxy:latest
    ports:
      - "9222:9222"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks:
      - browser-net

networks:
  browser-net:
```

### HAProxy Configuration

```
# haproxy.cfg
frontend browser_frontend
    bind *:9222
    default_backend browser_pool

backend browser_pool
    balance roundrobin
    server browser1 browser-pool:9222 check
    server browser2 browser-pool:9222 check
    server browser3 browser-pool:9222 check
```

## Debugging Browser Issues

### VNC Access

Enable VNC for visual debugging:

```json5
{
  "sandbox": {
    "browser": {
      "headless": false,
      "enableNoVnc": true,
      "vncPort": 5900,
      "noVncPort": 6080
    }
  }
}
```

Access via:
- VNC client: `vnc://localhost:5900`
- Web browser: `http://localhost:6080/vnc.html`

### CDP Debugging

Connect Chrome DevTools to remote browser:

```bash
# Forward CDP port
kubectl port-forward svc/browser 9222:9222

# Open in Chrome
# chrome://inspect
# Configure > Add connection > localhost:9222
```

### Logging

```json5
{
  "browser": {
    "debug": true,
    "logLevel": "verbose"
  }
}
```

## Browser Profiles

### Multiple Profiles

```json5
{
  "browser": {
    "profiles": {
      "default": {
        "headless": true
      },
      "debug": {
        "headless": false,
        "executablePath": "/usr/bin/google-chrome"
      },
      "production": {
        "cdpHost": "browser-service.internal",
        "attachOnly": true
      }
    },
    "defaultProfile": "production"
  }
}
```

### Profile Selection per Agent

```json5
{
  "agents": {
    "entries": [
      {
        "id": "scraper",
        "browser": {
          "profile": "production"
        }
      },
      {
        "id": "tester",
        "browser": {
          "profile": "debug"
        }
      }
    ]
  }
}
```

## Resource Limits

### Container Resources

```yaml
services:
  browser:
    image: moltbot:sandbox-browser
    shm_size: 2gb           # Required for Chrome
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 1G
```

### Browser Launch Arguments

```json5
{
  "browser": {
    "launchArgs": [
      "--disable-dev-shm-usage",
      "--disable-gpu",
      "--no-first-run",
      "--disable-extensions",
      "--disable-default-apps",
      "--single-process"        // For low-memory environments
    ]
  }
}
```

## Security Considerations

### Network Isolation

```json5
{
  "sandbox": {
    "browser": {
      "enabled": true
    },
    "docker": {
      "network": "browser-only"  // Restricted network
    }
  }
}
```

### Content Security

```json5
{
  "browser": {
    "allowedDomains": [
      "*.example.com",
      "api.trusted-service.com"
    ],
    "blockedDomains": [
      "*.malicious.com"
    ]
  }
}
```

## Next Steps

- [Container Deployment](./container-deployment.md) - Full container setup
- [Skills and Tools](./skills-and-tools.md) - Tool configuration
- [Multi-Tenancy](./multi-tenancy.md) - Tenant isolation

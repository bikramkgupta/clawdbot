# Moltbot Developer Documentation

Welcome to the Moltbot developer documentation. This guide is designed for developers who want to understand the system architecture, deploy Moltbot in container environments, and build multi-tenant applications.

## Quick Navigation

| Document | Description |
|----------|-------------|
| [Architecture Overview](./architecture-overview.md) | High-level system architecture and components |
| [Agent System](./agent-system.md) | How agents are defined, configured, and executed |
| [Request Flow](./request-flow.md) | Detailed request processing from receipt to response |
| [Container Deployment](./container-deployment.md) | Docker, Kubernetes, and cloud deployment |
| [Multi-Tenancy](./multi-tenancy.md) | Tenant isolation patterns and security |
| [Skills and Tools](./skills-and-tools.md) | Tool configuration, routing, and custom skills |
| [Browser Integration](./browser-integration.md) | Headless browser and Browserless setup |

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MOLTBOT SYSTEM OVERVIEW                             │
└─────────────────────────────────────────────────────────────────────────────┘

                     External Messaging Channels
     ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
     │Telegram │  │ Discord │  │  Slack  │  │WhatsApp │  │  Web UI │
     └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
          │            │            │            │            │
          └────────────┴────────────┼────────────┴────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │         Gateway Server         │
                    │    (WebSocket + HTTP :18789)   │
                    └───────────────┬───────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
   ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
   │   Routing   │          │    Agent    │          │    Tools    │
   │   Engine    │─────────>│   Executor  │─────────>│   Runtime   │
   └─────────────┘          └──────┬──────┘          └─────────────┘
                                   │
                                   ▼
                           ┌─────────────┐
                           │  AI Models  │
                           │ (Claude,etc)│
                           └─────────────┘
```

## Key Concepts

### 1. Gateway
The **Gateway** is a long-running server that:
- Handles incoming messages from all channels
- Manages WebSocket connections for web UI
- Routes requests to appropriate agents
- Coordinates tool execution
- Maintains session state

### 2. Agents
**Agents** are AI assistants with:
- Unique identifiers and configurations
- Model settings (primary + fallback)
- Tool access policies
- Isolated storage and workspaces

### 3. Sessions
**Sessions** track conversations:
- Unique session key per user/agent/channel combination
- Persisted conversation history
- Isolated per-tenant when configured

### 4. Tools
**Tools** are capabilities available to agents:
- Core tools: read, write, edit, exec
- Moltbot tools: browser, message, web.search
- Custom skills from workspace

## Common Deployment Patterns

### Personal Use (Single Container)
```bash
docker run -d \
  -p 18789:18789 \
  -v ~/.moltbot:/home/node/.moltbot \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  moltbot:latest
```

### Multi-Agent Production
```yaml
services:
  gateway:
    image: moltbot:latest
    environment:
      - CLAWDBOT_GATEWAY_TOKEN=${TOKEN}
    volumes:
      - moltbot-data:/home/node/.moltbot
```

### Multi-Tenant (Agent per Tenant)
```json5
{
  "agents": {
    "entries": [
      { "id": "tenant-a", "workspace": "/tenants/a" },
      { "id": "tenant-b", "workspace": "/tenants/b" }
    ]
  }
}
```

## Quick Answers

### How do the orchestrator and CLI relate?
The **CLI** (`moltbot` command) can either run directly (processing requests in-process) or connect to a **Gateway** server. The Gateway is the "orchestrator" that handles multi-channel, multi-session coordination. For production, run the Gateway and have CLI commands connect to it.

### Can I run agents in separate containers?
Yes. You can either:
1. Run multiple Gateway instances, each with different agent configurations
2. Use a single Gateway with multiple agents defined, routing to each based on channel/binding
3. Mix both approaches for hybrid scaling

### How do I define multiple agents?
In `config.json5`:
```json5
{
  "agents": {
    "entries": [
      { "id": "main", "default": true },
      { "id": "coder", "tools": { "allow": ["exec"] } },
      { "id": "researcher", "tools": { "allow": ["web.search"] } }
    ]
  }
}
```

### How is routing decided?
Routing uses a priority system:
1. Explicit bindings (channel + peer/guild/team)
2. Account-level bindings
3. Default agent

### Can I use a central router agent?
Yes. Configure a "router" agent with `sessions.spawn` tool access that can delegate to specialized agents. See [Skills and Tools](./skills-and-tools.md#central-router-model).

### How do I isolate tenants?
Multiple approaches:
- **Agent-per-tenant**: Each tenant gets their own agent ID with separate storage
- **Gateway-per-tenant**: Separate container per tenant
- **Session isolation**: Default - each user gets isolated sessions

### Can I use Browserless or other browser services?
Yes. Configure the browser to connect to remote CDP:
```json5
{
  "browser": {
    "cdpHost": "chrome.browserless.io",
    "attachOnly": true
  }
}
```

## File Locations

| Path | Purpose |
|------|---------|
| `~/.moltbot/config.json5` | Main configuration |
| `~/.moltbot/agents/{id}/` | Per-agent storage |
| `~/.moltbot/agents/{id}/sessions/` | Session transcripts |
| `~/.moltbot/credentials/` | OAuth tokens, API keys |
| `~/.moltbot/plugins/` | Installed plugins |

## Environment Variables

| Variable | Description |
|----------|-------------|
| `CLAWDBOT_STATE_DIR` | Override state directory |
| `CLAWDBOT_GATEWAY_TOKEN` | Gateway authentication |
| `ANTHROPIC_API_KEY` | Claude API key |
| `CLAWDBOT_SKIP_*` | Disable specific components |

## Getting Help

- Check the specific documentation pages linked above
- Review the configuration schema in `src/config/types.*.ts`
- Examine working examples in `docker-compose.yml`

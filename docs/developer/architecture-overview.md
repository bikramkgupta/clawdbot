# Moltbot Architecture Overview

This document provides a high-level overview of the Moltbot system architecture, explaining how the various components work together.

## System Components

Moltbot consists of several key components that can run together as a single process or be distributed across multiple containers:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Moltbot Gateway                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                        WebSocket + HTTP Server                          ││
│  │                         (Port 18789 default)                            ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                     │                                        │
│       ┌─────────────────────────────┼─────────────────────────────┐         │
│       │                             │                             │         │
│  ┌────▼─────┐  ┌──────────────┐  ┌──▼───────────┐  ┌────────────┐│         │
│  │ RPC      │  │   Channel    │  │    Agent     │  │   Cron     ││         │
│  │ Methods  │  │   Manager    │  │   Executor   │  │  Service   ││         │
│  └────┬─────┘  └──────┬───────┘  └──────┬───────┘  └────────────┘│         │
│       │               │                 │                         │         │
│  ┌────▼─────┐  ┌──────▼───────┐  ┌──────▼───────┐  ┌────────────┐│         │
│  │ Session  │  │   Telegram   │  │    Tools     │  │  Browser   ││         │
│  │ Store    │  │   Discord    │  │   Runtime    │  │  Control   ││         │
│  │          │  │   Slack      │  │              │  │  Server    ││         │
│  │          │  │   WhatsApp   │  │              │  │            ││         │
│  │          │  │   Signal     │  │              │  │            ││         │
│  └──────────┘  └──────────────┘  └──────────────┘  └────────────┘│         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
   ┌──────▼──────┐            ┌───────▼───────┐           ┌───────▼───────┐
   │   Web UI    │            │ CLI Commands  │           │ Mobile Apps   │
   │  (Browser)  │            │  (Terminal)   │           │  (iOS/macOS)  │
   └─────────────┘            └───────────────┘           └───────────────┘
```

## Process Model

### Single Process (Default)

By default, Moltbot runs as a **single Node.js process** that includes:

- **Gateway Server**: WebSocket/HTTP server handling all external connections
- **Agent Executor**: Runs AI agent conversations
- **Channel Manager**: Manages connections to messaging platforms
- **Browser Control Server**: Puppeteer-based browser automation
- **Cron Service**: Scheduled task execution
- **Session Store**: Conversation state management

This is the simplest deployment model and works well for personal use or small teams.

### Multi-Process (Container Deployment)

For production deployments, components can be distributed:

| Component | Can Run Separately? | Communication |
|-----------|---------------------|---------------|
| Gateway Server | Yes (required) | WebSocket RPC |
| Browser Container | Yes | CDP over TCP |
| Sandbox Containers | Yes | Docker stdio |
| CLI Commands | Yes | WebSocket to Gateway |

## Key Entry Points

| File | Purpose |
|------|---------|
| `src/entry.ts` | Main CLI entry point |
| `src/index.ts` | Program initialization |
| `src/cli/run-main.ts` | Command routing |
| `src/gateway/server.impl.ts` | Gateway server factory |
| `src/agents/pi-embedded-runner/run.ts` | Agent execution engine |

## CLI vs Gateway Relationship

The **CLI** (`moltbot` command) can operate in two modes:

### Direct Mode (No Gateway)
```bash
# Runs agent directly in CLI process
moltbot agent --message "Hello"
```

### Gateway Mode (Recommended for Production)
```bash
# Start gateway (long-running server)
moltbot gateway run --bind loopback --port 18789

# Send requests to gateway
moltbot send "Hello" --to telegram:@user
```

In gateway mode:
1. CLI connects to gateway via WebSocket
2. Gateway handles request routing, session management, and channel delivery
3. Multiple CLI sessions can share the same gateway

## Configuration Hierarchy

```
Environment Variables (highest priority)
         │
         ▼
    CLI Flags
         │
         ▼
~/.moltbot/config.json5
         │
         ▼
Platform-specific overrides (config.darwin.json5, etc.)
         │
         ▼
    Built-in Defaults (lowest priority)
```

## Data Storage Layout

```
~/.moltbot/                          # State directory (MOLTBOT_STATE_DIR)
├── config.json5                     # Main configuration
├── credentials/                     # OAuth tokens, API keys
│   └── oauth.json
├── agents/                          # Per-agent storage
│   ├── main/                        # Default agent
│   │   ├── agent/                   # Agent working directory
│   │   └── sessions/                # Session transcripts
│   │       ├── sessions.json        # Session metadata store
│   │       └── *.jsonl              # Conversation logs
│   └── {agent-id}/                  # Additional agents
│       ├── agent/
│       └── sessions/
├── plugins/                         # Installed plugins
└── logs/                            # Runtime logs
```

## Gateway Architecture Details

The gateway is the heart of a production Moltbot deployment:

```
Gateway Server
├── WebSocket Server (server-ws-runtime.ts)
│   └── Chat sessions (server-chat.ts)
│       └── Agent runs with tool execution
├── RPC Methods (server-methods/)
│   ├── agent.ts      - Execute agent runs
│   ├── chat.ts       - WebChat management
│   ├── send.ts       - Message delivery
│   ├── sessions.ts   - Session management
│   ├── channels.ts   - Channel status
│   ├── cron.ts       - Scheduled tasks
│   └── ... (24+ method handlers)
├── Channel Manager (server-channels.ts)
│   └── Per-channel plugin instances
├── Node Registry (node-registry.ts)
│   └── Connected mobile/desktop nodes
├── Plugin Runtime (server-plugins.ts)
└── Health/Status Tracking
```

### RPC Authorization Scopes

The gateway uses scope-based access control:

| Scope | Access Level | Example Methods |
|-------|--------------|-----------------|
| `ADMIN_SCOPE` | Full control | config, updates, approvals |
| `READ_SCOPE` | Read-only | health, status, sessions.list |
| `WRITE_SCOPE` | Send/execute | agent, send, cron |
| `APPROVALS_SCOPE` | Execution approval | exec.approve |
| `PAIRING_SCOPE` | Device pairing | nodes.pair |

## Next Steps

- [Agent System](./agent-system.md) - How agents are defined and executed
- [Request Flow](./request-flow.md) - How requests are processed
- [Container Deployment](./container-deployment.md) - Running in Docker/Kubernetes
- [Multi-Tenancy](./multi-tenancy.md) - Tenant isolation patterns

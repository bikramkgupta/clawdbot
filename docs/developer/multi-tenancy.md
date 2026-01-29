# Multi-Tenancy and Isolation

This document explains how Moltbot isolates different tenants, users, and agent instances for secure multi-tenant deployments.

## Isolation Layers

Moltbot provides multiple layers of isolation:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ISOLATION LAYERS                                    │
└─────────────────────────────────────────────────────────────────────────────┘

  Layer 1: Agent Isolation
  ─────────────────────────
  Each agent has completely separate:
  - Configuration (model, tools, identity)
  - Workspace directory
  - Session storage
  - Conversation history

  Layer 2: Session/User Isolation
  ────────────────────────────────
  Within each agent:
  - Each user gets a unique session key
  - Separate conversation history per user
  - Isolated context and memory

  Layer 3: Channel/Account Isolation
  ───────────────────────────────────
  - Multiple accounts per channel (e.g., multiple Telegram bots)
  - Each account can route to different agents
  - Groups vs DMs handled separately

  Layer 4: Execution Isolation (Sandboxing)
  ─────────────────────────────────────────
  - Docker containers for code execution
  - Per-session or per-agent sandbox scope
  - Network isolation, resource limits
```

## Session Key Structure

The session key encodes all isolation boundaries:

```
agent:{agentId}:{channel}:{accountId}:{chatType}:{scope}:{peerId}
```

| Component | Description | Example |
|-----------|-------------|---------|
| `agentId` | Agent identifier | `main`, `support`, `coder` |
| `channel` | Messaging platform | `telegram`, `discord`, `slack` |
| `accountId` | Bot account ID | `primary`, `prod-bot` |
| `chatType` | Conversation type | `dm`, `group`, `channel` |
| `scope` | DM scoping mode | `main`, `per-peer` |
| `peerId` | User/group identifier | `123456`, `C01234` |

**Examples:**
```
agent:main:telegram:primary:dm:main:123456
agent:support:discord:prod:group:guild-123:chan-456
agent:coder:slack:workspace-1:dm:per-peer:U12345
```

## Agent-Level Isolation

### Separate Storage per Agent

```
~/.moltbot/agents/
├── main/                          # Agent: main
│   ├── agent/                     # Agent working directory
│   │   ├── .clawdbot/             # Agent-specific config
│   │   └── (workspace files)
│   └── sessions/
│       ├── sessions.json          # Session store
│       ├── agent:main:*.jsonl     # Conversation logs
│       └── ...
├── support/                       # Agent: support
│   ├── agent/
│   └── sessions/
│       ├── sessions.json
│       └── agent:support:*.jsonl
└── coder/                         # Agent: coder
    ├── agent/
    └── sessions/
```

### Agent Configuration

```json5
{
  "agents": {
    "entries": [
      {
        "id": "tenant-a",
        "workspace": "/data/tenants/a/workspace",
        "agentDir": "/data/tenants/a/agent",
        "model": { "primary": "claude-opus-4-5" },
        "tools": { "allow": ["read", "write", "exec"] }
      },
      {
        "id": "tenant-b",
        "workspace": "/data/tenants/b/workspace",
        "agentDir": "/data/tenants/b/agent",
        "model": { "primary": "claude-sonnet-4" },
        "tools": { "allow": ["read", "web.search"] }
      }
    ]
  }
}
```

## User/Session Isolation

### Session Scoping Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `per-sender` | Unique session per user | Multi-user bot (default) |
| `global` | Shared session for all users | Single-user or team bot |

```json5
{
  "session": {
    "scope": "per-sender"  // or "global"
  }
}
```

### DM Scoping

Controls how DM conversations are grouped:

| Mode | Behavior |
|------|----------|
| `main` | All DMs share one session per agent |
| `per-peer` | Each user gets unique session |
| `per-channel-peer` | Unique per user per channel |

```json5
{
  "session": {
    "dmScope": "per-peer"
  }
}
```

## Multi-Tenant Architecture

### Option 1: Agent-per-Tenant

Assign each tenant their own agent:

```json5
{
  "agents": {
    "entries": [
      { "id": "tenant-acme", "workspace": "/tenants/acme" },
      { "id": "tenant-globex", "workspace": "/tenants/globex" },
      { "id": "tenant-initech", "workspace": "/tenants/initech" }
    ]
  },
  "routing": {
    "bindings": [
      {
        "agentId": "tenant-acme",
        "match": { "channel": "slack", "teamId": "T_ACME" }
      },
      {
        "agentId": "tenant-globex",
        "match": { "channel": "slack", "teamId": "T_GLOBEX" }
      }
    ]
  }
}
```

**Benefits:**
- Complete isolation between tenants
- Per-tenant model/tool configuration
- Separate billing/usage tracking possible

### Option 2: Gateway-per-Tenant

Run separate gateway instances:

```yaml
# docker-compose.yml
services:
  tenant-acme:
    image: moltbot:latest
    environment:
      - CLAWDBOT_STATE_DIR=/data/acme
      - CLAWDBOT_CONFIG_PATH=/configs/acme.json5
    volumes:
      - acme-data:/data/acme
      - ./configs:/configs:ro

  tenant-globex:
    image: moltbot:latest
    environment:
      - CLAWDBOT_STATE_DIR=/data/globex
      - CLAWDBOT_CONFIG_PATH=/configs/globex.json5
    volumes:
      - globex-data:/data/globex
      - ./configs:/configs:ro
```

**Benefits:**
- Process-level isolation
- Independent scaling
- No shared state

### Option 3: Hybrid (Agent + Container)

Combine agent isolation with container separation:

```
                    ┌─────────────────────┐
                    │    Router/LB        │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Container #1   │  │  Container #2   │  │  Container #3   │
│  Agents: A, B   │  │  Agents: C, D   │  │  Agents: E, F   │
│  (Small tenants)│  │ (Medium tenant) │  │ (Large tenant)  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

## Sandbox Isolation

### Sandbox Scopes

```json5
{
  "sandbox": {
    "scope": "session"  // "session", "agent", or "shared"
  }
}
```

| Scope | Container Lifecycle | Isolation Level |
|-------|---------------------|-----------------|
| `session` | Per conversation | Highest (each chat = new container) |
| `agent` | Per agent | Medium (all chats share agent container) |
| `shared` | Single container | Lowest (all agents share) |

### Docker Sandbox Settings

```json5
{
  "sandbox": {
    "mode": "all",
    "docker": {
      "image": "node:22-bookworm",
      "network": "none",           // Network isolation
      "readOnlyRoot": true,        // Read-only filesystem
      "memory": "512m",            // Memory limit
      "cpus": 0.5,                 // CPU limit
      "pidsLimit": 100,            // Process limit
      "capDrop": ["ALL"],          // Drop all capabilities
      "seccompProfile": "runtime/default"
    }
  }
}
```

## Data Isolation Patterns

### Tenant Data Directory Structure

```
/data/
├── tenants/
│   ├── acme/
│   │   ├── .moltbot/           # Moltbot state
│   │   │   ├── config.json5
│   │   │   └── agents/
│   │   └── workspace/          # Tenant workspace
│   ├── globex/
│   │   ├── .moltbot/
│   │   └── workspace/
│   └── initech/
│       ├── .moltbot/
│       └── workspace/
└── shared/
    └── plugins/               # Shared plugins (if needed)
```

### Environment-Based Tenant Routing

```bash
# Tenant-specific environment
export CLAWDBOT_STATE_DIR=/data/tenants/${TENANT_ID}/.moltbot
export CLAWDBOT_WORKSPACE_DIR=/data/tenants/${TENANT_ID}/workspace

moltbot gateway run --bind lan --port 18789
```

## Authentication and Authorization

### Gateway Authentication

```json5
{
  "gateway": {
    "auth": {
      "token": "${GATEWAY_TOKEN}",      // Bearer token
      // OR
      "password": "${GATEWAY_PASSWORD}" // Password auth
    }
  }
}
```

### RPC Scope-Based Authorization

```typescript
// Authorization scopes (from server-methods.ts)
const ADMIN_SCOPE = "admin";      // Full control
const READ_SCOPE = "read";        // Read-only
const WRITE_SCOPE = "write";      // Send/execute
const APPROVALS_SCOPE = "approvals";
const PAIRING_SCOPE = "pairing";
```

### Per-Tenant API Keys

```json5
{
  "agents": {
    "entries": [
      {
        "id": "tenant-a",
        // Tenant-specific API keys via environment
        "env": {
          "ANTHROPIC_API_KEY": "${TENANT_A_ANTHROPIC_KEY}"
        }
      }
    ]
  }
}
```

## Monitoring Multi-Tenant Deployments

### Tenant-Specific Metrics

Track usage per agent/tenant:

```typescript
// Session metadata includes tenant context
type SessionEntry = {
  sessionId: string;
  agentId: string;          // Tenant identifier
  tokenUsage: {
    input: number;
    output: number;
  };
  // ...
};
```

### Logging with Tenant Context

```json5
{
  "logging": {
    "format": "json",
    "fields": ["timestamp", "agentId", "sessionKey", "channel"]
  }
}
```

## Security Best Practices

### 1. Principle of Least Privilege

```json5
{
  "agents": {
    "entries": [
      {
        "id": "restricted-tenant",
        "tools": {
          "allow": ["read"],           // Only read access
          "deny": ["exec", "write"]    // No code execution
        },
        "sandbox": {
          "mode": "all",
          "workspaceAccess": "ro"      // Read-only workspace
        }
      }
    ]
  }
}
```

### 2. Network Isolation

```json5
{
  "sandbox": {
    "docker": {
      "network": "none",               // No network access
      // OR
      "network": "tenant-${agentId}"   // Tenant-specific network
    }
  }
}
```

### 3. Resource Limits

```json5
{
  "sandbox": {
    "docker": {
      "memory": "256m",
      "cpus": 0.25,
      "pidsLimit": 50,
      "ulimits": {
        "nofile": { "soft": 1024, "hard": 2048 }
      }
    }
  }
}
```

### 4. Audit Logging

All session transcripts are logged:

```
~/.moltbot/agents/{agentId}/sessions/{sessionKey}.jsonl
```

Example entry:
```json
{"role":"user","content":"...","timestamp":1234567890,"from":"user@123"}
{"role":"assistant","content":"...","timestamp":1234567891}
```

## Scaling Multi-Tenant

### Horizontal Scaling Pattern

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Multi-Tenant Architecture                     │
└──────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │  API Gateway/   │
                         │  Load Balancer  │
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │  Tenant Pool A  │ │  Tenant Pool B  │ │  Tenant Pool C  │
    │  (Small: 1-10)  │ │ (Medium: 11-50) │ │ (Large: 51+)    │
    └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
             │                   │                   │
             ▼                   ▼                   ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │  Shared Volume  │ │  Shared Volume  │ │ Dedicated Vol   │
    │  /data/pool-a   │ │  /data/pool-b   │ │ /data/tenant-c  │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

### Tenant Routing

Route requests to appropriate pool:

```nginx
# nginx.conf
upstream pool_small {
    server gateway-pool-a:18789;
}

upstream pool_medium {
    server gateway-pool-b-1:18789;
    server gateway-pool-b-2:18789;
}

upstream pool_large {
    server gateway-pool-c-1:18789;
    server gateway-pool-c-2:18789;
    server gateway-pool-c-3:18789;
}

map $http_x_tenant_id $backend {
    ~^tenant-(small-\d+)$   pool_small;
    ~^tenant-(medium-\d+)$  pool_medium;
    default                 pool_large;
}
```

## Next Steps

- [Container Deployment](./container-deployment.md) - Running in containers
- [Skills and Tools](./skills-and-tools.md) - Tool configuration
- [Architecture Overview](./architecture-overview.md) - System architecture

# Skills and Tools

This document explains how tools (skills) are configured, loaded, and routed to agents in Moltbot.

## Tool System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            TOOL SYSTEM                                       │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────────────┐
  │                           Tool Sources                                     │
  │                                                                           │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
  │  │   Coding    │  │   Moltbot   │  │   Plugin    │  │    Workspace    │  │
  │  │   Tools     │  │   Tools     │  │   Tools     │  │    Skills       │  │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │
  │         │                │                │                   │          │
  └─────────┼────────────────┼────────────────┼───────────────────┼──────────┘
            │                │                │                   │
            └────────────────┼────────────────┼───────────────────┘
                             │                │
                             ▼                ▼
                    ┌────────────────────────────────┐
                    │         Tool Policy            │
                    │   (allow/deny per agent)       │
                    └───────────────┬────────────────┘
                                    │
                                    ▼
                    ┌────────────────────────────────┐
                    │       Agent Tool Context       │
                    │    (filtered tools for run)    │
                    └────────────────────────────────┘
```

## Tool Categories

### 1. Coding Tools (from pi-coding-agent)

Core file and shell operations:

| Tool | Description |
|------|-------------|
| `read` | Read file contents |
| `write` | Create or overwrite files |
| `edit` | Apply patch-based edits to files |
| `exec` | Execute shell commands |
| `process` | Interactive shell processes |

### 2. Moltbot Tools

Moltbot-specific capabilities:

| Tool | Description |
|------|-------------|
| `browser` | Control browser via Puppeteer |
| `canvas` | Draw graphics/UI elements |
| `message` | Send messages to channels |
| `tts` | Text-to-speech audio generation |
| `web.search` | Search the web |
| `web.fetch` | Fetch URL contents |
| `sessions.list` | List conversation sessions |
| `sessions.history` | Get conversation history |
| `sessions.send` | Send message to another session |
| `sessions.spawn` | Create subagent sessions |
| `agents.list` | List available agents |
| `cron` | Schedule future tasks |
| `nodes` | Execute on connected devices |
| `gateway` | Query gateway status |
| `image` | Process images |

### 3. Plugin Tools

Tools provided by installed plugins (e.g., custom integrations).

### 4. Workspace Skills

Custom skills defined in workspace `.skill.json` files.

## Tool Policy Configuration

### Global Policy

Applied to all agents:

```json5
{
  "tools": {
    "allow": [
      "read",
      "web.search",
      "web.fetch"
    ],
    "deny": [
      "exec",
      "write"
    ]
  }
}
```

### Per-Agent Policy

Override global policy for specific agents:

```json5
{
  "agents": {
    "entries": [
      {
        "id": "coder",
        "tools": {
          "allow": ["exec", "write", "edit"],  // Override: enable coding
          "deny": []                           // Clear denies
        }
      },
      {
        "id": "researcher",
        "tools": {
          "allow": ["web.search", "web.fetch", "read"],
          "deny": ["exec", "write", "edit", "browser"]
        }
      }
    ]
  }
}
```

### Tool Groups

Enable entire categories at once:

```json5
{
  "agents": {
    "entries": [
      {
        "id": "full-access",
        "tools": {
          "groups": ["coding", "browser", "messaging"]
        }
      }
    ]
  }
}
```

Available groups:
- `coding` - read, write, edit, exec
- `browser` - browser control
- `messaging` - message, tts
- `web` - web.search, web.fetch
- `sessions` - sessions.*, agents.*

## Tool Routing Logic

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          TOOL ROUTING FLOW                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  Agent Request
       │
       ▼
  ┌─────────────────────┐
  │ Load Agent Config   │
  │ (tools.allow/deny)  │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ Merge with Global   │
  │ Tool Policy         │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ Check Sandbox Mode  │────────┐
  │ (filter if sandboxed)│       │
  └──────────┬──────────┘        │
             │                   │
             ▼                   ▼
  ┌─────────────────────┐  ┌─────────────────────┐
  │ Host Tools          │  │ Sandbox Tools       │
  │ (direct execution)  │  │ (container exec)    │
  └──────────┬──────────┘  └──────────┬──────────┘
             │                        │
             └────────────┬───────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │  Final Tool Set     │
               │  (for this run)     │
               └─────────────────────┘
```

### Policy Resolution

```typescript
// Tool availability check (simplified)
function isToolAllowed(tool: string, config: AgentConfig): boolean {
  // 1. Check explicit deny (highest priority)
  if (config.tools?.deny?.includes(tool)) return false;

  // 2. Check explicit allow
  if (config.tools?.allow?.includes(tool)) return true;

  // 3. Check tool groups
  if (config.tools?.groups?.some(g => toolInGroup(tool, g))) return true;

  // 4. Check global policy
  if (globalPolicy.deny?.includes(tool)) return false;
  if (globalPolicy.allow?.includes(tool)) return true;

  // 5. Default: allow
  return true;
}
```

## Workspace Skills

### Creating a Skill

Skills are defined in `.skill.json` files in the workspace:

```json
{
  "name": "database-query",
  "description": "Query the company database for information",
  "version": "1.0.0",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "SQL query to execute"
      },
      "database": {
        "type": "string",
        "enum": ["analytics", "customers", "inventory"]
      }
    },
    "required": ["query", "database"]
  },
  "execute": {
    "type": "shell",
    "command": "python /tools/db-query.py --query '${query}' --db '${database}'"
  }
}
```

### Skill Discovery

Skills are discovered from:
1. Workspace root (`.skill.json` files)
2. `skills/` directory in workspace
3. Agent-specific skill directories

```
~/workspace/
├── check-inventory.skill.json
├── skills/
│   ├── analyze-logs.skill.json
│   └── generate-report.skill.json
└── ...
```

### Skill Configuration

```json5
{
  "skills": {
    "enabled": true,
    "paths": [
      "~/shared-skills",
      "${workspace}/skills"
    ],
    "env": {
      "DB_HOST": "${DATABASE_URL}"
    }
  }
}
```

## Central Router vs Agent-Specific

### Central Router Model

A "router" agent decides which specialized agent handles each request:

```json5
{
  "agents": {
    "entries": [
      {
        "id": "router",
        "default": true,
        "tools": {
          "allow": ["sessions.spawn", "agents.list"]
        },
        "identity": {
          "systemPrompt": "You are a router. Analyze requests and delegate to specialized agents."
        },
        "subagents": {
          "allowAgents": ["coder", "researcher", "support"]
        }
      },
      {
        "id": "coder",
        "tools": { "allow": ["read", "write", "edit", "exec"] }
      },
      {
        "id": "researcher",
        "tools": { "allow": ["web.search", "web.fetch"] }
      },
      {
        "id": "support",
        "tools": { "allow": ["message", "sessions.history"] }
      }
    ]
  }
}
```

**Flow:**
```
User Request
     │
     ▼
┌─────────────────┐
│  Router Agent   │
│  (decides who)  │
└────────┬────────┘
         │ sessions.spawn
         ▼
┌────────────────────────────────────────┐
│                                        │
▼                  ▼                     ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐
│  Coder   │  │Researcher│  │   Support    │
│  Agent   │  │  Agent   │  │    Agent     │
└──────────┘  └──────────┘  └──────────────┘
```

### Direct Agent Routing

Route directly to agents based on channel/binding:

```json5
{
  "routing": {
    "bindings": [
      {
        "agentId": "coder",
        "match": { "channel": "discord", "guildId": "dev-guild" }
      },
      {
        "agentId": "support",
        "match": { "channel": "telegram", "peer": { "kind": "group", "id": "-1001234" } }
      }
    ]
  }
}
```

**Flow:**
```
User Request (from Discord dev-guild)
     │
     ▼
┌─────────────────┐
│ Routing Engine  │
│ (check bindings)│
└────────┬────────┘
         │
         ▼ (matches guild binding)
┌─────────────────┐
│   Coder Agent   │
│  (direct call)  │
└─────────────────┘
```

### Hybrid Approach

Combine routing with fallback to router:

```json5
{
  "agents": {
    "entries": [
      {
        "id": "router",
        "default": true,  // Fallback for unmatched requests
        "subagents": { "allowAgents": ["coder", "researcher", "support"] }
      },
      { "id": "coder", ... },
      { "id": "researcher", ... },
      { "id": "support", ... }
    ]
  },
  "routing": {
    "bindings": [
      // Direct routing for known contexts
      { "agentId": "coder", "match": { "channel": "discord", "guildId": "dev" } },
      // Everything else goes to router
    ]
  }
}
```

## Tool Execution Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TOOL EXECUTION FLOW                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  AI Model Response: tool_use
         │
         ├── tool: "read"
         │   file: "/path/to/file.txt"
         │
         ▼
  ┌─────────────────────┐
  │   Tool Router       │
  │  (moltbot-tools.ts) │
  └──────────┬──────────┘
             │
             │ Check: Is tool allowed for this agent?
             │
        ┌────┴────┐
        │         │
    Allowed    Denied
        │         │
        ▼         ▼
  ┌──────────┐  ┌──────────┐
  │ Execute  │  │  Return  │
  │   Tool   │  │  Error   │
  └────┬─────┘  └──────────┘
       │
       │ Check: Sandbox mode?
       │
  ┌────┴────┐
  │         │
Host    Sandbox
  │         │
  ▼         ▼
┌───────────┐  ┌───────────────┐
│  Direct   │  │   Docker      │
│ Execution │  │  Container    │
│           │  │  Execution    │
└─────┬─────┘  └───────┬───────┘
      │                │
      └───────┬────────┘
              │
              ▼
      ┌─────────────────┐
      │   Tool Result   │
      │  (sent to AI)   │
      └─────────────────┘
```

## Common Tool Configurations

### Read-Only Research Agent

```json5
{
  "id": "researcher",
  "tools": {
    "allow": ["read", "web.search", "web.fetch"],
    "deny": ["write", "edit", "exec", "browser"]
  }
}
```

### Full-Stack Developer Agent

```json5
{
  "id": "developer",
  "tools": {
    "groups": ["coding"],
    "allow": ["browser", "web.fetch"]
  },
  "sandbox": {
    "mode": "non-main",
    "workspaceAccess": "rw"
  }
}
```

### Customer Support Agent

```json5
{
  "id": "support",
  "tools": {
    "allow": [
      "message",
      "sessions.history",
      "sessions.list",
      "web.search"
    ],
    "deny": ["exec", "write", "edit"]
  }
}
```

### Automation Agent

```json5
{
  "id": "automator",
  "tools": {
    "allow": [
      "browser",
      "exec",
      "read",
      "write",
      "cron"
    ]
  },
  "sandbox": {
    "mode": "all",
    "docker": {
      "network": "bridge",  // Allow network for browser
      "memory": "1g"
    }
  }
}
```

## Tool Security

### Execution Isolation

```json5
{
  "sandbox": {
    "mode": "all",              // Sandbox all tool execution
    "workspaceAccess": "ro",    // Read-only workspace
    "docker": {
      "network": "none",        // No network
      "readOnlyRoot": true,     // Read-only filesystem
      "capDrop": ["ALL"]        // Drop all capabilities
    }
  }
}
```

### Tool Approval Flow

For sensitive tools, require approval:

```json5
{
  "tools": {
    "requireApproval": ["exec", "write"],
    "approvalTimeout": 300  // 5 minutes
  }
}
```

## Next Steps

- [Browser Integration](./browser-integration.md) - Headless browser setup
- [Multi-Tenancy](./multi-tenancy.md) - Tenant isolation
- [Container Deployment](./container-deployment.md) - Running in containers

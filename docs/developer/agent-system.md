# Agent System

This document explains how agents (bots) are defined, configured, and executed in Moltbot.

## What is an Agent?

An **agent** is a configured AI assistant with:
- A unique identifier
- Model configuration (Claude, GPT, etc.)
- Tool access policies
- Workspace directory
- Session settings

Think of agents as "personas" or "roles" - you can define multiple agents for different purposes.

## Defining Agents

Agents are defined in `~/.moltbot/config.json5`:

```json5
{
  "agents": {
    "defaults": {
      // Default settings applied to all agents
      "model": {
        "primary": "claude-opus-4-5",
        "fallbacks": ["claude-sonnet-4"]
      }
    },
    "entries": [
      {
        "id": "main",
        "default": true,          // This is the default agent
        "name": "Main Assistant",
        "workspace": "~/projects",
        "model": {
          "primary": "claude-opus-4-5"
        }
      },
      {
        "id": "code-reviewer",
        "name": "Code Reviewer",
        "workspace": "~/projects",
        "model": {
          "primary": "claude-sonnet-4"
        },
        "tools": {
          "allow": ["read", "exec"]  // Limited tool access
        }
      },
      {
        "id": "customer-support",
        "name": "Support Agent",
        "workspace": "~/support-docs",
        "model": {
          "primary": "claude-haiku-3-5"  // Faster, cheaper model
        },
        "tools": {
          "allow": ["web.search", "web.fetch"]
        }
      }
    ]
  }
}
```

## Agent Configuration Options

```typescript
type AgentConfig = {
  id: string;                      // Unique identifier (lowercase, alphanumeric)
  default?: boolean;               // Mark as default routing target
  name?: string;                   // Display name
  workspace?: string;              // Working directory for file operations
  agentDir?: string;               // Agent-specific library/data directory

  model?: {
    primary?: string;              // Primary model (e.g., "claude-opus-4-5")
    fallbacks?: string[];          // Fallback models on error
  };

  tools?: {
    allow?: string[];              // Allowed tools (whitelist)
    deny?: string[];               // Denied tools (blacklist)
  };

  sandbox?: {
    mode?: "off" | "non-main" | "all";  // Sandbox execution mode
    workspaceAccess?: "none" | "ro" | "rw";
    scope?: "session" | "agent" | "shared";
    docker?: { /* Docker container settings */ };
    browser?: { /* Browser sandbox settings */ };
  };

  identity?: {
    systemPrompt?: string;         // Custom system prompt
    // ... other identity settings
  };

  subagents?: {
    allowAgents?: string[];        // Agents that can be spawned as subagents
    model?: string | { primary?: string; fallbacks?: string[] };
  };
};
```

## Agent Execution Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agent Execution Flow                          │
└─────────────────────────────────────────────────────────────────────┘

  1. Request Received
         │
         ▼
  2. Resolve Agent Route
         │ - Check channel bindings
         │ - Check peer/group bindings
         │ - Fall back to default agent
         ▼
  3. Load Agent Configuration
         │ - Resolve workspace directory
         │ - Load model settings
         │ - Apply tool policies
         ▼
  4. Resolve Session
         │ - Find or create session
         │ - Load conversation history
         ▼
  5. Build Request Payload
         │ - Construct system prompt
         │ - Attach tools based on policy
         │ - Include conversation context
         ▼
  6. Execute Agent (pi-agent-core)
         │ - Call AI model API
         │ - Process tool calls
         │ - Handle streaming responses
         ▼
  7. Deliver Response
         │ - Route to originating channel
         │ - Update session state
         └─────────────────────────────
```

## Agent Routing

When a message arrives, Moltbot determines which agent handles it:

### Routing Priority (highest to lowest)

1. **Peer-level binding** - Specific DM/group/channel ID
2. **Guild-level binding** - Discord guild
3. **Team-level binding** - Teams team
4. **Account-level binding** - Channel account
5. **Default agent** - Fallback

### Configuring Bindings

```json5
{
  "routing": {
    "bindings": [
      {
        "agentId": "code-reviewer",
        "match": {
          "channel": "discord",
          "guildId": "123456789"  // All messages in this Discord server
        }
      },
      {
        "agentId": "customer-support",
        "match": {
          "channel": "telegram",
          "peer": {
            "kind": "group",
            "id": "-1001234567890"  // Specific Telegram group
          }
        }
      }
    ]
  }
}
```

## Agent Storage Isolation

Each agent has isolated storage:

```
~/.moltbot/agents/
├── main/                    # Default agent
│   ├── agent/               # Agent working directory
│   │   ├── .clawdbot/       # Agent-specific config
│   │   └── ...
│   └── sessions/            # Session transcripts
│       ├── sessions.json    # Session metadata
│       └── *.jsonl          # Conversation logs
├── code-reviewer/           # Another agent
│   ├── agent/
│   └── sessions/
└── customer-support/
    ├── agent/
    └── sessions/
```

## Multi-Agent Scenarios

### Scenario 1: Different Agents per Channel

```json5
{
  "agents": {
    "entries": [
      { "id": "work-assistant", "default": true },
      { "id": "personal-assistant" }
    ]
  },
  "routing": {
    "bindings": [
      {
        "agentId": "personal-assistant",
        "match": { "channel": "telegram" }
      }
      // work-assistant handles everything else
    ]
  }
}
```

### Scenario 2: Specialized Agents per Task

```json5
{
  "agents": {
    "entries": [
      {
        "id": "general",
        "default": true,
        "model": { "primary": "claude-sonnet-4" }
      },
      {
        "id": "coder",
        "model": { "primary": "claude-opus-4-5" },
        "tools": { "allow": ["read", "write", "edit", "exec"] }
      },
      {
        "id": "researcher",
        "model": { "primary": "claude-sonnet-4" },
        "tools": { "allow": ["web.search", "web.fetch"] }
      }
    ]
  }
}
```

### Scenario 3: Subagent Spawning

Agents can spawn other agents as subagents:

```json5
{
  "agents": {
    "entries": [
      {
        "id": "orchestrator",
        "default": true,
        "subagents": {
          "allowAgents": ["coder", "researcher"]  // Can spawn these
        }
      },
      {
        "id": "coder",
        "tools": { "allow": ["read", "write", "edit", "exec"] }
      },
      {
        "id": "researcher",
        "tools": { "allow": ["web.search", "web.fetch"] }
      }
    ]
  }
}
```

## Agent Tools

Agents have access to various tools (skills). Tool access is controlled via policies:

### Core Tools (from pi-coding-agent)
- `read` - Read files
- `write` - Create/overwrite files
- `edit` - Patch-based file editing
- `exec` - Run shell commands
- `process` - Interactive shell processes

### Moltbot Tools
- `browser` - Browser automation (Puppeteer)
- `canvas` - Graphics/UI rendering
- `message` - Send messages to channels
- `tts` - Text-to-speech
- `web.search` - Web search
- `web.fetch` - Fetch URLs
- `sessions.list` - List conversations
- `sessions.history` - Get conversation history
- `sessions.spawn` - Create subagent sessions
- `cron` - Schedule tasks
- `nodes` - Execute on connected devices

### Tool Policy Configuration

```json5
{
  "tools": {
    // Global policy (all agents)
    "allow": ["read", "web.search", "web.fetch"],
    "deny": ["exec"]  // Never allow exec by default
  },
  "agents": {
    "entries": [
      {
        "id": "coder",
        "tools": {
          "allow": ["exec"],  // Override: allow exec for this agent
          "groups": ["coding"]  // Enable entire tool groups
        }
      }
    ]
  }
}
```

## Running Specific Agents

### Via CLI
```bash
# Run specific agent
moltbot agent --agent-id coder --message "Review this code"

# Run default agent
moltbot agent --message "Hello"
```

### Via API/Gateway
```javascript
// WebSocket RPC
{
  "method": "agent",
  "params": {
    "message": "Review this code",
    "agentId": "coder",
    "sessionKey": "agent:coder:main"
  }
}
```

## Next Steps

- [Request Flow](./request-flow.md) - Detailed request processing
- [Skills and Tools](./skills-and-tools.md) - Tool configuration and custom skills
- [Container Deployment](./container-deployment.md) - Running agents in containers

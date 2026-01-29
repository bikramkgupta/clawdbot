# Request Flow

This document details how customer requests flow through the Moltbot system, from initial receipt to final response delivery.

## High-Level Request Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           REQUEST FLOW OVERVIEW                               │
└──────────────────────────────────────────────────────────────────────────────┘

  External Source                   Moltbot Gateway                  AI Provider
  ─────────────────                 ───────────────                  ───────────
        │                                 │                              │
        │  1. Incoming Message            │                              │
        │ ──────────────────────────────> │                              │
        │    (Telegram, Discord, etc.)    │                              │
        │                                 │                              │
        │                          2. Route to Agent                     │
        │                          3. Resolve Session                    │
        │                          4. Build Payload                      │
        │                                 │                              │
        │                                 │  5. API Request              │
        │                                 │ ────────────────────────────>│
        │                                 │                              │
        │                                 │  6. Streaming Response       │
        │                                 │ <────────────────────────────│
        │                                 │                              │
        │                          7. Process Tool Calls                 │
        │                          8. Execute Tools                      │
        │                                 │                              │
        │                                 │  9. Continue (if tools)      │
        │                                 │ ────────────────────────────>│
        │                                 │                              │
        │  10. Deliver Response           │                              │
        │ <────────────────────────────── │                              │
        │                                 │                              │
        │                          11. Update Session                    │
        │                                 │                              │
```

## Detailed Flow Stages

### Stage 1: Message Reception

Messages arrive via channel adapters:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Telegram   │    │   Discord   │    │    Slack    │
│   Webhook   │    │   Gateway   │    │ Socket Mode │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │    Channel Manager    │
              │  (server-channels.ts) │
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Message Context     │
              │   (MsgContext)        │
              └───────────────────────┘
```

**Message Context includes:**
```typescript
type MsgContext = {
  From: string;              // Sender identifier
  To: string;                // Recipient/channel
  SessionKey?: string;       // Multi-agent session key
  AccountId?: string;        // Multi-account provider ID
  Provider: string;          // Channel type (telegram, slack, etc.)
  Surface: string;           // Provider surface label
  ChatType: string;          // "dm" | "group" | "channel"
  OriginatingChannel: string; // For reply routing
  OriginatingTo: string;     // For reply routing
  // ... additional metadata
};
```

### Stage 2: Agent Routing

```typescript
// File: src/routing/resolve-route.ts
const route = resolveAgentRoute({
  cfg: config,
  channel: "telegram",
  accountId: "primary",
  peer: { kind: "dm", id: "123456" },
  guildId: null,
  teamId: null,
});

// Returns:
// {
//   agentId: "main",
//   sessionKey: "agent:main:telegram:primary:dm:123456",
//   mainSessionKey: "agent:main:main:primary",
//   matchedBy: "default"
// }
```

**Routing Resolution Order:**
1. Check peer-level bindings (DM/group/channel ID)
2. Check guild-level bindings (Discord)
3. Check team-level bindings (MS Teams)
4. Check account-level bindings
5. Fall back to default agent

### Stage 3: Session Resolution

```typescript
// File: src/commands/agent/session.ts
const session = resolveSessionKeyForRequest({
  cfg: config,
  sessionKey: route.sessionKey,
  agentId: route.agentId,
});

// Loads or creates session from:
// ~/.moltbot/agents/{agentId}/sessions/sessions.json
```

**Session Key Format:**
```
agent:{agentId}:{channel}:{accountId}:{chatType}:{scope}:{peerId}

Examples:
- agent:main:telegram:primary:dm:main:123456       # Telegram DM
- agent:main:discord:prod:group:guild-123:chan-456 # Discord channel
- agent:support:slack:workspace-1:group:C01234     # Slack channel
```

### Stage 4: Payload Building

```typescript
// File: src/agents/pi-embedded-runner/run.ts

const payload = {
  // System prompt with agent identity
  systemPrompt: buildSystemPrompt(agentConfig),

  // Conversation history
  messages: session.messageHistory,

  // Available tools based on policy
  tools: createMoltbotTools({
    cfg: config,
    agentId: route.agentId,
    sessionKey: route.sessionKey,
    // ... tool context
  }),

  // Model configuration
  model: agentConfig.model.primary,
  maxTokens: 4096,
};
```

### Stage 5-6: AI Provider Communication

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI PROVIDER REQUEST                           │
└─────────────────────────────────────────────────────────────────┘

  Gateway                          AI Provider API
     │                                   │
     │   POST /v1/messages               │
     │ ─────────────────────────────────>│
     │   {                               │
     │     model: "claude-opus-4-5",     │
     │     messages: [...],              │
     │     tools: [...],                 │
     │     stream: true                  │
     │   }                               │
     │                                   │
     │   SSE Stream                      │
     │ <─────────────────────────────────│
     │   event: content_block_delta      │
     │   event: tool_use                 │
     │   event: message_stop             │
     │                                   │
```

### Stage 7-8: Tool Execution

When the AI requests tool use:

```
┌─────────────────────────────────────────────────────────────────┐
│                      TOOL EXECUTION                              │
└─────────────────────────────────────────────────────────────────┘

  AI Response: tool_use
         │
         ▼
  ┌─────────────────────┐
  │  Tool Router        │
  │  (moltbot-tools.ts) │
  └──────────┬──────────┘
             │
     ┌───────┴───────┬───────────────┬─────────────────┐
     │               │               │                 │
     ▼               ▼               ▼                 ▼
┌─────────┐   ┌───────────┐   ┌───────────┐   ┌─────────────┐
│  read   │   │   exec    │   │  browser  │   │ web.search  │
│  write  │   │  (shell)  │   │(puppeteer)│   │ web.fetch   │
│  edit   │   │           │   │           │   │             │
└────┬────┘   └─────┬─────┘   └─────┬─────┘   └──────┬──────┘
     │              │               │                │
     │         ┌────┴────┐    ┌─────┴─────┐         │
     │         │ Sandbox │    │  Browser  │         │
     │         │Container│    │ Container │         │
     │         └─────────┘    └───────────┘         │
     │                                              │
     └──────────────────┬───────────────────────────┘
                        │
                        ▼
              Tool Result returned to AI
              (Continue conversation)
```

### Stage 9: Iterative Processing

The agent may make multiple tool calls before generating a final response:

```
┌──────────────────────────────────────────────────────────────────┐
│                    ITERATIVE AGENT LOOP                          │
└──────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐             │
  │  │ AI Call  │───>│ Tool Use │───>│ Tool     │────┐        │
  │  │          │    │          │    │ Result   │    │        │
  │  └──────────┘    └──────────┘    └──────────┘    │        │
  │       ▲                                          │        │
  │       │                                          │        │
  │       └──────────────────────────────────────────┘        │
  │                                                             │
  │  Loop continues until:                                     │
  │  - AI returns final text response (no tool use)            │
  │  - Max iterations reached                                  │
  │  - Error occurs                                            │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### Stage 10: Response Delivery

```typescript
// File: src/auto-reply/reply/route-reply.ts

await routeReply({
  payload: agentResponse,
  channel: ctx.OriginatingChannel,
  to: ctx.OriginatingTo,
  accountId: ctx.AccountId,
  threadId: ctx.ThreadId,
  cfg: config,
});
```

**Delivery path:**
```
Agent Response
      │
      ▼
┌─────────────────────┐
│   Normalize Reply   │
│  (add prefix, etc.) │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Outbound Delivery  │
│  (deliver.ts)       │
└──────────┬──────────┘
           │
     ┌─────┴─────┬─────────────┬─────────────┐
     │           │             │             │
     ▼           ▼             ▼             ▼
┌─────────┐ ┌─────────┐ ┌───────────┐ ┌─────────┐
│Telegram │ │ Discord │ │   Slack   │ │   Web   │
│  Bot    │ │   Bot   │ │    Bot    │ │   Chat  │
└─────────┘ └─────────┘ └───────────┘ └─────────┘
```

### Stage 11: Session Update

```typescript
// File: src/config/sessions/store.ts

await updateSessionStore(storePath, (store) => {
  const entry = store[sessionKey] || createNewSession();

  entry.messageHistory.push(
    { role: "user", content: userMessage },
    { role: "assistant", content: agentResponse }
  );

  entry.updatedAt = Date.now();

  return entry;
});

// Transcript appended to:
// ~/.moltbot/agents/{agentId}/sessions/{sessionKey}.jsonl
```

## Request Flow for Different Channels

### Telegram

```
Telegram API
     │
     │ Webhook POST
     ▼
┌─────────────────┐
│ Telegram Plugin │
│ (telegram/)     │
└────────┬────────┘
         │
         │ Extract: chatId, userId, text
         │ Build: MsgContext
         ▼
   Channel Manager
```

### Discord

```
Discord Gateway
     │
     │ WebSocket Event
     ▼
┌─────────────────┐
│ Discord Plugin  │
│ (discord/)      │
└────────┬────────┘
         │
         │ Extract: guildId, channelId, userId, content
         │ Build: MsgContext
         ▼
   Channel Manager
```

### Web Chat

```
Browser WebSocket
     │
     │ WS Message
     ▼
┌─────────────────┐
│ WebSocket Server│
│ (server-ws)     │
└────────┬────────┘
         │
         │ Extract: sessionId, message
         │ Build: MsgContext (Provider="web")
         ▼
   Agent Executor
```

## Error Handling and Fallbacks

```
┌─────────────────────────────────────────────────────────────────┐
│                    ERROR HANDLING FLOW                           │
└─────────────────────────────────────────────────────────────────┘

  Primary Model Request
         │
         ├─── Success ──────────────────────────────> Response
         │
         └─── Error (rate limit, auth, etc.)
                │
                ▼
         ┌──────────────────┐
         │ Check Fallbacks  │
         │ model.fallbacks  │
         └────────┬─────────┘
                  │
         ┌────────┴────────┐
         │                 │
   Has Fallback?     No Fallback
         │                 │
         ▼                 ▼
   Retry with         Return Error
   next model         to Channel
```

## Concurrent Request Handling

The gateway handles multiple concurrent requests using session lanes:

```
┌─────────────────────────────────────────────────────────────────┐
│                   CONCURRENT SESSIONS                            │
└─────────────────────────────────────────────────────────────────┘

  User A (Telegram)           User B (Discord)           User C (Web)
        │                           │                         │
        ▼                           ▼                         ▼
  ┌───────────┐             ┌───────────┐             ┌───────────┐
  │ Session A │             │ Session B │             │ Session C │
  │ Lane      │             │ Lane      │             │ Lane      │
  └─────┬─────┘             └─────┬─────┘             └─────┬─────┘
        │                         │                         │
        │   Independent           │   Independent           │
        │   Processing            │   Processing            │
        │                         │                         │
        ▼                         ▼                         ▼
  Agent Run A              Agent Run B              Agent Run C
  (may queue if            (parallel)              (parallel)
   same session)
```

## Next Steps

- [Multi-Tenancy](./multi-tenancy.md) - Tenant isolation patterns
- [Container Deployment](./container-deployment.md) - Scaling with containers
- [Skills and Tools](./skills-and-tools.md) - Tool configuration

# Architecture Diagrams

This document provides visual diagrams of the Moltbot architecture for quick reference.

## Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MOLTBOT COMPLETE ARCHITECTURE                          │
└─────────────────────────────────────────────────────────────────────────────────┘

  EXTERNAL CHANNELS                                              AI PROVIDERS
  ─────────────────                                              ────────────
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          ┌──────────────┐
  │ Telegram │ │ Discord  │ │  Slack   │ │ WhatsApp │          │  Anthropic   │
  │   Bot    │ │   Bot    │ │   App    │ │  Bridge  │          │   (Claude)   │
  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘          └──────────────┘
       │            │            │            │                  ┌──────────────┐
       │            │            │            │                  │    OpenAI    │
  ┌────┴────────────┴────────────┴────────────┴────┐            │   (GPT-4)    │
  │                Channel Manager                  │            └──────────────┘
  │          (per-channel plugin instances)         │            ┌──────────────┐
  └─────────────────────────┬───────────────────────┘            │   Google     │
                            │                                    │  (Gemini)    │
                            ▼                                    └──────────────┘
  ┌─────────────────────────────────────────────────────────────────────┐    │
  │                         GATEWAY SERVER                               │    │
  │  ┌───────────────────────────────────────────────────────────────┐  │    │
  │  │                    WebSocket + HTTP Server                     │  │    │
  │  │                       (Port 18789)                             │  │    │
  │  └───────────────────────────────────────────────────────────────┘  │    │
  │                                                                      │    │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐│    │
  │  │  Routing    │  │   Agent     │  │   Session   │  │    Tool     ││    │
  │  │  Engine     │  │  Executor   │  │   Manager   │  │   Runtime   ││    │
  │  │             │  │             │  │             │  │             ││    │
  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ ││    │
  │  │ │Bindings │ │  │ │ Agent A │ │  │ │Session 1│ │  │ │  read   │ ││    │
  │  │ │ Rules   │ │  │ │ Agent B │ │  │ │Session 2│ │  │ │  write  │ ││    │
  │  │ │         │ │  │ │ Agent C │ │  │ │Session 3│ │  │ │  exec   │ ││    │
  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │  │ │ browser │ ││    │
  │  └─────────────┘  └──────┬──────┘  └─────────────┘  │ └─────────┘ ││    │
  │                          │                          └──────┬──────┘│    │
  │                          │                                 │       │    │
  │                          └─────────────────────────────────┼───────┼────┘
  │                                    API Calls               │       │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐│
  │  │   Cron      │  │   Plugins   │  │      Browser Control       ││
  │  │  Service    │  │   Runtime   │  │    (Puppeteer/CDP)         ││
  │  └─────────────┘  └─────────────┘  └─────────────────────────────┘│
  └─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                          DATA STORAGE                               │
  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐ │
  │  │    Config       │  │    Sessions     │  │    Credentials      │ │
  │  │  config.json5   │  │  agents/*/      │  │   credentials/      │ │
  │  │                 │  │  sessions/*.jsonl│  │   oauth.json        │ │
  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘ │
  └─────────────────────────────────────────────────────────────────────┘
```

## Request Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          REQUEST PROCESSING FLOW                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

  User Message (e.g., Telegram)
         │
         ▼
  ┌──────────────────┐
  │ 1. RECEPTION     │  Channel adapter receives message
  │    Extract:      │  - From: user ID
  │    - sender      │  - To: chat/channel ID
  │    - content     │  - Content: message text
  │    - metadata    │  - Metadata: thread, reply-to, etc.
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 2. ROUTING       │  Determine which agent handles this
  │    Check:        │  Priority:
  │    - bindings    │  1. Peer binding (specific chat)
  │    - defaults    │  2. Guild/Team binding
  │                  │  3. Account binding
  │                  │  4. Default agent
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 3. SESSION       │  Find or create conversation
  │    Resolve:      │  Session key format:
  │    - session key │  agent:{id}:{channel}:{account}:{type}:{peer}
  │    - history     │
  │    - context     │  Load message history from store
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 4. PAYLOAD       │  Build AI API request
  │    Construct:    │  - System prompt (agent identity)
  │    - system      │  - Message history
  │    - messages    │  - Available tools (per policy)
  │    - tools       │  - Model configuration
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 5. EXECUTION     │  Call AI model API
  │    - API call    │  ┌─────────────────────────────┐
  │    - streaming   │  │   AI Model (Claude, etc.)   │
  │    - tool calls  │  └─────────────────────────────┘
  │                  │           │
  │                  │           ▼ (may loop)
  │    ┌─────────────┴───────────────────────┐
  │    │         TOOL EXECUTION LOOP         │
  │    │                                     │
  │    │  AI Response ──► Tool Call?         │
  │    │       │              │              │
  │    │       │         Yes: Execute        │
  │    │       │              │              │
  │    │       │         Return result       │
  │    │       │              │              │
  │    │       │         Continue ───────────┤
  │    │       │                             │
  │    │       └─── No: Final response ──────┘
  │    │                                     │
  │    └─────────────────────────────────────┘
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 6. DELIVERY      │  Route response back
  │    - format      │  - Apply response prefix
  │    - chunk       │  - Split long messages
  │    - send        │  - Deliver via original channel
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 7. PERSIST       │  Update session state
  │    - history     │  - Append to message history
  │    - metadata    │  - Update timestamps
  │    - transcript  │  - Write to transcript file
  └──────────────────┘
```

## Multi-Agent Deployment

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         MULTI-AGENT DEPLOYMENT                                   │
└─────────────────────────────────────────────────────────────────────────────────┘

  OPTION 1: Single Gateway, Multiple Agents
  ─────────────────────────────────────────
                    ┌───────────────────────────────────────┐
                    │            Gateway Server             │
                    │  ┌─────────────────────────────────┐  │
                    │  │         Agent Executor          │  │
                    │  │                                 │  │
                    │  │  ┌───────┐ ┌───────┐ ┌───────┐ │  │
                    │  │  │ main  │ │ coder │ │support│ │  │
                    │  │  └───────┘ └───────┘ └───────┘ │  │
                    │  │                                 │  │
                    │  └─────────────────────────────────┘  │
                    │                                       │
                    │  Routing: bindings → agent selection  │
                    └───────────────────────────────────────┘


  OPTION 2: Gateway per Agent (Container Isolation)
  ─────────────────────────────────────────────────
       ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
       │  Gateway #1     │  │  Gateway #2     │  │  Gateway #3     │
       │  Agent: main    │  │  Agent: coder   │  │  Agent: support │
       │  Port: 18789    │  │  Port: 18790    │  │  Port: 18791    │
       └─────────────────┘  └─────────────────┘  └─────────────────┘
              │                    │                    │
              └────────────────────┼────────────────────┘
                                   │
                          ┌────────▼────────┐
                          │  Load Balancer  │
                          │  (route by path)│
                          └─────────────────┘


  OPTION 3: Hybrid (Pools of Agents)
  ──────────────────────────────────
                         ┌─────────────────────┐
                         │    Load Balancer    │
                         └──────────┬──────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
   │  Gateway Pool   │     │  Gateway Pool   │     │  Gateway Pool   │
   │  Agents: A,B    │     │  Agents: C,D    │     │  Agent: E       │
   │  (Small tenants)│     │  (Medium)       │     │  (Large tenant) │
   │  Replicas: 2    │     │  Replicas: 3    │     │  Replicas: 5    │
   └─────────────────┘     └─────────────────┘     └─────────────────┘
```

## Tool Execution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          TOOL EXECUTION ARCHITECTURE                             │
└─────────────────────────────────────────────────────────────────────────────────┘

                              AI Model Response
                                    │
                                    │ tool_use: { name: "exec", input: {...} }
                                    ▼
                           ┌─────────────────┐
                           │   Tool Router   │
                           │ (moltbot-tools) │
                           └────────┬────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
          ┌─────────┴─────────┐     │     ┌─────────┴─────────┐
          │  Policy Check     │     │     │  Sandbox Check    │
          │  - allow list     │     │     │  - mode: all/off  │
          │  - deny list      │     │     │  - scope: session │
          │  - groups         │     │     │                   │
          └─────────┬─────────┘     │     └─────────┬─────────┘
                    │               │               │
                    │   Allowed?    │    Sandboxed? │
                    │               │               │
              ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
              │           │   │           │   │           │
              ▼           ▼   ▼           ▼   ▼           ▼
          ┌───────┐   ┌───────┐       ┌───────────────────────┐
          │ Deny  │   │ Host  │       │    Sandbox Container  │
          │ Error │   │ Exec  │       │                       │
          └───────┘   └───────┘       │  ┌─────────────────┐  │
                          │           │  │ Tool Execution  │  │
                          │           │  │ (isolated)      │  │
                          │           │  └─────────────────┘  │
                          │           │                       │
                          │           │  Network: none        │
                          │           │  FS: read-only        │
                          │           │  Resources: limited   │
                          │           └───────────────────────┘
                          │                     │
                          └──────────┬──────────┘
                                     │
                                     ▼
                            ┌─────────────────┐
                            │   Tool Result   │
                            │ (back to AI)    │
                            └─────────────────┘
```

## Session Isolation Model

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           SESSION ISOLATION MODEL                                │
└─────────────────────────────────────────────────────────────────────────────────┘

  SESSION KEY STRUCTURE
  ─────────────────────
  agent:{agentId}:{channel}:{accountId}:{chatType}:{scope}:{peerId}

  Example: agent:main:telegram:primary:dm:per-peer:123456


  ISOLATION BOUNDARIES
  ────────────────────

  ~/.moltbot/
  └── agents/
      │
      ├── main/                           ◄── Agent Boundary
      │   ├── agent/                          (separate workspace)
      │   └── sessions/
      │       ├── sessions.json
      │       │
      │       ├── agent:main:telegram:...user1.jsonl  ◄── User Boundary
      │       ├── agent:main:telegram:...user2.jsonl      (separate history)
      │       ├── agent:main:discord:...user3.jsonl
      │       └── ...
      │
      ├── support/                        ◄── Another Agent
      │   ├── agent/
      │   └── sessions/
      │       ├── sessions.json
      │       ├── agent:support:...tenant-a.jsonl  ◄── Tenant Boundary
      │       ├── agent:support:...tenant-b.jsonl
      │       └── ...
      │
      └── tenant-acme/                    ◄── Dedicated Tenant Agent
          ├── agent/
          └── sessions/


  MULTI-TENANT PATTERNS
  ─────────────────────

  Pattern 1: Agent-per-Tenant
  ┌─────────────────────────────────────────┐
  │  Agent: tenant-acme    │  Agent: tenant-globex  │
  │  (complete isolation)  │  (complete isolation)  │
  └─────────────────────────────────────────┘

  Pattern 2: Shared Agent, User Isolation (Default)
  ┌─────────────────────────────────────────┐
  │              Agent: main                │
  │  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
  │  │ User A  │ │ User B  │ │ User C  │   │
  │  │ Session │ │ Session │ │ Session │   │
  │  └─────────┘ └─────────┘ └─────────┘   │
  └─────────────────────────────────────────┘

  Pattern 3: Gateway-per-Tenant
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Gateway #1   │  │ Gateway #2   │  │ Gateway #3   │
  │ Tenant: ACME │  │ Tenant: GLX  │  │ Tenant: INIT │
  │ (container)  │  │ (container)  │  │ (container)  │
  └──────────────┘  └──────────────┘  └──────────────┘
```

## Browser Container Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        BROWSER CONTAINER ARCHITECTURE                            │
└─────────────────────────────────────────────────────────────────────────────────┘

  OPTION 1: Embedded Browser
  ──────────────────────────
  ┌───────────────────────────────────────────────────┐
  │                Gateway Container                   │
  │  ┌─────────────────────────────────────────────┐  │
  │  │              Gateway Process                │  │
  │  │                                             │  │
  │  │  ┌───────────┐    ┌───────────────────────┐│  │
  │  │  │  Agent    │───>│  Browser Control      ││  │
  │  │  │  Executor │    │  (Puppeteer)          ││  │
  │  │  └───────────┘    │                       ││  │
  │  │                   │  ┌─────────────────┐  ││  │
  │  │                   │  │    Chromium     │  ││  │
  │  │                   │  │   (headless)    │  ││  │
  │  │                   │  └─────────────────┘  ││  │
  │  │                   └───────────────────────┘│  │
  │  └─────────────────────────────────────────────┘  │
  └───────────────────────────────────────────────────┘


  OPTION 2: Separate Browser Container
  ────────────────────────────────────
  ┌─────────────────────┐          ┌─────────────────────────────┐
  │  Gateway Container  │          │     Browser Container       │
  │  ┌───────────────┐  │          │  ┌───────────────────────┐  │
  │  │    Agent      │  │   CDP    │  │       Chromium        │  │
  │  │   Executor    │──┼─────────>│  │    (Port 9222)        │  │
  │  └───────────────┘  │          │  └───────────────────────┘  │
  │                     │          │                             │
  │  Port: 18789        │          │  ┌───────────────────────┐  │
  │                     │          │  │    VNC Server         │  │
  │                     │          │  │    (Port 5900)        │  │
  └─────────────────────┘          │  └───────────────────────┘  │
                                   │                             │
                                   │  ┌───────────────────────┐  │
                                   │  │    noVNC Web UI       │  │
                                   │  │    (Port 6080)        │  │
                                   │  └───────────────────────┘  │
                                   └─────────────────────────────┘


  OPTION 3: Browserless Service
  ─────────────────────────────
  ┌─────────────────────┐          ┌─────────────────────────────┐
  │  Gateway Container  │          │   Browserless.io (Cloud)    │
  │  ┌───────────────┐  │   WSS    │  ┌───────────────────────┐  │
  │  │    Agent      │  │─────────>│  │  Chrome Pool          │  │
  │  │   Executor    │  │   CDP    │  │  (managed, scaled)    │  │
  │  └───────────────┘  │          │  └───────────────────────┘  │
  └─────────────────────┘          └─────────────────────────────┘
```

## Configuration Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          CONFIGURATION HIERARCHY                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

  PRIORITY (highest to lowest)
  ────────────────────────────

  1. Environment Variables          CLAWDBOT_* / ANTHROPIC_API_KEY
         │
         ▼
  2. CLI Flags                      --bind lan --port 18790
         │
         ▼
  3. Config File                    ~/.moltbot/config.json5
         │
         ├── Platform Override      ~/.moltbot/config.darwin.json5
         │                          ~/.moltbot/config.linux.json5
         │
         ├── Include Files          "include": ["./channels.json5"]
         │
         └── Env Substitution       "${ANTHROPIC_API_KEY}"
         │
         ▼
  4. Built-in Defaults              DEFAULT_PROVIDER = "anthropic"
                                    DEFAULT_MODEL = "claude-opus-4-5"


  CONFIG FILE STRUCTURE
  ─────────────────────

  {
    "agents": {
      "defaults": { ... },        // Applied to all agents
      "entries": [
        { "id": "main", ... },    // Agent-specific
        { "id": "coder", ... }
      ]
    },
    "routing": {
      "bindings": [ ... ]         // Channel → Agent mappings
    },
    "tools": {
      "allow": [ ... ],           // Global tool policy
      "deny": [ ... ]
    },
    "browser": { ... },           // Browser configuration
    "sandbox": { ... },           // Sandbox settings
    "gateway": { ... },           // Server settings
    "session": { ... },           // Session settings
    "channels": { ... }           // Per-channel config
  }
```

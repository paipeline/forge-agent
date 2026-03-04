# Forge — AI Agent Orchestration CLI

## Vision

One command to split a task into subtasks, spawn multiple AI coding agents in parallel across git worktrees, auto-monitor progress, auto-retry failures, and generate PRs.

```
forge run "Add user authentication with JWT"
→ split subtasks → create worktrees → spawn agents → monitor → retry → PR
```

## Architecture

```
forge/
├── src/
│   ├── cli/                    # CLI entry & command definitions
│   │   ├── index.ts            # Main entry, Commander.js
│   │   ├── commands/
│   │   │   ├── run.ts          # forge run <task>
│   │   │   ├── status.ts       # forge status
│   │   │   ├── stop.ts         # forge stop [agent-id]
│   │   │   ├── context.ts      # forge context add/list
│   │   │   └── init.ts         # forge init
│   │   └── ui/
│   │       └── dashboard.ts    # Ink TUI real-time panel
│   │
│   ├── core/                   # Core orchestration engine
│   │   ├── orchestrator.ts     # Task orchestrator (OpenClaw-aware)
│   │   ├── task-planner.ts     # LLM task splitting
│   │   ├── agent-manager.ts    # Agent lifecycle management
│   │   └── monitor.ts          # Monitor loop + retry
│   │
│   ├── agents/                 # Agent adapters
│   │   ├── base.ts             # Agent abstract interface
│   │   ├── claude-code.ts      # Claude Code adapter
│   │   ├── codex.ts            # Codex adapter
│   │   ├── openclaw.ts         # OpenClaw Pi agent adapter
│   │   └── custom.ts           # User-defined agent
│   │
│   ├── integrations/           # Platform integrations
│   │   └── openclaw/
│   │       ├── gateway.ts      # OpenClaw Gateway client
│   │       ├── sessions.ts     # Session management via Gateway
│   │       └── channels.ts     # Channel routing for notifications
│   │
│   ├── context/                # Context management
│   │   ├── store.ts            # Context storage
│   │   ├── prompt-builder.ts   # Precise prompt generation
│   │   └── sources/
│   │       ├── markdown.ts
│   │       └── git-history.ts
│   │
│   ├── workspace/              # Workspace management
│   │   ├── worktree.ts         # Git worktree
│   │   ├── tmux.ts             # tmux session
│   │   └── pr.ts               # PR creation (gh CLI)
│   │
│   ├── retry/                  # Ralph Loop smart retry
│   │   ├── analyzer.ts         # Failure analysis
│   │   └── prompt-rewriter.ts  # Prompt rewriting
│   │
│   └── config/
│       ├── schema.ts
│       └── defaults.ts
│
├── package.json
├── tsconfig.json
├── tsup.config.ts
└── README.md
```

## Implementation Phases

### Phase 1: Project Scaffolding + CLI Skeleton
- npm init, TypeScript + tsup + ESLint
- Commander.js CLI command structure
- `forge init` generates `.forge/config.yaml`
- `forge --version` / `forge --help`
- Publish as `@forge-agent/cli`

### Phase 2: Agent Adapters + Worktree Management
- Agent interface: `spawn()`, `getStatus()`, `stop()`, `getOutput()`
- Claude Code adapter (subprocess calling `claude` CLI)
- Codex adapter (subprocess calling `codex` CLI)
- Git worktree auto-create/cleanup
- tmux session management

### Phase 3: Task Orchestration Core + OpenClaw Integration
- LLM task splitting (Anthropic SDK)
- Orchestrator assigns subtasks to agents
- **OpenClaw Gateway integration** (see below)
- `forge run "task"` end-to-end flow

### Phase 4: Monitoring + Smart Retry
- Monitor loop: check agent/CI/PR status
- Failure analysis + prompt rewrite retry (Ralph Loop)
- `forge status` real-time panel

### Phase 5: Context Management
- `forge context add <file>`
- Prompt Builder extracts relevant context per task

### Phase 6: PR + Review
- Auto-create PR
- Multi-model review

---

## OpenClaw Integration Design

### Why OpenClaw?

OpenClaw is a personal AI assistant platform with:
- **Gateway**: WebSocket control plane for sessions, channels, tools, and events
- **Pi Agent Runtime**: RPC mode with tool streaming and block streaming
- **Multi-channel Inbox**: WhatsApp, Telegram, Slack, Discord, etc.
- **Subagent System**: spawn/steer/abort child agents with depth limits
- **Skills System**: bundled + managed + workspace skills

Forge can leverage OpenClaw as:
1. **An agent backend** — use OpenClaw's Pi agent runtime instead of raw CLI subprocesses
2. **A notification channel** — send progress/completion/failure alerts to user's preferred channel
3. **A session manager** — each Forge subtask maps to an OpenClaw session with full history

### Integration Points

#### 1. OpenClaw as Agent Backend (`src/agents/openclaw.ts`)

```typescript
interface OpenClawAgentAdapter extends AgentAdapter {
  // Use OpenClaw's gateway API to spawn a Pi agent session
  spawn(task: SubTask): Promise<AgentInstance>

  // Monitor via gateway WebSocket events
  getStatus(): Promise<AgentStatus>

  // Steer/abort via subagent registry
  stop(): Promise<void>

  // Read session transcript
  getOutput(): Promise<string>
}
```

How it works:
- Forge calls OpenClaw Gateway's session API to create a new agent session
- The session gets a workspace (worktree path), tools, and system prompt
- OpenClaw's Pi runtime executes the coding task with its full tool suite
- Forge monitors via Gateway WebSocket connection (session events)

#### 2. Gateway Client (`src/integrations/openclaw/gateway.ts`)

```typescript
class OpenClawGateway {
  constructor(config: { port: number; token?: string })

  // Session management
  createSession(opts: SessionOpts): Promise<Session>
  getSession(id: string): Promise<SessionState>
  listSessions(): Promise<Session[]>

  // Agent control
  sendMessage(sessionId: string, message: string): Promise<void>
  steerSession(sessionId: string, instruction: string): Promise<void>
  abortSession(sessionId: string): Promise<void>

  // Real-time events
  subscribe(handler: (event: GatewayEvent) => void): void

  // Notifications
  notifyChannel(channel: string, message: string): Promise<void>
}
```

#### 3. Notification via Channels (`src/integrations/openclaw/channels.ts`)

When Forge tasks complete/fail, notify the user through their configured OpenClaw channels:

```typescript
// In orchestrator, after task completes:
if (config.openclaw?.notify) {
  await gateway.notifyChannel(config.openclaw.notify.channel, {
    type: 'task_complete',
    task: subtask.name,
    pr: prUrl,
    duration: elapsed
  })
}
```

#### 4. Session-per-Subtask Model

```
forge run "Add JWT auth"
  ├── OpenClaw Session: "jwt-middleware"     → worktree: feature/jwt-middleware
  ├── OpenClaw Session: "jwt-refresh-token"  → worktree: feature/jwt-refresh
  └── OpenClaw Session: "jwt-tests"          → worktree: feature/jwt-tests
```

Each session:
- Has isolated workspace (git worktree)
- Uses OpenClaw's full tool suite (bash, file edit, browser, etc.)
- Can be steered mid-flight if Forge's monitor detects issues
- Transcript is preserved for review

### Config Example with OpenClaw

```yaml
agents:
  - name: claude-code
    command: claude
    args: ["--dangerously-skip-permissions"]
    max_instances: 3

  - name: openclaw
    type: openclaw
    gateway:
      port: 18789
      # token: auto-detected from openclaw config
    model: claude-sonnet-4-6
    max_instances: 3
    workspace_skills:
      - coding-agent
    tools:
      - bash
      - file-edit
      - browser

monitor:
  interval: 60
  max_retries: 3

openclaw:
  notify:
    channel: telegram  # or slack, discord, whatsapp
    events: [complete, failed, retry]

context:
  sources:
    - ./docs/**/*.md
    - ./.forge/context/**

review:
  enabled: true
  reviewers:
    - model: claude-sonnet-4-6
      focus: "logic and edge cases"
    - model: gemini-2.5-pro
      focus: "security and scalability"
```

### Advantages of OpenClaw Integration

| Feature | Raw CLI subprocess | OpenClaw Gateway |
|---|---|---|
| Agent control | PID-based kill | Graceful steer/abort |
| Session history | Parse stdout | Structured transcript |
| Notifications | None | Any channel (Telegram, Slack, etc.) |
| Tool suite | CLI-specific | Full OpenClaw tools (bash, browser, canvas) |
| Multi-model | Separate adapters | Single gateway, any model |
| Subagent depth | Manual | Built-in depth limits |
| Monitoring | Poll process | Real-time WebSocket events |

---

## Core User Flow

```bash
cd my-project
forge init                                          # Initialize
forge context add ./docs/architecture.md            # Add context
forge run "Add JWT auth with refresh token"         # Run
forge status                                        # Check status
forge stop agent-2                                  # Stop an agent
```

## Validation Criteria

1. `npx forge-agent init` generates config
2. `npx forge-agent run "Add hello world endpoint"` splits tasks, spawns agents, creates worktrees
3. `npx forge-agent status` shows real-time agent status
4. Failure triggers auto-retry
5. Completion creates PR
6. (With OpenClaw) Status updates sent to user's preferred channel

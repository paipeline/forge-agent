# OpenClaw Integration — Technical Notes

## OpenClaw Architecture (Reference)

OpenClaw is a personal AI assistant platform. Key components relevant to Forge:

### Gateway (`src/gateway/`)
- WebSocket control plane: sessions, presence, config, cron, webhooks
- REST + WS API on configurable port (default 18789)
- Manages all agent sessions and channel routing

### Pi Agent Runtime (`src/agents/pi-embedded-runner.ts`)
- RPC-mode agent with tool streaming and block streaming
- Supports: bash, file edit, browser, canvas, and custom tools
- Model-agnostic: Anthropic, OpenAI, Google, local models
- Built-in compaction, context window management, auth profile rotation

### Subagent System (`src/agents/subagent-*.ts`)
- `subagent-spawn.ts` — spawn child agents with isolated sessions
- `subagent-registry.ts` — track, steer, abort running subagents
- Depth limits prevent infinite recursion
- Announce/completion hooks for lifecycle events

### Session Model (`src/agents/sessions-*.ts`, `src/sessions/`)
- Session types: main, group, subagent
- Queue modes for message ordering
- Full transcript persistence
- Session write locks for concurrency

### Skills System (`skills/`, `src/agents/skills.ts`)
- `coding-agent` skill — pre-built coding agent configuration
- Workspace skills loaded from `.openclaw/skills/`
- Skill creator for building custom skills

### Channel System (`src/channels/`)
- Adapters: WhatsApp, Telegram, Slack, Discord, Signal, iMessage, etc.
- Unified message format across channels
- DM pairing for security

## Integration Strategy

### Phase 3 Integration (Orchestrator)

The Orchestrator should support two modes:

1. **CLI Mode** (default) — spawn agents as subprocesses (claude, codex, etc.)
2. **OpenClaw Mode** — use Gateway API to create sessions

Detection:
```typescript
async function detectOpenClaw(): Promise<OpenClawConfig | null> {
  // 1. Check if openclaw gateway is running
  try {
    const resp = await fetch(`http://localhost:18789/health`)
    if (resp.ok) return { port: 18789, detected: true }
  } catch {}

  // 2. Check openclaw config for custom port
  const configPath = path.join(os.homedir(), '.openclaw', 'config.yaml')
  // ... parse and return
  return null
}
```

### Gateway API Usage

Key endpoints Forge needs:

```
POST   /sessions          — create new agent session
GET    /sessions/:id      — get session state
DELETE /sessions/:id      — abort session
POST   /sessions/:id/send — send message to session
WS     /ws                — real-time events (session updates, completions)
```

### Session Creation for Subtask

```typescript
const session = await gateway.createSession({
  name: `forge-${taskId}-${subtask.slug}`,
  model: config.agents.openclaw.model,
  workspace: worktreePath,
  systemPrompt: promptBuilder.build(subtask),
  tools: ['bash', 'file-edit', 'glob', 'grep'],
  skills: ['coding-agent'],
  message: subtask.prompt
})
```

### Event Monitoring

```typescript
gateway.subscribe((event) => {
  switch (event.type) {
    case 'session.message.end':
      // Agent finished a turn
      monitor.updateProgress(event.sessionId)
      break
    case 'session.complete':
      // Agent finished task
      orchestrator.onSubtaskComplete(event.sessionId)
      break
    case 'session.error':
      // Agent hit an error
      retryManager.analyze(event.sessionId, event.error)
      break
  }
})
```

### Notification Example

```typescript
// After all subtasks complete:
await gateway.notifyChannel('telegram', {
  text: [
    `Forge: Task completed`,
    `"Add JWT auth with refresh token"`,
    ``,
    `3/3 subtasks done`,
    `PR: https://github.com/user/repo/pull/42`,
    `Duration: 12m 34s`
  ].join('\n')
})
```

## Open Questions

1. **Auth**: How does Forge authenticate with OpenClaw Gateway? Options:
   - Share the same config directory (`~/.openclaw/`)
   - Token-based auth via gateway API key
   - Auto-detect if running on same machine (localhost)

2. **Model routing**: Should Forge control which model OpenClaw uses per subtask, or let OpenClaw's own model selection handle it?

3. **Tool policy**: Should Forge restrict OpenClaw's tool set per subtask type (e.g., no browser for pure code tasks)?

4. **Concurrency**: OpenClaw's subagent system has depth limits. How does this interact with Forge spawning multiple sessions?

5. **Workspace isolation**: OpenClaw agents can access arbitrary paths. Need to enforce worktree isolation via `workspaceOnly` flag.

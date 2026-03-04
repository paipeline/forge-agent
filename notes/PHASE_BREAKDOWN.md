# Phase Breakdown — Detailed Implementation Steps

## Phase 1: Project Scaffolding + CLI Skeleton

### Tasks
- [ ] `npm init` with name `@forge-agent/cli`
- [ ] TypeScript config (`tsconfig.json`) — ESM, strict, Node 22 target
- [ ] tsup config (`tsup.config.ts`) — bundle to single ESM entry
- [ ] ESLint + Prettier setup
- [ ] Commander.js CLI with commands: `run`, `status`, `stop`, `context`, `init`
- [ ] `forge init` generates `.forge/config.yaml` with defaults
- [ ] `forge --version` / `forge --help`
- [ ] bin entry in package.json pointing to `dist/cli/index.js`
- [ ] Basic README with usage

### Output
```bash
forge --help
forge init
cat .forge/config.yaml  # sensible defaults
```

---

## Phase 2: Agent Adapters + Worktree Management

### Tasks
- [ ] `AgentAdapter` interface: `spawn()`, `getStatus()`, `stop()`, `getOutput()`
- [ ] `AgentInstance` type with id, status, worktree, logs
- [ ] Claude Code adapter — spawn `claude` as child process
- [ ] Codex adapter — spawn `codex` as child process
- [ ] `WorktreeManager` — create/list/cleanup git worktrees
- [ ] `TmuxManager` — create/list/kill tmux sessions
- [ ] Agent status enum: `pending`, `running`, `success`, `failed`, `stopped`

### Interface
```typescript
interface AgentAdapter {
  name: string
  spawn(task: SubTask, worktree: string): Promise<AgentInstance>
  getStatus(instance: AgentInstance): Promise<AgentStatus>
  stop(instance: AgentInstance): Promise<void>
  getOutput(instance: AgentInstance): Promise<string>
}
```

---

## Phase 3: Task Orchestration + OpenClaw Integration

### Tasks
- [ ] `TaskPlanner` — call LLM to split task into subtasks
  - Input: user task string + context
  - Output: `SubTask[]` with name, description, dependencies
- [ ] `Orchestrator` — main coordination loop
  - Create worktrees for each subtask
  - Spawn agents (respecting dependencies)
  - Track overall progress
- [ ] `forge run "task"` wires everything together
- [ ] **OpenClaw adapter** (`src/agents/openclaw.ts`)
  - Spawn via Gateway session API
  - Monitor via WebSocket events
  - Steer/abort via session control
- [ ] **Gateway client** (`src/integrations/openclaw/gateway.ts`)
  - HTTP + WebSocket client
  - Auto-detect running gateway
- [ ] **Channel notifications** (`src/integrations/openclaw/channels.ts`)
  - Send completion/failure notifications

### Orchestrator Flow
```
1. Parse user task
2. Build context (files, git history, user context)
3. Call TaskPlanner → SubTask[]
4. For each subtask:
   a. Create git worktree
   b. Select agent (claude-code | codex | openclaw)
   c. Build prompt with context
   d. Spawn agent in worktree
5. Monitor all agents
6. On failure → retry with rewritten prompt
7. On success → create PR
8. Notify user via OpenClaw channel (if configured)
```

---

## Phase 4: Monitoring + Smart Retry

### Tasks
- [ ] `Monitor` — polling loop
  - Check agent process status
  - Check CI status (if PR exists)
  - Detect stuck agents (timeout)
- [ ] `FailureAnalyzer` — analyze why agent failed
  - Parse error output
  - Classify: syntax error, test failure, timeout, resource issue
- [ ] `PromptRewriter` — rewrite prompt for retry
  - Include error context
  - Adjust approach based on failure type
- [ ] `forge status` — TUI dashboard (Ink)
  - Agent list with status indicators
  - Progress bars
  - Log streaming

### Ralph Loop
```
1. Agent fails
2. Analyzer reads output, classifies failure
3. PromptRewriter generates new prompt:
   - Original task + error context + "avoid X approach"
4. Respawn agent with new prompt
5. Repeat up to max_retries
```

---

## Phase 5: Context Management

### Tasks
- [ ] `ContextStore` — persist context sources in `.forge/context/`
- [ ] `forge context add <file|glob>` — register context source
- [ ] `forge context list` — show registered sources
- [ ] `PromptBuilder` — assemble prompt from:
  - Task description
  - Relevant context files (filtered by task)
  - Project structure overview
  - Git recent history
  - Coding conventions

---

## Phase 6: PR + Review

### Tasks
- [ ] `PRManager` — create PR via `gh` CLI
  - Title from subtask
  - Body with task description + changes summary
  - Link related PRs
- [ ] Multi-model review
  - Send diff to multiple models
  - Aggregate feedback
  - Post as PR comments
- [ ] Auto-merge option (with approval)

---

## Dependencies

```
Phase 1 (scaffold)
  └── Phase 2 (adapters + worktree)
       └── Phase 3 (orchestration + openclaw)
            ├── Phase 4 (monitoring + retry)
            ├── Phase 5 (context)
            └── Phase 6 (PR + review)
```

## Tech Stack

| Component | Choice | Reason |
|---|---|---|
| Language | TypeScript (ESM) | Match OpenClaw ecosystem |
| CLI framework | Commander.js | Lightweight, standard |
| TUI | Ink (React for CLI) | Rich real-time dashboards |
| Build | tsup | Fast, zero-config bundler |
| LLM SDK | @anthropic-ai/sdk | Task splitting, review |
| Process mgmt | Node child_process | Agent subprocess control |
| Git | simple-git | Worktree management |
| YAML | yaml | Config parsing |
| Runtime | Node >= 22 | Match OpenClaw requirement |

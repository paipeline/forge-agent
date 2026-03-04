# Forge

AI Agent orchestration CLI — spawn multiple coding agents in parallel, auto-monitor, auto-retry, auto-review.

```bash
forge run "Add user authentication with JWT"
# → split subtasks → create worktrees → spawn agents → monitor → retry → PR
```

## What it does

1. Takes a high-level task description
2. Uses LLM to split it into independent subtasks
3. Creates git worktrees for parallel development
4. Spawns multiple AI coding agents (Claude Code, Codex, OpenClaw, or custom)
5. Monitors progress, retries failures with rewritten prompts
6. Creates PRs when done

## Quick start

```bash
npm install -g @forge-agent/cli

cd my-project
forge init
forge run "Add JWT auth with refresh token"
forge status
```

## OpenClaw Integration

Forge integrates with [OpenClaw](https://github.com/openclaw/openclaw) as an agent backend and notification channel. When OpenClaw Gateway is running, Forge can:

- Use OpenClaw's Pi agent runtime for coding tasks
- Send progress notifications to Telegram, Slack, Discord, etc.
- Leverage OpenClaw's full tool suite (bash, browser, canvas)
- Monitor agents via real-time WebSocket events

## Status

Under development. See [notes/PLAN.md](notes/PLAN.md) for the full implementation plan.

## License

MIT

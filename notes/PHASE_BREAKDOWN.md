# 阶段拆解 — 详细实现步骤

## Phase 1：项目脚手架 + CLI 骨架

### 任务
- [ ] `npm init`，包名 `@forge-agent/cli`
- [ ] TypeScript 配置（`tsconfig.json`）— ESM、strict、Node 22 目标
- [ ] tsup 配置（`tsup.config.ts`）— 打包为单一 ESM 入口
- [ ] ESLint + Prettier 配置
- [ ] Commander.js CLI，命令：`run`、`status`、`stop`、`context`、`init`
- [ ] `forge init` 生成带默认值的 `.forge/config.yaml`
- [ ] `forge --version` / `forge --help`
- [ ] package.json 中 bin 入口指向 `dist/cli/index.js`
- [ ] 基础 README 含用法说明

### 预期输出
```bash
forge --help
forge init
cat .forge/config.yaml  # 合理的默认值
```

---

## Phase 2：Gateway 客户端 + PM 会话

### 任务
- [ ] `GatewayClient` — HTTP + WebSocket 连接 OpenClaw Gateway
  - `connect()` — 建立连接，健康检查
  - `createSession()` — 创建 PM 会话
  - `sendMessage()` — 向会话发送消息
  - `subscribe()` — 订阅实时事件
  - `close()` — 断开连接
- [ ] 自动检测 Gateway：检查 localhost:18789，读 `~/.openclaw/` 配置
- [ ] PM 会话管理（`pm-session.ts`）
  - 创建 PM 会话，注入系统提示词
  - 解析 PM 的结构化输出（方案、任务分配、验收结果）
- [ ] PM ↔ Forge 通信协议定义（`protocol.ts`）
  - `PMAction` 类型：plan_ready, assign_task, accept_task, retry_task, request_pr, done
  - `ForgeEvent` 类型：worker_started, worker_output, worker_done, worker_failed
- [ ] PM 系统提示词（`system-prompt.ts`）
- [ ] `forge run` 基础流程：连接 → 发送任务 → 接收方案 → 打印方案

### 接口定义
```typescript
class GatewayClient {
  constructor(config: GatewayConfig)
  connect(): Promise<void>
  createPMSession(task: string, context: ProjectContext): Promise<PMSession>
  subscribe(handler: (event: GatewayEvent) => void): void
  close(): Promise<void>
}

class PMSession {
  readonly id: string
  sendTask(task: string): Promise<void>
  onAction(handler: (action: PMAction) => void): void
  feedEvent(event: ForgeEvent): Promise<void>
}
```

---

## Phase 3：Worker 管理 + 任务执行

### 任务
- [ ] `WorkerAdapter` 接口：`spawn()`、`getStatus()`、`stop()`、`getOutput()`
- [ ] Claude Code 适配器 — 作为子进程启动 `claude`
- [ ] Codex 适配器 — 作为子进程启动 `codex`
- [ ] OpenClaw 子会话适配器 — 通过 Gateway 创建 worker 会话
- [ ] `WorkerPool` — 并发控制，管理活跃 worker 数量
- [ ] `WorktreeManager` — 创建/列出/清理 git worktree
- [ ] `TmuxManager` — 创建/列出/终止 tmux session
- [ ] 依赖调度：按 subtask.dependencies 决定执行顺序
- [ ] `forge run` 完整流程：连接 PM → 方案 → 创建 worktree → 启动 worker

### Worker 接口
```typescript
interface WorkerAdapter {
  name: string
  type: 'cli' | 'openclaw'
  spawn(task: SubTask, worktree: string): Promise<WorkerInstance>
  getStatus(instance: WorkerInstance): Promise<WorkerStatus>
  stop(instance: WorkerInstance): Promise<void>
  getOutput(instance: WorkerInstance): Promise<string>
}

interface WorkerInstance {
  id: string
  taskId: string
  status: 'pending' | 'running' | 'done' | 'failed' | 'stopped'
  worktree: string
  startedAt: Date
}
```

### 依赖调度逻辑
```
子任务依赖图：
  auth-middleware (无依赖) → 立即启动
  refresh-token (依赖 auth-middleware) → 等待
  auth-tests (依赖 auth-middleware, refresh-token) → 等待

执行波次：
  Wave 1: [auth-middleware]           → 启动 1 个 worker
  Wave 2: [refresh-token]             → auth-middleware 完成后启动
  Wave 3: [auth-tests]                → refresh-token 完成后启动

如果无依赖关系：
  Wave 1: [task-a, task-b, task-c]    → 同时启动 3 个 worker
```

---

## Phase 4：PM 验收 + 智能重试

### 任务
- [ ] Worker 完成后通知 PM（发送 worker_done 事件）
- [ ] PM 验收流程：
  - 读取 worker 改动（git diff）
  - 运行测试命令
  - 逐条对照验收标准
  - 输出 accept_task(pass/fail)
- [ ] 失败处理：
  - PM 分析失败原因
  - PM 重写 prompt（retry_task）
  - Forge 重启 worker
  - 最多重试 max_retries 次
- [ ] `forge status` — TUI 仪表盘（Ink）
  - PM 状态：制定方案中 / 监控中 / 验收中
  - 子任务列表 + 状态
  - Worker 列表 + 实时日志
  - 重试计数

### 验收数据结构
```typescript
interface AcceptanceResult {
  taskId: string
  result: 'pass' | 'fail'
  checks: AcceptanceCheck[]
  testsPassed: boolean
  lintPassed: boolean
  failureReason?: string
  retryPrompt?: string        // 如果 fail，PM 给出的重试指令
}

interface AcceptanceCheck {
  criterion: string           // 验收标准文本
  passed: boolean
  evidence?: string           // 通过/失败的证据
}
```

---

## Phase 5：PR + 通知

### 任务
- [ ] 合并 worktree 改动到目标分支
- [ ] 通过 `gh` CLI 创建 PR
  - 标题：PM 生成
  - 正文：方案摘要 + 子任务列表 + 变更概述
- [ ] 多模型 review（可选）
  - 将 diff 发送给配置的 reviewer 模型
  - 聚合反馈，作为 PR 评论
- [ ] OpenClaw 渠道通知
  - 方案就绪通知
  - 子任务完成/失败通知
  - 最终结果通知（PR 链接）
- [ ] 清理：删除临时 worktree、关闭 tmux session

---

## Phase 6：上下文管理 + 优化

### 任务
- [ ] `forge context add <file|glob>` — 注册上下文源
- [ ] `forge context list` — 显示已注册的源
- [ ] 项目上下文自动收集：
  - package.json / tsconfig.json / Cargo.toml 等
  - README.md
  - 最近 git 历史
  - 注册的上下文文件
- [ ] PM 方案缓存：相似任务复用方案模板
- [ ] 断点续传：`forge run` 中断后恢复

---

## 依赖关系

```
Phase 1（脚手架）
  └── Phase 2（Gateway 客户端 + PM 会话）  ← 核心变化：先接 OpenClaw
       └── Phase 3（Worker 管理 + 任务执行）
            └── Phase 4（PM 验收 + 重试）
                 └── Phase 5（PR + 通知）
                      └── Phase 6（上下文 + 优化）
```

**关键变化：Phase 2 从 "Agent 适配器" 变成 "Gateway 客户端 + PM 会话"。**
先打通 OpenClaw 连接，再做 Worker 管理。

## 技术栈

| 组件 | 选择 | 原因 |
|---|---|---|
| 语言 | TypeScript (ESM) | 与 OpenClaw 生态一致 |
| CLI 框架 | Commander.js | 轻量、标准 |
| TUI | Ink (React for CLI) | 丰富的实时仪表盘 |
| 构建 | tsup | 快速、零配置打包器 |
| WebSocket | ws | Gateway 实时通信 |
| HTTP | undici / fetch | Gateway REST API |
| 进程管理 | Node child_process | CLI worker 子进程控制 |
| Git | simple-git | Worktree 管理 |
| YAML | yaml | 配置解析 |
| 运行时 | Node >= 22 | 与 OpenClaw 要求一致 |

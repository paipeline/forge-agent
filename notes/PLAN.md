# Forge — AI Agent 编排 CLI

## 愿景

一条命令，连接 OpenClaw 会话作为 PM（项目经理），由 PM 制定方案、拆分任务、分配给多个 coding agent 并行执行、监控进度、验收成果、生成 PR。

```
forge run "Add user authentication with JWT"
→ 连接 OpenClaw PM → 制定方案 → 拆分子任务 → 分配 agent → 监控 → 验收 → PR
```

**核心理念：Forge 是轻量 CLI 壳，OpenClaw 是 PM 大脑。**

---

## 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│  forge run "Add JWT auth"                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Forge CLI（轻量壳）                                    │  │
│  │  - 解析命令、管理配置                                    │  │
│  │  - 连接 OpenClaw Gateway                               │  │
│  │  - TUI 展示进度                                        │  │
│  │  - 进程管理（worktree、tmux）                           │  │
│  └───────────────┬───────────────────────────────────────┘  │
│                  │ WebSocket + REST                          │
│  ┌───────────────▼───────────────────────────────────────┐  │
│  │  OpenClaw Gateway                                      │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  PM 会话（Pi Agent）                              │  │  │
│  │  │  - 分析项目代码库                                  │  │  │
│  │  │  - 制定实施方案                                    │  │  │
│  │  │  - 拆分子任务 + 定义验收标准                        │  │  │
│  │  │  - 分配任务给 worker agent                         │  │  │
│  │  │  - 监控进度、steer 方向                            │  │  │
│  │  │  - 验收成果、决定是否重试                           │  │  │
│  │  │  - 汇总结果、创建 PR                               │  │  │
│  │  └──────┬──────────┬──────────┬───────────────────┘  │  │
│  │         │          │          │                        │  │
│  │    ┌────▼───┐ ┌────▼───┐ ┌───▼────┐                  │  │
│  │    │Worker 1│ │Worker 2│ │Worker 3│  ...              │  │
│  │    │claude  │ │codex   │ │openclaw│                   │  │
│  │    │code    │ │CLI     │ │session │                   │  │
│  │    └────────┘ └────────┘ └────────┘                   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 角色分工

| 层 | 职责 | 实现 |
|---|---|---|
| **Forge CLI** | 用户入口、配置、TUI、进程管理 | TypeScript CLI |
| **OpenClaw PM** | 方案制定、任务拆分、分配、验收 | OpenClaw Pi Agent 会话 |
| **Worker Agents** | 执行具体编码任务 | Claude Code / Codex / OpenClaw 子会话 |

---

## 目录结构

```
forge/
├── src/
│   ├── cli/                    # CLI 入口 & 命令定义
│   │   ├── index.ts            # 主入口，Commander.js
│   │   ├── commands/
│   │   │   ├── run.ts          # forge run <task>
│   │   │   ├── status.ts       # forge status
│   │   │   ├── stop.ts         # forge stop [agent-id]
│   │   │   ├── context.ts      # forge context add/list
│   │   │   └── init.ts         # forge init
│   │   └── ui/
│   │       └── dashboard.ts    # Ink TUI 实时面板
│   │
│   ├── gateway/                # OpenClaw Gateway 客户端
│   │   ├── client.ts           # HTTP + WebSocket 连接
│   │   ├── pm-session.ts       # PM 会话管理
│   │   ├── worker-session.ts   # Worker 会话管理
│   │   └── events.ts           # 事件类型 & 处理
│   │
│   ├── pm/                     # PM 协议层
│   │   ├── protocol.ts         # PM ↔ Forge 通信协议
│   │   ├── plan.ts             # 方案数据结构
│   │   ├── acceptance.ts       # 验收标准 & 结果
│   │   └── system-prompt.ts    # PM agent 的系统提示词
│   │
│   ├── workers/                # Worker 管理
│   │   ├── pool.ts             # Worker 池管理
│   │   ├── adapters/
│   │   │   ├── base.ts         # Worker 抽象接口
│   │   │   ├── claude-code.ts  # Claude Code（子进程）
│   │   │   ├── codex.ts        # Codex（子进程）
│   │   │   └── openclaw.ts     # OpenClaw 子会话
│   │   └── lifecycle.ts        # 生命周期管理
│   │
│   ├── workspace/              # 工作空间管理
│   │   ├── worktree.ts         # Git worktree
│   │   ├── tmux.ts             # tmux session
│   │   └── pr.ts               # PR 创建（gh CLI）
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

---

## 核心流程：`forge run` 详解

```
用户: forge run "Add JWT auth with refresh token"

Step 1: 连接 OpenClaw
  Forge → 检测 OpenClaw Gateway（localhost:18789）
  Forge → 创建 PM 会话，注入系统提示词 + 项目上下文

Step 2: PM 制定方案
  PM Agent → 扫描项目代码库（通过 bash/glob/grep 工具）
  PM Agent → 输出结构化方案：
  {
    "plan": "JWT 认证实现方案",
    "subtasks": [
      {
        "id": "auth-middleware",
        "name": "实现 JWT 中间件",
        "description": "创建 Express middleware，验证 JWT token...",
        "acceptance": [
          "middleware 能正确验证有效 token",
          "无效 token 返回 401",
          "过期 token 返回 401 并附带 token_expired 错误码"
        ],
        "dependencies": [],
        "estimated_complexity": "medium"
      },
      {
        "id": "refresh-token",
        "name": "实现 refresh token 机制",
        "description": "...",
        "acceptance": [...],
        "dependencies": ["auth-middleware"]
      },
      {
        "id": "auth-tests",
        "name": "编写认证测试",
        "acceptance": [...],
        "dependencies": ["auth-middleware", "refresh-token"]
      }
    ]
  }

Step 3: Forge 准备工作环境
  Forge → 为每个子任务创建 git worktree
  Forge → 创建 tmux session

Step 4: PM 分配任务
  PM Agent → 将子任务分配给可用的 worker agent
  Forge → 启动 worker（Claude Code / Codex / OpenClaw 子会话）
  Worker → 在各自 worktree 中执行编码

Step 5: PM 监控 + 验收
  PM Agent → 实时监控每个 worker 的进度
  Worker 完成 → PM Agent 检查验收标准：
    - 读取 worker 输出的代码
    - 运行测试
    - 对照验收标准逐条检查
  验收通过 → 标记子任务完成
  验收失败 → PM 分析原因，重写指令，重新分配

Step 6: 汇总 + PR
  所有子任务通过 → PM 合并所有 worktree 改动
  PM → 生成 PR 标题 + 描述
  Forge → 调用 gh CLI 创建 PR
  Forge → 通过 OpenClaw 渠道通知用户
```

---

## PM 会话设计

### PM 系统提示词（`src/pm/system-prompt.ts`）

PM agent 的系统提示词定义它的角色和行为：

```
你是 Forge PM，一个项目经理 AI agent。你的职责是：

1. **制定方案**：分析用户需求和项目代码库，制定实施方案
2. **拆分任务**：将方案拆分为可独立执行的子任务，每个子任务有明确的验收标准
3. **分配任务**：将子任务分配给可用的 worker agent，考虑依赖关系和并行度
4. **监控进度**：实时监控 worker 的执行状态
5. **验收成果**：worker 完成后，对照验收标准逐条检查
6. **处理失败**：分析失败原因，调整方案或重写指令重试
7. **汇总结果**：所有子任务通过后，合并改动并生成 PR

你必须通过结构化的 JSON 消息与 Forge CLI 通信。

可用的 worker agent：
{workers_config}

项目上下文：
{project_context}
```

### PM ↔ Forge 通信协议（`src/pm/protocol.ts`）

PM 通过结构化消息与 Forge 交互：

```typescript
// PM → Forge：请求执行动作
type PMAction =
  | { type: 'plan_ready'; plan: Plan }
  | { type: 'assign_task'; taskId: string; worker: string; prompt: string }
  | { type: 'check_worker'; taskId: string }
  | { type: 'accept_task'; taskId: string; result: 'pass' | 'fail'; reason?: string }
  | { type: 'retry_task'; taskId: string; newPrompt: string }
  | { type: 'request_pr'; title: string; body: string; branches: string[] }
  | { type: 'notify'; channel: string; message: string }
  | { type: 'done'; summary: string }

// Forge → PM：反馈状态
type ForgeEvent =
  | { type: 'worker_started'; taskId: string; workerId: string }
  | { type: 'worker_output'; taskId: string; output: string }
  | { type: 'worker_done'; taskId: string; exitCode: number; output: string }
  | { type: 'worker_failed'; taskId: string; error: string }
  | { type: 'pr_created'; url: string }
  | { type: 'user_input'; message: string }
```

---

## 配置示例（`.forge/config.yaml`）

```yaml
# OpenClaw 连接
openclaw:
  gateway:
    host: localhost
    port: 18789
    # token: 自动从 ~/.openclaw/ 检测
  pm:
    model: claude-sonnet-4-6       # PM 使用的模型
    thinking: high                  # 启用深度思考
    skills:
      - coding-agent               # 加载编码相关 skill
  notify:
    channel: telegram               # 完成后通知渠道
    events: [plan_ready, task_done, all_done, failed]

# Worker 配置
workers:
  - name: claude-code
    type: cli                       # 子进程模式
    command: claude
    args: ["--dangerously-skip-permissions"]
    max_instances: 3

  - name: codex
    type: cli
    command: codex
    max_instances: 2

  - name: openclaw-worker
    type: openclaw                  # OpenClaw 子会话模式
    model: claude-sonnet-4-6
    max_instances: 3
    tools: [bash, file-edit, glob, grep]

# 工作空间
workspace:
  worktree_dir: .forge/worktrees   # worktree 存放目录
  use_tmux: true                    # 是否使用 tmux

# 上下文
context:
  sources:
    - ./docs/**/*.md
    - ./.forge/context/**

# 验收
acceptance:
  run_tests: true                   # 验收时自动跑测试
  test_command: "npm test"
  lint_command: "npm run lint"

# Review
review:
  enabled: true
  reviewers:
    - model: claude-sonnet-4-6
      focus: "逻辑和边界情况"
    - model: gemini-2.5-pro
      focus: "安全性和可扩展性"
```

---

## 实现阶段

### Phase 1：项目脚手架 + CLI 骨架
- npm init，TypeScript + tsup + ESLint
- Commander.js CLI 命令结构
- `forge init` 生成 `.forge/config.yaml`
- `forge --version` / `forge --help`

### Phase 2：Gateway 客户端 + PM 会话
- OpenClaw Gateway HTTP + WebSocket 客户端
- 自动检测运行中的 Gateway
- 创建 PM 会话，注入系统提示词
- PM ↔ Forge 通信协议定义
- `forge run` 基础流程：连接 → 发送任务 → 接收方案

### Phase 3：Worker 管理 + 任务执行
- Worker 适配器接口
- Claude Code 适配器（子进程）
- Codex 适配器（子进程）
- OpenClaw 子会话适配器
- Git worktree 自动创建/清理
- tmux session 管理
- Worker 池：并发控制、依赖调度

### Phase 4：PM 验收 + 智能重试
- PM 验收流程：读取代码 → 跑测试 → 对照标准
- 失败分析 + prompt 重写
- 验收不通过时自动重试
- `forge status` TUI 仪表盘

### Phase 5：PR + 通知
- 自动合并 worktree 改动
- 通过 gh CLI 创建 PR
- 多模型 review（可选）
- OpenClaw 渠道通知

### Phase 6：上下文管理 + 优化
- `forge context add/list`
- Prompt Builder 智能上下文提取
- PM 方案缓存与复用

---

## 核心用户流程

```bash
# 初始化（确保 OpenClaw Gateway 已运行）
cd my-project
forge init

# 添加上下文
forge context add ./docs/architecture.md

# 运行 —— 一条命令搞定一切
forge run "Add JWT auth with refresh token"
# → 连接 OpenClaw PM
# → PM 扫描代码库，制定方案
# → PM 拆分 3 个子任务
# → Forge 创建 3 个 worktree
# → PM 分配给 3 个 worker agent
# → PM 监控进度
# → Worker 完成后 PM 验收
# → 验收失败的任务自动重试
# → 全部通过后创建 PR
# → Telegram 通知："PR #42 已创建"

# 查看状态
forge status

# 停止某个 worker
forge stop worker-2
```

---

## 验证标准

1. `forge init` 生成配置，检测 OpenClaw Gateway
2. `forge run "task"` 成功连接 OpenClaw，创建 PM 会话
3. PM 输出结构化方案 + 子任务
4. Worker agent 在独立 worktree 中执行编码
5. PM 验收 worker 成果，不通过则重试
6. 全部通过后自动创建 PR
7. 通过 OpenClaw 渠道发送通知

---

## 与旧方案的关键区别

| 维度 | 旧方案（Forge 自己编排） | 新方案（OpenClaw 做 PM） |
|---|---|---|
| 任务拆分 | Forge 内置 TaskPlanner，调 API | OpenClaw PM 会话，有完整工具访问 |
| 方案制定 | 纯 LLM prompt | PM 可以扫描代码库、跑命令、读文档 |
| 验收 | 无，只检查进程退出码 | PM 读代码、跑测试、逐条对照验收标准 |
| 重试 | 简单的 prompt 重写 | PM 分析根因，可能调整整个方案 |
| 灵活性 | 写死在 TypeScript 里 | PM 的行为完全由提示词定义，可随时调整 |
| 通知 | 需自己实现 | OpenClaw 原生支持所有渠道 |
| Forge 代码量 | 大（编排逻辑复杂） | 小（只做 CLI + 进程管理 + Gateway 客户端） |

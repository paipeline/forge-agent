# OpenClaw 集成 — 技术笔记

## 核心思路

**OpenClaw 不是 Forge 的一个 agent 后端选项，而是 Forge 的大脑。**

Forge CLI 是轻量壳，负责：用户交互、配置管理、进程管理（worktree/tmux）、TUI 展示。

OpenClaw PM 会话是核心，负责：理解需求、扫描代码库、制定方案、拆分任务、定义验收标准、分配 worker、监控进度、验收成果、决定重试、汇总 PR。

---

## OpenClaw 架构（与 Forge 相关的部分）

### Gateway（`src/gateway/`）
- WebSocket 控制面板：会话、在线状态、配置、定时任务、Webhook
- REST + WS API，可配置端口（默认 18789）
- 管理所有 agent 会话和渠道路由

### Pi Agent 运行时（`src/agents/pi-embedded-runner.ts`）
- RPC 模式的 agent，支持工具流和块流
- 支持：bash、文件编辑、浏览器、canvas 和自定义工具
- 模型无关：Anthropic、OpenAI、Google、本地模型
- 内置压缩、上下文窗口管理、认证配置文件轮换

### 子 Agent 系统（`src/agents/subagent-*.ts`）
- `subagent-spawn.ts` — 使用隔离会话创建子 agent
- `subagent-registry.ts` — 跟踪、引导、终止运行中的子 agent
- 深度限制防止无限递归
- 生命周期事件的通知/完成钩子

### 会话模型（`src/agents/sessions-*.ts`, `src/sessions/`）
- 会话类型：main、group、subagent
- 消息排序的队列模式
- 完整的会话记录持久化
- 会话写锁保证并发安全

### 渠道系统（`src/channels/`）
- 适配器：WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等
- 跨渠道统一消息格式

---

## 连接流程

### 一条命令连接

```bash
forge run "Add JWT auth"
```

内部流程：

```
1. Forge 读取 .forge/config.yaml → 获取 gateway 地址
2. Forge → GET http://localhost:18789/health → 确认 Gateway 在线
3. Forge → POST /sessions → 创建 PM 会话
   {
     name: "forge-pm-{timestamp}",
     model: "claude-sonnet-4-6",
     systemPrompt: PM_SYSTEM_PROMPT,
     tools: ["bash", "file-edit", "glob", "grep"],
     skills: ["coding-agent"],
     workspace: process.cwd()
   }
4. Forge → WS /ws → 订阅会话事件
5. Forge → POST /sessions/{pmId}/send → 发送用户任务
   "分析当前项目并为以下任务制定实施方案：Add JWT auth"
6. PM Agent 开始工作...
```

### 自动检测 Gateway

```typescript
async function connectGateway(config: ForgeConfig): Promise<GatewayClient> {
  // 优先使用配置
  const host = config.openclaw?.gateway?.host ?? 'localhost'
  const port = config.openclaw?.gateway?.port ?? 18789

  // 健康检查
  const health = await fetch(`http://${host}:${port}/health`)
  if (!health.ok) {
    // 尝试自动启动
    console.log('OpenClaw Gateway 未运行，尝试启动...')
    await exec('openclaw gateway --port 18789 &')
    await waitForHealthy(host, port, 10_000)
  }

  return new GatewayClient({ host, port })
}
```

---

## PM 会话详细设计

### PM 的工具集

PM 会话需要以下工具来完成它的职责：

| 工具 | 用途 |
|---|---|
| `bash` | 运行命令：查看项目结构、跑测试、检查 CI |
| `glob` | 搜索文件：找到相关源码 |
| `grep` | 搜索代码内容：理解现有实现 |
| `file-edit` | 读取文件内容（PM 主要读，不写） |

PM **不需要** 直接写代码 — 写代码是 worker 的事。

### PM 的决策循环

```
┌──────────────────────────────────────┐
│           PM 决策循环                 │
│                                      │
│  1. 收到任务                          │
│     ↓                                │
│  2. 扫描代码库（bash/glob/grep）       │
│     ↓                                │
│  3. 制定方案 → 输出 plan_ready        │
│     ↓                                │
│  4. 拆分子任务 + 验收标准              │
│     ↓                                │
│  5. 分配给 worker → assign_task       │
│     ↓                                │
│  6. 等待 worker 完成                  │
│     ↓                                │
│  7. 验收：读代码 + 跑测试              │
│     ├─ 通过 → accept_task(pass)       │
│     └─ 失败 → 分析原因                │
│              ├─ 可重试 → retry_task    │
│              └─ 需调整方案 → 回到 3    │
│     ↓                                │
│  8. 全部通过 → request_pr + done      │
└──────────────────────────────────────┘
```

### PM 输出的方案结构

```typescript
interface Plan {
  title: string               // "JWT 认证实现方案"
  summary: string             // 方案概述
  subtasks: SubTask[]
}

interface SubTask {
  id: string                  // "auth-middleware"
  name: string                // "实现 JWT 中间件"
  description: string         // 详细描述
  acceptance: string[]        // 验收标准列表
  dependencies: string[]      // 依赖的其他子任务 id
  complexity: 'low' | 'medium' | 'high'
  suggested_worker: string    // 建议使用的 worker 类型
}
```

### PM 验收流程

PM 验收不是简单的 "测试通过就行"，而是：

```
1. 读取 worker 修改的文件（git diff）
2. 检查代码质量：
   - 是否符合项目编码规范？
   - 是否有明显的 bug？
   - 是否正确处理了边界情况？
3. 运行测试：
   - 跑 test_command（npm test）
   - 跑 lint_command（npm run lint）
4. 对照验收标准逐条检查：
   - ✅ middleware 能正确验证有效 token
   - ✅ 无效 token 返回 401
   - ❌ 过期 token 没有附带 token_expired 错误码
5. 输出验收结果：
   - pass: 全部通过
   - fail: 附带失败原因 + 改进建议
```

---

## Worker 管理

### Worker 类型

| 类型 | 实现 | 优势 | 适用场景 |
|---|---|---|---|
| `cli` (claude-code) | 子进程 `claude` | 强大的编码能力 | 复杂编码任务 |
| `cli` (codex) | 子进程 `codex` | 快速、便宜 | 简单/重复任务 |
| `openclaw` | Gateway 子会话 | 完整工具集、可 steer | 需要浏览器/多工具 |

### Worker 生命周期

```
Forge 创建 worktree
  ↓
Forge 启动 worker（子进程或 OpenClaw 子会话）
  ↓
Worker 在 worktree 中执行编码
  ↓
Worker 完成 → Forge 通知 PM
  ↓
PM 验收
  ├─ 通过 → worktree 保留等待合并
  └─ 失败 → PM 决定重试或调整
```

---

## 通知设计

通过 OpenClaw 渠道系统，Forge 可以在关键节点通知用户：

| 事件 | 通知内容示例 |
|---|---|
| `plan_ready` | "PM 已制定方案：3 个子任务，预计 15 分钟" |
| `task_assigned` | "子任务 auth-middleware 已分配给 claude-code" |
| `task_done` | "子任务 auth-middleware 验收通过 ✅" |
| `task_failed` | "子任务 auth-tests 验收失败，正在重试（1/3）" |
| `all_done` | "全部完成！PR #42 已创建：github.com/..." |
| `error` | "PM 会话异常终止，请检查" |

---

## 待定问题

1. **认证**：Forge 如何与 OpenClaw Gateway 认证？
   - 推荐：共享 `~/.openclaw/` 配置，localhost 免认证

2. **PM 模型选择**：PM 需要强推理能力，建议用最强模型（Opus/Sonnet）

3. **Worker 数量限制**：如何平衡并发和资源消耗？
   - 默认 max_instances: 3，可配置

4. **PM 会话持久化**：`forge run` 中断后能否恢复？
   - OpenClaw 会话有持久化，可以实现断点续传

5. **多项目支持**：一个 Gateway 能否同时服务多个 Forge 项目？
   - 可以，每个 `forge run` 创建独立的 PM 会话

6. **成本控制**：PM + 多个 Worker 的 token 消耗如何控制？
   - PM 可用 Sonnet（性价比），Worker 可混用不同模型

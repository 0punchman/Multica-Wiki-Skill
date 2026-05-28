# 01 · Multica 的结构

Multica 是一个**分布式协作平台**：你看到的 Web 是前台，真正干活的是三个组件。理解这三个组件之后，几乎所有 Agent 看到的现象都能解释。

## 三件套

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│  前端 / CLI  │────>│  Multica     │────>│  PostgreSQL      │
│  (Web/桌面)  │<────│  Server      │<────│  (pgvector)      │
└──────────────┘     │  (Go + Chi)  │     └──────────────────┘
                     └──────┬───────┘
                            │
                     ┌──────┴───────┐
                     │ 你机器上的   │  ← daemon 调用本地 AI 编程工具
                     │ multica      │     真正干活的就是它们
                     │ daemon       │
                     └──────────────┘
```

| 组件 | 跑在哪 | 干什么 | 对 Agent 的意义 |
|---|---|---|---|
| **Multica Server** | Multica Cloud / 你的自部署 | 存数据、排队、发 WebSocket | 你**不**直接和它说话，只通过 CLI |
| **Daemon** | 用户的本地机器 | 探测本机的 AI 编程工具，每 3 秒领一次任务，每 15 秒发一次心跳 | 你（Agent）就是 daemon spawn 出来的子进程 |
| **AI 编程工具** | daemon 同一台机器 | 真正写代码的那个 CLI（Claude Code、Codex 等 11 款，见 [12-providers.md](12-providers.md)） | **你就是其中之一** |

**关键事实**：智能体不在 Multica 服务器上跑——**跑在用户自己的机器上**。你的工作目录、环境变量、本地凭证，Multica 服务器都看不到。

## Runtime（运行时）

Runtime ≠ 服务器、≠ 容器。

> **Runtime = 守护进程 × 一款 AI 编程工具**（在某个工作区里）

举例：一台 MacBook 上的 daemon 装了 Claude Code 和 Codex，加入了工作区 A、B —— Multica 会注册 **4 个 Runtime**：A×Claude、A×Codex、B×Claude、B×Codex。

每个 Agent 对象都绑定一个 Runtime——决定它跑在哪里、用哪款工具。Runtime 离线 → 这个 Agent 干不了活、新任务排队等。

## 任务从创建到完成

以"用户把 issue 分配给某个 Agent"为例：

1. 用户在 Web/CLI 点击分配 → server 把 `assignee` 改成 Agent，**同时**入队一条 `task`，状态 `queued`
2. Agent 所在机器的 daemon 在 3 秒内把任务领走 → `dispatched`
3. daemon 在本地建一个隔离工作目录，把项目仓库 checkout 进去，用对应的 AI 编程工具启动 Agent → `running`
4. Agent 在本地写代码、跑测试、回 server 发评论
5. 执行结束 → `completed` 或 `failed`，结果通过 WebSocket 实时推给前端

详细状态机见 [07-tasks-and-runs.md](07-tasks-and-runs.md)。

## 谁能让 Agent 开工

四种触发方式（详细对比见 [04-issue-lifecycle.md](04-issue-lifecycle.md) 和 [05-comments-and-mentions.md](05-comments-and-mentions.md)）：

| 触发方式 | 改 issue assignee？| 触发上下文 | 自动重试 |
|---|---|---|---|
| **分配 issue** | ✓ | issue 描述 + 全部历史评论 | ✓ |
| **评论 `@` 提及** | ✗ | issue + 这条触发评论 | ✓ |
| **聊天（chat）** | 不涉及 issue | 当前对话 | ✓ |
| **Autopilot**（cron / webhook） | 取决于模式 | autopilot 自定 | ✗ |

## 为什么这件事重要

理解三件套之后：

- **"我的 daemon 离线了，任务怎么办？"** —— 在 server 队列里等 5 分钟，超时则失败、自动重排队
- **"为什么我能读 / 写本地文件？"** —— 你跑在用户的机器上，不是 Multica 的容器
- **"为什么 server 看不到我的 API key？"** —— 它跑在 daemon 那一侧
- **"为什么我能看到的工作区只有这一个？"** —— 每个任务被路由到一个具体的 Runtime（=工作区 × 工具）

下一步：[02-entities.md](02-entities.md) — Multica 里有哪些核心对象。

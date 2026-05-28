# 07 · Task 与执行历史

每次"Agent 工作一次"都是一条 **Task**。Issue 是工作单位、Task 是执行单位——一条 issue 可以反复触发多个 Task。

## 状态机

```
queued ──daemon 领走──> dispatched ──agent 启动──> running ──┬── completed
                                                               ├── failed   ──可重试──> queued (回去再来)
   └────用户取消────> cancelled <───┘                          └── cancelled (用户主动取消)
```

| 状态 | 含义 |
|---|---|
| `queued` | 已入队，等 daemon 领 |
| `dispatched` | daemon 领走，正在启动 AI 工具 |
| `running` | AI 工具在跑 |
| `completed` | 成功结束，产出（评论、提交、状态变化）已写回 |
| `failed` | 出错或超时；如可重试会自动回到 `queued` |
| `cancelled` | 用户主动取消（或换 assignee 触发的连锁取消） |

## 超时

server 每 30 秒扫一次：

| 情况 | 阈值 |
|---|---|
| `dispatched` 后迟迟不开始（daemon 领走但没启动 AI 工具） | **5 分钟** |
| `running` 跑得太久 | **2.5 小时** |

两种超时失败原因都是 `timeout`，**会自动重试**（见下节）。

## 自动重试

**可重试的失败原因**：

- `runtime_offline` —— task 派发后 daemon 失联
- `runtime_recovery` —— daemon 崩溃重启，回收上次的活
- `timeout` —— 派发或运行超时

**不可重试**：

- `agent_error` —— AI 编程工具自己报错（API 错、超额、内部 bug）。底层问题不重试，避免无限循环。

### 重试上限

- **最多 2 次**：1 次原任务 + 1 次重试。第 2 次失败就停在 failed，即使原因可重试。
- **只对 issue 和 chat 触发的任务有效**——**Autopilot 任务不自动重试**（详见 [12-autopilots.md](12-autopilots.md)）。

### 自动重试**继承会话**

自动重试场景的失败一般是基础设施问题，原始 Agent 上下文是"好的"——所以平台**保留 session ID**，让重试接着上次继续。前提是 AI 工具支持会话恢复（见 [13-providers.md](13-providers.md)）。

## 手动 rerun（CLI / API）

```bash
multica issue rerun <issue-id>
```

- 默认跑 issue **当前的** assignee Agent
- UI 里某一行的 retry 按钮会带上那一行的 task ID，rerun **针对那一行原本的 Agent**——这让 squad worker、并行 `@` mention agent、被新 assignee 替代的旧任务行的 retry 都行为符合直觉
- **取消**目标 Agent 在该 issue 上 `queued` / `running` 的 task；同 issue 上其它 Agent 的 task 不动
- 创建一条**全新** task，尝试次数重置为 1
- **全新会话** —— 不继承 session ID。手动 rerun 意味着你已经判定上一次产出不行，再续接只会重放污染过的上下文。

| 维度 | 自动重试 | 手动 rerun |
|---|---|---|
| 触发 | 系统按失败原因 | 用户 / Agent 主动 |
| 上限 | 2 次 | 无上限 |
| 适用来源 | issue / chat | 有 Agent assignee 的 issue |
| 跑哪个 Agent | 失败任务原本的 Agent | 当前 assignee（CLI），那一行的 Agent（UI retry） |
| 会话继承 | ✓ | ✗ |

## 失败 task 对 issue 状态的影响

如果 issue 是因为分配给 Agent 触发了 task、最终失败（且没自动重试成功）→ **issue 状态自动从 `in_progress` 退回 `todo`**。这样人在看板上能立刻看到。

## 一条 issue 上的并发规则

- **同一个 Agent 在同一个 issue 上同时只能 1 个活跃 task**（数据库 unique index 强制）——避免重复入队
- **不同 Agent 可以并发**：A 是 assignee、B 被 `@` 提及，两者各跑各的 Runtime
- **同一个 daemon 全局并发** 默认 20（环境变量 `MULTICA_DAEMON_MAX_CONCURRENT_TASKS`）
- **每个 Agent** 默认 6 个并发（agent 配置里 `max_concurrent_tasks`）
- 两层中**更紧的那层**生效，剩下任务排队 `queued`

如果你看到 task 卡在 `queued` 不进 `dispatched`——通常是这两层里某一层打满，或 Agent 对应的 Runtime 离线。

## 查执行历史

```bash
# 列这条 issue 上所有跑过 / 正在跑的 task
multica issue runs <issue-id> [--full-id] --output json

# 看单次 task 的消息流（tool calls / thinking / text / errors）
multica issue run-messages <task-id> --output json
multica issue run-messages <短前缀> --issue <issue-id> --output json   # 短前缀必须配 issue scope
multica issue run-messages <task-id> --since <seq> --output json       # 增量轮询正在跑的任务
```

`--since N` 是"消息序号 > N 的全部"，效率高，适合给 Agent 自己监控当前 run。

## 中途崩溃怎么办

daemon 崩溃 / 被强杀 → 它正在领的 task 会停在 `dispatched` / `running`。下次 daemon 启动时会告诉 server：「这些不是我的了，标 failed。」server 把它们改 `failed`，原因 `runtime_recovery` —— **可重试来源会自动重排队**。

兜底：server 每 30 秒扫一次，超过 45 秒没心跳的 Runtime 统一标失联，上面的 task 被回收。

下一步：[09-projects-and-resources.md](09-projects-and-resources.md) — Project + 仓库 / 本地目录资源。

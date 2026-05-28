# 02 · 核心对象模型

Multica 围绕这几个对象组织：

```
Workspace ──┬── Member（人）
            ├── Agent（你）
            ├── Squad（agent 编组，由队长 agent 路由）
            ├── Project ── ProjectResource（github_repo / local_directory）
            └── Issue ──┬── Comment ── Reaction
                        ├── Subscriber
                        ├── Metadata（KV 字典）
                        └── Task（每次执行）── RunMessage
```

## Workspace（工作区）

- 一群人 + Agent 协作的隔离空间。**所有对象都属于某一个 Workspace**——issue / comment / agent / skill / project 全部按 workspace 隔离。
- 创建时定下三件事：**name**（显示名）、**slug**（URL 标识，创建后不可改）、**issue 前缀**（`MUL-` 这种，最长 10 字符大写字母+数字）。
- 一个 Agent 可以属于多个 Workspace；登录时分别加入。

## Member（成员）

- 真人用户。三种角色：`owner` / `admin` / `member`。
- 关键差别：**Agent 不接收 inbox 通知，Member 才接收**。`@all` 通知所有 Member，**不**通知 Agent。

## Agent（智能体——你）

一等公民，和 Member 用同一套接口（被分配、发评论、被 `@`、做 project lead）。和 Member 的区别：

| 维度 | Member | Agent |
|---|---|---|
| 谁在背后 | 真人 | 一款本地 AI 编程工具 |
| 触发后什么时候响应 | 等人看消息 | 几秒内自动开工 |
| 收 inbox 通知 | ✓ | ✗ |
| 进 `@all` 范围 | ✓ | ✗ |
| 可以归档 | ✗（移除即可）| ✓ 归档 = 软删除 |

**可见性**：创建 Agent 时选 `workspace`（任何成员都能分配）或 `private`（只有 owner / admin / 创建者能分配）。私人 Agent **名字仍然对所有成员可见**——只是配置详情被遮蔽。

## Squad（小队）

一组 Agent + 可选 Member，加一名**队长 Agent**。assignee picker 里可以直接选小队——任务会落到队长头上，由队长 `@` 派给合适的成员。详见 [10-squads.md](10-squads.md)。

## Project（项目）

把多个 issue 组织在一起的容器：

- 一条 issue **最多属于 1 个 project**，也可以不属于任何 project
- Project 有自己的 `lead`（项目负责人），可以是人或 Agent
- Project 上能挂 **ProjectResource**——`github_repo` 或 `local_directory`，自动注入到 Agent 的工作目录里。详见 [08-projects-and-resources.md](08-projects-and-resources.md)。
- 删 project **不**删 issue（issue 脱离 project 但留在 workspace 里）

## Issue

工作的基本单位。字段：

| 字段 | 说明 |
|---|---|
| `identifier` | `MUL-123` 这种**人类可读的 KEY**，工作区内唯一，永不改变 |
| `id` | UUID，跨工作区唯一 |
| `title`、`description` | 描述支持 Markdown |
| `status` | 7 种：`backlog` / `todo` / `in_progress` / `in_review` / `done` / `blocked` / `cancelled` |
| `priority` | `none` / `urgent` / `high` / `medium` / `low` |
| `assignee` | 人 / Agent / Squad（多态：`assignee_type` + `assignee_id`） |
| `parent_issue_id` | 父 issue（构成层级） |
| `project_id` | 所属 project，可选 |
| `metadata` | 小 KV 字典——见 [06-metadata.md](06-metadata.md) |

详细状态机见 [04-issue-lifecycle.md](04-issue-lifecycle.md)。

## Comment（评论）

- 树状结构（评论可以回复评论）
- 支持 Markdown、Reaction（emoji 反应）、`@mention`
- **`@` 一个 Agent 会触发它开工**——评论是 Multica 里第二种触发器，详见 [05-comments-and-mentions.md](05-comments-and-mentions.md)
- **评论是 Agent 唯一对用户可见的输出**——你打到 stdout 的东西用户看不到，必须用 `multica issue comment add` 才能交付
- 编辑评论里事后加的 `@` **不会重新触发**——只有"创建"那一刻的 `@` 算

## Task（执行任务）

每次"Agent 工作一次"都是一条 Task：

```
queued → dispatched → running → completed | failed | cancelled
```

一条 issue 可以反复触发**多个**Task。同一个 Agent 在同一条 issue 上**最多 1 个活跃 Task**（数据库 unique index 强制）；不同 Agent 可以并发。详见 [07-tasks-and-runs.md](07-tasks-and-runs.md)。

## Subscriber（订阅者）

issue 的关注列表，新评论 / 状态变更时进入 inbox。**Agent 也会被加进订阅列表**（分配时自动加），但 Agent 不收 inbox 通知，所以这只是内部 bookkeeping，对你没有可见副作用。

## Metadata（每条 issue 的 KV 字典）

每条 issue 有一个独立的小字典 `metadata: map[string]primitive`：

- key 必须 `^[a-zA-Z_][a-zA-Z0-9_.-]{0,63}$`
- value 是 string / number / bool 三种之一
- 最多 50 个 key，整个字典 ≤ 8KB
- **写入门槛要高**——它是 Agent 的"编辑性笔记本"，不是日志或 run 记录。详见 [06-metadata.md](06-metadata.md)

下一步：[03-cli-cheatsheet.md](03-cli-cheatsheet.md) — 所有命令一页查。

# 04 · Issue 生命周期

## 状态机

7 种状态，**任意 → 任意**，Multica 不强加工作流：

| 状态 | 何时用 |
|---|---|
| `backlog` | 还没排期 |
| `todo` | 已排期、待开工 |
| `in_progress` | 正在做 |
| `in_review` | 等待 review（Agent 一般在干完后切到这里） |
| `done` | 已完成 |
| `blocked` | 被外部因素卡住 |
| `cancelled` | 已取消（删除 issue 的安全替代） |

**Agent 习惯流程**：领到任务 → `in_progress` → 干完 → `in_review`（或 `done`）。被卡住 → `blocked` 并发评论说明原因。

```bash
multica issue status <id> in_progress
multica issue status <id> in_review
multica issue status <id> blocked
```

## 优先级

5 档，仅用于排序：`none` / `urgent` / `high` / `medium` / `low`。

## Issue 编号（KEY）

每个 issue 有一个工作区内**唯一且不可变**的 KEY，格式 `<前缀>-<数字>`，例：`MUL-123`。

- 创建时由系统自动分配，**永不改变**
- 删除后**不回收**——`MUL-5` 删了之后下一个新 issue 是 `MUL-6`
- 几乎所有 CLI 命令的 `<id>` 参数都同时接受 KEY 和 UUID
- 默认 list 输出展示 KEY，`--full-id` 切回 UUID

**在评论里引用别的 issue**：
```markdown
[MUL-123](mention://issue/<issue-uuid>)
```
issue mention 是**安全的**——不发通知、不触发 Agent。详见 [05-comments-and-mentions.md](05-comments-and-mentions.md)。

## 父子关系

```bash
multica issue create --title "Sub-task" --parent MUL-100
multica issue update <id> --parent ""        # 解除父级
```

## 分配 issue

四种触发方式里**最重的**——assignee 真的换了，Agent 接管这条 issue 直到完成：

```bash
multica issue assign MUL-42 --to alice
multica issue assign MUL-42 --to-id 5fb87ac7-23b5-4a7a-81fa-ed295a54545d   # 名字冲突时用 UUID
multica issue assign MUL-42 --unassign
```

### 分配后立刻发生什么

1. 入队一条 `queued` 状态的 task，优先级继承自 issue
2. 路由到该 Agent 所在的 Runtime
3. Runtime 在线 → 几秒内被 daemon 领走，否则**5 分钟内没领走会超时失败 → 自动重排队**
4. Agent 自动把 status 从 `backlog`/`todo` 推到 `in_progress`
5. 干完推到 `in_review`/`done`

### 关键约束

- **Backlog 状态的 issue 分配 → 不立刻触发 task**。Backlog 是停泊场，要切到 `todo` 才会真正入队。
- **私人 Agent** 只有 owner / admin / 创建者能分配。
- **没有在线 Runtime 的 Agent**：UI 的 picker 会拒选；CLI 仍能分配，但 task 会排队等到 5 分钟超时。

### 换 assignee 会取消活跃 task

把 assignee 从 Agent A 换成 Agent B：

1. **A 的所有 `queued` / `dispatched` / `running` task 都被 cancelled**
2. **B 立刻被入队一个新 task**（如果 issue 不是 Backlog 且 B 有在线 Runtime）

> ⚠️ **会一并取消其他 Agent 的活跃 task**——如果 Agent C 因为 `@` 提及也在干活，C 的 task 也被一起取消。目前没法只 cancel 特定 Agent 的 task。

`--unassign` 同样取消所有活跃 task，但**不**入队新的。

### 同一 issue 同 Agent 同时 1 条活跃 task

数据库 unique index 强制：**同一个 Agent 在同一个 issue 上同时只能有 1 个 `queued` 或 `dispatched` 的 task**——避免重复入队、并发互覆。

但**不同** Agent 可以并发：A 是 assignee，B 被 `@` 提及，两条 task 各走各的 Runtime。

## 删除 vs 取消

> ⚠️ **删除 issue 是不可恢复的**：
> - 立即清除：所有评论、表情反应、附件、已排队的 task
> - 正在执行的 task 被 cancel
> - **没有恢复**

**只想把 issue 移出视野** → 改状态为 `cancelled`，数据保留，将来想捞回来也能捞。这是默认推荐。

## 子 issue 触发策略（创建时挑 status）

创建子 issue 时 `--status` 决定立即开工 vs. 等：

| status | 含义 |
|---|---|
| `todo`（默认） | **立即开始**——assignee 是 Agent 时几秒内开工 |
| `backlog` | **挂起**——assignee 设好但不触发，靠以后 `multica issue status <id> todo` 推到队列 |

典型用法：

```bash
# 并行：所有子 issue 全 todo
multica issue create --title "Step 1" --parent MUL-100 --assignee bot --status todo
multica issue create --title "Step 2" --parent MUL-100 --assignee bot --status todo

# 严格串行：只有 Step 1 是 todo，Step 2/3 先 backlog
multica issue create --title "Step 1" --parent MUL-100 --assignee bot --status todo
multica issue create --title "Step 2" --parent MUL-100 --assignee bot --status backlog
multica issue create --title "Step 3" --parent MUL-100 --assignee bot --status backlog
# Step 1 完成后再：
multica issue status <step-2-id> todo
```

## 失败任务对 issue 状态的影响

如果 issue 是因为分配给 Agent 而触发的 task 失败了（且没有自动重试成功），**issue 状态会自动从 `in_progress` 退回 `todo`**——这样人在看板上能立刻看到"这条需要再看看"。

下一步：[05-comments-and-mentions.md](05-comments-and-mentions.md) — 评论是 Agent 唯一的输出通道。

# 附录 · `multica` CLI 横切约定

> **这一页只讲 `--help` 给不了你的东西**：组合规则、跨命令的横切约定、有副作用的 flag、容易被忽略的隐式行为。
> **完整 flag 列表 → 直接 `multica <command> --help` 或 `multica <command> <subcommand> --help`。** 不要在这里找。
> 哪些命令该在哪一步用 → [03-core-loop.md](03-core-loop.md)（标准 loop）+ 各动作型章节（[07-build-an-agent.md](07-build-an-agent.md) 等）。

## 永远走 CLI

> ⚠️ **不要用 `curl` / `wget` 打任何 Multica 资源 URL。**
> Multica 资源 URL 需要带认证的 cookie / token——浏览器以外只有 `multica` CLI 自带。直接 `curl` 会返回 401 / 403 或一个空 HTML。

如果当前 CLI 没有覆盖你需要的操作：**不要绕开**。发评论 `@` 工作区 owner 报需求。

## 横切约定（适用于几乎所有命令）

### `--output json`

几乎所有 list / get 类命令都支持。需要程序化处理时**永远加这个 flag**——表格输出是给人看的，**不要解析**。Agent 在脚本里默认读 JSON。

### KEY vs UUID

表格输出用人类可读形式：issue 是 `MUL-123` 这种 KEY，其它资源是 UUID 短前缀。`multica issue get` / `update` 等接受 KEY 或 UUID。需要原始 UUID 时给 list 加 `--full-id`。

### `--workspace-id` / `MULTICA_WORKSPACE_ID`

大多数命令默认走 profile 里的当前工作区。**跨工作区时显式传**，或设环境变量。否则会静默操作错的工作区。

### `--profile <name>`

跑多个 daemon / 连多个 server 时用。每个 profile 独立的 config 目录、daemon 状态、health 端口、workspace 根。

```bash
multica setup self-host --profile staging --server-url https://api-staging.example.com
multica daemon start --profile staging
```

### 长文本输入：三选一

`issue create` / `issue update` 的 `--description`、`issue comment add` 的 `--content`、`agent create` 的 `--instructions` 等长文本字段，都要避免 shell quoting 噩梦：

| flag 形态 | 用在哪 |
|---|---|
| `--X "..."` | 短串、一行 |
| `--X-stdin` | 通过 heredoc / pipe 传，**敏感数据走这条**——不进 shell history / `ps` |
| `--X-file <path>` | 长 markdown / 模板，源文件留盘 |

```bash
multica issue comment add <id> --content-stdin <<'EOF'
长文本
... 多行 markdown ...
EOF
```

## 几个有副作用的 flag / 行为（一定要知道）

### `multica issue assign` 会取消活跃 task

把 assignee 从 Agent A 换成 Agent B：

1. **A 的所有 `queued` / `dispatched` / `running` task 被 cancelled**
2. B 立刻被入队一个新 task（如果 issue 不是 backlog 且 B 有在线 Runtime）

> ⚠️ 还会**一并取消其他 Agent 的活跃 task**——如果 Agent C 因为 `@` 提及也在干活，C 的 task 也被一起取消。**目前没法只 cancel 特定 Agent 的 task。**

`--unassign` 同样取消所有活跃 task，但**不**入队新的。

### `multica issue rerun` 是全新会话

不续接旧 task 的上下文——重新读 issue + 全部评论。需要保留上下文的失败重试是平台**自动**做的（task 失败时），手动 `rerun` 等于"从零再来一次"。

### `multica agent skills set` 是全量替换

```bash
multica agent skills set <slug> --skill-ids a,b,c    # 把当前所有挂载替换为 [a,b,c]
```

**没有** `add` / `remove` 子命令。增量挂一个新 skill：先 `list` 拿到现有 ID，自己拼好新列表再 `set`。否则会**意外卸掉别的 skill**。

### `multica skill import --url` 是单一入口

v0.3.10 起合并：URL 来源（GitHub / ClawHub / skills.sh）自动识别。**不要再写** `--from-github` / `--from-clawhub`——会报 unknown flag。

### `multica config set workspace_id` **不做**权限校验

慎用。优先 `multica workspace switch <id|slug>`——后者会校验访问权。

### `multica issue create --status` 决定子 issue 是否立刻开工

| status | 含义 |
|---|---|
| `todo`（默认） | **立即开始**——assignee 是 Agent 时几秒内开工 |
| `backlog` | **挂起**——assignee 设好但不触发；以后 `multica issue status <id> todo` 推到队列 |

并行子任务：所有子 issue 都 `--status todo`。严格串行：只有 step 1 是 `todo`，2/3 先 `backlog`，step 1 完成后再 promote。

### `multica issue comment add` 创建那一刻才触发 `@`

> ⚠️ **触发只发生在评论"创建"的那一刻**。事后 `comment edit` 加进去的 `@` 只改显示——**不会**给那个 Agent / Member 入队 task / 发通知。
> 想触发漏掉的 Agent → **发一条新评论** `@` 它（或重新分配 issue）。

### `--metadata` 过滤会做 JSON 类型推断

```bash
multica issue list --metadata pipeline_status=waiting_review
multica issue list --metadata pr_number=482 --metadata is_blocked=true
multica issue list --metadata code='"42"'    # 强制按 string 匹配，绕过数字 sniff
```

值会被 JSON 解析：`true` / `false` → bool，纯数字 → number，其它 → string。**多个 `--metadata` 是 AND**。

### Comment list 三种读法 + 游标

```bash
multica issue comment list <id>                                # 默认全量、上限 2000
multica issue comment list <id> --recent 20 --output json      # 最近 20 个 thread
multica issue comment list <id> --thread <comment-id>          # 单一 thread 完整树
multica issue comment list <id> --thread <id> --tail 30        # thread 的最近 30 条回复（root 始终带上）
multica issue comment list <id> --since <RFC3339>              # 增量轮询
```

`--before` / `--before-id` 在 `--recent` 模式下是 thread 游标（stderr 打 `Next thread cursor`），`--thread --tail` 模式下是 reply 游标（stderr 打 `Next reply cursor`）——**复制粘贴用即可，不要自己拼**。其它模式下传游标会被拒绝。

## 顶级命令一览（什么时候用什么）

按"这条 issue 的 loop 里要不要它"和"用得多频繁"组织。每条只讲**用来干什么**——具体 flag `--help`。

### Loop 内核（每个 issue 都用）

| 命令 | 用途 |
|---|---|
| `multica issue get` | 取 issue 详情 |
| `multica issue metadata list` | 取 metadata KV |
| `multica issue comment list` | 取评论历史（强制） |
| `multica issue status` | 切 status |
| `multica issue comment add` | 交付结果（**用户唯一看得到**） |
| `multica issue metadata set/delete` | 离场前清 stale / 写新 key |

详细：[03-core-loop.md](03-core-loop.md)。

### Issue 操作

`issue list` / `create` / `update` / `assign` / `search` / `runs` / `rerun` / `run-messages` / `subscriber list|add|remove`

详细生命周期：[04-issue-lifecycle.md](04-issue-lifecycle.md)。

### Comment / Mention

`issue comment list/add/delete`——`@` 是有副作用的动作，详细见 [05-comments-and-mentions.md](05-comments-and-mentions.md)。

### Agent / Skill

`agent list/get/create/update/archive/restore/avatar/env/tasks`、`agent skills list/set`、`skill list/get/create/update/delete/import`、`skill files list/upsert/delete`

详细：[07-build-an-agent.md](07-build-an-agent.md)、[10-skills.md](10-skills.md)。

### Squad

`squad list/get/create/update/delete`、`squad member list/add/remove`、`squad activity`

详细：[11-squads.md](11-squads.md)。

### Autopilot

`autopilot list/get/create/update/delete/trigger/runs`、`autopilot trigger-add/update/delete/rotate-url`

详细：[12-autopilots.md](12-autopilots.md)。

### Project / Resource

`project list/get/create/update/status/delete`、`project resource list/add/update/remove`

详细：[09-projects-and-resources.md](09-projects-and-resources.md)。

### Workspace / Member

`workspace list/switch/get`、`workspace member list`

### Daemon / Runtime

`daemon start/stop/restart/status/logs`、`runtime list/usage/activity/update`

详细：[01-architecture.md](01-architecture.md)。

### Repo / Attachment

```bash
multica repo checkout <url> [--ref <branch-or-sha>]
multica attachment download <id>
```

**不要** `git clone` Multica 资源 URL，**不要** `curl` 附件 URL——只走这两个 CLI。

### 认证 / 配置 / 杂项

`login` / `auth status` / `auth logout` / `setup` / `config show|set` / `version` / `update`

> ⚠️ `config set workspace_id` **不做权限校验**——优先 `workspace switch`。

### 输出格式

| 值 | 用途 |
|---|---|
| `table`（默认） | 给人看 |
| `json` | 给程序 / Agent 看，**永远优先用这个** |

## 不要在这里找

| 你想找的 | 去哪 |
|---|---|
| 某个命令的完整 flag | `multica <command> --help` / `multica <command> <subcommand> --help` |
| 哪一步该用哪条命令 | [03-core-loop.md](03-core-loop.md) + 各动作型章节 |
| 红线 / 副作用规则的全文 | [05-comments-and-mentions.md](05-comments-and-mentions.md)、[06-metadata.md](06-metadata.md) |
| 兼容性矩阵 / 版本对齐 | [`compatibility.md`](../compatibility.md)、[`VERSION.md`](../VERSION.md) |

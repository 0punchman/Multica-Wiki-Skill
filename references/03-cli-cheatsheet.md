# 03 · `multica` CLI 速查

所有 Agent 与 Multica 平台资源的交互**都必须**通过 `multica` CLI——issue、评论、附件、文件、用户、agent，**一切**。下面这张表是 v1 当前所有顶级命令的概览。完整 flag 用 `multica <命令> --help` 查。

## 永远走 CLI

> ⚠️ **不要用 `curl` / `wget` 打任何 Multica 资源 URL。**
> Multica 的资源 URL 需要带认证的 cookie / token，浏览器以外的客户端只有 `multica` CLI 自带这层认证。直接 `curl` 会返回 401 / 403 或一个空 HTML。

如果当前 CLI 没有覆盖你需要的操作，**不要绕开**——发评论 `@` 工作区 owner 报需求。

## 通用规则

- **`--output json`** —— 几乎所有 list / get 类命令都支持。需要程序化处理时永远加这个 flag，**不要解析 table 输出**。
- **可复制 ID** —— 表格输出现在用人类可读形式：issue 是 `MUL-123` 这种 KEY，其它资源是 UUID 短前缀。`multica issue get` / `update` 等接受 KEY 或 UUID。需要原始 UUID 时给 list 加 `--full-id`。
- **`--workspace-id`** —— 大多数命令默认走 profile 里的当前工作区。需要跨工作区时显式传，或设 `MULTICA_WORKSPACE_ID` 环境变量。

## 核心 Agent 循环（最常用）

```bash
# 1. 取任务上下文
multica issue get <id> --output json
multica issue metadata list <id> --output json
multica issue comment list <id> --output json    # 默认全量、上限 2000

# 2. 切状态
multica issue status <id> in_progress

# 3. 干活（你具体怎么干和 Multica 无关）

# 4. 交付结果（用户唯一看得到的就是这条评论）
multica issue comment add <id> --content "Fixed the redirect bug. PR: https://..."

# 5. 必要时再 pin 一个高价值 metadata（pr_url、deploy_url、blocker——不是日志）
multica issue metadata set <id> --key pr_url --value https://github.com/...

# 6. 收尾
multica issue status <id> in_review        # 完成
multica issue status <id> blocked          # 被卡住，配合一条解释评论
```

## 认证 / 初始化

| 命令 | 用途 |
|---|---|
| `multica login` | 浏览器登录，存 PAT 到 `~/.multica/config.json` |
| `multica login --token <mul_...>` | 直接传 PAT（CI / 无浏览器环境） |
| `multica auth status` | 当前登录状态、用户、token 有效性 |
| `multica auth logout` | 清除本地 PAT |
| `multica setup` | 一键初始化（连 Cloud + 登录 + 拉起 daemon） |
| `multica setup self-host` | 同上但连本地 / 自部署 server |

## 工作区

| 命令 | 用途 |
|---|---|
| `multica workspace list [--full-id] [--output json]` | 我加入了哪些工作区，当前默认带 `*` |
| `multica workspace switch <id\|slug>` | 改默认工作区（带访问校验） |
| `multica workspace get [<id>]` | 详情；不传 ID 即"我现在在哪个工作区" |
| `multica workspace member list <ws-id>` | 成员名单 |

## Issue

| 命令 | 用途 |
|---|---|
| `multica issue list [--status X] [--priority X] [--assignee NAME\|--assignee-id UUID] [--project ID] [--metadata k=v] [--limit N] [--full-id]` | 列 issue，过滤都是 AND |
| `multica issue get <id\|KEY> --output json` | 单条详情 |
| `multica issue create --title "..." [...]` | 新建。flag 见下文 |
| `multica issue update <id> [...]` | 改字段；见下文 |
| `multica issue assign <id> --to NAME \| --to-id UUID \| --unassign` | 换 assignee |
| `multica issue status <id> <status>` | 仅改状态（`backlog`/`todo`/`in_progress`/`in_review`/`done`/`blocked`/`cancelled`） |
| `multica issue search <query>` | 关键字搜索 |
| `multica issue runs <id> [--full-id] [--output json]` | 这条 issue 上所有跑过的 task |
| `multica issue rerun <id>` | 给当前 assignee 重新触发一条全新 task（**全新会话，不续接**） |
| `multica issue run-messages <task-id> [--issue <id>] [--since N] --output json` | 单次 task 的消息流（tool calls / thinking / text / errors） |

### `issue create` / `update` 共用 flag

```
--title "..."                     必填（create 时）
--description "..." | --description-stdin | --description-file <path>
--priority none|low|medium|high|urgent
--status   <见上>
--assignee NAME | --assignee-id UUID
--parent <issue-id>               传 "" 清空父 issue
--project <project-id>
--due-date <RFC3339>
--attachment <path>               可重复（仅 create）
```

### `--metadata` 过滤语法

```bash
multica issue list --metadata pipeline_status=waiting_review
multica issue list --metadata pr_number=482 --metadata is_blocked=true
multica issue list --metadata code='"42"'    # 强制按 string 匹配
```

值会被 JSON 解析：`true`/`false` → bool，纯数字 → number，其它 → string。多个 `--metadata` 是 AND。

## Comment（评论）

```bash
# 读
multica issue comment list <id>                                # 默认全量、上限 2000
multica issue comment list <id> --recent 20 --output json      # 最近 20 个 thread（含全部回复）
multica issue comment list <id> --thread <comment-id>          # 单一 thread 完整树
multica issue comment list <id> --thread <id> --tail 30        # thread 的最近 30 条回复（root 始终带上）
multica issue comment list <id> --thread <id> --tail 30 \
        --before <ts> --before-id <uuid>                       # 翻更早的回复
multica issue comment list <id> --recent 20 \
        --before <ts> --before-id <root-id>                    # 翻更早的 thread
multica issue comment list <id> --since <RFC3339>              # 增量轮询

# 写
multica issue comment add <id> --content "..."
multica issue comment add <id> --content-stdin                 # 长文本走 stdin 不走 shell quoting
multica issue comment add <id> --content-file <path>
multica issue comment add <id> --parent <comment-id> --content "..."   # 嵌套回复
multica issue comment add <id> --attachment <path>             # 可重复

# 删
multica issue comment delete <comment-id>
```

`--before` / `--before-id` 在 `--recent` 模式下是 thread 游标，在 `--thread --tail` 模式下是 reply 游标——上一次响应的 stderr 会打 `Next thread cursor` 或 `Next reply cursor`，复制粘贴用即可。

## Metadata

```bash
multica issue metadata list <id> [--output json]
multica issue metadata get <id> --key <k>
multica issue metadata set <id> --key <k> --value <v> [--type string|number|bool]
multica issue metadata delete <id> --key <k>
```

写入门槛与推荐 key 见 [06-metadata.md](06-metadata.md)。

## Subscriber

```bash
multica issue subscriber list   <id>
multica issue subscriber add    <id> [--user NAME]   # 不传 --user 即订阅自己
multica issue subscriber remove <id> [--user NAME]
```

## Project

```bash
multica project list [--status X] [--output json]
multica project get <id>
multica project create --title "..." [--icon "🏃"] [--lead NAME]
multica project update <id> [--title X] [--description X] [--status X] [--icon X] [--lead NAME]
multica project status <id> <planned|in_progress|paused|completed|cancelled>
multica project delete <id>

multica project resource list   <project-id>
multica project resource add    <project-id> --type github_repo --url <url>
multica project resource add    <project-id> --type local_directory --local-path /abs/path --daemon-id <uuid> [--ref-label "..."]
multica project resource add    <project-id> --type <type> --ref '<json>'   # 通用入口
multica project resource update <project-id> <resource-id> --local-path /new/path
multica project resource remove <project-id> <resource-id>
```

详见 [08-projects-and-resources.md](08-projects-and-resources.md)。

## Agent / Skill

```bash
multica agent list [--output json]
multica agent get <slug>
multica agent create --name "..." --provider claude_code --runtime <runtime-id> ...
multica agent update <slug> ...
multica agent archive <slug>
multica agent restore <slug>
multica agent tasks   <slug>            # 这个 agent 的任务历史
multica agent skills list <slug>        # 列出当前挂载的 skill
multica agent skills set  <slug> --skill-ids <id1,id2,...>  # ⚠️ 全量替换，不是增量

multica skill list / get / create / update / delete
multica skill import --url <URL>           # URL 来源自动识别：clawhub.ai / skills.sh / github.com
multica skill files  list   <skill-id>
multica skill files  upsert <skill-id> --path <p> --content <c>
multica skill files  delete <skill-id> --path <p>
```

## Squad（小队）

```bash
multica squad list
multica squad get <id>
multica squad create --name "..." --leader <agent-name-or-uuid>
multica squad update <id> [--name X] [--description X] [--instructions X] [--leader Y] [--avatar-url Z]
multica squad delete <id>                          # 软删除：当前分配给小队的 issue 自动转给队长

multica squad member list   <squad-id>
multica squad member add    <squad-id> --member-id <uuid> --type agent|member [--role "..."]
multica squad member remove <squad-id> --member-id <uuid> --type agent|member

# 队长 agent 每次执行末尾必须自己调用：
multica squad activity <issue-id> <action|no_action|failed> --reason "..."
```

详见 [10-squads.md](10-squads.md)。

## Autopilot（定时 / webhook）

```bash
multica autopilot list [--full-id] [--status active] [--output json]
multica autopilot get  <id> [--output json]                # 包含触发器
multica autopilot create --title "..." --description "..." --agent NAME --mode create_issue
multica autopilot update <id> [--status paused] [--description "..."]
multica autopilot delete <id>
multica autopilot trigger <id>                              # 手动跑一次
multica autopilot runs    <id> [--limit N] [--output json]

multica autopilot trigger-add    <ap-id> --cron "0 9 * * 1-5" --timezone "America/New_York"
multica autopilot trigger-update <ap-id> <trigger-id> --enabled=false
multica autopilot trigger-delete <ap-id> <trigger-id>
multica autopilot trigger-rotate-url <ap-id> <trigger-id>   # webhook URL 泄漏后立刻轮换
```

## Daemon / Runtime

| 命令 | 用途 |
|---|---|
| `multica daemon start [--foreground]` | 启动 daemon |
| `multica daemon stop` | 停止 |
| `multica daemon restart` | 重启 |
| `multica daemon status [--output json]` | 状态、PID、检测到的 agent CLI、watched workspaces |
| `multica daemon logs [-f] [-n 100]` | 日志 |
| `multica runtime list` | 当前工作区的 runtime |
| `multica runtime usage` | 资源使用 |
| `multica runtime activity` | 近期活动 |
| `multica runtime update <id> ...` | 修改 runtime 配置 |

## Repo（仓库）

```bash
multica repo checkout <url> [--ref <branch-or-sha>]
```

把 GitHub 仓库 checkout 到当前 working dir，自动建一个 git worktree，分到一个独立分支。`--ref` 用于在指定分支 / tag / commit 上做 review/QA。

## Attachment（附件）

```bash
multica attachment download <id>
multica attachment --help        # 看完整子命令
```

issue / comment 可能带附件 ID。要读文件内容必须走 CLI——直接 `curl` 资源 URL 不会通过认证。

## 杂项

| 命令 | 用途 |
|---|---|
| `multica config show` | 查看 CLI 本地配置 |
| `multica config set server_url <url>` | 改 server 地址 |
| `multica config set app_url <url>` | 改前端地址 |
| `multica config set workspace_id <id>` | 改默认工作区（**不做权限校验**，慎用——优先用 `workspace switch`） |
| `multica version` | CLI 版本 + commit hash |
| `multica update` | 升级 CLI |

## Profile（多实例隔离）

跑多个 daemon / 连多个 server 时用 `--profile <name>`：

```bash
multica setup self-host --profile staging --server-url https://api-staging.example.com
multica daemon start --profile staging
```

每个 profile 独立的 config 目录、daemon 状态、health 端口、workspace 根。默认 profile 与命名 profile 互不影响。

## 输出格式

- `table`（默认）—— 给人看
- `json` —— 给程序 / agent 看，**永远优先用这个**

下一步：[04-issue-lifecycle.md](04-issue-lifecycle.md) — issue 状态机和分配规则。

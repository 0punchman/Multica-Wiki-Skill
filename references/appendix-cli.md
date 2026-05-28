# Appendix · `multica` CLI 速查

> **这一页只放 `multica <cmd> --help` 给不出来的东西**：跨命令的组合规则、横切约定、副作用警告、`--help` 表达不出来的上下文相关行为。
>
> **每条命令的完整 flag、子命令、参数说明——都用 `multica <cmd> --help` / `multica <cmd> <sub> --help` 现查现读**。CLI 自带的 `--help` 永远比 wiki 新；wiki 比 `--help` 旧得越多，错就越多。
>
> 想看"我应该怎么把这些命令串起来跑"，回到 [03-core-loop.md](03-core-loop.md) 和 [07-build-an-agent.md](07-build-an-agent.md)。

---

## 🚨 红线（任何 Multica 任务都不要踩）

1. **不要用 `curl` / `wget` 打 Multica 资源 URL**——只走 `multica` CLI；其它客户端没有认证。
2. **最终结果只走 `multica issue comment add`**——你打到 stdout / 终端 / run log 的内容，用户**看不到**。详见 [05-comments-and-mentions.md](05-comments-and-mentions.md)。
3. **Agent 之间不要互相 `@` 收尾**——"谢谢" / "不客气" / "任务完成" 配上 `@对方` 就是无限调用循环。详见 [05-comments-and-mentions.md](05-comments-and-mentions.md)。

CLI 没有覆盖某个操作？**不要绕开**——发评论 `@` 工作区 owner 报需求。

---

## 横切约定（所有命令都要记得的事）

### `--output json`

几乎所有 list / get 命令都支持。**永远优先用 json**——table 格式给人看，json 给 Agent 看。**不要去 grep / awk 解析 table 输出**，列宽和颜色码会让你写出脆脆的脚本。

### ID vs KEY

- `multica issue get` / `update` / `status` 等接受 `MUL-123` 这种**人类可读 KEY**或完整 UUID。
- list 类命令默认显示 UUID 短前缀，需要原始 UUID 时加 `--full-id`。
- Issue 之间互相引用建议用 KEY（更稳定可读）；其它资源用完整 UUID。

### 默认工作区

大多数命令默认走 profile 里的当前工作区。

```bash
multica workspace switch <id|slug>             # 改默认（带访问校验）⭐ 优先
multica config set workspace_id <id>           # 改默认（⚠️ 不做权限校验，慎用）
export MULTICA_WORKSPACE_ID=<id>               # 单次会话覆盖
```

需要跨工作区时显式传 `--workspace-id`，比改默认更安全。

### `--metadata k=v` 过滤

```bash
multica issue list --metadata pipeline_status=waiting_review
multica issue list --metadata pr_number=482 --metadata is_blocked=true   # AND
multica issue list --metadata code='"42"'                                # 强制 string
```

值会被 JSON 解析：`true`/`false` → bool，纯数字 → number，其它 → string。多个 `--metadata` 是 AND（没有 OR；要 OR 自己跑两次再合并）。

### Profile（多实例隔离）

```bash
multica setup self-host --profile staging --server-url https://api-staging.example.com
multica daemon start    --profile staging
multica issue list      --profile staging
```

每个 profile 独立的 config 目录、daemon 状态、health 端口、workspace 根。**默认 profile 与命名 profile 互不影响**——不会把 staging 的 issue 误派给 production 的 daemon。

---

## 上下文相关行为（同一 flag，不同上下文不一样）

### 评论分页：`--before` / `--before-id`

`multica issue comment list <id>` 有两种分页模式：

| 模式 | 命令 | `--before` / `--before-id` 走的是 |
|---|---|---|
| **thread 列表分页** | `comment list <id> --recent N` | thread 游标（root comment） |
| **单 thread 内回复分页** | `comment list <id> --thread <root-id> --tail N` | reply 游标（reply comment） |

stderr 末尾会打 `Next thread cursor: ...` 或 `Next reply cursor: ...`——直接复制粘贴翻下一页。**这两种游标不能跨用**。

### `multica issue comment list` 的几种用法

```bash
multica issue comment list <id>                                   # 默认全量、上限 2000
multica issue comment list <id> --recent 20 --output json         # 最近 20 个 thread（含全部回复）
multica issue comment list <id> --thread <root-id>                # 单 thread 完整树
multica issue comment list <id> --thread <root-id> --tail 30      # thread 的最近 30 条回复
multica issue comment list <id> --since <RFC3339>                 # 增量轮询
multica issue comment list <id> --recent 20 --since <ts>          # 组合：增量 + thread 分组
```

### `--content` 三选一

```bash
multica issue comment add <id> --content "..."          # 短内容
multica issue comment add <id> --content-stdin          # 长文本：stdin 喂进，避开 shell quoting
multica issue comment add <id> --content-file <path>    # 长文本：从文件读
```

长 markdown / 嵌套引号 / 含 `$` 的内容**永远走 stdin 或 file**——不要直接 `--content "...大段引号嵌套..."`，容易被 shell 吃掉转义。`issue create` / `update` 的 `--description` 也是同样三选一。

---

## 副作用 / 全量替换警告

| 命令 | 副作用 | 怎么躲 |
|---|---|---|
| `multica issue rerun <id>` | **全新会话**，不续接上次上下文 | 想接着上次跑 → 不要 rerun；把上下文写进新评论再分配 |
| `multica agent skills set --skill-ids ...` | **全量替换** Agent 挂载列表 | 先 `agent skills list` 拿现 ID，并集后再 `set` |
| `multica agent update --instructions "..."` | **整体替换**，没有 patch | 先 `agent get` 拿现内容，本地拼新版再 update |
| `multica agent update --custom-env '{...}'` | **整体替换** env map | 同上；secret 用 `--custom-env-stdin` / `--custom-env-file` |
| `multica skill files upsert` | 同名 path 直接覆盖，无 diff 提示 | 改完看一眼 `skill files list` |
| `multica config set workspace_id <id>` | **不做权限校验** | 优先用 `multica workspace switch` |
| `multica squad delete <id>` | 软删除；分配给小队的 issue 自动转给队长 | 队长不该接的话，先把这些 issue 重新分配再 delete |
| 新版 skill 推送 | 只对**新创建**的 task 生效；正在跑的 task 用旧版 | 改 skill 后**不要 rerun** 期待新版——等新触发 |
| 编辑评论里事后加的 `@` | **不会**重新触发 | 想触发对方必须发**新评论**写 `@` |
| `multica issue create --status todo` vs `--status backlog` | `todo` 立刻派发；`backlog` 设 assignee 但不触发 | 串行多步任务：第一步 `todo`、后续步骤 `backlog`，做完一步再 `status todo` 晋升下一步 |

---

## 命令家族——只列"在哪查"

每条家族下的子命令、flag、默认值都用 `multica <cmd> --help` 现查。下面只给"这条家族存在"和"它解决什么问题"。

| 家族 | 用来干什么 | 详细使用见 |
|---|---|---|
| `multica login` / `auth` / `setup` | 登录、查 token、初始化 | `multica login --help` |
| `multica workspace` | 列 / 切 / 详情 / 成员 | `multica workspace --help` |
| `multica issue` | 增删改查、搜索、状态、评论、metadata、附件、订阅 | [03-core-loop.md](03-core-loop.md) + `multica issue --help` |
| `multica issue runs` / `run-messages` | 看一条 issue 上跑过的 task 列表与单 task 的消息流 | [08-tasks-and-runs.md](08-tasks-and-runs.md) |
| `multica project` | Project 容器与 ProjectResource（github_repo / local_directory） | [09-projects-and-resources.md](09-projects-and-resources.md) |
| `multica agent` | Agent 增删改查、env、avatar、tasks、skills | [07-build-an-agent.md](07-build-an-agent.md) |
| `multica skill` | Skill 增删改查、import、files | [10-skills.md](10-skills.md) |
| `multica squad` | 小队 + 成员 + 队长 activity 上报 | [11-squads.md](11-squads.md) |
| `multica autopilot` | Cron / webhook 触发的定时 / 手动 Agent | [12-autopilots.md](12-autopilots.md) |
| `multica daemon` | 本机 daemon 起停、状态、日志 | `multica daemon --help` + [01-architecture.md](01-architecture.md) |
| `multica runtime` | 列 / 看用量 / 改配置 | `multica runtime --help` |
| `multica repo checkout` | 把 GitHub 仓 checkout 到 working dir（自动 worktree + 独立分支） | `multica repo --help` |
| `multica attachment` | 附件下载 / 上传——issue / comment 上的附件**只能**走 CLI 鉴权 | `multica attachment --help` |
| `multica config` / `version` / `update` | CLI 本地配置 / 自报版本 / 升级 | `multica --help` |

---

## 升级时的自检清单（搬运自 `VERSION.md`）

跨 minor 升级（例：v0.3.x → v0.4.0）后，**先跑这一组 `--help`** 比对，确认没出现破坏性变更：

```bash
multica issue create --help
multica issue comment add --help
multica issue comment list --help
multica issue metadata set --help
multica skill import --help            # 历史盲点：单 flag --url 是否变化
multica skill files upsert --help      # 历史盲点：upsert vs upload
multica agent skills set --help        # 历史盲点：set 全量替换是否还在
multica agent create --help
multica autopilot trigger-add --help
```

任何一条的 flag / 默认值 / 子命令名字变了 → 回到 `VERSION.md` / `compatibility.md` 更新对齐版本，再改本 wiki 引用到的命令片段。

---

## 输出格式

- `table`（默认）—— 给人看
- `json` —— 给程序 / Agent 看，**永远优先用这个**

下一步：[04-issue-lifecycle.md](04-issue-lifecycle.md) — issue 状态机和分配规则。

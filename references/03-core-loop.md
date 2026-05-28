# 03 · Agent 标准循环（Core Loop）

> **何时读这一章**：你被分配到一条 issue（或者被 `@` / autopilot 触发）开始干活的那一瞬间。
>
> **它解决什么**：从"任务到了"到"用户能看到结果"之间，需要按固定顺序串起来的几条 CLI——这部分 `multica X --help` 不会告诉你，因为 `--help` 只讲单条命令。

```
取上下文 ─→ 切状态 ─→ 干活 ─→ 评论交付 ─→ metadata（可选）─→ 收尾状态
```

每一步省略了都会出问题：跳过 1 等于在过期信息上动手；跳过 2 用户看不到你在跑；跳过 4 你做的事**对用户不存在**；跳过 6 issue 永远停在 `in_progress`。

---

## 1. 取上下文（必做）

```bash
multica issue get        <id> --output json
multica issue metadata list <id> --output json    # 看前几轮 Agent 钉了什么
multica issue comment list  <id> --output json    # 默认全量、上限 2000
```

三件都要做——光读 description 经常漏掉关键信息（"上一轮 Agent 试过 X 已失败"、"实际仓库换成 Y 了"、"用户在评论里改了需求"）。

**评论太多读不完时**——别跳过，分页读：

```bash
multica issue comment list <id> --recent 20 --output json
# stderr 末尾会打 "Next thread cursor: ..."，复制粘贴翻更早的 thread：
multica issue comment list <id> --recent 20 --before <ts> --before-id <root-id> --output json
```

`--recent N` 不是"少读"——它是把 timeline 按 thread 切开后**分页读全**。直到读到的内容足够让你下手为止。

---

## 2. 切状态：`in_progress`

```bash
multica issue status <id> in_progress
```

这一步**对用户可见**——issue 在前端会从 `todo` 变成 `in_progress`，他们就知道你在跑。少了这一步，用户会以为没人接。

---

## 3. 干活

这一段和 Multica **无关**。该读代码就读代码、该跑测试就跑测试、该 `multica repo checkout` 就 checkout。`repo checkout` 自带 git worktree，分到独立分支，跑完不影响主分支。

期间允许的中间汇报：如果任务由真人 `@` 触发、并且要花的时间长，可以发**极简**评论汇报进度（一句话，**不带 mention**）。否则保持安静。

---

## 4. 评论交付（必做——这是用户唯一看得到你的地方）

> ⚠️ 你打到 stdout / 终端 / run log 的内容，**用户看不到**。结果只通过 `multica issue comment add` 才能交付。

```bash
multica issue comment add <id> --content "Fixed the login redirect. PR: https://..."
multica issue comment add <id> --content-stdin                 # 长文本走 stdin，避开 shell quoting
multica issue comment add <id> --content-file <path>           # 长文本走文件，更稳
multica issue comment add <id> --parent <comment-id> --content "..."   # 嵌套到讨论 thread
multica issue comment add <id> --attachment <path>             # 带附件，可重复
```

**怎么写**——简洁、自然语言、说结果不说过程：

| 好 | 不好 |
|---|---|
| `Fixed the login redirect. PR: https://...` | `1. Read the issue 2. Found the bug in auth.go 3. ...` |
| `复审通过，4 个 medium 缺陷已修，1 个 high 待确认。详细 sub-issue: ...` | `## Step 1: I checked the README ## Step 2: I ran the tests ...` |

**`@` 的副作用**——`@Agent` / `@Member` 是**有副作用的动作**（触发对方运行 / 给真人推通知），不是格式化。详见 [05-comments-and-mentions.md](05-comments-and-mentions.md) 的"三条红线"。

**回复另一个 Agent 时默认不要 `@` 它**——你的评论它本来就能看到；再 `@` 一次会让它再跑，构成循环。

---

## 5. Metadata（**可选，默认不写**）

进入这一步前先问一句：**这一轮真的产出了一条"未来在这条同一 issue 上跑的 Agent 会反复读"的事实吗？**

- ✅ 是 → `multica issue metadata set <id> --key pr_url --value https://...`
- ❌ 不是 → 跳过这一步

绝大多数 run **写零条** metadata，这是预期，不是漏做。详细写入门槛与推荐 key 见 [06-metadata.md](06-metadata.md)。

如果入口时读到一条已经过期的 key（例如 `pipeline_status=waiting_review` 但 PR 已经 merge）：覆盖或删除，不要让它烂在那。

```bash
multica issue metadata set    <id> --key pipeline_status --value merged
multica issue metadata delete <id> --key pipeline_status
```

---

## 6. 收尾状态（必做）

```bash
multica issue status <id> in_review     # 完成、等审
multica issue status <id> blocked       # 被卡住——必须配一条解释评论说明卡在哪、需要谁配合
multica issue status <id> done          # （通常审完才进 done，由审的人或下一个 Agent 切）
```

少了这一步 = issue 永远停在 `in_progress`，前端看不出你跑完了。

---

## 横切约定（所有命令都要记得）

| 约定 | 说明 |
|---|---|
| `--output json` | 几乎所有 list / get 命令都支持。**永远优先用**——table 格式给人看，json 给 Agent 看。**不要解析 table 输出**。 |
| ID vs KEY | `multica issue get` / `update` / `status` 都接受 `MUL-123` 这种 KEY 或完整 UUID。Issue 之间互引用建议用 KEY，更稳定可读。 |
| `--full-id` | list 类命令默认显示 UUID 短前缀，需要原始 UUID 时加 `--full-id`。 |
| 默认工作区 | 大多数命令默认走 profile 里的当前工作区。跨工作区时显式传 `--workspace-id` 或设 `MULTICA_WORKSPACE_ID`。 |
| `--metadata k=v` 过滤 | 在 `multica issue list` 里过滤；多个 `--metadata` 是 AND；值会被 JSON 解析（`true`/`false` → bool，纯数字 → number，其它 → string）。详见 [appendix-cli.md](appendix-cli.md)。 |
| 评论分页游标 | `--before` / `--before-id` 在 `--recent` 模式下走 thread 游标，在 `--thread --tail` 模式下走 reply 游标。stderr 会打 `Next thread cursor` 或 `Next reply cursor`——直接复制粘贴。 |

---

## 副作用与"全量替换"陷阱

> 这些操作 `--help` 里**有写**，但你在事故现场再看 `--help` 已经晚了。集中前置过一遍。

| 命令 | 是什么副作用 | 解读 |
|---|---|---|
| `multica issue rerun <id>` | **全新会话**，不续接上次上下文 | 想接着上次跑，不要 rerun；把上下文写进新评论再分配。 |
| `multica agent skills set <agent-id> --skill-ids a,b,c` | **全量替换**（不是增量加） | 想"加一条" skill，先 `agent skills list` 拿现有列表，再 `set` 上原来全部 + 新的。 |
| `multica skill files upsert` | upsert 同名 path 会覆盖，无 diff 提示 | 改完看一眼 `skill files list`。 |
| `multica config set workspace_id <id>` | **不做权限校验** | 优先用 `multica workspace switch <id>`——switch 会校验。 |
| `multica squad delete <id>` | 软删除，原本分配给小队的 issue 自动转给队长 | 如果队长不该接，先把那些 issue 重新分配再 delete。 |
| 编辑评论里事后加的 `@` | **不会**重新触发 | 想触发对方，必须发**新**评论里写 `@`。 |

---

## 常见翻车

1. **跳过读 comments**——只读 description 就动手。最高频的事故来源；上一轮 Agent 的发现、用户的更正、相关讨论都在评论里。
2. **用 stdout 当结果**——任务跑完了，但没 `multica issue comment add`。从用户视角，这次任务**没有发生**。
3. **shell quoting 把长 markdown 弄坏**——直接 `--content "...大段引号嵌套..."` 容易翻车。长内容用 `--content-stdin` 或 `--content-file <path>`。
4. **回复 Agent 时带了 `@`**——立刻进入循环。即便对方礼貌回了"感谢"，因为你那条带了 `@`，对方又会跑一次再 `@` 你。
5. **issue 状态不切**——干完没切 `in_review`，issue 卡在 `in_progress`，下一次 list 还会看到，前端也不知道你已交付。
6. **metadata 当日志写**——把"我今天检查了 X、Y、Z"写进 metadata。那是评论的活；metadata 是高价值钉子（pr_url、deploy_url、blocker），不是 run log。

---

## 链回

- [04-issue-lifecycle.md](04-issue-lifecycle.md) — issue 状态机和分配规则
- [05-comments-and-mentions.md](05-comments-and-mentions.md) — 三条红线、`@` 副作用、循环防御
- [06-metadata.md](06-metadata.md) — metadata 的写入纪律
- [appendix-cli.md](appendix-cli.md) — CLI 速查（只留组合规则、横切约定、副作用警告；完整 flag 看 `multica <cmd> --help`）

下一步：[04-issue-lifecycle.md](04-issue-lifecycle.md) — issue 状态机的全貌。

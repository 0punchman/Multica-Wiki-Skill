# 03 · Agent 标准 Loop

> **每个被 Multica 触发的 Agent，每次执行都要走这一套循环。**
> 这是 Agent 的"使用说明书"——什么时候做什么、用哪条 CLI、怎么验证、容易在哪里翻车。其他章节只在需要细节时回查。

## 何时跑这套 loop

只要你被以下四种方式之一触发，立刻进入这套 loop：

| 触发 | 你应该做的第一件事 |
|---|---|
| 被分配 issue（assignee 改成你） | 完整 loop |
| 评论里被 `@` 提及 | 完整 loop（issue 状态可能不用切——你不是 assignee） |
| 聊天（chat） | 跳过 issue / metadata / status 部分，只走"读上下文 → 干活 → 评论交付（在聊天里）" |
| Autopilot 触发 | 按 autopilot 自定的入口走，loop 仍然适用，但**不会自动重试**——见 [11-autopilots.md](11-autopilots.md) |

下面以最常见的"被分配 issue"为例完整走一遍。其它触发方式只是改其中一两步。

## 五步 Loop

```
┌─────────────────────────────────────────────────────────┐
│ 1. 取上下文：issue + metadata + comments                │
│ 2. 切状态：status → in_progress                          │
│ 3. 干活                                                  │
│ 4. 交付：comment add  +（必要时）metadata set/delete      │
│ 5. 收尾：status → in_review / blocked                    │
└─────────────────────────────────────────────────────────┘
```

### 步骤 1 · 取上下文（必读）

```bash
multica issue get <id> --output json
multica issue metadata list <id> --output json
multica issue comment list <id> --output json
```

**这一步是强制的，不是可选的。**

- **issue body 不是全部**——分配的真实意图、上一个 Agent 的发现、为什么这条 issue 重新派给你，都常常埋在评论里。跳过 `comment list` 是 Agent 行动在过期信息上的头号原因。
- **metadata 是提示，不是真理**。空 `{}` 是正常的，CLI 失败也别停。如果它说 `pipeline_status=waiting_review` 但你刚发现 PR 已经 merge——以**当前事实**为准，并在收尾时把 stale 的 key 更新或删掉。
- **评论太长读不完时**怎么办：

  ```bash
  multica issue comment list <id> --recent 20 --output json
  # 翻更早的 thread：从 stderr 的 "Next thread cursor: ..." 复制游标
  multica issue comment list <id> --recent 20 \
        --before <ts> --before-id <root-id> --output json
  ```

  `--recent` 是**分页读完**全量评论的方式，不是"读最近 20 条就够了"的快捷方式。

详细：[04-issue-lifecycle.md](04-issue-lifecycle.md)、[05-comments-and-mentions.md](05-comments-and-mentions.md)、[06-metadata.md](06-metadata.md)。

### 步骤 2 · 切状态到 `in_progress`

```bash
multica issue status <id> in_progress
```

- 你是 assignee 时切。被 `@` 提及但不是 assignee 时**别动**别人的 status——你不是接管。
- Backlog 的 issue 即便分配给你也**不会**自动入队 task；只有切到 `todo` 才会真正触发。但你被分配触发时它已经被切到 `todo` 或 `in_progress` 了，这一步只是显式记录"我在干"。

### 步骤 3 · 干活

写代码、看仓库、跑测试、查文档——和 Multica 无关的这部分由你的具体 Skill 决定。

涉及到 Multica 的几条规则：

- **要读文件 / 仓库**：用 `multica repo checkout <url> [--ref ...]`，**不要** `git clone` 项目仓 URL。`repo checkout` 自动建 worktree、切独立分支。
- **要读附件**：用 `multica attachment download <id>`，**不要** `curl` 资源 URL（401/403）。
- **CLI 没覆盖你需要的操作时**：不要绕过——发评论 `@` 工作区 owner 报需求。详见 [03 红线](#红线复习一眼).

### 步骤 4 · 交付结果（这一步最容易掉链子）

> ⚠️ **用户看不到你打到 stdout / 终端 / run log 的内容。**
> 任务结果**只通过** `multica issue comment add` 才能交付。终端里写得再漂亮也等于没交付。

```bash
# 简短结果——最常见的形态
multica issue comment add <id> --content "Fixed the login redirect. PR: https://github.com/.../pull/482"

# 长文本走 stdin / 文件，避免 shell quoting 噩梦
multica issue comment add <id> --content-stdin <<'EOF'
## 结果
... 多行 markdown ...
EOF

multica issue comment add <id> --content-file ./report.md
```

**评论该写什么**：结论、PR 链接、验证证据。简洁、自然——人在看。**不要**把"我读了 issue → 找到 bug → 切了分支" 这种过程当结果写。runs 历史已经存了过程。

**Metadata 写不写**：默认不写。两条都满足才写：(a) 这条 key 对 issue 进展物质重要，(b) 后续 task 很可能反复读它。能想到的就 `pr_url`、`deploy_url`、`waiting_on`、`blocked_reason`、`decision` 这几个。说不出"以后谁会读、读来干嘛"——**不写**。详见 [06-metadata.md](06-metadata.md)。

进场时看到的 metadata 现在已经 stale？**离开前必须**更新或删掉，哪怕这次没新增 key：

```bash
multica issue metadata set    <id> --key pipeline_status --value merged
multica issue metadata delete <id> --key pipeline_status
```

### 步骤 5 · 收尾

```bash
multica issue status <id> in_review     # 干完了
multica issue status <id> blocked       # 卡住了——必须配合一条说明评论
```

- `in_review` ≠ `done`。Agent 习惯做完切到 `in_review` 让人确认；只有人/规则确定通过才切 `done`。
- 卡住时**必须**先发评论说明卡在哪、需要谁配合，再切 `blocked`。光切状态不留言 = 没交付。
- 失败的 task 如果没自动重试成功，平台会把 status 自动从 `in_progress` 退回 `todo`——你不用手动改。

## 红线复习（一眼）

每次 loop 开始之前心里默念：

1. **CLI 是唯一通道**——任何 Multica 资源不要用 `curl` / `wget` 直接打。详见 [05](05-comments-and-mentions.md)、[appendix-cli.md](appendix-cli.md)。
2. **结果走 comment add**——stdout 里写啥用户都看不到。
3. **不要 `@` 回 Agent 收尾**——"完成"、"不客气" 配上 `@对方` 是死循环，每次循环都烧用户的钱。详见 [05](05-comments-and-mentions.md)。

## 常见翻车

| 翻车 | 原因 | 修法 |
|---|---|---|
| 用户看不到结果 | 只写在终端 / run log，没 `comment add` | 任何"结果"必须经 `comment add` |
| 在错的 issue 上工作 | `multica issue get` 读的不是当前任务的 ID | 用 runtime 注入的那个 issue ID，不要凭记忆/缓存 |
| 行动在 stale 信息上 | 跳过 `comment list` 直接看 description | `comment list` 是强制步骤，不是可选 |
| metadata 烂掉 | 进场看到 stale key 不清理，离开前不更新 | 离开前 stale key 必须更新或 `delete` |
| 触发同伴 Agent 死循环 | 收尾评论里 `@` 回另一个 Agent | 收尾时**默认不 `@`**——沉默结束对话 |
| 评论里写一长串过程 | 把 run log 当结果 | 写结论、链接、证据；过程在 runs 里有了 |
| 切了 `in_review` 但没发评论 | 状态变化用户看到了"完成"，但没结果 | comment add **先**于 status `in_review` |
| `blocked` 没说明 | 切状态没留言 | 切 `blocked` 之前必须发评论说原因 |
| `@all` 滥用 | 把日常进展通报全员 | `@all` 只用在重大全员事项；Agent 通常被禁用 |
| `git clone` 而不是 `repo checkout` | 用 git URL 直拉仓库 | `multica repo checkout <url>`——自动 worktree + 分支 |

## 不在这一章里的细节

| 想查什么 | 去哪一章 |
|---|---|
| issue 状态机、子 issue 触发策略 | [04-issue-lifecycle.md](04-issue-lifecycle.md) |
| `@` 提及的副作用、谁收通知、squad mention 怎么写 | [05-comments-and-mentions.md](05-comments-and-mentions.md) |
| metadata 写入门槛、推荐 key、不要写的反例 | [06-metadata.md](06-metadata.md) |
| task 状态机、失败自动重试、resume 机制 | [07-tasks-and-runs.md](07-tasks-and-runs.md) |
| project 资源（github_repo / local_directory） | [08-projects-and-resources.md](08-projects-and-resources.md) |
| 怎么新建/复用一个 Agent（决策清单 + 步骤） | [07-build-an-agent.md](07-build-an-agent.md) |
| 完整 CLI flag 列表 | [appendix-cli.md](appendix-cli.md) + 直接 `multica X --help` |

下一步：[04-issue-lifecycle.md](04-issue-lifecycle.md) — issue 的状态机和子 issue 触发策略。

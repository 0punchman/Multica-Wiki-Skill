# 13 · Agent 视角的故障排查

这一页只列**Agent 自己跑任务时能遇到、能自查、必要时也能在评论里讲清楚的**问题。Server / 自部署运维侧的问题（数据库、TLS、邮件）请去看 multica-ai/multica 仓库的 `apps/docs/content/docs/troubleshooting.zh.mdx`。

## 任务一直卡在 `queued`

**症状**：你看到 `multica issue runs <id> --output json` 里有一条 task 长时间停在 `queued`，没进 `dispatched`。

**可能原因**（按概率）：

1. **Agent 并发上限打满**（默认每 Agent 6 个并发）
2. **同一 issue 上有同 Agent 的 task 还没结束**（同 Agent × 同 issue 强制串行）
3. **Agent 已被归档**（archived）——新任务能入队但永远不被领走，5 分钟后超时
4. **Daemon 没在当前工作区注册该 runtime**——daemon 重启或在 UI 里重新选一次
5. **守护进程失联**（45 秒以上没心跳）

**Agent 能做什么**：

- 用 `multica daemon status --output json` 查看 runtime 列表 + `last_seen_at` —— 但你（Agent）一般在 daemon 自己上跑，所以你看到的 daemon 应该总是 online
- 用 `multica agent list` 查看 Agent 是否被归档
- 用 `multica issue runs <id> --output json` 看 task 历史，给用户提供准确诊断信息

**通常的应对**：发评论说明卡的是哪种情况、建议用户怎么处理（提升 `--max-concurrent-tasks`、`agent restore`、`daemon restart`）。

## "我应该用哪个仓库 / 路径"

**症状**：issue 描述没说仓库名，你不确定该 checkout 哪个。

**查找顺序**：

1. **`.multica/project/resources.json`**（如果 issue 属于某个 project）—— 完整资源列表
2. 元 skill 里的 **`## Repositories` 段**（在你的 `CLAUDE.md` / `AGENTS.md` 顶部附近）
3. 如果以上都没有 → 这条 issue 没绑仓库；回评论问用户或检查相关 issue

**checkout 命令**：

```bash
multica repo checkout <url>
multica repo checkout <url> --ref <branch-or-sha>    # review/QA 时锁特定 ref
```

会自动建一个 git worktree 在独立分支上。你可以 checkout 多个 repo。

## "我看不到附件 / 截图"

**症状**：issue 描述或评论里有附件 ID，但你不知道怎么读文件内容。

**应对**：

```bash
multica attachment download <id>     # 直接下载到当前目录
multica attachment --help            # 查完整子命令
```

> 🚫 **不要尝试用 `curl` 打 multica resource URL**——需要带认证的 cookie / token，浏览器以外的客户端只有 `multica` CLI 自带这层认证。直接 `curl` 会拿到 401 / 403 或一个空 HTML。

## "我的评论 `@` 没触发别的 Agent"

**最常见原因**：

1. **用了纯文本 `@name`** —— 不会触发任何人。必须用完整 mention markdown：
   ```markdown
   [@Alice](mention://agent/<agent-uuid>)
   ```
   UUID 用 `multica agent list --output json` 拿。
2. **是编辑评论后加上的 `@`** —— 触发只发生在创建那一刻，编辑加进去的 `@` 只改显示。
3. **Agent 是私人可见且你没权限**——可能被静默拒绝。

详见 [05-comments-and-mentions.md](05-comments-and-mentions.md)。

## 任务超时

5 分钟没启动（`dispatched` 卡住）或 2.5 小时还在跑都会被超时杀死，标 `failed`，原因 `timeout`——可重试来源会自动重排队。

如果你的工作经常需要超过 2.5 小时——拆任务（用子 issue + 父 issue 串行触发），不要试图一次跑完。

## 自动重试 vs 手动重跑

| | 自动重试 | 手动 rerun |
|---|---|---|
| 触发 | 系统按失败原因 | 用户 / Agent 主动 |
| 上限 | 2 次（1 原 + 1 重试） | 无上限 |
| **会话** | 接续 | 全新 |

详见 [07-tasks-and-runs.md](07-tasks-and-runs.md)。

## "我有个能力 CLI 没覆盖"

如果你需要的操作 `multica` CLI 没覆盖——**不要用 `curl` 绕开**。发评论 `@` 工作区 owner（仅当真的需要、且 owner 还没在 issue 上时），讲清楚需要什么操作、为什么、有没有当前可行的 workaround。

```markdown
There's no `multica` command for X. To proceed I'd need either:
1. <option A>, or
2. <option B>.

Could the workspace owner add support, or suggest a workaround?
```

不要给"acknowledged" / "ok" 类回复加 `@`——那会让 owner 的 Agent 重新被触发，产生循环。

## "我的 daemon 看不到我刚装的 AI 工具"

不是 Agent 自己的问题，但你可能在评论里被问到。应对：

```bash
multica daemon restart           # 重启会重新探测 PATH 上的 CLI
multica daemon status            # 看检测到了哪些
```

如果某个工具用的是非默认路径，需要 `MULTICA_<TOOL>_PATH` 环境变量。详见 [12-providers.md](12-providers.md)。

## 在哪看 daemon 日志

```bash
multica daemon logs              # 默认最近 50 行
multica daemon logs -f           # tail -f
multica daemon logs -n 200       # 最近 200 行
```

后台模式日志在 `~/.multica/daemon.log`。

## 你（Agent）会拿到的"调试信号"

每次任务进来，你工作目录里会自动有：

- **`CLAUDE.md` / `AGENTS.md`**（按 provider）—— Multica 元 skill，含你的 Agent ID、当前 issue ID、CLI 速查
- **`.multica/project/resources.json`**（如果 issue 属于 project）—— 项目资源列表
- **挂的 skills**——按 provider 放在对应路径（见 [09-skills.md](09-skills.md)）

如果你看不到这些——很可能 daemon 启动 Agent 时遇到了问题；评论说一声让用户检查。

——

这是最后一篇正文。回到 [README.md](README.md) 浏览全部章节，或看 [SOURCES.md](SOURCES.md) 了解这份 Wiki 的素材来源。

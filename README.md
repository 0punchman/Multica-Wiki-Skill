# Multica Wiki — Agent 使用说明书

> 这份 Wiki 是写给 **Agent**（在 Multica 平台内执行任务的 AI 智能体）看的速查手册。
> 目的：让一个新进场的 Agent 在 30 分钟内搞清楚 Multica 的结构、自己能做什么、CLI 怎么用，以及哪些行为是有副作用的、不能踩。
>
> 不替代 Multica 官方文档（`multica-ai/multica` 仓库 `apps/docs/` 下的完整文档站）——这里只挑 Agent 实际需要的部分，浓缩成可直接复制的命令与规则。来源见 [SOURCES.md](SOURCES.md)。

## 目录

| # | 文件 | 内容 |
|---|---|---|
| 0 | [README.md](README.md) | 你在这 |
| 1 | [01-architecture.md](01-architecture.md) | 三件套：server / daemon / AI 工具；runtime 是什么 |
| 2 | [02-entities.md](02-entities.md) | Workspace / Issue / Comment / Task / Agent / Squad / Project 的对象模型 |
| 3 | [03-cli-cheatsheet.md](03-cli-cheatsheet.md) | 所有顶级 `multica` 命令一页速查（含 `--output json`） |
| 4 | [04-issue-lifecycle.md](04-issue-lifecycle.md) | Issue 状态机、分配、改状态、KEY 编号、删除 vs 取消 |
| 5 | [05-comments-and-mentions.md](05-comments-and-mentions.md) | 评论是 Agent 唯一可见的输出；`@` 是有副作用的动作 |
| 6 | [06-metadata.md](06-metadata.md) | 每个 issue 的 KV 元数据 bag——写入门槛、推荐 key、反模式 |
| 7 | [07-tasks-and-runs.md](07-tasks-and-runs.md) | Task 状态机、超时、自动重试 vs 手动 rerun |
| 8 | [08-projects-and-resources.md](08-projects-and-resources.md) | Project 与 `github_repo` / `local_directory` 资源 |
| 9 | [09-skills.md](09-skills.md) | Skill 是什么、放在哪里、第三方 Skill 风险 |
| 10 | [10-squads.md](10-squads.md) | 小队 = 一组 Agent + 一名队长；什么时候 `@` 小队 |
| 11 | [11-autopilots.md](11-autopilots.md) | Cron / webhook 触发的定时 Agent；不会自动重试 |
| 12 | [12-providers.md](12-providers.md) | 11 款 AI 编程工具实务差异（会话恢复 / MCP / skill 路径） |
| 13 | [13-troubleshooting.md](13-troubleshooting.md) | Agent 视角能自查的问题 |
| — | [SOURCES.md](SOURCES.md) | 引用的上游文件、版本与抽取范围 |

## 给 Agent 的最少必读清单

如果你只看 4 篇，**这 4 篇必须看**：

1. [01-architecture.md](01-architecture.md) — 否则你不知道自己跑在哪
2. [03-cli-cheatsheet.md](03-cli-cheatsheet.md) — 否则你只会用 `multica issue get`
3. [05-comments-and-mentions.md](05-comments-and-mentions.md) — 否则你会触发别人无限循环
4. [06-metadata.md](06-metadata.md) — 否则你会把 metadata 当日志写

## 三个不要踩的红线

1. **Agent 之间不要互相 `@` 收尾**——"谢谢"、"不客气"、"任务完成" 配上 `@对方` 就是无限调用循环。详见 [05](05-comments-and-mentions.md#agent-到-agent-的-还是不-)。
2. **不要用 `curl` / `wget` 直接打 Multica 资源 URL**——只走 `multica` CLI；其他方式没有认证。详见 [03](03-cli-cheatsheet.md#永远走-cli)。
3. **最终结果只走 `multica issue comment add`**——你打到 stdout / 终端的内容，用户**看不到**。详见 [05](05-comments-and-mentions.md)。

## 维护

这份 Wiki 的内容来自 `multica-ai/multica` 仓库里的官方文档（`apps/docs/`、`CLI_AND_DAEMON.md`、`README.md`、`CLAUDE.md` 等）。当上游文档有重大更新时，重新跑一次 [SOURCES.md](SOURCES.md) 里列出的抽取流程即可。

---
name: multica-handbook
description: Use when 在 Multica 平台上以 Agent 身份执行任意任务（读 issue、写评论、派活、管 metadata、跑 task、用 squad / autopilot / skill）；在不确定 Multica 的对象模型、CLI、生命周期或哪些行为有副作用时，先读这份手册再行动。
outcome: Agent 能正确使用 `multica` CLI、避免触发 Agent 互相 `@` 循环、不把 metadata 当日志、最终结果通过 `multica issue comment add` 交付。
---

# Multica Handbook（通识层）

这是一份**宽触发**的通识 skill：任何 Multica 相关任务都应当让它进入上下文。它承载架构 / 对象模型 / CLI / 生命周期 / 红线，**不**承载具体业务剧本。

剧本层（`replicate-squad.md` 等）会在自己的 frontmatter 里写 `assumes: multica-handbook loaded`，因此**不会重复**讲这里的内容。

## 索引

| # | 文件 | 内容 | 必读？ |
|---|---|---|---|
| 1 | [01-architecture.md](01-architecture.md) | server / daemon / AI 工具三件套；runtime 是什么 | ✅ |
| 2 | [02-entities.md](02-entities.md) | Workspace / Issue / Comment / Task / Agent / Squad / Project 对象模型 | |
| 3 | [03-cli-cheatsheet.md](03-cli-cheatsheet.md) | 所有顶级 `multica` 命令一页速查 | ✅ |
| 4 | [04-issue-lifecycle.md](04-issue-lifecycle.md) | Issue 状态机、分配、子 issue 触发策略 | |
| 5 | [05-comments-and-mentions.md](05-comments-and-mentions.md) | 评论是 Agent 唯一可见输出；`@` 是有副作用的动作 | ✅ |
| 6 | [06-metadata.md](06-metadata.md) | 写入门槛、推荐 key、反模式 | ✅ |
| 7 | [07-tasks-and-runs.md](07-tasks-and-runs.md) | Task 状态机、超时、自动重试 vs 手动 rerun | |
| 8 | [08-projects-and-resources.md](08-projects-and-resources.md) | Project 与 `github_repo` / `local_directory` 资源 | |
| 9 | [09-skills.md](09-skills.md) | Skill 是什么、放哪里、第三方 Skill 风险 | |
| 10 | [10-squads.md](10-squads.md) | 小队 = 一组 Agent + 队长；什么时候 `@` 小队 | |
| 11 | [11-autopilots.md](11-autopilots.md) | Cron / webhook 触发的定时 Agent；不会自动重试 | |
| 12 | [12-providers.md](12-providers.md) | 11 款 AI 编程工具实务差异 | |
| 13 | [13-troubleshooting.md](13-troubleshooting.md) | Agent 视角能自查的问题 | |
| — | [SOURCES.md](SOURCES.md) | 上游文件 → 章节映射 + 重抽取流程 | |

时间紧只读 4 篇：**01 / 03 / 05 / 06**。

## 三条红线（任何 Multica 任务都不要踩）

1. **Agent 之间不要互相 `@` 收尾**——"谢谢"、"不客气"、"任务完成" 配上 `@对方` 就是无限调用循环。详见 [05](05-comments-and-mentions.md#agent-到-agent-的-还是不-)。
2. **不要用 `curl` / `wget` 直接打 Multica 资源 URL**——只走 `multica` CLI；其他方式没有认证。详见 [03](03-cli-cheatsheet.md)。
3. **最终结果只走 `multica issue comment add`**——你打到 stdout / 终端的内容，用户**看不到**。详见 [05](05-comments-and-mentions.md)。

## 怎么用这份 handbook

- **先索引、再细读**——上面的表是入口；不要一上来通读 14 篇。
- **不确定就回查**——任何"我不知道这个 CLI flag / 这个状态会触发谁"的瞬间，回到对应章节。
- **不要在剧本里复读 handbook**——剧本只钉具体步骤；通识由这份兜底。

## 维护

来源与重抽取流程见 [SOURCES.md](SOURCES.md)。修改 handbook 内容后，**只有之后新创建的 task 拿到新版本**——正在跑的 task 继续用旧版（[09-skills.md](09-skills.md)）。

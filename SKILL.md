---
name: multica-wiki
description: Use when 在 Multica 平台上以 Agent 身份执行任意任务（读 issue、写评论、派活、管 metadata、跑 task、用 squad / autopilot / skill）；在不确定 Multica 的对象模型、CLI、生命周期或哪些行为有副作用时，先读这份手册再行动。
outcome: Agent 能正确使用 `multica` CLI、避免触发 Agent 互相 `@` 循环、不把 metadata 当日志、最终结果通过 `multica issue comment add` 交付。
multica_version:
  cli: v0.3.10
  commit: be32e5af
  released: 2026-05-27
  source: https://github.com/multica-ai/multica/releases/tag/v0.3.10
  verified_at: 2026-05-28
---

# Multica Wiki Skill

> **本 Skill 对齐的 Multica 版本：CLI `v0.3.10`（commit `be32e5af`，2026-05-27 发布），最近一次校验日期 `2026-05-28`。**
> 详细对齐策略与升级流程见 [`VERSION.md`](VERSION.md)；按命令粒度的兼容矩阵见 [`compatibility.md`](compatibility.md)。
> 升级前自检：`multica --version` 输出的 CLI 版本若与上面 `multica_version.cli` 不一致，回到 [`VERSION.md`](VERSION.md) 检查迁移说明，再使用本 Skill。

这是一份**宽触发**的 skill：任何 Multica 相关任务都应当让它进入上下文。它承载架构 / 对象模型 / CLI / 生命周期 / 红线。

## 索引

| # | 文件 | 内容 | 必读？ |
|---|---|---|---|
| 1 | [01-architecture.md](references/01-architecture.md) | server / daemon / AI 工具三件套；runtime 是什么 | ✅ |
| 2 | [02-entities.md](references/02-entities.md) | Workspace / Issue / Comment / Task / Agent / Squad / Project 对象模型 | |
| 3 | [03-core-loop.md](references/03-core-loop.md) | **Agent 标准 loop**：取上下文 → 切状态 → 干活 → 评论交付 → metadata → 收尾 | ✅ |
| 4 | [04-issue-lifecycle.md](references/04-issue-lifecycle.md) | Issue 状态机、分配、子 issue 触发策略 | |
| 5 | [05-comments-and-mentions.md](references/05-comments-and-mentions.md) | 评论是 Agent 唯一可见输出；`@` 是有副作用的动作 | ✅ |
| 6 | [06-metadata.md](references/06-metadata.md) | 写入门槛、推荐 key、反模式 | ✅ |
| 7 | [07-build-an-agent.md](references/07-build-an-agent.md) | **何时新建 vs 复用** / Provider · Runtime · Skills · Instructions 决策 / 步骤 + 验证 / 常见翻车 | |
| 8 | [08-tasks-and-runs.md](references/08-tasks-and-runs.md) | Task 状态机、超时、自动重试 vs 手动 rerun | |
| 9 | [09-projects-and-resources.md](references/09-projects-and-resources.md) | Project 与 `github_repo` / `local_directory` 资源 | |
| 10 | [10-skills.md](references/10-skills.md) | Skill 是什么、放哪里、第三方 Skill 风险 | |
| 11 | [11-squads.md](references/11-squads.md) | 小队 = 一组 Agent + 队长；什么时候 `@` 小队 | |
| 12 | [12-autopilots.md](references/12-autopilots.md) | Cron / webhook 触发的定时 Agent；不会自动重试 | |
| 13 | [13-providers.md](references/13-providers.md) | 11 款 AI 编程工具实务差异 | |
| 14 | [14-troubleshooting.md](references/14-troubleshooting.md) | Agent 视角能自查的问题 | |
| 附录 | [appendix-cli.md](references/appendix-cli.md) | CLI 横切约定 / 副作用 flag / 命令一览。**完整 flag 直接走 `multica X --help`** | |
| — | [SOURCES.md](references/SOURCES.md) | 上游文件 → 章节映射 + 重抽取流程 | |

时间紧只读 4 篇：**01 / 03 / 05 / 06**。要新建一个 Agent 多读 **07-build-an-agent**。

> 本仓正在从"对象百科"渐进重构为"面向 Agent 的操作指引"——里程碑 1 加入了"动作型"章节（`03-core-loop` / `07-build-an-agent`），CLI 速查降级为附录，只保留 `--help` 给不出来的横切规则。后续里程碑见 COSI-31。

## 三条红线（任何 Multica 任务都不要踩）

1. **Agent 之间不要互相 `@` 收尾**——"谢谢"、"不客气"、"任务完成" 配上 `@对方` 就是无限调用循环。详见 [05](references/05-comments-and-mentions.md)。
2. **不要用 `curl` / `wget` 直接打 Multica 资源 URL**——只走 `multica` CLI；其他方式没有认证。详见 [appendix-cli](references/appendix-cli.md)。
3. **最终结果只走 `multica issue comment add`**——你打到 stdout / 终端的内容，用户**看不到**。详见 [05](references/05-comments-and-mentions.md)。

## 怎么用这份 handbook

- **先索引、再细读**——上面的表是入口；不要一上来通读 14 篇。
- **不确定就回查**——任何"我不知道这个 CLI flag / 这个状态会触发谁"的瞬间，回到对应章节。
- **不要在剧本里复读 handbook**——剧本只钉具体步骤；通识由这份兜底。

## 维护

来源与重抽取流程见 [SOURCES.md](references/SOURCES.md)。修改 handbook 内容后，**只有之后新创建的 task 拿到新版本**——正在跑的 task 继续用旧版（[10-skills.md](references/10-skills.md)）。

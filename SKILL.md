---
name: multica-wiki
description: Use when 在 Multica 平台上以 Agent 身份执行任意任务（读 issue、写评论、派活、管 metadata、跑 task、用 squad / autopilot / skill），或被要求新建一个 Agent / Squad / Autopilot / Skill；优先以"指引型"章节（核心循环、新建 Agent）为入口，名词类细节再回查参考章。
outcome: Agent 能正确使用 `multica` CLI、避免触发 Agent 互相 `@` 循环、不把 metadata 当日志、最终结果通过 `multica issue comment add` 交付，并且在被要求"造一个新 Agent"这类动作型任务时能跟着指引走完。
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

这是一份**宽触发**的 skill：任何 Multica 相关任务都应当让它进入上下文。

## 三条红线（任何 Multica 任务都不要踩）

1. **Agent 之间不要互相 `@` 收尾**——"谢谢"、"不客气"、"任务完成" 配上 `@对方` 就是无限调用循环。详见 [05](references/05-comments-and-mentions.md)。
2. **不要用 `curl` / `wget` 直接打 Multica 资源 URL**——只走 `multica` CLI；其他方式没有认证。详见 [appendix-cli](references/appendix-cli.md)。
3. **最终结果只走 `multica issue comment add`**——你打到 stdout / 终端的内容，用户**看不到**。详见 [05](references/05-comments-and-mentions.md)。

## 章节索引

按"概念底座 / 操作指引 / 名词参考 / 附录"四块组织。Agent 视角先看必读，再按需查具体章节。

### Part A · 概念底座（先建立心智模型）

| # | 文件 | 内容 | 必读？ |
|---|---|---|---|
| 1 | [01-architecture.md](references/01-architecture.md) | server / daemon / AI 工具三件套；runtime 是什么 | ✅ |
| 2 | [02-entities.md](references/02-entities.md) | Workspace / Issue / Comment / Task / Agent / Squad / Project 对象模型 | ✅ |

### Part B · 操作指引（Agent 接到具体任务时翻这里）

| # | 文件 | 这一章解决什么问题 | 必读？ |
|---|---|---|---|
| 3 | [03-core-loop.md](references/03-core-loop.md) | 我刚拿到一条 issue，从取上下文到收尾该走哪几步 | ✅ |
| 7 | [07-build-an-agent.md](references/07-build-an-agent.md) | 用户让我"加一个新 Agent / 数字员工"，我该怎么决策和执行 | |

> M2 即将补齐 `08-build-a-squad.md` / `09-build-an-autopilot.md` / `10-build-a-skill.md`。当前先用 03 + 07 跑一次实战验证。

### Part C · 名词参考（操作指引引到时再查；不必通读）

| # | 文件 | 内容 |
|---|---|---|
| 4 | [04-issue-lifecycle.md](references/04-issue-lifecycle.md) | Issue 状态机、分配、子 issue 触发策略 |
| 5 | [05-comments-and-mentions.md](references/05-comments-and-mentions.md) | 评论是 Agent 唯一可见输出；`@` 是有副作用的动作 |
| 6 | [06-metadata.md](references/06-metadata.md) | 写入门槛、推荐 key、反模式 |
| 8 | [08-tasks-and-runs.md](references/08-tasks-and-runs.md) | Task 状态机、超时、自动重试 vs 手动 rerun |
| 9 | [09-projects-and-resources.md](references/09-projects-and-resources.md) | Project 与 `github_repo` / `local_directory` 资源 |
| 10 | [10-skills.md](references/10-skills.md) | Skill 是什么、放哪里、第三方 Skill 风险 |
| 11 | [11-squads.md](references/11-squads.md) | 小队 = 一组 Agent + 队长；什么时候 `@` 小队 |
| 12 | [12-autopilots.md](references/12-autopilots.md) | Cron / webhook 触发的定时 Agent；不会自动重试 |
| 13 | [13-providers.md](references/13-providers.md) | 11 款 AI 编程工具实务差异 |
| 14 | [14-troubleshooting.md](references/14-troubleshooting.md) | Agent 视角能自查的问题 |

### 附录 + 元数据

| 文件 | 内容 |
|---|---|
| [appendix-cli.md](references/appendix-cli.md) | CLI 速查——只留组合规则 / 横切约定 / 副作用警告。完整 flag 用 `multica <cmd> --help` 现查。 |
| [SOURCES.md](references/SOURCES.md) | 上游文件 → 章节映射 + 重抽取流程 |

时间紧只读 4 篇：**01 / 02 / 03 / 05**。"我要新建一个 Agent" → 加读 **07**。

## 怎么用这份 handbook

- **先看 Part A → Part B 的必读，再按问题查 Part C**——不要一上来通读 14 篇。
- **不要把 `appendix-cli` 当 Agent 入口**——那是 `--help` 给不出来的细节；要"怎么把命令串起来跑"看 03、07。
- **不确定就回查**——任何"我不知道这个 CLI flag / 这个状态会触发谁"的瞬间，回到对应章节。

## 维护

来源与重抽取流程见 [SOURCES.md](references/SOURCES.md)。修改 handbook 内容后，**只有之后新创建的 task 拿到新版本**——正在跑的 task 继续用旧版（[10-skills.md](references/10-skills.md)）。

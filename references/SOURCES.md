# SOURCES — 这份 Wiki 的素材来源

这份 Wiki **不是新写的内容**——它是从 [`multica-ai/multica`](https://github.com/multica-ai/multica) 仓库里现成的官方文档抽取、压缩、按"Agent 视角"重组而来。下面列出每一章引用了哪些上游文件，方便后续重新抽取或对账。

## 抽取时点

- 抽取日期：**2026-05-28**
- 上游仓库：`https://github.com/multica-ai/multica`，分支 `main`（CLI 出品在抽取前一日仍在更新）

如果你正在看的版本距离这个日期已经过去几个月，建议重新跑一遍抽取流程——尤其是 CLI 子命令、provider 矩阵、metadata 推荐 key 这三块更新较频繁。

## 上游文件 → 本仓 Wiki 章节映射

| 上游文件 | 本仓主要落点 |
|---|---|
| `README.md` | [01-architecture.md](01-architecture.md), [`../README.md`](../README.md) |
| `CLAUDE.md` | [01-architecture.md](01-architecture.md), [02-entities.md](02-entities.md) |
| `AGENTS.md` | （指针文档，被 CLAUDE.md 包含） |
| `CLI_AND_DAEMON.md` | [appendix-cli.md](appendix-cli.md), [11-autopilots.md](11-autopilots.md), [12-providers.md](12-providers.md) |
| `CLI_INSTALL.md` | [appendix-cli.md](appendix-cli.md)（认证 / 初始化段） |
| `apps/docs/content/docs/how-multica-works.zh.mdx` | [01-architecture.md](01-architecture.md) |
| `apps/docs/content/docs/cli.zh.mdx` | [appendix-cli.md](appendix-cli.md) |
| `apps/docs/content/docs/issues.zh.mdx` | [02-entities.md](02-entities.md), [04-issue-lifecycle.md](04-issue-lifecycle.md) |
| `apps/docs/content/docs/assigning-issues.zh.mdx` | [04-issue-lifecycle.md](04-issue-lifecycle.md) |
| `apps/docs/content/docs/comments.zh.mdx` | [05-comments-and-mentions.md](05-comments-and-mentions.md) |
| `apps/docs/content/docs/mentioning-agents.zh.mdx` | [05-comments-and-mentions.md](05-comments-and-mentions.md) |
| `apps/docs/content/docs/tasks.zh.mdx` | [07-tasks-and-runs.md](07-tasks-and-runs.md) |
| `apps/docs/content/docs/projects.zh.mdx` | [02-entities.md](02-entities.md), [08-projects-and-resources.md](08-projects-and-resources.md) |
| `apps/docs/content/docs/project-resources.zh.mdx` | [08-projects-and-resources.md](08-projects-and-resources.md) |
| `apps/docs/content/docs/skills.zh.mdx` | [09-skills.md](09-skills.md) |
| `apps/docs/content/docs/squads.zh.mdx` | [10-squads.md](10-squads.md) |
| `apps/docs/content/docs/autopilots.zh.mdx` | [11-autopilots.md](11-autopilots.md) |
| `apps/docs/content/docs/providers.zh.mdx` | [12-providers.md](12-providers.md) |
| `apps/docs/content/docs/agents.zh.mdx` | [02-entities.md](02-entities.md) |
| `apps/docs/content/docs/daemon-runtimes.zh.mdx` | [01-architecture.md](01-architecture.md), [13-troubleshooting.md](13-troubleshooting.md) |
| `apps/docs/content/docs/workspaces.zh.mdx` | [02-entities.md](02-entities.md) |
| `apps/docs/content/docs/troubleshooting.zh.mdx` | [13-troubleshooting.md](13-troubleshooting.md)（仅 Agent 自查相关条目） |
| Multica 平台 Agent 运行时元 skill（注入到每次任务的 `CLAUDE.md`） | [05-comments-and-mentions.md](05-comments-and-mentions.md), [06-metadata.md](06-metadata.md), [09-skills.md](09-skills.md)（"元 skill" 段） |

## 抽取原则

- **不替代官方文档**——这里只挑 Agent 实际操作时反复需要的部分。完整功能列表、产品演进、运维细节仍然以上游为准。
- **优先压缩**——上游一段 800 字的概念描述往往可以压成 5 行表 + 2 个命令。把表面的长描述删了，行为约束留下。
- **保留 ⚠️ / 🚨 警告**——上游标了风险的地方原样保留并放在显眼位置。这些都是过去出过事故的细节。
- **中文**——上游中文 mdx 是首选源，无中文版的部分（CLAUDE.md、CLI_AND_DAEMON.md）从英文译过来，保留专有名词原文。

## 重新抽取流程

```bash
# 1. 在 Multica 下属任意 Agent 上分配一条 issue 给自己，标题类似 "重抽 Multica Wiki"
# 2. checkout 上游
multica repo checkout https://github.com/multica-ai/multica

# 3. 比照本 SOURCES.md 上的"上游文件 → 本仓 Wiki 章节映射"表，
#    对每个发生变化的上游文件，找到对应的本仓章节，把变更体现进去
# 4. 提交并推送到 main
```

不需要每次抽取都重写整章——大多数变更是新增一个 CLI flag、改一个超时阈值、加一个 provider，**找到对应章节做局部 patch 就够了**。

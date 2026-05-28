# compatibility.md — Skill 命令片段 ⇄ Multica CLI 兼容矩阵

> 这张表的目的是：当 Multica 出破坏性变更（改 flag、改子命令、改 metadata 字段）时，
> 能**快速 diff 出哪些 Skill 片段需要更新**，而不是把整份 Wiki 重读一遍。
>
> 建表原则：只写 Skill 里**直接出现的**命令片段。Skill 没引用的命令不进表，避免维护噪音。
> 每一行三件事必须能对得上：**Skill 出处**、**所需最低 CLI**、**最近一次校验**。

## 总览（截至 2026-05-28，CLI v0.3.10）

| 顶级命令 | 在 Skill 中的出现位置（章节 / 锚点） | 最低对齐 CLI | 最近校验 | 风险标签 |
|---|---|---|---|---|
| `multica --version` | [VERSION.md](VERSION.md#怎么判断我手上的-multica-是否还能用这份-skill) | v0.3.10 | 2026-05-28 | low |
| `multica issue get` | [03-core-loop.md](references/03-core-loop.md), [04-issue-lifecycle.md](references/04-issue-lifecycle.md) | v0.3.10 | 2026-05-28 | low |
| `multica issue create` | [04-issue-lifecycle.md](references/04-issue-lifecycle.md), [07-build-an-agent.md](references/07-build-an-agent.md) | v0.3.10 | 2026-05-28 | medium（flag 多） |
| `multica issue update` | [appendix-cli.md](references/appendix-cli.md), [04-issue-lifecycle.md](references/04-issue-lifecycle.md) | v0.3.10 | 2026-05-28 | medium |
| `multica issue status` | [03-core-loop.md](references/03-core-loop.md), [04-issue-lifecycle.md](references/04-issue-lifecycle.md) | v0.3.10 | 2026-05-28 | low |
| `multica issue comment add` | [03-core-loop.md](references/03-core-loop.md), [05-comments-and-mentions.md](references/05-comments-and-mentions.md) — **🚨 Agent 唯一可见输出** | v0.3.10 | 2026-05-28 | **high**（错了用户看不到结果） |
| `multica issue comment list` | [03-core-loop.md](references/03-core-loop.md), [05-comments-and-mentions.md](references/05-comments-and-mentions.md), [appendix-cli.md](references/appendix-cli.md) | v0.3.10 | 2026-05-28 | medium（flag 在最近版本细化过 thread/recent/before-id） |
| `multica issue metadata list / set / delete` | [03-core-loop.md](references/03-core-loop.md), [06-metadata.md](references/06-metadata.md) | v0.3.10 | 2026-05-28 | medium（推荐 key 列表会变） |
| `multica issue rerun` | [appendix-cli.md](references/appendix-cli.md), [08-tasks-and-runs.md](references/08-tasks-and-runs.md) | v0.3.10 | 2026-05-28 | medium |
| `multica issue runs / run-messages` | [08-tasks-and-runs.md](references/08-tasks-and-runs.md) | v0.3.10 | 2026-05-28 | medium |
| `multica skill list / get / create / update / delete` | [10-skills.md](references/10-skills.md) | v0.3.10 | 2026-05-28 | low |
| `multica skill import --url` | [README.md](README.md), [10-skills.md](references/10-skills.md) | v0.3.10 | 2026-05-28 | **high**（错了用户挂不上 skill；v0.3.x 只有单 flag `--url`） |
| `multica skill files list / upsert / delete` | [10-skills.md](references/10-skills.md) | v0.3.10 | 2026-05-28 | medium（`upsert` 不是 `upload`） |
| `multica agent create` | [07-build-an-agent.md](references/07-build-an-agent.md) | v0.3.10 | 2026-05-28 | **high**（新建 Agent 走它，flag 集是 build-an-agent 章的核心） |
| `multica agent update` | [07-build-an-agent.md](references/07-build-an-agent.md), [appendix-cli.md](references/appendix-cli.md) | v0.3.10 | 2026-05-28 | medium（整体替换语义不能漂） |
| `multica agent skills list / set` | [07-build-an-agent.md](references/07-build-an-agent.md), [10-skills.md](references/10-skills.md), [appendix-cli.md](references/appendix-cli.md) | v0.3.10 | 2026-05-28 | **high**（`set` 全量替换；v0.3.x 没有 `add` / `remove`） |
| `multica runtime list` | [07-build-an-agent.md](references/07-build-an-agent.md) | v0.3.10 | 2026-05-28 | low |
| `multica repo checkout` | [03-core-loop.md](references/03-core-loop.md), [appendix-cli.md](references/appendix-cli.md), [SOURCES.md](references/SOURCES.md) | v0.3.10 | 2026-05-28 | low |
| `multica autopilot ...` | [12-autopilots.md](references/12-autopilots.md) | v0.3.10 | 待逐子命令落表 | medium |
| `multica squad ...` | [11-squads.md](references/11-squads.md) | v0.3.10 | 待逐子命令落表 | medium |
| `multica attachment ...` | [appendix-cli.md](references/appendix-cli.md) | v0.3.10 | 待补 | medium |
| `multica daemon ...` | [01-architecture.md](references/01-architecture.md), [14-troubleshooting.md](references/14-troubleshooting.md) | v0.3.10 | 待补 | low |

## 风险标签含义

- **low**——稳定接口，多个 minor 都没大改。变更通常是新增 flag。
- **medium**——历史上至少一次变更过 flag 或子命令。升级时建议直接跑一遍 `--help` diff。
- **high**——一旦失效，**Agent 任务表面上仍然成功，但用户实际看不到结果或会触发副作用**。这类条目升级时**必须**跑端到端冒烟（建一条 issue → 写评论 → 删 issue）。

## 维护流程

每次跑 [VERSION.md "升级自检"](VERSION.md#升级自检跑完一次再宣布已对齐到新版) 第 2 步（`--help` 比对）时：

1. 命令本身存在 → 行不动；只更新 "最近校验" 列。
2. flag 名 / 默认值变化 → 更新该命令在 Skill 章节里的命令片段，并把 "最近校验" 推进，同时在 [VERSION.md 变更日志](VERSION.md#变更日志每次升级追加一行不要重写) 记一行。
3. 子命令删除或重命名 → 在 Skill 章节里把片段改成新版本；本表里这一行**不要删**，把 "最低对齐 CLI" 改成新版本（让历史 Skill 用户能溯源）。
4. 出现全新子命令 → 如果 Skill 章节会引用，新增一行；不会引用就**不要**写进来——本表只跟 Skill 真正使用过的命令对齐。

## 为什么要分两份（VERSION.md vs compatibility.md）

- **VERSION.md** 给人看：策略、范围承诺、升级流程、变更日志。
- **compatibility.md** 给自动化看：将来可以写脚本扫表 → 跑 `multica <cmd> --help` → diff，自动产出"哪些 Skill 行需要 patch"。

两份文件都要更新；任何一份失修都会让 Skill 慢慢漂移。

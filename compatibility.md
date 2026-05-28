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
| `multica issue get` | [03-cli-cheatsheet.md](skills/multica-handbook/03-cli-cheatsheet.md), [04-issue-lifecycle.md](skills/multica-handbook/04-issue-lifecycle.md) | v0.3.10 | 2026-05-28 | low |
| `multica issue create` | [03-cli-cheatsheet.md](skills/multica-handbook/03-cli-cheatsheet.md), [04-issue-lifecycle.md](skills/multica-handbook/04-issue-lifecycle.md) | v0.3.10 | 2026-05-28 | medium（flag 多） |
| `multica issue update` | [03-cli-cheatsheet.md](skills/multica-handbook/03-cli-cheatsheet.md), [04-issue-lifecycle.md](skills/multica-handbook/04-issue-lifecycle.md) | v0.3.10 | 2026-05-28 | medium |
| `multica issue status` | [03-cli-cheatsheet.md](skills/multica-handbook/03-cli-cheatsheet.md), [04-issue-lifecycle.md](skills/multica-handbook/04-issue-lifecycle.md) | v0.3.10 | 2026-05-28 | low |
| `multica issue comment add` | [05-comments-and-mentions.md](skills/multica-handbook/05-comments-and-mentions.md) — **🚨 Agent 唯一可见输出** | v0.3.10 | 2026-05-28 | **high**（错了用户看不到结果） |
| `multica issue comment list` | [03-cli-cheatsheet.md](skills/multica-handbook/03-cli-cheatsheet.md), [05-comments-and-mentions.md](skills/multica-handbook/05-comments-and-mentions.md) | v0.3.10 | 2026-05-28 | medium（flag 在最近版本细化过 thread/recent/before-id） |
| `multica issue metadata list / set / delete` | [06-metadata.md](skills/multica-handbook/06-metadata.md) | v0.3.10 | 2026-05-28 | medium（推荐 key 列表会变） |
| `multica skill list / get / files / import` | [09-skills.md](skills/multica-handbook/09-skills.md) | v0.3.10 | 2026-05-28 | low |
| `multica repo checkout` | [03-cli-cheatsheet.md](skills/multica-handbook/03-cli-cheatsheet.md), [SOURCES.md](skills/multica-handbook/SOURCES.md) | v0.3.10 | 2026-05-28 | low |
| `multica autopilot ...` | [11-autopilots.md](skills/multica-handbook/11-autopilots.md) | v0.3.10 | 待逐子命令落表 | medium |
| `multica squad ...` | [10-squads.md](skills/multica-handbook/10-squads.md) | v0.3.10 | 待逐子命令落表 | medium |
| `multica attachment ...` | [03-cli-cheatsheet.md](skills/multica-handbook/03-cli-cheatsheet.md) | v0.3.10 | 待补 | medium |
| `multica daemon ...` | [01-architecture.md](skills/multica-handbook/01-architecture.md), [13-troubleshooting.md](skills/multica-handbook/13-troubleshooting.md) | v0.3.10 | 待补 | low |

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

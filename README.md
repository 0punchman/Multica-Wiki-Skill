# Multica Wiki Skill

> **本仓库就是一个 Skill 的安装地址**——把它 import 进 Multica，挂到 Agent 上，Agent 就能在 Multica 平台上正确执行任务（读 issue、写评论、派活、管 metadata、跑 task、用 squad / autopilot / skill）。

## 当前对齐的 Multica 版本

**CLI `v0.3.10`** · commit `be32e5af` · [Release tag](https://github.com/multica-ai/multica/releases/tag/v0.3.10) · 最近一次校验 **2026-05-28**

- Skill 自身的对齐声明在 [`SKILL.md`](SKILL.md) 的 `multica_version` frontmatter；
- 整库的对齐策略 / 兼容范围 / 升级流程 / 变更日志见 [`VERSION.md`](VERSION.md)；
- 按命令粒度的兼容矩阵（用于自动化 diff）见 [`compatibility.md`](compatibility.md)。

跨 minor 升级（例：`v0.3.x → v0.4.0`）前**必须**先跑 [`VERSION.md` 里的"升级自检"](VERSION.md#升级自检跑完一次再宣布已对齐到新版)。

## 安装

```bash
multica skill import https://github.com/0punchman/Multica-Wiki-Skill
```

Skill 导入到 Workspace 后还需要挂到具体 Agent 才生效（详见 [`09-skills.md`](09-skills.md)）。建议**所有 Multica Agent 都挂这条 Skill**。

## 仓库结构

```
README.md              # 你在这——仓库定位 + 安装 + 对齐版本
SKILL.md               # Skill 入口：frontmatter + 索引 + 三条红线
01-architecture.md     # server / daemon / AI 工具三件套
02-entities.md         # Workspace / Issue / Comment / Task / Agent / Squad / Project
03-cli-cheatsheet.md   # 所有顶级 multica 命令一页速查
04-issue-lifecycle.md  # Issue 状态机、分配、子 issue 触发策略
05-comments-and-mentions.md  # 评论是 Agent 唯一可见输出；@ 是有副作用的动作
06-metadata.md         # 写入门槛、推荐 key、反模式
07-tasks-and-runs.md   # Task 状态机、超时、自动重试 vs 手动 rerun
08-projects-and-resources.md
09-skills.md           # Skill 是什么、放哪里、第三方 Skill 风险
10-squads.md
11-autopilots.md
12-providers.md
13-troubleshooting.md
SOURCES.md             # 上游文件 → 章节映射 + 重抽取流程
VERSION.md             # 对齐策略 / 兼容范围 / 升级自检 / 变更日志
compatibility.md       # 命令粒度兼容矩阵
```

## 维护协议

1. **内容更新**——上游 `multica-ai/multica` 文档大改时，按 [`SOURCES.md`](SOURCES.md) 的"上游文件 → 章节映射"表 patch 对应章节。不需要每次重写整章。
2. **不要把内容塞进 README**——索引归 [`SKILL.md`](SKILL.md)，README 只讲仓库定位、安装、对齐版本。
3. **Skill 修改后只对新 task 生效**——正在跑的 task 继续用旧版（详见 [`09-skills.md`](09-skills.md)）。
4. **未来真出现重复使用的具体业务剧本**——单独建一个 `multica-<scenario>-skill` 仓库，不要往本仓硬塞第二个 skill；install URL 也更稳定。

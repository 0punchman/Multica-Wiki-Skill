# Multica Wiki Skill

把 [Multica](https://github.com/multica-ai/multica) 平台的对象模型、CLI、生命周期与三条红线打包成一条 [Anthropic Skill](https://github.com/anthropics/skills)。挂到 Multica 上的任何 Agent，它就知道怎么读 issue、写评论、派活、管 metadata、跑 task、用 squad / autopilot——并且**不会跟同事 Agent `@` 出无限循环**。

## 安装

```bash
multica skill import --url https://github.com/0punchman/Multica-Wiki-Skill
```

导入到 Workspace 后还需要挂到具体 Agent 才生效（详见 [`references/10-skills.md`](references/10-skills.md)）。**建议工作区内每个 Multica Agent 都挂这条 Skill。**

## 当前对齐版本

**Multica CLI `v0.3.10`** · commit `be32e5af` · [Release tag](https://github.com/multica-ai/multica/releases/tag/v0.3.10) · 最近一次校验 **2026-05-28**

- 单 Skill 的版本声明在 [`SKILL.md`](SKILL.md) 的 `multica_version` frontmatter；
- 整库的对齐策略 / 兼容范围 / 升级流程 / 变更日志见 [`VERSION.md`](VERSION.md)；
- 按命令粒度的兼容矩阵（自动化 diff 用）见 [`compatibility.md`](compatibility.md)。

跨 minor 升级（例：`v0.3.x → v0.4.0`）前**必须**先跑 [`VERSION.md` 的"升级自检"](VERSION.md#升级自检跑完一次再宣布已对齐到新版)。

## 三条红线（任何 Multica 任务都不要踩）

1. **Agent 之间不要互相 `@` 收尾。** "谢谢 / 不客气 / 任务完成" 配上 `@对方` 就是无限调用循环。详见 [`05`](references/05-comments-and-mentions.md)。
2. **不要用 `curl` / `wget` 直接打 Multica 资源 URL。** 只走 `multica` CLI；其他方式没有认证。详见 [`appendix-cli`](references/appendix-cli.md)。
3. **最终结果只走 `multica issue comment add`。** 你打到 stdout / 终端的内容，用户**看不到**。详见 [`05`](references/05-comments-and-mentions.md)。

## 仓库布局

```
SKILL.md               # Skill 入口：frontmatter + 索引 + 红线（Agent 加载本 skill 时一定看这里）
README.md              # GitHub 落地页：定位、安装、对齐版本（你正在看的）
VERSION.md             # 对齐策略 / 兼容范围 / 升级自检 / 变更日志
compatibility.md       # 命令粒度兼容矩阵（机读）
references/            # 渐进披露式参考文档：Agent 用到再读，不会一次性塞进上下文
  01-architecture.md            # server / daemon / AI 工具三件套（概念底座）
  02-entities.md                # Workspace / Issue / Comment / Task / Agent / Squad / Project（概念底座）
  03-core-loop.md               # ⭐ 操作指引：Agent 接到 issue 后的标准 loop
  04-issue-lifecycle.md         # 名词参考
  05-comments-and-mentions.md   # 名词参考（含三条红线之一）
  06-metadata.md                # 名词参考
  07-build-an-agent.md          # ⭐ 操作指引：新建一个 Agent
  08-tasks-and-runs.md          # 名词参考
  09-projects-and-resources.md  # 名词参考
  10-skills.md                  # 名词参考
  11-squads.md                  # 名词参考
  12-autopilots.md              # 名词参考
  13-providers.md               # 名词参考（11 款 AI 编程工具）
  14-troubleshooting.md         # 名词参考
  appendix-cli.md               # CLI 速查（只留组合规则/横切约定/副作用警告；flag 看 --help）
  SOURCES.md                    # 上游文件 → 章节映射 + 重抽取流程
```

布局遵循 Anthropic Skill [官方约定](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)——`SKILL.md` 顶层、`references/` 放渐进披露的参考文档。Agent 加载本 skill 时只需要 `SKILL.md` 进入上下文，`references/*` 在指向时再按需读，避免一次性塞 50KB 进 prompt。

## 维护协议

1. **内容更新**——上游 `multica-ai/multica` 文档大改时，按 [`references/SOURCES.md`](references/SOURCES.md) 的"上游文件 → 章节映射"表 patch 对应章节。不需要每次重写整章。
2. **不要把章节内容塞进 README。** 索引归 `SKILL.md`，README 只讲定位 / 安装 / 对齐版本。
3. **Skill 修改后只对新 task 生效。** 正在跑的 task 继续用旧版（[`references/10-skills.md`](references/10-skills.md)）。
4. **未来真出现重复使用的具体业务剧本**——单独建一个 `multica-<scenario>-skill` 仓库，不要往本仓硬塞第二个 skill；install URL 也更稳定。
5. **每次 `multica` 发新 tag 后**——指派 [`Skill 质量工程师`](https://multica.ai)（或所有者指定的 Agent）跑 [`VERSION.md` 的"升级自检"](VERSION.md#升级自检跑完一次再宣布已对齐到新版)。

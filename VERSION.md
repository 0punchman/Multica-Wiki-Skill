# VERSION — 这个仓库每条 Skill 当前对齐的 Multica 版本

> Skill 不是普通 Wiki：它要被 Agent **照着执行**。一旦上游 `multica-ai/multica` 改了 CLI flag、metadata key、或某个对象的状态机，Skill 里的命令片段就会从"正确"变成"偷偷出错"。所以这里必须**讲清楚每条 Skill 对齐的是哪个 Multica 版本，以及越界时的行为**。

## 当前对齐版本（2026-05-28 校验）

| Skill | 对齐 CLI 版本 | 对齐 commit | Release tag | 校验日期 |
|---|---|---|---|---|
| [`SKILL.md`](SKILL.md)（仓库根 = `multica-wiki` skill） | `v0.3.11` | `063e9711` | [v0.3.11](https://github.com/multica-ai/multica/releases/tag/v0.3.11) | 2026-05-28 |

`SKILL.md` 的 frontmatter 里也带 `multica_version` 字段——这张表是**索引**，frontmatter 是**单条 Skill 的真相**；两边不一致以 frontmatter 为准。

## 兼容范围承诺

- **同一 minor（`v0.3.x`）**：CLI flag 一般只增不删，行为兼容。这份 Skill 在 `v0.3.11 ≤ x` 区间内默认仍可用，但建议每次升级后跑一次本文档底部"升级自检"。
- **跨 minor（`v0.3.x → v0.4.0`）**：可能引入破坏性变更（flag 改名、metadata 字段移除、子命令重命名）。**不要直接套用**——先按"重新对齐流程"跑一遍。
- **跨 major（`v0.x → v1.0`）**：默认假设全部失效，必须重抽 + 全量评测。

## 怎么判断我手上的 Multica 是否还能用这份 Skill

```bash
multica --version
# multica vX.Y.Z (commit: ABCDEFGH, built: ...)
```

把输出里的 `vX.Y.Z` 跟上面表格对照：

- 完全相等 → 直接用。
- 只差 patch（例：表格 `v0.3.11`，本机 `v0.3.13`）→ 直接用，但出现"CLI 报 unknown flag / unknown command"时立刻停手，去 [release notes](https://github.com/multica-ai/multica/releases) 查变更，再回写本文件。
- 差 minor 或更多 → 走"重新对齐流程"。

## 升级自检（跑完一次再宣布"已对齐到新版"）

每次 `multica` 发新 tag 后，由 **Skill 质量工程师**（或所有者指定的 Agent）跑一次：

1. **拉新版 CLI**——确认 `multica --version` 与目标 tag 一致。
2. **逐条 `--help` 比对**——按 [`references/appendix-cli.md`](references/appendix-cli.md)（"命令家族"段）和 [`compatibility.md`](compatibility.md) 里出现过的每个命令跑 `multica <cmd> --help`，把 flag/子命令差异记到下面的"变更日志"。**必须包含 `skill import` / `skill files upsert` / `agent skills set`**——这三条命令一旦漂移会让用户"装/挂 skill"流程整体跑不通，是历史盲点。
3. **跑触发评测**——按 `skill-creator` 的 description-optimization 流程（`scripts/run_loop.py`）抽 10-20 条 trigger query，确认 handbook 还能在该触发的场景被命中。
4. **更新 frontmatter**——把每条 SKILL.md 的 `multica_version.cli` / `commit` / `released` / `verified_at` 改到新版本。
5. **更新本表**——把上面那张"当前对齐版本"刷成新版。
6. **写迁移说明**——见下一节。

未跑完上述 1-6 步**不要**声称 Skill 已对齐到新版本——这是 `skill-creator` 方法论"用证据说话"的最低门槛。

## 重新对齐流程（minor 及以上跨版本）

```bash
# 1. 在本仓重抽分支
git checkout -b realign/v0.4.0

# 2. 拉上游
multica repo checkout https://github.com/multica-ai/multica --ref v0.4.0

# 3. 按 references/SOURCES.md 的"上游文件 → 章节映射"
#    diff 每个上游文件的 v0.3.10..v0.4.0 范围，把变更落到对应章节
# 4. 执行"升级自检"1-6 步
# 5. 提 PR，PR 描述里 @ Skill 质量工程师 走评测
```

## 变更日志（每次升级追加一行；不要重写）

| 日期 | 从 → 到 | 触发人 | 主要变更点 | 评测结果（pass_rate / 备注） |
|---|---|---|---|---|
| 2026-05-28 | — → `v0.3.10` | Skill 质量工程师 | 首次建立版本对齐声明（仓库已有内容此前未明示版本） | 待跑首轮评测 |
| 2026-05-28 | `v0.3.10` 内修 | 通用智能体(Opus4.7) | 修正 `skill import` / `skill files` / `agent skills` 在 README 与 03/09 章节的命令片段（5 处漂移），与 v0.3.10 实测 `--help` 完全对齐；compatibility.md 拆出 high-risk 行 | n/a（仅命令片段对齐） |
| 2026-05-28 | `v0.3.10` 重构 M1 | 通用智能体(Opus4.7) | 从"对象百科"向"操作指引"渐进过渡（保留 wiki 属性）：新增 `03-core-loop.md`（Agent 标准 loop）和 `07-build-an-agent.md`（新建 Agent 决策与步骤）；旧 03-cli-cheatsheet 降级 + 瘦身为 `appendix-cli.md`，仅保留 `--help` 给不出来的组合规则 / 横切约定 / 副作用警告；新 07 占用导致 07-13 整体下移到 08-14；SKILL.md 索引保持扁平 14+附录（先建立心智模型再行动），不切 Part A/B/C。COSI-31 跟踪。 | n/a（结构重构 + 新章节，未跑评测） |
| 2026-05-28 | `v0.3.10` → `v0.3.11` | Skill 质量工程师 | 跑完"升级自检"1-2 步：逐条 `--help` 比对了 compatibility.md 全部 24 条命令（含 `skill import` / `skill files upsert` / `agent skills set` 三条历史盲点），命令拼写、子命令、flag、副作用清单与 v0.3.11 实测完全一致。仅观察到一处**纯增量**变更：`multica daemon` 多了新子命令 `disk-usage`（旧的 5 条 `start/stop/restart/status/logs` 不变）；wiki 暂不引用 `disk-usage`，按 compatibility.md 维护流程"全新子命令 → 不引用就不写进来"，本次不扩大改动范围，已在 COSI-44 评论里报告等队长裁决。frontmatter / 总览表 / README 当前对齐段一并刷到 `v0.3.11` (commit `063e9711`)。 | n/a（仅版本回写，未跑评测） |

> **维护规则**：日志只追加，不删除。哪怕回滚版本，也写一行"回滚 X→Y, 原因 ..."。这样 6 个月后能复盘 Skill 质量在哪一次对齐里抖动。

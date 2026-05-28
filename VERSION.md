# VERSION — 这个仓库每条 Skill 当前对齐的 Multica 版本

> Skill 不是普通 Wiki：它要被 Agent **照着执行**。一旦上游 `multica-ai/multica` 改了 CLI flag、metadata key、或某个对象的状态机，Skill 里的命令片段就会从"正确"变成"偷偷出错"。所以这里必须**讲清楚每条 Skill 对齐的是哪个 Multica 版本，以及越界时的行为**。

## 当前对齐版本（2026-05-28 校验）

| Skill | 对齐 CLI 版本 | 对齐 commit | Release tag | 校验日期 |
|---|---|---|---|---|
| [`SKILL.md`](SKILL.md)（仓库根 = `multica-wiki` skill） | `v0.3.10` | `be32e5af` | [v0.3.10](https://github.com/multica-ai/multica/releases/tag/v0.3.10) | 2026-05-28 |

`SKILL.md` 的 frontmatter 里也带 `multica_version` 字段——这张表是**索引**，frontmatter 是**单条 Skill 的真相**；两边不一致以 frontmatter 为准。

## 兼容范围承诺

- **同一 minor（`v0.3.x`）**：CLI flag 一般只增不删，行为兼容。这份 Skill 在 `v0.3.10 ≤ x` 区间内默认仍可用，但建议每次升级后跑一次本文档底部"升级自检"。
- **跨 minor（`v0.3.x → v0.4.0`）**：可能引入破坏性变更（flag 改名、metadata 字段移除、子命令重命名）。**不要直接套用**——先按"重新对齐流程"跑一遍。
- **跨 major（`v0.x → v1.0`）**：默认假设全部失效，必须重抽 + 全量评测。

## 怎么判断我手上的 Multica 是否还能用这份 Skill

```bash
multica --version
# multica vX.Y.Z (commit: ABCDEFGH, built: ...)
```

把输出里的 `vX.Y.Z` 跟上面表格对照：

- 完全相等 → 直接用。
- 只差 patch（例：表格 `v0.3.10`，本机 `v0.3.12`）→ 直接用，但出现"CLI 报 unknown flag / unknown command"时立刻停手，去 [release notes](https://github.com/multica-ai/multica/releases) 查变更，再回写本文件。
- 差 minor 或更多 → 走"重新对齐流程"。

## 升级自检（跑完一次再宣布"已对齐到新版"）

每次 `multica` 发新 tag 后，由 **Skill 质量工程师**（或所有者指定的 Agent）跑一次：

1. **拉新版 CLI**——确认 `multica --version` 与目标 tag 一致。
2. **逐条 `--help` 比对**——按 [`references/appendix-cli.md`](references/appendix-cli.md) 里出现过的每个命令跑 `multica <cmd> --help`，把 flag/子命令差异记到下面的"变更日志"。**必须包含 `skill import` / `skill files upsert` / `agent skills set`**——这三条命令一旦漂移会让用户"装/挂 skill"流程整体跑不通，是历史盲点。
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

> **维护规则**：日志只追加，不删除。哪怕回滚版本，也写一行"回滚 X→Y, 原因 ..."。这样 6 个月后能复盘 Skill 质量在哪一次对齐里抖动。

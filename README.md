# Multica Skills（仓库定位）

> 这个仓库里的所有内容都以 **Skill** 形式组织，给在 Multica 平台上跑任务的 Agent 用。
> 仓库当前名字仍是 `Multica_Wiki`（外链兼容），改名 `Multica_Skills` 留作后续单独 issue。

## 两层结构（不可互替）

| 层 | 触发宽度 | 作用 | 例子 |
|---|---|---|---|
| **通识层** | 宽（任何 Multica 相关任务都命中） | 让 Agent 在模糊场景**有自主判断的底座**——架构、对象模型、CLI、红线、生命周期 | [`skills/multica-handbook/`](skills/multica-handbook/) |
| **剧本层** | 窄（限定到具体业务场景） | 在高风险或重复场景**约束 Agent 不要自由发挥**——只钉步骤 + CLI + 红线 + 验证 + 回滚 | [`skills/replicate-squad.md`](skills/replicate-squad.md) |

剧本层的 frontmatter 会写 `assumes: multica-handbook loaded`，因此**不重复**讲对象模型——读剧本前 handbook 已经在上下文里。

## 目录

```
README.md                              # 你在这——仓库定位 + 维护协议
skills/
  multica-handbook/                    # 通识层（一个大 skill）
    SKILL.md                           # 入口：frontmatter + 索引 + 三条红线
    01-architecture.md ... 13-troubleshooting.md
    SOURCES.md                         # 上游文件 → 章节映射
  replicate-squad.md                   # 剧本层（每篇一个独立 skill）
  （后续按需新增）
```

## 单 skill 文件骨架

```markdown
---
name: <skill-name>
description: Use when ...               # 触发条件：handbook 写宽，剧本写窄
outcome: ...                            # 跑完之后的可观测产物
assumes: multica-handbook loaded        # 仅剧本层需要
---

何时用 / 何时不用
Step 1..N（含可直接复制的 CLI 模板）
失败回滚
深入参考（链接到 handbook 章节）
```

## 维护协议

1. **通识层更新**——上游 `multica-ai/multica` 文档大改时，按 `skills/multica-handbook/SOURCES.md` 的"上游文件 → 章节映射"表 patch 对应章节。不需要每次重写整章。
2. **剧本层新增**——发现某类业务流程被多个 issue 反复踩坑，就抽成剧本。剧本必须写**步骤、CLI、红线、验证、回滚**五段，缺一不可。
3. **不要把内容塞进 README**——索引归 `multica-handbook/SKILL.md`，剧本归各自文件，README 只讲仓库结构和维护协议。
4. **Skill 修改后只对新 task 生效**——正在跑的 task 继续用旧版（详见 `09-skills.md`）。

## 怎么把这份仓库挂到 Agent

把仓库当作**多个 GitHub Skill 来源**：

```bash
# 通识层（整个目录作为一个 skill）
multica skill import --from-github https://github.com/0punchman/Multica_Wiki/tree/main/skills/multica-handbook

# 单篇剧本
multica skill import --from-github https://github.com/0punchman/Multica_Wiki/blob/main/skills/replicate-squad.md
```

Skill 导入后必须挂到具体 Agent 才生效（`multica agent skills <slug> add --skill-id <id>`）。建议**所有 Multica Agent 都挂 `multica-handbook`**；剧本按场景挂。

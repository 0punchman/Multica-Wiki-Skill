# 15 · 创建一个 Skill

> **本章是动作型指引**——什么时候新建一个 Skill、什么时候导入第三方、新建时怎么决定形态与触发面、怎么验证 Agent 真的"用上了"它、踩过哪些坑。
> **只想知道 Skill 是什么 / 工作区 vs 本机 / 第三方风险有哪些类型，看 [`10-skills.md`](10-skills.md)。** 名词章是这一章的对象底座；动作章是把它落到 CLI 上。

## 何时该新建一个 Skill

Skill 是给 Agent 的**专业知识包**——`SKILL.md` + 可选支持文件，告诉 Agent "遇到某类任务怎么想、怎么做"。一次性指令塞 Agent `--instructions` 就够了；**只有当同一段知识会被多个 Agent / 多次任务复用**，才值得封成 Skill。

| 你的场景 | 应该 |
|---|---|
| 同一段操作规范要发给团队里多个 Agent | **新建 Skill**——挂到每个 Agent，未来改一处生效全队 |
| 知识只服务于这一个 Agent、几乎不变 | 写到 `agent --instructions` 里，**不要**单独建 Skill |
| 想用 Anthropic / ClawHub / skills.sh 上现成的能力 | **导入第三方**（先按 [`10-skills.md`](10-skills.md) 的"第三方 Skill 安全"过一遍审查） |
| 业务剧本、复用 1 次以上、需要被独立版本管理 | **新建独立仓库**的 Skill（`multica-<scenario>-skill`），不要往已有 Skill 仓硬塞第二个 |
| 只是想给 Agent 加一段背景资料 | 评估能不能塞到 `agent --instructions` 或 issue description；够用就别建 |

> 经验法则：**复用面 ≥ 2 个 Agent / ≥ 3 次任务**才建 Skill。低于这个量级，instructions 更轻、改起来更快、不会拖维护负担。

### 复用时怎么找到现成的 Skill

```bash
multica skill list --output json
multica skill get <skill-id>
multica skill files list <skill-id>     # 看包内具体文件，决定它是否覆盖你的场景
```

确认它的内容真贴你的任务，再决定是用现成的还是 fork 一份做局部改动。

## 决策清单（新建之前定好这 6 件事）

| 决策 | 选项 | 怎么挑 |
|---|---|---|
| **形态** | 操作指引型（动作章为主） / 名词参考型（百科为主） / 混合 | 给 Agent 实际**执行**用——优先操作指引型；纯查询用——名词参考型也可以。看本仓 `references/03-core-loop` vs `references/02-entities` 的对比。 |
| **触发面（description）** | 一句话写清"什么时候应该读它" | Skill 的 `description` 决定它能否被自动命中。**写"使用场景"，不要写"功能列表"**——例如 ✅"当在 Multica 平台上以 Agent 身份执行任务时" ✗"包含 Multica CLI 命令" |
| **来源** | 自写 / fork 第三方 / 直接 import 第三方 | 涉敏感数据、关键流程：自写或 fork。一次性辅助、来源可信：直接 import |
| **文件结构** | 单文件 `SKILL.md` / 顶层 `SKILL.md` + `references/` 渐进披露 | 内容 < 5 KB：单文件；内容 ≥ 5 KB 或要分章节：照 Anthropic Skill 约定，顶层放索引 + 红线，细节进 `references/`（本仓即范例） |
| **frontmatter 必填字段** | `name` / `description` / `outcome` 至少凑齐 | `name` 是 slug；`description` 决定触发；`outcome` 写"读完这个 Skill 之后 Agent 能/不能做什么"，让评测有锚点 |
| **装载范围** | 工作区 Skill / 本机 Skill | 团队共享：工作区 Skill（默认）。涉敏感本地内容 / 还在调试：本机 Skill（在 daemon 端扫描后选入） |

> ⚠️ **`description` 是 Skill 最值钱的一行**——它直接决定"什么场景下 Agent 会自动加载这条 Skill"。写"在 Multica 平台上以 Agent 身份执行任意任务时"比写"包含 CLI、对象模型、生命周期"命中率高得多。建 Skill 后跑一次 `skill-creator` 的 description-optimization 流程是值得的（参考本仓 `VERSION.md` 升级自检第 3 步）。

## 步骤 + 必备 CLI

### 1. 选一种来源

#### A. 从外部仓库 / 市场导入（最快路径）

```bash
multica skill import --url <URL>
# 来源识别由 CLI 自动完成：
#   - GitHub:   https://github.com/<owner>/<repo>
#               https://github.com/<owner>/<repo>/tree/<ref>/skills/<name>
#   - ClawHub:  https://clawhub.ai/skills/<slug>
#   - skills.sh: https://skills.sh/<slug>
```

> ⚠️ v0.3.10 起合并为单 flag `--url`——**不要**写 `--from-github` / `--from-clawhub`，会报 unknown flag。
> 🚨 **导入前必读 [`10-skills.md`](10-skills.md) 的"第三方 Skill 安全"段**：Multica 不做签名/沙盒；ClawHavoc 教训犹在。审查 `SKILL.md` 与所有附带文件再 import。

#### B. 在 CLI 直接新建（当外部仓库不存在时）

```bash
multica skill create \
  --name "..." \
  --description "..." \
  --output json
# 记下返回的 skill id

multica skill files upsert <skill-id> --path SKILL.md     --content "$(cat ./SKILL.md)"
multica skill files upsert <skill-id> --path references/01-foo.md --content "$(cat ./references/01-foo.md)"
multica skill files list   <skill-id>
multica skill files delete <skill-id> --path <relative-path>
```

> ⚠️ `skill files` 只有 `list` / `upsert` / `delete`——**没有** `upload`。`--content` 走命令行，**长度有限**：长 markdown 用 shell heredoc / 命令替换 / 外部脚本拼内容（GitHub 单文件上限约 1 MB，Skill 包总大小有限制）。

#### C. 本机 Skill

写到本机指定目录（如 Claude Code 的 `~/.claude/skills/<slug>/`），让 daemon 扫描后在 UI 里选入工作区。详见 [`10-skills.md`](10-skills.md)。

### 2. 写 `SKILL.md`（如果是自写 / 自建仓）

最小骨架——本仓的 `SKILL.md` 是范例：

```markdown
---
name: my-skill-slug
description: Use when <一句话写清楚什么场景该读它>
outcome: <读完这个 Skill 之后 Agent 能/不能做什么>
---

# 标题

> 一段定位 + 红线

## 索引（如果有 references/）

| # | 文件 | 内容 | 必读？ |
| 1 | references/01-foo.md | ... | ✅ |
```

参考 Anthropic Skill 官方约定：`SKILL.md` 顶层放索引 + 红线，`references/*` 放渐进披露的参考文档。Agent 加载 Skill 时只读 `SKILL.md` 进 prompt，避免一次性塞 50 KB。

### 3. 挂到 Agent

Skill 导入到工作区后**必须挂到具体 Agent 才生效**：

```bash
multica agent skills list <agent-slug>
multica agent skills set  <agent-slug> --skill-ids <id1>,<id2>,<new-skill-id>
```

> ⚠️ **`agent skills set` 是全量替换**——传给它的 `--skill-ids` 列表会**覆盖**该 Agent 当前所有挂载。增量挂一个新 Skill：先 `list` 拿到现有 ID，自己拼好新列表再 `set`，否则会**意外卸掉别的 Skill**。详见 [`07-build-an-agent.md`](07-build-an-agent.md)。

### 4. 验证：跑一条 smoke issue

```bash
# 建一个最小测试 issue 分给挂了新 Skill 的 Agent
multica issue create \
  --title "Smoke test for <my-skill>" \
  --description "<一段只有读了这条 Skill 才能正确处理的指令>" \
  --assignee-id <agent-uuid> \
  --status todo

multica issue runs <issue-id> --output json
multica issue run-messages <task-id> --issue <issue-id> --output json
multica issue comment list <issue-id> --output json
```

**通过条件**：

- Agent 在执行轨迹里**引用了 Skill 的内容**（点出名词、走对了步骤、用了正确的命令片段）
- 行为符合 Skill 里写的红线 / 决策 / 验证步骤
- Skill 的 `outcome` 在这条 issue 上达成

不达标说明 description 没命中、内容没装进 prompt、或挂载没生效——进"常见翻车"。

## 常见翻车

| 翻车 | 原因 | 修法 |
|---|---|---|
| `skill import --from-github <URL>` 报 unknown flag | v0.3.10 起合并为单一 `--url` | 改成 `multica skill import --url <URL>` |
| 第三方 Skill 装上后 Agent 行为异常 | 包里夹带恶意指令（参见 [`10-skills.md`](10-skills.md) 的 ClawHavoc 教训） | 删掉，重新走 import 前的审查；只用信任来源 |
| 改了 Skill 内容、Agent "看不到" | Skill 修改**只对之后新建的 task 生效**，正在跑的 task 继续用旧版 | 重发评论 / 重新分配触发新 task；UI 看 task 启动时间确认拿到了新版 |
| `agent skills set` 后旧 Skill 不见了 | `set` 是**全量替换**，`--skill-ids` 没带上旧的 ID | 永远先 `agent skills list` 取现有 ID，拼好完整列表再 `set` |
| `skill files upsert --content` 把内容截断 | 命令行长度限制 / 引号转义吃掉了一段 | 改用 `--content "$(cat path)"` 或 shell heredoc 或外部脚本拼内容 |
| Skill 导入了但 Agent 不命中 | `description` 写成了"功能列表"而不是"使用场景" | 改写成"Use when ..."；按 `skill-creator` 的 description-optimization 跑一轮评测 |
| Gemini / Hermes / OpenClaw 用不上 Skill | 这三款走通用 fallback 路径 `.agent_context/skills/`，工具自己是否真读取由工具决定 | 改用 Claude Code 等原生发现的工具，或在 instructions 里显式让 Agent 自己读 fallback 路径 |
| Skill 里塞了 Multica MCP 配置但没生效 | **MCP 当前只有 Claude Code 真消费**——其余 10 款会接收但不实际用 | 真要 MCP 就用 Claude Code；其它 provider 别在 Skill 里依赖 MCP |
| 导入 GitHub 单文件 > 1 MB 失败 | GitHub 单文件 ~1 MB 上限 / Skill 包总大小限制 | 拆分文件，把大块内容挪到 references；二进制资源走附件不要塞 Skill |
| 多个 Skill 重叠互相打架 | description 触发面重叠 + 内容矛盾 | 收紧每个 Skill 的 description 触发面（"什么场景"而不是"包含什么"）；让 Agent 一次只命中一个 |

## 把它接进工作流

新 Skill 上线后，常见接法：

1. **挂给若干 Agent**——`agent skills list`/`set`，覆盖所有需要这条知识的 Agent。
2. **跑评测 / smoke**——按"步骤 4"建测试 issue 反复跑，直到 Skill `outcome` 在多种场景上都达成。
3. **写到 README / 文档里**——告诉团队"什么场景应该挂这条 Skill"。
4. **进入 [`VERSION.md`](../VERSION.md) / [`compatibility.md`](../compatibility.md)**（如果是本仓内 Skill）——把对齐版本与命令片段登记起来，后续 Multica 升级时跑 `--help` diff 才有据可查。

## 不在这一章里的细节

| 想查什么 | 去哪一章 |
|---|---|
| Skill 是什么 / 工作区 vs 本机 / 文件存放路径 | [10-skills.md](10-skills.md) |
| Skill 与 MCP 的区别 / Multica 元 Skill | [10-skills.md](10-skills.md) |
| 11 款工具谁原生发现 Skill / 谁走 fallback | [10-skills.md](10-skills.md) + [13-providers.md](13-providers.md) |
| 改了 Skill 后什么时候生效 | [10-skills.md](10-skills.md) + [08-tasks-and-runs.md](08-tasks-and-runs.md) |
| 把新 Skill 挂到 Agent 的全量替换语义 | [07-build-an-agent.md](07-build-an-agent.md) |
| 命令完整 flag | `multica skill <subcommand> --help` / `multica agent skills <subcommand> --help` |

下一步：[16-build-a-squad.md](16-build-a-squad.md) — 把多个 Agent 组织成可路由的小队。

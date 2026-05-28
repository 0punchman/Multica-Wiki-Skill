# 09 · Skills（技能包）

> **本章是名词参考型**——Skill 是什么、放哪里、第三方风险有哪些。
> **要新建 / 复用 / 导入 Skill，看动作章 [`15-build-a-skill.md`](15-build-a-skill.md)。**

Skill 是给 Agent 的**专业知识包**——一个 `SKILL.md` + 可选的支持文件（脚本、配置、参考模板），告诉 Agent "遇到某类任务时该怎么想、怎么做"。

Multica 采用 [Anthropic Agent Skills](https://agentskills.io) 开放标准——所有符合规范的 Skill（Anthropic 官方、ClawHub、skills.sh 上的包）都能直接导入。

## 工作区 Skill vs 本机 Skill

| 类型 | 存哪里 | 怎么用 |
|---|---|---|
| **工作区 Skill** | Multica server | 挂到 Agent 后，任务执行时 daemon 自动同步到本机。**团队共享 skill 的标准方式**。 |
| **本机 Skill** | 用户本机指定目录（如 Claude Code 的 `~/.claude/skills/`） | daemon 按请求扫描本机，发现后由用户手动选入工作区 |

大多数情况用**工作区 Skill**：导一次，团队所有 Agent 都能用。本机 Skill 适合先在本地测试或涉及敏感本地内容。

## 导入 Skill

```bash
# 新建（在 UI 里直接写）—— CLI 也支持：
multica skill create --name "..." --description "..."
multica skill files upsert <skill-id> --path <relative-path> --content "<file contents>"
multica skill files list   <skill-id>
multica skill files delete <skill-id> --path <relative-path>

# 从外部仓库 / 市场导入：单一 flag --url，URL 来源自动识别
#   - GitHub:  https://github.com/<owner>/<repo>            (仓根 = SKILL 根)
#              https://github.com/<owner>/<repo>/tree/<ref>/skills/<name>
#   - ClawHub: https://clawhub.ai/skills/<slug>
#   - skills.sh: https://skills.sh/<slug>
multica skill import --url <URL>

# 从本机扫描后选入（daemon 端发起）
# 在 UI 里走"扫描本机 skills"流程
```

> ⚠️ v0.3.10 的 `skill import` 只有一个 `--url` flag——别再写 `--from-github` / `--from-clawhub`，会报 unknown flag。
> ⚠️ `skill files` 子命令是 `list` / `upsert` / `delete`，没有 `upload`；`upsert` 的内容必须通过 `--content` 传字符串（命令行长度有限，长文件请用 shell 替换或外部脚本）。

GitHub 单文件上限约 **1 MB**；包总大小有限制——超过会在导入时报错。

## 挂到 Agent

Skill 导入后必须**挂载到具体 Agent** 才生效。一个 Agent 能挂多个 Skill，一个 Skill 能挂多个 Agent。

```bash
multica agent skills list <agent-slug>
multica agent skills set  <agent-slug> --skill-ids <id1,id2,...>
```

> ⚠️ v0.3.10 的 `agent skills` 只有 `list` / `set` 两个子命令——**没有** `add` / `remove`。
> `set` 是**全量替换**：传给它的 `--skill-ids` 列表会覆盖该 agent 当前所有挂载。增量挂载要先 `list` 拿到现有 ID，再把新 ID 拼进列表一次性传给 `set`，否则会**意外卸掉别的 skill**。

## Agent 在哪里读到 skill 文件

每款 AI 工具有自己的 skill 发现路径，daemon 会在任务启动前把 workspace skill 复制到对应路径：

| 工具 | 路径 | 是否原生发现 |
|---|---|---|
| Claude Code | `.claude/skills/` | ✅ |
| Codex | `$CODEX_HOME/skills/` | ✅ |
| Copilot | `.github/skills/` | ✅ |
| Cursor | `.cursor/skills/` | ✅ |
| Kimi | `.kimi/skills/` | ✅ |
| Kiro CLI | `.kiro/skills/` | ✅ |
| OpenCode | `.opencode/skills/` | ✅ |
| Pi | `.pi/skills/` | ✅ |
| **Gemini** | `.agent_context/skills/` | ⚠️ 通用 fallback |
| **Hermes** | `.agent_context/skills/` | ⚠️ 通用 fallback |
| **OpenClaw** | `.agent_context/skills/` | ⚠️ 通用 fallback |

> ⚠️ **Gemini / Hermes / OpenClaw 走的是通用 fallback 路径**——这些工具是否真从这里读 skill 取决于工具本身。如果你的 skill 对它们没起效，先查这条。

## 修改 Skill 后

修改 skill 内容后，**只有之后新创建的 task 拿到新版本**——正在跑的 task 继续用旧版。

## 第三方 Skill 安全

> 🚨 **从 GitHub 或 ClawHub 导入的 Skill 可能包含脚本和可执行内容。**
> Multica 本身**不做签名验证、不做代码审查、不做沙盒隔离**——Skill 内容原封交给对应 AI 工具，工具怎么用（是否执行）由工具决定。

历史教训：**ClawHavoc 事件（2026-02）**——有人在热门 Skill 包里植入恶意指令，受害用户的 API key 被窃取。ClawHub 之后加了 VirusTotal 扫描，但**自动扫描不能替代审查**。

**导入前必须**：

1. 审查 `SKILL.md` 和它附带的所有文件
2. 只从信任的来源导入
3. 涉敏感数据的项目优先用自己写的本机 Skill

## Skill 与 MCP 的区别

两者都是"增强 Agent 能力"，方向不同：

| | Skill | MCP |
|---|---|---|
| 是什么 | 结构化的**知识包**（静态内容 + 指令） | **工具通道** |
| Agent 怎么用 | 读 SKILL.md 学"遇到 X 怎么想" | 调用外部 API / DB / 文件系统 |

Multica 当前 **MCP 支持只有 Claude Code 真的消费**——其他工具会接收 MCP 配置但不实际用。详见 [13-providers.md](13-providers.md)。

## 元 Skill（Multica 内置）

每次任务启动时，daemon 会把一个 Multica 元 skill 注入 Agent 工作目录——它就是你看到的 `CLAUDE.md` / `AGENTS.md` 这一份，包含：

- Agent 身份（ID + 名字）
- 当前 issue 的 ID 和工作流程清单
- 可用的 `multica` CLI 命令
- 项目资源（如果该 issue 属于某个 project）
- 评论 / mention / metadata 的写入纪律

这一份不是用户写的 Skill——是平台自动生成的、给 Agent 的运行手册。**这份 Wiki 是它的延伸版**。

下一步：[11-squads.md](11-squads.md) — 小队如何路由工作。

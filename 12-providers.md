# 12 · AI 编程工具对照（11 款）

Multica 内置支持 **11 款 AI 编程工具**——它们都实现同一套接口（排队、派发、执行、结果回传），但**能力细节差异很大**。这一页是实务对照。

## 能力矩阵

| 工具 | 厂商 | 会话恢复 | MCP | Skill 注入路径 | 模型选择 |
|---|---|---|---|---|---|
| **Claude Code** | Anthropic | ✅ | **✅（11 款里唯一真用）** | `.claude/skills/` | 静态 + flag |
| **Codex** | OpenAI | ⚠️ 代码存在但不可达 | ❌ | `$CODEX_HOME/skills/` | 静态 |
| **Copilot** | GitHub | ✅ | ❌ | `.github/skills/` | 静态（账号权益决定） |
| **Cursor** | Anysphere | ⚠️ 代码存在但不可用 | ❌ | `.cursor/skills/` | 动态发现 |
| **Gemini** | Google | ❌ | ❌ | `.agent_context/skills/` | 静态 |
| **Hermes** | Nous Research | ✅ | ❌ | `.agent_context/skills/`（fallback） | 动态发现 |
| **Kimi** | Moonshot | ✅ | ❌ | `.kimi/skills/` | 动态发现 |
| **Kiro CLI** | Amazon | ✅ | ❌ | `.kiro/skills/` | 动态发现 |
| **OpenCode** | SST | ✅ | ❌ | `.opencode/skills/` | 动态发现 |
| **OpenClaw** | 开源 | ✅ | ❌ | `.agent_context/skills/`（fallback） | 绑定在 Agent 上，不能在任务里切换 |
| **Pi** | Inflection AI | ✅（session 是文件路径） | ❌ | `.pi/skills/` | 动态发现 |

## 会话恢复——谁真的支持

会话恢复机制：失败自动重试时把 session ID 传回去，Agent 接着上次上下文继续。详细见 [07-tasks-and-runs.md](07-tasks-and-runs.md)。

| 状态 | 工具 | 含义 |
|---|---|---|
| ✅ **真用** | Claude Code, Copilot, Hermes, Kimi, Kiro CLI, OpenCode, OpenClaw, Pi | 传 resume id，会接着上次上下文 |
| ⚠️ **代码存在但不可达** | Codex, Cursor | 代码里有 resume 路径但实际走不到（Codex 静默回落、Cursor session id 不回传）——**当作不支持** |
| ❌ **无** | Gemini | CLI 无 resume 机制 |

> **决策**：如果你的工作流需要 Agent 在多次任务之间保持上下文（失败重试、手动重跑、对话式迭代），只选 ✅ 那一行的工具。

## MCP——只有 Claude Code 真的读

> ⚠️ **11 款里只有 Claude Code 实际消费 `mcp_config`**。其他 10 款会接收这个字段但**完全忽略**——不报错、不警告，配置就是不生效。

如果你在 Agent 配置里设了 `mcp_config` 但选了 Claude Code 之外的工具——你的 MCP server 对这个 Agent **没有效果**。

## 每款工具的定位（30 秒读完）

### Claude Code（Anthropic）

**新用户首选**——功能最完整。会话恢复真用，是 11 款里**唯一真读 MCP**。支持 `--max-turns`、`--append-system-prompt` 等细调。需要 Anthropic API key。

### Codex（OpenAI）

JSON-RPC 2.0 协议，状态化更强，approve 机制更细（手动批准 `exec_command` 和 `patch_apply`）。**会话恢复代码存在但当前不可达**——需要 resume，选 Claude Code 或 ACP 系列。

### Copilot（GitHub）

模型路由走 GitHub 账号权益——工具自己不做模型选择，由 GitHub 决定给你用哪个模型。skill 放 `.github/skills/` 是 GitHub CLI 的原生发现机制。

### Cursor（Anysphere）

Cursor 编辑器的 CLI 对应物。**会话恢复代码存在但实际不工作**——Cursor CLI 事件流里不回传 session ID，传的 resume 值永远无效。要 resume 选别的。

### Gemini（Google）

支持 Gemini 2.5 和 3 系列。**不支持会话恢复也不支持 MCP**——适合一次性、不需要长上下文记忆的任务。

### Hermes（Nous Research）

ACP 协议（和 Kimi 共享传输层）。会话恢复真用。但 **skill 注入路径是通用 fallback**（`.agent_context/skills/`）——如果 Hermes CLI 本身不读这路径，skill 对它可能不起作用。需要实测。

### Kimi（Moonshot）

中国市场向。和 Hermes 共享 ACP 协议，但 skill 路径 `.kimi/skills/` 是**原生**——和 Hermes 的 fallback 不一样。

### Kiro CLI（Amazon）

通过 `kiro-cli acp` 用 ACP stdio 协议。会话恢复走 ACP `session/load`，模型选择走 `session/set_model`。

### OpenCode（SST）

开源。动态发现可用模型（扫 CLI 的配置文件）。会话恢复真用。**适合爱折腾、想自定义模型目录**的开发者。

### OpenClaw（开源）

CLI agent 编排器。**模型绑定在 Agent 层**（`openclaw agents add --model`）——不能在单次任务里覆盖。配置严格受控：用户不能传 `--model` 或 `--system-prompt`，由 Agent 注册时的配置决定。

### Pi（Inflection AI）

极简主义。**会话恢复机制特殊**——session ID 是磁盘上的文件路径（`~/.pi/...`），而不是字符串 ID。其他工具里 resume id 是 CLI 返回的字符串，Pi 里就是会话文件本身。

## 给 Agent 配置工具时的环境变量

每款都能用环境变量覆盖二进制路径、模型、默认参数：

| 变量族 | 例 |
|---|---|
| `MULTICA_<TOOL>_PATH` | `MULTICA_CLAUDE_PATH=/usr/local/bin/claude` |
| `MULTICA_<TOOL>_MODEL` | `MULTICA_CODEX_MODEL=gpt-5-codex` |
| `MULTICA_<TOOL>_ARGS` | `MULTICA_CLAUDE_ARGS='--model "claude-opus-4-7" --max-turns 50'` |

`MULTICA_CLAUDE_ARGS` / `MULTICA_CODEX_ARGS` 用 POSIX shellword 解析——值像 shell 命令行一样切分。

参数应用顺序：**Multica 内置默认 → daemon-wide env → 任务里 Agent 的 `custom_args`**。

下一步：[13-troubleshooting.md](13-troubleshooting.md) — Agent 视角能自查的问题。

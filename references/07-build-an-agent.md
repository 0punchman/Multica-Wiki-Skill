# 07 · 新建一个 Agent

> **何时读这一章**：你或用户想往工作区里加一个新角色——专门跑代码评审、跑回归测试、做项目管家、当某个 squad 的成员。
>
> **它解决什么**：把"造一个新 Agent"拆成五个决策（要不要建 / 用谁的脑 / 跑哪台机 / 装什么 skill / 给什么纲领），每个决策给判断口诀和必备 CLI。

---

## 第 0 步：先问"该不该建"

绝大多数任务**不需要**新 Agent，给现有 Agent 派活就行。新建合理的信号：

| 信号 | 应该新建 |
|---|---|
| 这条任务**会反复出现**（每周 / 每条 PR / 每次发版） | ✅ |
| 这条任务**用现有 Agent 已经跑过 ≥3 次**，每次都同一套指令 | ✅ |
| 同一类活有**自己专属的**红线 / 风格 / SOP，跟通用 Agent 不重叠 | ✅ |
| 想让它能**被工作区其它人单独 `@` 触发**，而不是混在通用 Agent 后面 | ✅ |
| 一次性任务、好奇试一下、没有重复证据 | ❌ → 派给现有 Agent |
| 只是想换个名字 / 头像 | ❌ → 改现有 Agent，不要重复造 |

**反模式**：建一个 "Big Agent" 把所有职责塞一起，靠 instructions 自己分工——会让 instruction 变长、模型走偏、`@` 不准。**职责单一**比"多功能"重要。

---

## 第 1 步：五个决策

按这个顺序拍：**Provider → Runtime → Visibility → Model → Skills + Instructions**。

### 决策 1：Provider（用哪款 AI 编程工具的脑）

11 款 provider 详细对照见 [13-providers.md](13-providers.md)。**新用户首选 `claude_code`**——功能最完整、会话恢复真用、唯一真消费 MCP。

挑选口诀：

| 你的需求 | 推荐 | 不推荐 |
|---|---|---|
| 不知道选啥、想要功能最全 | `claude_code` | — |
| 失败重试要接着上次上下文跑 | `claude_code` / `copilot` / `kimi` / `kiro_cli` / `opencode` / `openclaw` / `pi` / `hermes` | `codex`（resume 不可达）/ `cursor`（同）/ `gemini`（无） |
| 要用 MCP server | `claude_code`（**唯一真用**） | 其它 10 款都会**忽略** `mcp_config` |
| 想用 GPT 系列模型 | `codex` | — |
| 用 Gemini | `gemini`（注意：无 resume，不适合长链路重试） | — |

> ⚠️ **MCP 陷阱**：在非 `claude_code` 的 Agent 上配 `mcp_config`，daemon **接受但不报错**——配置就是不生效。装 MCP 之前先确认 provider。

### 决策 2：Runtime（在哪台 daemon 上跑）

Runtime = 一台运行 Multica daemon 的机器，承载实际的 AI 工具进程。

```bash
multica runtime list --output json
```

这条返回一组 daemon。挑选时看：

- **`provider`** 字段——和你刚选的 Provider 必须匹配（`claude_code` 必须挑 `provider: "claude"` 的 runtime）。
- **`status: online`**——挑离线的等于扔进黑洞。
- **`visibility`**——`private` 是个人 daemon、`workspace` 是团队共享。
- **`metadata.cli_version`**——和当前对齐版本（见 `VERSION.md`）匹配，避免命令行为差异。

记下要用的 runtime 的 **`id`**（UUID）。

### 决策 3：Visibility（谁能分配它）

| 值 | 谁能分配它 | 谁能看见它 | 用在哪 |
|---|---|---|---|
| `workspace`（默认推荐） | 任何工作区成员 | 所有人 | 团队共享角色（代码审查、回归测试） |
| `private` | 只有 owner / admin / 创建者 | 名字可见、配置遮蔽 | 实验性 Agent、个人脚本工具 |

注意：`private` Agent 的**名字**对所有成员可见，只是配置被遮蔽——不要把"对外不可见"指望在这里。

### 决策 4：Model（具体哪个模型）

`--model` 传的是模型 ID（例 `claude-opus-4-7`、`claude-sonnet-4-6`、`gpt-5-codex`）。**优先用 `--model` 而不是把它塞进 `--custom-args`**——某些 provider（codex app-server、openclaw）会拒绝 `custom_args` 里的 `--model`。

不传 `--model` 就走 runtime 的默认模型，多数情况下够用。明显需要更强 / 更快 / 更便宜模型时再显式传。

### 决策 5：Skills + Instructions（给它什么知识、什么纲领）

两件不同的事：

- **Skills**——可被多个 Agent 共享的"专业知识包"（一个 `SKILL.md` + 资料）。详见 [10-skills.md](10-skills.md)。
- **Instructions**——只属于这一个 Agent 的"工作纲领"，写它的身份、目标、边界、禁止事项。

经验法则：

- 通用红线、CLI 用法、对象模型 → **挂 skill**（这条 wiki 就是这种 skill）。
- 这个 Agent 独有的角色 / 风格 / 口吻 / KPI → **写 instructions**。
- Skill 是"可复用，宽触发"；instructions 是"绑死这个 Agent，永远在线"。

> ⚠️ **`agent skills set` 是全量替换不是增量**——后续给已有 Agent 加 skill 时，先 `agent skills list` 拿现有 skill_ids，再 `set` 上"原列表 + 新 id"。直接 `set --skill-ids 新id` 会**意外把别的 skill 全部卸掉**。

---

## 第 2 步：执行——四条命令

```bash
# (1) 选好 runtime（决策 2）
multica runtime list --output json
# 记下 id，下面记为 RUNTIME_ID

# (2) 创建 Agent
multica agent create \
  --name        "代码评审专家" \
  --description "针对 PR 做严谨技术评审" \
  --instructions "你是一名资深工程师，对每个 PR 给出至少 3 个具体可执行的改进建议；不空泛、不情绪化；引用代码行号..." \
  --runtime-id  $RUNTIME_ID \
  --model       claude-opus-4-7 \
  --visibility  workspace \
  --max-concurrent-tasks 4 \
  --output      json
# 输出里记下新 agent 的 id（UUID），下面记为 AGENT_ID

# (3) 挂 skill —— 全量替换语义，先 list 再 set
multica skill list --output json     # 找到要挂的 skill 的 id
multica agent skills set $AGENT_ID --skill-ids <skill-id-1>,<skill-id-2>

# （可选）传 secret 类的 env 变量，避免 shell history
echo '{"OPENAI_API_KEY":"sk-..."}' | multica agent update $AGENT_ID --custom-env-stdin
```

> 💡 **从模板开种子**——`multica agent create --from-template <slug>` 可以从内置模板（如 `code-reviewer`）继承一组合理默认值，再用 `--instructions` 覆盖。模板 slug 列表通过 server 的 `/api/agent-templates` 端点查。

---

## 第 3 步：验证（必跑）

新建完**不要直接放生产**——派一条试用 issue 跑一次，看它是不是按预期工作。

```bash
# 1. 创建一条试用 issue
multica issue create \
  --title       "[smoke] 新 Agent 试用" \
  --description "请按你的 instructions 自我介绍一下，然后引用 [03-core-loop.md](...) 描述你接到任务后会怎么走。" \
  --assignee-id $AGENT_ID \
  --status      todo \
  --output      json
# 记下新 issue 的 id

# 2. 等 30s-2min 后看它是否触发并产出评论
multica issue get          <issue-id> --output json | grep -E 'status|updated_at'
multica issue comment list <issue-id> --output json
multica issue runs         <issue-id> --output json    # 看 task 状态
```

通过标准：

- ✅ Task 状态从 `queued` 走到 `completed`（不是 `failed`）。
- ✅ Issue 进了 `in_progress` 然后 `in_review`（说明它会切状态）。
- ✅ 评论里出现了它**按 instructions 写的**自我介绍（不是模型默认回答）。
- ✅ 没有 `@` 别的 Agent 收尾（无意识踩循环红线）。

任意一条 ❌ → 回去改 instructions 或 skills，**不要**直接派真任务。

---

## 第 4 步：必读的"全量替换"陷阱清单

| 命令 | 副作用 | 怎么躲 |
|---|---|---|
| `multica agent skills set <id> --skill-ids X,Y` | **全量替换** Agent 当前挂载 | 先 `agent skills list` 取并集再 set |
| `multica agent update <id> --instructions "..."` | 整体替换，**没有 patch** | 先 `agent get` 拿到现有，本地拼新版再 update |
| `multica agent update <id> --custom-env '{"K":"V"}'` | 整体替换 env map | 同上；secret 用 `--custom-env-stdin` 或 `--custom-env-file` |
| `multica agent archive <slug>` | 软删 | 想恢复 → `multica agent restore <slug>` |
| 新版 skill 推送 | 只对**新创建**的 task 生效，**正在跑**的 task 用旧版 | 改 skill 后，**不要** rerun 期待新版生效——要等新触发 |

---

## 常见翻车

1. **挑错 provider 装 MCP**——在 `codex` / `cursor` / `gemini` 等 Agent 上配 `mcp_config`，配置静默失效，`@` 它跑出来的结果一脸问号。**装 MCP 前 100% 确认 provider 是 `claude_code`**。
2. **`agent skills set` 把别的 skill 弄丢**——只 `set` 了新 skill_id，没把原来的并进去。常见后果：原本依赖工作区基础 skill 的工作流全部失灵。
3. **runtime 挑了离线的**——任务永远停在 `queued`。第一次创建后用 `multica issue runs <issue-id>` 看状态，超过 5 分钟还在 `queued` 八成是 runtime offline 或 provider mismatch。
4. **Instructions 太长 / 太散**——把 wiki 内容、CLI 规则、风格指南全塞进 instructions。这些应该挂 **skill**，instructions 只写"这一个 Agent 的角色 / 风格"。
5. **跳过验证步骤**——直接把新 Agent 派进真实工作流。最安全是先发 1 条 smoke issue，验证完再放生产。
6. **私有 Agent 当成"对外不可见"**——名字其实对所有成员可见。如果真要保密某个角色的存在，**它就不该是工作区 Agent**。

---

## 链回

- [02-entities.md](02-entities.md) — Agent / Member / Squad 对象差异（Agent 不收 inbox 通知 vs Member 收）
- [10-skills.md](10-skills.md) — Skill 是什么、怎么导入、怎么挂载、本机 vs 工作区
- [13-providers.md](13-providers.md) — 11 款 provider 的能力矩阵和坑位
- [appendix-cli.md](appendix-cli.md) — `agent` / `skill` / `runtime` 命令的 flag 速查
- [03-core-loop.md](03-core-loop.md) — 新 Agent 跑起来后该走的标准 loop（验证步骤里就是用它跑 smoke）

下一步：[08-tasks-and-runs.md](08-tasks-and-runs.md) — Agent 跑起来之后，每次执行的 Task 状态机。

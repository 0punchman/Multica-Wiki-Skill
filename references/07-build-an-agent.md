# 07 · 创建一个 Agent

> **本章是动作型指引**——什么时候新建一个 Agent、什么时候复用现成的、新建时要决定哪些字段、怎么验证它真的能干活、踩过哪些坑。
> Agent 的对象模型在 [02-entities.md](02-entities.md)；运行时机制在 [01-architecture.md](01-architecture.md)；11 款 AI 编程工具的差异在 [13-providers.md](13-providers.md)。

## 何时该新建一个 Agent

Agent 是**身份 + 配置 + 挂载的 Skills**——不是"一次执行"。一次执行是 Task。新建 Agent 之前先问自己：

| 你的场景 | 应该 |
|---|---|
| 已有 Agent 能干、只是想让它做一次 | **复用**——分配 issue 或 `@` 它，不新建 |
| 需要不同的**指令集**（不同的 system prompt / persona） | 新建 |
| 需要不同的**模型 / 工具**（如团队默认 Claude，这个任务要 Codex） | 新建（绑定不同 Runtime） |
| 需要不同的**Skill 组合**（一个 reviewer-only、一个 fullstack） | 新建（一个 Agent 一组 skill 设定） |
| 需要不同的**可见性**（敏感任务限 owner/admin 调度） | 新建（`--visibility private`） |
| 同一个人想跑多个并发会话 | **不需要新建**——同一 Agent 在不同 issue 上天然并发；同 issue 同 Agent 数据库强制 1 条活跃 task |

> 经验法则：**身份/指令/工具/可见性有任何一项不同 → 新建。否则复用。** 用大量"近亲" Agent 互相 `@` 来拼工作流是反模式——优先用 Squad（[11-squads.md](11-squads.md)）或子 issue（[04-issue-lifecycle.md](04-issue-lifecycle.md)）拼。

### 复用时怎么找到现成的 Agent

```bash
multica agent list --output json
multica agent get <slug>
multica agent skills list <slug>     # 看挂的 skill 清单决定能不能干
```

确认它的 `runtime_id` 对应的 Runtime 在线（`multica runtime list --output json` 看 `status: online`），不在线分配过去也只能排队。

## 决策清单（新建之前定好这 6 件事）

| 决策 | 选项 | 怎么挑 |
|---|---|---|
| **Provider / 工具** | Claude Code / Codex / Copilot / Cursor / Gemini / Hermes / Kimi / Kiro CLI / OpenCode / OpenClaw / Pi | 看 [13-providers.md](13-providers.md)。**默认 Claude Code**：会话恢复真用、是 11 款里**唯一真消费 MCP** 的。需要 Gemini/Codex 才考虑别的。 |
| **Runtime** | 工作区里某个 `online` 的 runtime ID | `multica runtime list --output json` 选状态 `online` 的。Runtime = daemon × 工具，离线则任务排队，超时失败。 |
| **Model** | 静态，如 `claude-opus-4-7` / `claude-sonnet-4-6`；空 = runtime 默认 | 偏好 `--model` 这个 flag，**不要**塞到 `--custom-args`（codex / openclaw 会拒）。 |
| **Skills** | 一组 skill ID（**全量替换**，详见 [10-skills.md](10-skills.md)） | 至少把 multica-wiki 这种"通识 skill"挂上；任务专长 skill 按需追加 |
| **Instructions** | 一段 markdown，描述这个 Agent 的人格 / 边界 / 风格 | 写"任务边界 + 红线 + 收尾要求"；不要重写 wiki 里的通识——交叉引用即可 |
| **Visibility** | `private`（默认）/ `workspace` | 普通团队工具用 `workspace`；触敏感数据 / 需要 owner-only 触发的用 `private` |

> ⚠️ **Skill 挂载是全量替换**：`multica agent skills set <slug> --skill-ids a,b,c` 会**覆盖**当前所有挂载——只想加一个，先 `list` 拿到现有 ID，自己拼好新列表再 `set`。否则会**意外卸掉别的 skill**。

## 步骤 + 必备 CLI

### 1. 选 Runtime

```bash
multica runtime list --output json
# 看 provider、status、device_info 选一个 status:"online" 的，记 id
```

### 2. 列 Skills（先把要挂的找好）

```bash
multica skill list --output json
# 没现成的 skill 包：要么不挂，要么先 multica skill import --url <github/clawhub/skills.sh URL>
```

### 3. 创建 Agent

```bash
multica agent create \
  --name "Code Reviewer (Opus)" \
  --description "Reviews PRs against project conventions, posts inline + summary." \
  --runtime-id <runtime-uuid> \
  --model "claude-opus-4-7" \
  --visibility workspace \
  --instructions "$(cat ./instructions.md)" \
  --output json
# 输出里记下新 agent 的 id 和 slug
```

可选 flag：

- `--from-template <slug>`——从内置模板（`code-reviewer` 等）seed
- `--max-concurrent-tasks <N>`——同时跑几个 task，默认 6
- `--custom-args '["--max-turns","50"]'`——以 JSON 数组传额外 CLI 参数
- `--custom-env-stdin` / `--custom-env-file`——传敏感环境变量。**不要用 `--custom-env`** 直接传值，会进 shell history / `ps`。
- `--runtime-config '{"...":"..."}'`——runtime 级配置（少用）

### 4. 挂 Skills（创建后单独走一次）

```bash
multica agent skills set <agent-slug> --skill-ids <id1>,<id2>,<id3>
multica agent skills list <agent-slug>     # 验证挂上了
```

### 5. 设 avatar / 环境（可选）

```bash
multica agent avatar  <slug> --file ./avatar.png
multica agent env     <slug> --get
multica agent env     <slug> --set KEY=value     # 见 multica agent env --help（写入会被审计）
```

### 6. 验证：分一个简单 issue 试一次

```bash
# 建一个最小测试 issue 分给新 Agent
multica issue create \
  --title "Smoke test for new Agent" \
  --description "回一条评论，说你能看到这条 issue 的描述。" \
  --assignee-id <new-agent-uuid> \
  --status todo

# 几秒内 Runtime 应该领走任务
multica issue runs <issue-id> --output json
multica issue run-messages <task-id> --issue <issue-id> --output json     # 看消息流
```

**通过条件**：

- `multica issue runs` 里出现一条 `running` 或 `completed` 的 task
- 该 task 完成后，issue 上多了一条由新 Agent 发出的评论（`multica issue comment list`）
- issue status 被推到了 `in_review`

任何一条不满足都说明 Agent 没正常起来，进"常见翻车"。

## 常见翻车

| 翻车 | 原因 | 修法 |
|---|---|---|
| `agent create` 报 `runtime not found` | `--runtime-id` 给错了 / 那个 runtime 不在当前 workspace | 重新 `multica runtime list --output json` 选当前 ws 的 |
| 分配后 5 分钟没动静、task 超时 | Runtime 在 list 里看着 online 但实际不能领（守护进程挂了） | `multica daemon status` / `daemon logs -f` 在那台机器上查 |
| 任务跑起来了但 Agent "看不到" skill | Skill 没用 `agent skills set` 挂；或挂了一组但被后续 set 覆盖了 | `agent skills list` 验证；增量挂载先 list 再 set |
| `skill files upsert` 想上传大文件失败 | `--content` 走命令行，长度有限 | 用 shell heredoc 或外部脚本拼内容；GitHub 单文件上限 ~1 MB |
| MCP 配置不生效 | 选了 Claude Code 之外的工具——其它 10 款**忽略** `mcp_config` | 真要 MCP 就用 Claude Code，否则不要依赖 |
| 修了 skill 内容、新 task 还是旧版 | Skill 修改**只对之后新建的 task** 生效 | 重发一条 issue 评论 / 重新分配触发新 task |
| `--model` 写到 `--custom-args` 里 | codex app-server / openclaw **会拒** | 用顶层 `--model` flag |
| 私人 Agent 名字仍能被看到 | 私人 Agent 只是隐藏配置详情、**名字仍然全员可见** | 真要彻底不可见的能力，重新设计 |
| skill import 用 `--from-github` | v0.3.10 已经合并成单一 `--url`，URL 来源自动识别 | `multica skill import --url <URL>` |
| `agent skills` 想 add / remove | v0.3.10 只有 `list` / `set` | `set` 全量替换；增量自己拼 |

## 把它接进工作流

新 Agent 起来之后，常见接法：

1. **直接分配 issue**——四种触发里最重的，详见 [04-issue-lifecycle.md](04-issue-lifecycle.md)。
2. **评论里 `@` 它**——不改 assignee，独立入队 task。慎用：和现有 Agent 互 `@` 是死循环高发地。详见 [05-comments-and-mentions.md](05-comments-and-mentions.md)。
3. **加进 Squad**——做小队成员，由队长 Agent 路由分派。详见 [11-squads.md](11-squads.md)。
4. **Autopilot 触发**——cron / webhook 周期性触发它。**不会自动重试**。详见 [12-autopilots.md](12-autopilots.md)。

## 不在这一章里的细节

| 想查什么 | 去哪一章 |
|---|---|
| Skill 怎么写 / 导入 / 挂载顺序 / 风险 | [10-skills.md](10-skills.md) |
| 11 款工具的细节差异 / 会话恢复 / MCP | [13-providers.md](13-providers.md) |
| Task 状态机 / 失败重试 / resume | [08-tasks-and-runs.md](08-tasks-and-runs.md) |
| `@` 一个 Agent 的副作用 | [05-comments-and-mentions.md](05-comments-and-mentions.md) |
| `agent` 命令完整 flag | `multica agent <subcommand> --help` |

下一步：[08-tasks-and-runs.md](08-tasks-and-runs.md) — task 状态机和重试规则。

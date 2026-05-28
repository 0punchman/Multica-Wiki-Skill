# 17 · 创建一个 Autopilot

> **本章是动作型指引**——什么时候建一个 Autopilot、cron / webhook 怎么挑、`create_issue` vs `run_only` 怎么选、怎么验证它真的会按时开工、踩过哪些坑。
> **只想知道 Autopilot 是什么 / 失败回退到 todo / webhook URL 即凭证 / 状态码语义有哪些类型，看 [`12-autopilots.md`](12-autopilots.md)。** 名词章是这一章的对象底座；动作章是把它落到 CLI 上。

## 何时该新建一个 Autopilot

Autopilot = **时间驱动 / 事件驱动**地让 Agent 自动开工。和分配 / `@` 提及 / chat（用户主动喊一声）相比，它适合**周期性、不需要人触发**的场景。

| 你的场景 | 应该 |
|---|---|
| 每天 / 每周 / 每月跑一次的巡检 / 报告 | **新建 Autopilot**（cron 触发器） |
| 外部系统（GitHub / GitLab / 自建 webhook）变化时让 Agent 响应 | **新建 Autopilot**（webhook 触发器） |
| 想"每条新 PR 都让 reviewer Agent 看一眼" | webhook 触发器 + `create_issue` 模式 |
| 一次性任务、几小时内就要跑 | **不要 Autopilot**——直接建 issue 分给 Agent |
| 强一致性 / 严格 SLA / 失败必须自动重试 | **不要 Autopilot**——Autopilot **不自动重试**，详见下方红线 |
| 工作要求秒级精度触发 | **不要 Autopilot**——server 每 30 秒扫一次到期触发器，最多延迟 30 秒 |

> 经验法则：**Autopilot 适合"晚一点 / 漏一次也没事"的常驻任务。** 真要可靠的定时执行 + 重试 + 监控，自己搭专门的调度器，让 Multica 只承担"派活给 Agent"那一段。

## Autopilot 的硬约束（建之前必读）

> 🚨 **Autopilot 触发的 task 不自动重试。** 失败原因可重试也不会被系统重排队；下一次 cron 到点会重新触发**新的 run**，但这次失败的工作不会自动补跑。

> 🚨 **Autopilot 失败会让 issue 状态从 `in_progress` 自动回退到 `todo`**——这是用户在看板上唯一的失败信号；**没有 inbox 通知**。要监控请自己设计：让 Agent 在成功时发评论，靠"缺少评论"反推失败。

> 🚨 **Webhook URL 即 bearer 凭证**——拿到的人都能触发这个 Autopilot。**不要贴公开 issue / 评论 / 截图 / 聊天记录**。泄漏后立刻 `multica autopilot trigger-rotate-url` 轮换。

把这三条钉在脑子里再继续——大多数 Autopilot 翻车都是踩了其中之一。

## 决策清单（新建之前定好这 6 件事）

| 决策 | 选项 | 怎么挑 |
|---|---|---|
| **执行 Agent** | 工作区里某个 Agent | 选会处理这类任务的 Agent；可挂上专属 Skill。如果还没有，先按 [`07-build-an-agent.md`](07-build-an-agent.md) 建。 |
| **执行模式** | `create_issue`（默认） / `run_only` | **默认 `create_issue`**——所有工作在看板上有可见 issue + 评论 + 状态。`run_only` 只在不想污染看板时用（看板上看不到）。 |
| **触发器类型** | `schedule`（cron） / `webhook` / 两者都开 | 周期性巡检：cron。响应外部事件：webhook。也可以同一个 Autopilot 挂多个触发器。 |
| **Cron 表达式 + 时区** | 5 字段 cron + IANA 时区 | **必须配时区**——不写默认会按 UTC，跨时区上线很容易跑错时间。最小粒度 1 分钟（无秒级）。 |
| **`create_issue` 模式的标题模板** | 标题字符串，可用占位符 `{{date}}` | **只有 `{{date}}` 一个占位符**（UTC `YYYY-MM-DD`）。**其它 `{{...}}` 会在创建时被拒**——避免拼错以后悄无声息把原文当 issue 标题。 |
| **Webhook 事件过滤（如果用 webhook）** | 0 或多行规则：事件名 + 可选 action 列表 | 留空 = 接受所有事件。明确想接的事件就写规则；规则匹配的是**推断出来的事件 / action**，不是任意 body 字段。详见下方 webhook 段。 |

> ⚠️ **Cron 别太密**：每 30 秒扫一次的 server 给 `* * * * *`（每分钟）配 `webhook` 一拥而上时容易堆积。先用 `*/5` 跑稳再考虑加密度。

## 步骤 + 必备 CLI

### 1. 选 / 建执行 Agent

```bash
multica agent list --output json
# 选状态正常、Runtime 在线的 Agent。如果还没有合适的，按 07-build-an-agent.md 建一个。
```

### 2. 创建 Autopilot

```bash
multica autopilot create \
  --title "Nightly bug triage" \
  --description "扫一遍 todo issue，按优先级排序并打标签。" \
  --agent triage-bot \
  --mode create_issue \
  --output json
# 记下返回的 autopilot id

# 可选 flag：
#   --priority urgent|high|medium|low|none   # 给生成的 task 继承的优先级
#   --issue-title-template "Triage {{date}}" # create_issue 模式的标题模板（仅 {{date}}）
#   完整 flag 直接 multica autopilot create --help
```

### 3. 加触发器

#### A. Cron 触发器

```bash
multica autopilot trigger-add <autopilot-id> \
  --cron "0 9 * * 1-5" \
  --timezone "Asia/Shanghai"
# 这条 = 工作日 09:00 北京时间触发一次
```

常用 cron 例子：

| Cron | 时区 | 含义 |
|---|---|---|
| `0 9 * * 1-5` | `Asia/Shanghai` | 工作日北京时间早 9 点 |
| `*/30 * * * *` | `UTC` | 每 30 分钟一次 |
| `0 3 * * *` | `UTC` | 每天 UTC 凌晨 3 点 |

> ⚠️ server **每 30 秒**扫一次到期触发器——触发时刻**最多延迟 30 秒**，不是秒级精准。
> ⚠️ server 重启时会**补扫漏掉的触发**——错过 N 个点就立即补跑 N 次（不丢但会瞬时堆积）。

#### B. Webhook 触发器

```bash
multica autopilot trigger-add <autopilot-id> --type webhook
# 输出里会给你一个形如 https://<host>/api/webhooks/autopilots/awt_... 的 URL
# 这就是触发凭证——保管好
```

发请求测试：

```bash
curl -X POST -H 'Content-Type: application/json' \
  -d '{"event":"github.pull_request.opened","eventPayload":{"pr":42}}' \
  https://<host>/api/webhooks/autopilots/awt_...
# 返回 200 + {"status":"accepted","run_id":"..."} 就触发成功
```

> 🚨 **Webhook URL 即 bearer**——见上方红线。泄漏立刻 `multica autopilot trigger-rotate-url <ap-id> <trigger-id>`，旧 URL 立即失效。
> ⚠️ Multica **当前 v1 仅 bearer**——没有 HMAC 签名校验、没有 IP allowlist。

加事件过滤（可选）：

```bash
# UI 里给 webhook 触发器配规则；CLI 直接 trigger-update 也行
multica autopilot trigger-update <autopilot-id> <trigger-id> \
  --filter "workflow_run:completed,failed"
# 只放行 X-GitHub-Event=workflow_run + body.action ∈ {completed, failed}
```

事件名推断顺序（不带 `event` 字段时）：

1. body envelope 里 `event` 字符串字段
2. `X-GitHub-Event` + body `action` → `github.<event>.<action>`
3. `X-Gitlab-Event` → `gitlab.<event>`
4. `X-Event-Type` → 原样
5. body 顶层 `event` / `type` / `action`
6. 默认 `webhook.received`

> 💡 **常见误区**：规则 `event=trigger / action=true` 不是"body 里出现 `trigger: true` 就放行"——匹配的是**推断出来的事件 / action**，不是任意 body 字段。要命中它请用 `X-Event-Type: trigger.true` 头或 body envelope。
> 💡 **action 必须是 JSON 字符串**才被列为候选——`{"action": true}` 里的 `true` 是 bool，永远命中不了 `action=true` 这条规则。

### 4. 验证：手动触发一次

```bash
# 走和 schedule 完全一样的流程，只是 source = manual
multica autopilot trigger <autopilot-id>

# 看 run 记录
multica autopilot runs <autopilot-id> --output json --limit 5
```

**通过条件**：

- `autopilot runs` 里出现一条 `status: completed`（或至少 `running`）的 run
- `create_issue` 模式：`runs[0].issue_id` 指向一条新生成的 issue，issue 已分配给目标 Agent，且过几秒内就有 Agent 评论
- `run_only` 模式：`runs[0].task_id` 存在，task 状态进了 `running` / `completed`
- 失败时 `runs[0].error_reason` 写明原因（runtime 离线 / agent 不存在 / payload 解析失败 / ...）

任何一条不满足进"常见翻车"。

## 常见翻车

| 翻车 | 原因 | 修法 |
|---|---|---|
| Cron 配了但没触发 | 时区没传，默认 UTC，本地时间还没到 / 已经过 | `trigger-update` 显式补 `--timezone "Asia/Shanghai"`；或重新算 UTC 表达式 |
| 一启动就跑了一堆次 | server 重启会**补扫漏掉的触发** | 已知行为；如果不能补跑，临时 `trigger-update --enabled=false` 等过当前周期再开 |
| 标题里 `{{user}}` 没插值 | **只支持 `{{date}}`**，其它占位符直接报错或被原样拒绝 | 换成 `{{date}}` 或把动态信息写到 issue **描述**里（描述无插值，由 Agent 解析 webhook payload 后自己写进去） |
| Webhook POST 返回 200 但 task 没起来 | 看 body：`{"status":"skipped","reason":"agent runtime is offline at dispatch time"}` 这类**正常 no-op** 也是 200，避免外部 webhook 反复重试 | 看 `autopilot runs` 拿真实状态；正常 200 状态详见 [`12-autopilots.md`](12-autopilots.md) 的状态码表 |
| Webhook 返回 400 | 无效 JSON / scalar body / 空 body | body 必须是合法 JSON 对象 / 数组——发空 body 或 `null` 都会被拒 |
| Webhook 返回 413 | 请求体超过 256 KiB | 把大 payload 拆小，或在 Agent instructions 里告诉 Agent 去外部拉详情（payload 只带关键 ID） |
| Webhook 返回 429 | 单 token 速率限制（默认 60 / 分钟） | 上游降低请求率；或同一个 webhook 被多源滥发——换一个 token 给不同源 |
| 失败了但用户没察觉 | Autopilot **没有 inbox 通知**；`failed` 只在 runs 里能看到 | 显式监控：让 Agent 成功时发评论，看缺评论反推失败；或定一条更高频的 cron 检查 runs |
| 失败后 issue 又回到 `todo`，反复重派 | Autopilot 失败会让 `in_progress` 回退到 `todo`，下一轮 cron 又派 | 这是设计如此（让看板上有失败信号）；要避免反复重派就让目标 Agent 自己识别"已经失败过"——查 metadata / 评论历史前置判断 |
| 私 webhook URL 误贴到 Slack / 评论 | URL 即凭证，被任意人触发 | 立刻 `multica autopilot trigger-rotate-url <ap-id> <trigger-id>`——旧 URL 立即失效 |
| 用了"API 类型触发器" | schema 里保留了 `api` 类型但**没有入站路由会触发它**，UI 给历史记录打 Deprecated | 不要新建；已有的全部改 cron / webhook |

## 把它接进工作流

新 Autopilot 上线后，常见接法：

1. **跑稳 1 周再扩频率**——头几次 run 看 `autopilot runs --output json` 排查触发延迟、payload 解析、Agent 行为。
2. **挂 Skill 到执行 Agent**——把 Autopilot 拉进来的工作和 Agent 长期挂的通用 Skill（multica-wiki 等）配合。Agent 收到的 issue 描述是 webhook payload + 你写的 description，必要时让 Agent 用 [`15-build-a-skill.md`](15-build-a-skill.md) 建专属"webhook payload 解析"Skill。
3. **配监控反推**：定一条更高频的 cron Autopilot 跑"扫描所有 Autopilot 最近 N 次 run"，发现连续 `failed` 时发评论 `@` workspace owner。这是当前没有 inbox 通知的兜底。
4. **轮换 webhook URL 进运维清单**——Token 轮换不是事故才做，定期做。

## 不在这一章里的细节

| 想查什么 | 去哪一章 |
|---|---|
| Autopilot 模型 / 失败回退 / 状态码语义 | [12-autopilots.md](12-autopilots.md) |
| webhook 事件名推断顺序 / 事件过滤规则的全文 | [12-autopilots.md](12-autopilots.md) |
| 执行 Agent 的新建 / Skill 挂载 | [07-build-an-agent.md](07-build-an-agent.md), [15-build-a-skill.md](15-build-a-skill.md) |
| 把 Autopilot 派到小队（按主题路由） | [16-build-a-squad.md](16-build-a-squad.md), [11-squads.md](11-squads.md) |
| Task 重试规则 / 为什么 Autopilot 不重试 | [08-tasks-and-runs.md](08-tasks-and-runs.md) |
| `--metadata` 过滤、`--output json`、其它 CLI 横切约定 | [appendix-cli.md](appendix-cli.md) |
| 命令完整 flag | `multica autopilot <subcommand> --help` |

至此 **里程碑 2** 操作指引三章完成（Skill / Squad / Autopilot）——回到 [SKILL.md](../SKILL.md) 索引看完整地图。

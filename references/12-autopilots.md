# 11 · Autopilots（定时 / Webhook 触发）

Autopilots 让 Agent **按调度自动开工**——cron 到点 / webhook 到达 / 用户手动触发。和分配 / `@` 提及 / chat（都是用户主动喊一声）相比，Autopilots 是**时间驱动**的常驻指令，适合定期巡检、周期性报告、凌晨清理任务。

## 关键性质（必读）

> ⚠️ **Autopilot 触发的 task 不自动重试**——和分配 / `@` 提及不同。
>
> 失败原因可重试也不会被系统重排队；下一次 cron 到点会重新触发**新的 run**，但这次失败的工作不会自动补跑。
>
> **没有 inbox 通知**——失败只在运行历史里留 `failed` 记录。要监控请自己设计（让 Agent 在成功时发评论，靠"缺少评论"反推失败）。

> ⚠️ **Autopilot 失败会让 issue 状态从 `in_progress` 自动回退到 `todo`**——这是用户在看板上唯一的失败信号。

## 配置一个 Autopilot

要定下：

- **名字** —— 显示名
- **执行 Agent** —— 到点派给谁
- **优先级** —— 继承给它产生的 task
- **描述 / Prompt** —— Agent 每次执行拿到的工作说明
- **执行模式** —— 见下节
- **触发器** —— 至少 1 个：`schedule`（cron + 时区）或 `webhook`

```bash
multica autopilot create \
  --title "Nightly bug triage" \
  --description "Scan todo issues and prioritize." \
  --agent triage-bot \
  --mode create_issue
```

## 执行模式

| 模式 | 行为 | 推荐 |
|---|---|---|
| **`create_issue`** | 每次触发先建 issue（标题支持占位符 `{{date}}`，会插值成 UTC `YYYY-MM-DD`），按分配流程派给 Agent。所有工作落在看板上，历史 / 评论 / 状态和手动分配的 issue 完全一致。 | ✅ **默认推荐** |
| **`run_only`** | 不建 issue，直接入队 task。看板上看不到，只能在 Autopilot 运行历史里看。 | 当你不想污染看板时 |

> 标题占位符**只支持 `{{date}}`**。其它 `{{...}}` 形式会在创建时被拒，避免拼错以后悄无声息地把原文当 issue 标题。

## Cron 触发器

- 标准 5 字段（分 时 日 月 周）
- 最小粒度 **1 分钟**（没有秒级）
- 时区用 IANA 格式（`Asia/Shanghai`、`UTC`、`America/New_York`）
- server **每 30 秒**扫一次到期触发器——**触发时刻最多延迟 30 秒**，不是秒级精准
- server 重启时如果错过触发点，启动时会**补扫漏掉的触发**（不丢但会立即补跑）

```bash
multica autopilot trigger-add <autopilot-id> --cron "0 9 * * 1-5" --timezone "America/New_York"
multica autopilot trigger-update <autopilot-id> <trigger-id> --enabled=false
multica autopilot trigger-delete <autopilot-id> <trigger-id>
```

例子：

| Cron | 时区 | 含义 |
|---|---|---|
| `0 9 * * 1-5` | `Asia/Shanghai` | 工作日北京时间早上 9 点 |
| `*/30 * * * *` | `UTC` | 每 30 分钟一次 |
| `0 3 * * *` | `UTC` | 每天 UTC 凌晨 3 点 |

## 手动触发

```bash
multica autopilot trigger <autopilot-id>
```

走和 schedule 触发**完全相同**的执行流程，只是运行记录里 `source = manual`。

## Webhook 触发器

```
https://<你的 Multica host>/api/webhooks/autopilots/awt_…
```

向这个 URL POST 任意 JSON → Multica 记一条 `source = webhook` 的 run，把请求体保存为 `trigger_payload`，按和 schedule 触发器**完全一致**的方式派发给 Agent。

`create_issue` 模式：入站 payload 附在新 issue 描述里供 Agent 读到。
`run_only` 模式：payload 也会随 run 一并交给 daemon。

### Payload 形态

可以发自己的封装：

```json
{ "event": "github.pull_request.opened", "eventPayload": { "..." : "..." } }
```

也可以发任意 JSON 对象 / 数组，Multica 会规范化为内部封装：

```json
{
  "event": "<推断>",
  "eventPayload": <你的 body>,
  "request": { "receivedAt": "<rfc3339>", "contentType": "application/json" }
}
```

不带 `event` 字段时按以下顺序推断事件名：

1. body envelope 里 `event` 字符串字段
2. `X-GitHub-Event` + body `action` → `github.<event>.<action>`
3. `X-Gitlab-Event` → `gitlab.<event>`
4. `X-Event-Type` → 原样
5. body 顶层 `event` / `type` / `action`
6. 默认 `webhook.received`

### 事件过滤

webhook 触发器上能配过滤规则——一行规则 = 事件名 + 可选逗号分隔的 action 列表，**任一行命中放行**：

| 事件名 | Actions | 命中范围 |
|---|---|---|
| `workflow_run` | `completed, failed` | 只 `action=completed` 或 `failed` 的 `workflow_run` |
| `workflow_run` | _(空)_ | 所有 `workflow_run` |
| `push` | _(空)_ | 所有 `push` |

留空所有规则 = 接受所有事件。

> 💡 **常见误区**：规则 `Event name: trigger / Actions: true` 不是"body 里出现 `trigger: true` 就放行"——过滤匹配的是**推断出来的事件 / action**，不是任意 body 字段。要命中它，请用 `X-Event-Type: trigger.true` 头或 body envelope。

> 💡 **action 必须是 JSON 字符串**才被列为候选。`{"action": true}` 里的 `true` 是 bool 不是字符串，永远命中不了 `action=true` 这条规则。

### 状态码语义

正常的 no-op 路径都返回 `200 OK`（避免外部 webhook 重试机制反复打）：

| Body | 含义 |
|---|---|
| `{"status":"accepted","run_id":"…"}` | 已派发一次 run |
| `{"status":"skipped","reason":"agent runtime is offline at dispatch time"}` | Runtime 离线，记 skipped run |
| `{"status":"ignored","reason":"trigger_disabled"}` | 触发器已禁用 |
| `{"status":"ignored","reason":"autopilot_paused"}` | Autopilot 已暂停 |
| `{"status":"ignored","reason":"autopilot_archived"}` | Autopilot 已归档 |
| `{"status":"ignored","reason":"event_filtered"}` | 被事件过滤拦下 |

非 2xx：

| 状态码 | 原因 |
|---|---|
| `400` | 无效 JSON / scalar body / 空 body |
| `404` | 未知 token (`webhook not found`) |
| `413` | 请求体超过 256 KiB |
| `429` | 单 token 速率限制（默认 60 / 分钟） |

### 安全：URL 即 bearer

> 🚨 **生成的 webhook URL 就是凭证**——拿到的人都能触发这个 Autopilot。按 token 对待：
>
> - **不要贴到公开 issue / 评论 / 截图 / 聊天记录**
> - **泄漏后立刻轮换**：`multica autopilot trigger-rotate-url <ap-id> <trigger-id>`，旧 URL 立即失效
> - 当前 v1 仅 bearer，**无 HMAC 签名校验、无 IP allowlist**

## 运行历史

```bash
multica autopilot runs <id> [--limit N] --output json
```

每次触发产生一条 run 记录：

- 触发源（`schedule` / `manual` / `webhook`）
- 开始 / 完成时间
- 状态（`issue_created` / `running` / `completed` / `failed` / `skipped`）
- 关联的 issue（`create_issue` 模式）或 task（`run_only`）
- 失败 / 跳过原因

## 目前不可用的能力

- **API 类型触发器**——schema 里保留了 `api` 类型但没有入站路由会触发它；UI 给已有的此类记录打 Deprecated 标签
- **Per-trigger HMAC 签名校验、IP allowlist、按提供方的事件预设**——后续工作

下一步：[13-providers.md](13-providers.md) — 11 款 AI 编程工具的实务差异。

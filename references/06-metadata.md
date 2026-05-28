# 06 · Issue Metadata Bag

每条 issue 都有一个小 KV 字典 `metadata`，是 Agent 之间传递"高价值结构化笔记"的标准位置——但**写入门槛要高**。它是编辑性笔记本，不是 run log，不是 description 替代品，更不是评论的另一种发表方式。

## 硬约束

- key 模式：`^[a-zA-Z_][a-zA-Z0-9_.-]{0,63}$`（snake_case ASCII，长度 ≤ 64）
- value 类型：string / number / bool 三种之一（不能存 array / object）
- 每条 issue **最多 50 个 key，blob 总大小 ≤ 8KB**
- 写入是**单 key 原子**——不同 key 的并发写不会互相覆盖

## 进入 issue 时——读

每次任务的开头：

```bash
multica issue metadata list <issue-id> --output json
```

读出来当作"上一次同伴留下的提示"。空 `{}` 是正常情况，不是错误。CLI 失败也别停。

**Metadata 不是真理，是提示**：如果 metadata 说 `pipeline_status=waiting_review`，但你刚发现 PR 已经 merge 了——以**当前的事实**为准，并顺手把 stale 的 key 更新或删掉。

## 退出 issue 前——写（很少）

> ⚠️ **大多数任务不写新 key——这是预期的常态，不是失败**。

写一个 key 之前，必须**两个条件同时满足**：

1. **对这条 issue 的进展是物质重要的事实**，且
2. **后续在同一条 issue 上的 task 很可能反复读它**，而不是从最新评论 / 代码 / PR 里就能拿到

如果你说不出"这条 key 以后谁会读、读来干嘛"——**不要写**。

### 推荐 key 名（复用、不要发明同义词）

| Key | 含义 |
|---|---|
| `pr_url` | 这次任务对应的 PR URL |
| `pr_number` | PR 编号（数字，便于 list 过滤） |
| `pipeline_status` | CI / 流程状态（`waiting_review` / `merged` / `reverted` 等） |
| `deploy_url` | 已部署的预览 / 生产环境 URL |
| `external_issue_url` | 关联的外部工单（Linear / Jira / GitHub issue） |
| `waiting_on` | 当前在等谁 / 什么 |
| `blocked_reason` | 卡住的原因（短，不超过一行） |
| `decision` | 一次决策的简短结论（"采用方案 A" / "暂缓") |

新 key 名只在没有合适的现成 key 时才造，造时仍走 snake_case ASCII。

### 离开前清理 stale

如果你进场时看到 `pipeline_status=waiting_review`，但你刚发现 PR 已经 merge 了，**离开前必须把这条 key 更新或删除**：

```bash
multica issue metadata set <id> --key pipeline_status --value merged
# 或者
multica issue metadata delete <id> --key pipeline_status
```

不清理 = 让 metadata 继续腐烂，变成"考古工作"，把这个机制本身的价值毁掉。哪怕你这次没写新 key，**stale 清理也是预期之内的工作**。

## 不要写的东西

| 类别 | 反例 |
|---|---|
| 密钥 / token | `api_key`, `bearer_token` |
| 长日志 / 长摘要 | `last_run_log`, `error_dump`（评论才是该写的地方） |
| description / comment 摘要 | `summary`（这是 issue 描述自己的工作） |
| Run 簿记 | `attempts`, `last_run_at`, `agent_id`（平台已经有 `runs` 接口） |
| 单次任务的细节 | `file_edited`, `test_added`（评论 + diff 就够了） |
| 新闻播报 | `last_status_change_reason`（评论） |

> 简单的判断：**离开 issue 时，如果删掉这条 metadata，未来的 Agent 是否会真的"重新读不到"必要信息？** 不会 → 不要写。

## 用 metadata 过滤 issue 列表

写好 key 之后，可以在 list 里过滤：

```bash
multica issue list --metadata pipeline_status=waiting_review
multica issue list --metadata pr_number=482 --metadata is_blocked=true
multica issue list --metadata code='"42"'    # 强制 string 匹配，绕过数字 sniff
```

值会被 JSON 解析：`true`/`false` → bool，纯数字 → number，其它 → string。多 `--metadata` 是 AND。这是 metadata 的**消费侧**——它的存在意义就是为了未来能这样查得到。

## 命令复习

```bash
multica issue metadata list   <id> [--output json]
multica issue metadata get    <id> --key <k>
multica issue metadata set    <id> --key <k> --value <v> [--type string|number|bool]
multica issue metadata delete <id> --key <k>
```

`--type` 一般不用——CLI 会自动推断。需要强制 string（避免被识别成 number / bool）时显式传。

下一步：[07-build-an-agent.md](07-build-an-agent.md) — 何时该新建一个 Agent，怎么建。

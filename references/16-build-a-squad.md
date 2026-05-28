# 16 · 创建一个 Squad

> **本章是动作型指引**——什么时候建一个小队、队长 / 成员怎么挑、`instructions` 写什么、怎么验证它真的会路由、踩过哪些坑。
> **只想知道 Squad 是什么 / 队长 briefing 三段 / 触发模型，看 [`11-squads.md`](11-squads.md)。** 名词章是这一章的对象底座；动作章是把它落到 CLI 上。

## 何时该新建一个 Squad

Squad = **一组 Agent / Member + 一个队长 Agent**。assignee 选小队 → 触发落到队长 → 队长在评论里 `@` 派给合适的成员。

| 你的场景 | 应该 |
|---|---|
| 一个领域有多名候选 Agent / 人，事先不知道哪条 issue 该归谁 | **新建 Squad**——交给队长判断 |
| 想"按主题派活"，将来扩队后已有 issue 自动用上新人 | **新建 Squad**——`@FrontendTeam` 比 `@alice-or-bob-or-carol` 稳定 |
| 工作范围明确、就是某个 Agent 干 | **不要建 Squad**——直接分配给那个 Agent，省一层路由 |
| 想串行执行 step1 → step2 → step3 | **不要 Squad**——用子 issue + `--status backlog` 串行（详见 [04-issue-lifecycle.md](04-issue-lifecycle.md)） |
| 想让多个 Agent 并行评审同一个产物 | **不要 Squad**——并行子 issue 或 `@` 多个 Agent |

> 经验法则：**不知道谁干 + 候选 ≥ 2** 才建 Squad。否则直接分配/`@` 更轻、路径更短。

### 复用时怎么找到现成的 Squad

```bash
multica squad list --output json
multica squad get <squad-id>
multica squad member list <squad-id>     # 看现有成员是否已经覆盖你的需求
```

如果一个现成小队的领域 + 成员都贴你的场景，**优先扩成员**（`squad member add`）而不是建新小队。

## 决策清单（新建之前定好这 6 件事）

| 决策 | 选项 | 怎么挑 |
|---|---|---|
| **领域 / 触发面** | 一个明确的主题（"Frontend"、"DB Migrations"、"Bug Triage"） | 名字 + 描述要让人 `@` 时一秒判断该不该选；和已有小队**不要重叠** |
| **队长 Agent** | 必须是 Agent，不能是 Member | 选**会路由 + 会克制**的——队长的天职是派活，不是亲自动手干。挑通用型 reviewer / triage Agent，挂上路由相关的 Skill |
| **成员构成** | Agent / Member 任意组合，至少 1 个（队长可以是唯一成员） | 同一个 Agent 能加入多个小队；先列出"接到这个领域的 issue 时谁能干"，按这个集合配 |
| **Squad Instructions（队长私货）** | 一段 markdown：路由规则 / 上报策略 | 写"DB 派给 @Alice，前端派给 @Bob，模糊的回评论问 reporter"。**不要**重写 Squad Operating Protocol（那段是硬编码的，会自动追加） |
| **成员 role** | 每位成员一段 freeform 描述（"Owns the migrations"） | 给队长当**路由依据**——队长 briefing 的 Roster 会带上这段，写得越清越准 |
| **触发面 vs 子 issue** | 让 reporter `@` 小队 / 直接 assign 小队 | 想让 issue 仍归原 owner、只让小队挑个人回答一下：在评论里 `@` 小队（不改 assignee）。想把 issue 整条交给小队跟到底：assignee 直接选小队 |

> ⚠️ **创建 / 更新 / 归档 / 增删成员、改 leader 都要 workspace owner 或 admin 权限**——你（普通成员 / Agent）能列、能在评论里 `@`、但不能改成员。需要时在评论里 `@` workspace owner 报需求，不要绕开权限模型。

## 步骤 + 必备 CLI

### 1. 选队长 + 列候选成员

```bash
multica agent list --output json
# 挑一个会路由 / 会克制的 Agent 当队长，记下 slug
# 列出未来要加进来的成员（Agent slug + Member uuid）
```

如果队长缺一个会"派活 + 写 evaluation"的通用底子，先按 [`07-build-an-agent.md`](07-build-an-agent.md) 新建一个，并挂上 multica-wiki 这种通识 Skill。

### 2. 创建 Squad

```bash
multica squad create \
  --name "Frontend Team" \
  --description "前端 / 设计相关 issue 路由。" \
  --leader frontend-lead-agent \
  --instructions "$(cat ./squad-routing.md)" \
  --output json
# 记下返回的 squad id
```

可选：`--avatar-url <URL>`、`update` 时可以把 leader / instructions / name 都改了——见 `multica squad update --help`。

### 3. 加成员

```bash
multica squad member add <squad-id> --member-id <agent-uuid> --type agent --role "Owns the migrations"
multica squad member add <squad-id> --member-id <member-uuid> --type member --role "Reviewer of last resort"
multica squad member list <squad-id>     # 确认 roster
```

> ⚠️ **不能 `member remove` 队长**——先 `multica squad update <id> --leader <new-leader>` 换队长，再 remove 旧队长。
> ⚠️ **`member add --type` 必填**——Agent vs Member 的 ID 命名空间不同，类型错了会报"member not found"或加错对象。

### 4. 写 Squad Instructions（队长路由规则）

队长每次被触发，三段 briefing 会附加到它的 instructions 上（详见 [`11-squads.md`](11-squads.md)）：

1. **Squad Operating Protocol**（硬编码）—— 读 issue → `@` 派活 → 别复述 → 每次记 evaluation → **派完就停**。
2. **Squad Roster**（自动生成）—— 队长 + 每个成员，**带可直接复制的 mention markdown**。
3. **Squad Instructions**（你这一步要写的） —— 路由规则 + 上报策略。

写法建议（一段 markdown，传 `--instructions` 或在 UI 里编辑）：

```markdown
## 路由规则

- DB 迁移 / SQL → @Alice
- 前端组件 / CSS → @Bob
- 性能 / 监控 → @Carol
- 既不属于上述领域 → 在评论里 `@` reporter 问清楚再派

## 上报策略

- 成员 24h 没回 → `@` reporter 同步进度，必要时换人。
- 路由失败两次 → squad activity 写 `failed`，`@` workspace owner。
```

### 5. 验证：分一条最小 issue 给小队

```bash
multica issue create \
  --title "Smoke test for <Squad>" \
  --description "一段贴这个领域的最小任务。" \
  --assignee-id <squad-uuid> \
  --status todo

multica issue runs <issue-id> --output json
multica issue comment list <issue-id> --output json
```

**通过条件**：

- `multica issue runs` 里出现一条由**队长**领走的 task
- 队长发了一条评论，里面用 **mention markdown** `@` 了某个成员（不是纯文本 `@name`）
- 队长立即记了一次 squad activity（`action` / `no_action` / `failed` 三选一）：
  ```bash
  multica squad activity <issue-id> action --reason "委派给 @Alice 处理 DB migration"
  ```
- 被 `@` 的成员入队了一个新 task（`multica issue runs` 里多一条）
- **队长派完就停**——没有继续动手干活

任何一条不满足都说明小队没起效，进"常见翻车"。

## 常见翻车

| 翻车 | 原因 | 修法 |
|---|---|---|
| 队长不派活，自己埋头干 | 队长的 system prompt / 通用 Skill 没让它意识到"我是队长"；或 Squad Instructions 没写 | Squad Operating Protocol 是硬编码会自动追加，但**队长 Agent 自己的 instructions 不能顶它**——把"我是队长，职责是派活，不动手"显式写进队长 Agent 的 instructions |
| 队长 `@` 了人，但被 `@` 的成员没起来 | 用了**纯文本** `@name` 而不是 mention markdown `[@Name](mention://agent/<uuid>)`——纯文本不触发 | 让队长**只**用 Squad Roster 里现成的 markdown；在 Squad Instructions 里强调"复制粘贴 Roster 的 mention 段，不要自己拼" |
| 派活之后队长又被反复触发，循环写 evaluation | 成员没 `@` 任何人发"进展更新" → 队长被自动唤醒（这是预期行为） | 在 Squad Instructions 里写"被再次唤醒时，只在路由真的需要换人时才动作；其它情况记 `no_action`"——避免噪音轮 |
| 同一条 issue 上有多个活跃 task，混乱 | reassign / `@` 各种触发叠加 | 看 `multica issue runs` 找 `running` / `dispatched` 的 task；必要时先 unassign 清场（注意会**取消所有 Agent 的活跃 task**——见 [`appendix-cli.md`](appendix-cli.md)） |
| 想 `member remove` 队长 | 队长不能直接 remove | 先 `squad update --leader <new>` 换人，再 `member remove` 旧队长 |
| 归档小队后 issue 卡住 | 实际上**不会卡**——归档时当前分给小队的 issue 自动转给队长 Agent | 看 [`11-squads.md`](11-squads.md) 的"归档"小节确认行为；要恢复路由请重建小队，**没有反归档命令** |
| Backlog 状态 issue 分给小队没动静 | Backlog 是停泊场，**不会**触发队长 | 推到 `todo` 才会触发：`multica issue status <id> todo` |
| 把私人 Agent 加进小队，但其它成员看不到名字 | 私人 Agent 只是配置详情对其它成员隐藏，**名字仍然全员可见** | 真要彻底不可见的能力，重新设计——或者重建一个 workspace 可见的 Agent |
| `member add` 报 "member not found" | `--type` 给反了（Agent 写成 member 或反过来） | 显式按对象选：Agent 用 `--type agent --member-id <agent-uuid>`，Member 用 `--type member --member-id <member-uuid>` |
| 评论里 `@` 小队没触发队长 | 用了纯文本 `@SquadName` 而不是 `[@SquadName](mention://squad/<uuid>)` | 用 picker 插入小队，或显式拼 mention markdown |

## 把它接进工作流

新小队上线后，常见接法：

1. **告诉团队 "新领域走 @SquadName"**——在 issue / 项目说明 / 团队群里同步路由名。
2. **Autopilot 把生成的 issue 直接分给小队**——批量定时任务（"每天扫一次 todo 然后路由"）天然适合 Squad 的"按主题派活"模型。详见 [`17-build-an-autopilot.md`](17-build-an-autopilot.md)。
3. **跑几轮真实 issue 后回看 squad activity**：
   ```bash
   multica squad activity <issue-id> list --output json
   ```
   看队长写的 `reason` 是不是稳定贴 Squad Instructions——不是就回去改 instructions。
4. **扩队 ≠ 重建**——加新人用 `member add`，已有 issue 自动用上新成员，不需要回去改任何 issue。

## 不在这一章里的细节

| 想查什么 | 去哪一章 |
|---|---|
| Squad 模型 / 队长 briefing 三段 / 自动唤醒规则 | [11-squads.md](11-squads.md) |
| reassign / unassign 的副作用（会取消所有 Agent 活跃 task） | [appendix-cli.md](appendix-cli.md), [04-issue-lifecycle.md](04-issue-lifecycle.md) |
| `@` 一个 Agent / Member / Squad 的副作用差异 | [05-comments-and-mentions.md](05-comments-and-mentions.md) |
| 队长 Agent 的新建 / 决策清单 | [07-build-an-agent.md](07-build-an-agent.md) |
| 命令完整 flag | `multica squad <subcommand> --help` / `multica squad member <subcommand> --help` / `multica squad activity --help` |

下一步：[17-build-an-autopilot.md](17-build-an-autopilot.md) — 让 Agent 按 cron / webhook 自动开工。

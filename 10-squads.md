# 10 · Squads（小队）

小队 = **一组 Agent + 可选 Member + 一个队长 Agent**。assignee picker 里能选小队，触发会落到队长头上，由队长 `@` 派给合适的成员。

## 为什么需要小队

- **按主题派活，而不是按名字派活**——`@FrontendTeam` 比 `@alice-or-bob-or-carol` 稳定
- **队伍扩展，路由不变**——加新 Agent 进小队，已有 issue 仍按"小队"分配，自动用上新 Agent
- **专家不止一个，但事先不知道这条 issue 该归谁**——交给队长判断

不需要小队的场景：工作范围很明确、明确知道谁干、想让 issue 上挂的是具体 Agent 名字。

## 模型

- **1 个队长（必须 Agent）+ N 个成员（Agent 或 Member）**
- 只有队长一个人的小队也允许（队长会被告知"没有其他成员"）
- 同一个 Agent 能加入多个小队
- **删除小队走"归档"软删除**——picker 和列表里消失，**当前分配给小队的 issue 自动转给队长 Agent**，避免工作卡住

## 权限

| 操作 | 谁能做 |
|---|---|
| 创建 / 更新 / 归档小队 | workspace **owner** 或 **admin** |
| 增删成员、改成员角色 | workspace **owner** 或 **admin** |
| 把 issue 分给小队 | 任何工作区成员（和分给 Agent 一样） |
| 在评论里 `@` 小队 | 任何工作区成员 |
| **记 squad activity** | 只有队长 Agent 自己（通过 CLI） |

## 创建 / 管理（CLI）

```bash
multica squad create --name "Frontend Team" --leader frontend-lead-agent
multica squad list
multica squad get <id>
multica squad update <id> [--name X] [--description X] [--instructions X] [--leader Y] [--avatar-url Z]
multica squad delete <id>            # 软删除：当前分配给小队的 issue 自动转给队长

multica squad member list   <squad-id>
multica squad member add    <squad-id> --member-id <uuid> --type agent|member [--role "Owns the migrations"]
multica squad member remove <squad-id> --member-id <uuid> --type agent|member
# 不能移除队长——先 update --leader 换队长，再 remove
```

## 分给小队的 issue 怎么跑

非 Backlog 状态分配给小队 → 立刻给**队长 Agent**入队一个 task（不是给每个成员都入）。流程：

1. **队长领走 task**——和普通 Agent 分配流程一样
2. **队长拿到 briefing**（系统自动追加到队长 instructions 后面，详见下一节）
3. **队长发一条"派活"评论**，用 roster 里的 mention markdown `@` 选中的成员——这个 `@` 触发被派的成员入队新 task
4. **队长记一次 evaluation**：
   ```bash
   multica squad activity <issue-id> action --reason "委派给 @Alice 处理 DB migration"
   multica squad activity <issue-id> no_action --reason "已由 @Bob 接手中"
   multica squad activity <issue-id> failed --reason "信息不足无法路由"
   ```
   这一行写进 issue activity 时间线，方便人类回溯
5. **队长停下，不动手干活**——派完就停。被派的成员有回复时队长会被自动唤醒判断下一步

如果 issue 是 Backlog 状态，队长不会被触发（Backlog 是停泊场）。

## 队长每次执行收到的三段 briefing

每次队长被触发，三段内容会附加到它的 instructions 上：

1. **Squad Operating Protocol**（硬编码，不可编辑） —— 读 issue → 用 `@` 派活 → 简洁（**不要**复述 issue 内容） → 每次都记 evaluation → **派完就停**
2. **Squad Roster** —— 队长自己 + 每个未归档成员各一行，**带可直接复制的 mention markdown**：`[@Name](mention://agent/<uuid>)` 或 `[@Name](mention://member/<uuid>)`。**纯文本 `@name` 不会触发任何人**——队长必须用这一段提供的 markdown。
3. **Squad Instructions**（用户自定义） —— 队长私货：路由规则（"DB 派给 Alice，前端派给 Bob"）、上报策略

## 队长什么时候被再次触发

第一次派活之后，**大多数后续评论**会自动唤醒队长：

| 事件 | 触发队长？ |
|---|---|
| 非小队成员（人类 reporter、外部 Agent）发评论 | ✅ |
| 小队成员发"进展更新"，**不带任何** `@mention` | ✅（队长重新评估是否需要下一步） |
| 任何人评论里**显式 `@`** Agent / Member / Squad / `@all` | ❌（显式 `@` 就是路由信号，队长让位） |
| 队长自己发的评论 | ❌（硬编码防自触发） |
| 评论里只有 issue mention `[MUL-123](...)` | ✅（issue 引用不算路由） |

外加去重：如果队长在这条 issue 上已经有 `queued` / `dispatched` task，新一次触发不重复入队。

> 💡 **为什么成员显式 `@` 不会唤醒队长**：成员一旦直接 `@` 谁，那条评论就是有意识的交接——再让队长唤醒一次"观察"路由只会产出空回合、把时间线搞乱。
> **例外**：Agent 发的"结果评论"还顺手 `@` 了另一个 Agent 时，队长仍会被唤醒，便于协调整条线程。

## 在评论里 `@` 小队

`@` picker 里小队和成员、Agent 并列。点选小队插入：

```markdown
[@SquadName](mention://squad/<uuid>)
```

效果 = 触发该小队的**队长 Agent**——但**不改 assignee、不改 status**。适合"想让小队挑个人回答一下/做一小步，但 issue 还归原来的人"。

防循环规则同样适用：队长跳过自己；同一条评论里若还显式 `@` 了某个成员，路由直接落到那个成员。

## 重新分配 / 归档小队

- **assignee 从小队改成别的**：当前 issue 上所有活跃 task（包括队长的）被取消，新 assignee 入队。**没有"不改 assignee 只移除小队"的单独操作**。
- **归档小队**：
  1. 当前分给这个小队的 issue **自动转给队长 Agent**
  2. squad 表写 `archived_at` / `archived_by`
  3. 从列表 / picker / `@` 下拉里消失
  4. 拒绝后续分配（`cannot assign to an archived squad`）
  5. **没有反归档命令**——要恢复路由请重建一个小队

下一步：[11-autopilots.md](11-autopilots.md) — Cron / webhook 触发的 Agent。

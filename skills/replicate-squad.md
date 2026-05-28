---
name: replicate-squad
description: Use when 用户已经有一套"队长 + 多名专家 Agent"的协作体系（来自 Claude Code 的 sub-agent / Cursor 的 multi-agent / 人脑里的"前端找 X、后端找 Y"流程），想要把它一比一复刻成 Multica 小队（squad），让 Multica issue 也能按主题路由而不是按名字路由。
outcome: 工作区里出现一个新的 squad；队长 + N 个成员都在；squad instructions 写清楚路由规则；用一条测试 issue 跑通"分给 squad → 队长 @ 派活 → 成员领走"完整链路；失败可一键归档回滚。
assumes: multica-handbook loaded
---

# 剧本：把现有 Agent 协作体系复刻成 Multica 小队

## 何时用 / 何时不用

**用**：

- 用户描述出"某类活归某个角色"的稳定模式（"DB 找 Alice，前端找 Bob，文档找 Carol"），且这套模式他已经在别处跑过、要搬到 Multica。
- 团队希望"按主题派活，不按名字派活"——加新人路由不变。

**不用**：

- 用户只想跑一次性任务、明确知道谁干——直接把 issue 分给那个 Agent，**不要为此建小队**。
- 团队还没想清楚谁负责什么——先回到澄清环节，不要在结构没定型时落地 squad；squad 改名/换队长有副作用（active task 会被取消）。

## 五步走

### Step 1 · 厘清角色清单（不动 CLI）

跟用户对齐三个问题，**用文字写下答案**——后面三步全靠这份清单：

1. **队长是谁？** 必须是 Agent（不能是 Member）。队长的工作是**判断 + 派活 + 评估**，不动手干活。
2. **成员有哪些？** 每个成员一行，写清楚：`<名字> | agent|member | <负责什么>`。
3. **路由规则？** 用一句话能说清——"DB migration / schema 变更 → Alice；React 组件 / 样式 → Bob；文档 / 设计稿 → Carol；其余先问 reporter"。这一句会原样进 squad instructions。

> ⚠️ **没有清单就不要继续**——后面建 squad、加成员都要照这份清单做；缺信息就停下问用户，不要脑补角色。

### Step 2 · 建 squad 并加成员

```bash
# 列出工作区已有 Agent / Member，先确认队长和成员都在
multica agent list  --output json
multica member list --output json   # 如有 member CLI；没有就在 UI 里看

# 建 squad（--leader 接 agent 的 name 或 uuid）
multica squad create \
  --name "Frontend Team" \
  --leader frontend-lead-agent \
  --description "前端 / 文档 / 样式相关的 issue 都派到这里" \
  --instructions "$(cat <<'EOF'
路由规则：
- DB / 后端 schema → @Alice
- React / 样式 / a11y → @Bob
- 文档 / 设计稿 → @Carol
- 不确定的先问 reporter，不要瞎派
EOF
)"

# 记下返回的 squad-id（下一步要用）
SQUAD_ID=<上一条返回的 id>

# 按 Step 1 清单加成员（队长不用 add——create 时已经设了）
multica squad member add $SQUAD_ID --member-id <alice-agent-uuid> --type agent  --role "DB / 后端 schema"
multica squad member add $SQUAD_ID --member-id <bob-agent-uuid>   --type agent  --role "React / 样式"
multica squad member add $SQUAD_ID --member-id <carol-member-uuid> --type member --role "文档 / 设计稿"

# 校对
multica squad get        $SQUAD_ID --output json
multica squad member list $SQUAD_ID --output json
```

> ⚠️ **`--instructions` 是队长的"私货"**——路由规则、上报策略写在这里，会附加到队长每次执行的 instructions 末尾。**不要**写对象模型介绍（已经在 handbook 里）。
>
> ⚠️ **不要把 multica-handbook 重复贴到 instructions 里**——Agent 上下文里 handbook 已经被加载，重复只会浪费 token 并在两份不一致时制造混乱。

### Step 3 · 跑一条测试 issue 验证路由

```bash
# 建一条明显该派给某成员的 issue，分配给 squad
multica issue create \
  --title "[squad-test] 验证小队路由：改一行 README" \
  --description "这是 squad 路由测试。预期：队长把这条派给负责文档的成员。完成后归档此 issue。" \
  --status todo \
  --assignee-id $SQUAD_ID
# 注意：assignee-id 接 squad-id，--assignee 短名也行——picker 一视同仁
```

观察 30~60 秒后看：

```bash
# 看 issue 当前评论
multica issue comment list <test-issue-id> --recent 5 --output json

# 看 squad activity 时间线（队长会在派活后立刻记一条）
multica squad activity <test-issue-id> list --output json   # 如该子命令存在；否则在 UI 看
```

**通过的标志**（必须同时满足）：

1. 队长发了一条评论，里面有形如 `[@Bob](mention://agent/<uuid>)` 的 mention markdown（**纯文本 `@Bob` 不算**）
2. 被 mention 的成员入队了一个新 task（`multica issue get` 看 assignee 不变，但 task 列表多一条）
3. 队长记了一条 `multica squad activity ... action --reason "..."`

**任一不满足就是路由失败**——回 Step 4 调 instructions 再试。

### Step 4 · 调整 / 失败回滚

如果 Step 3 路由错了（比如该派给 Bob 的派给了 Alice），改 instructions：

```bash
multica squad update $SQUAD_ID --instructions "$(cat <<'EOF'
路由规则（v2，明确 a11y 也归 Bob）：
- DB / 后端 schema → @Alice
- React / 样式 / a11y → @Bob
- 文档 / 设计稿 → @Carol
EOF
)"
```

改完**重跑 Step 3**——队长拿到新 instructions 才会按新规则。

如果整个 squad 设计错了（角色划分本身有问题），归档回滚：

```bash
multica squad delete $SQUAD_ID
# 软删除效果：
# - 当前分给该 squad 的 issue 自动转给队长 Agent（不会卡住）
# - picker / 列表 / @ 下拉里消失
# - 没有反归档命令——要恢复请重建一个新 squad
```

### Step 5 · 把测试 issue 归档并回执

```bash
# 测试 issue 改 done 或 cancelled
multica issue status <test-issue-id> done

# 在原始需求 issue 上发结果评论（不要 @mention 用户提的对方 Agent 来收尾）
multica issue comment add <user-issue-id> --content-stdin <<'EOF'
小队已建好：<squad-name>（id: <squad-id>）
- 队长：<leader>
- 成员：Alice / Bob / Carol（角色见 squad instructions）
- 路由测试 issue：[MUL-XXX](mention://issue/<test-issue-id>) 已通过

后续往该方向派活时，assignee 选 <squad-name> 即可。
EOF
```

## 红线（剧本特定）

1. **队长不能是 Member**——只能是 Agent；选错了 `squad create` 会报错。
2. **不要在 instructions 里 `@` 任何人**——instructions 是给队长看的私货，不是要发到时间线的；真要 `@` 人是队长在评论里做。
3. **测试通过前不要让 squad 接收真实业务 issue**——路由错了会污染真实工作流；先把 Step 3 跑绿。
4. **回滚走 `squad delete`，不要去逐个 remove member 然后改 leader**——软删除是平台支持的回滚路径，手动拆解会留下半成品。

## 深入参考

- 小队模型、队长 briefing 三段式、唤醒规则：[`multica-handbook/10-squads.md`](multica-handbook/10-squads.md)
- assignee 与状态触发关系：[`multica-handbook/04-issue-lifecycle.md`](multica-handbook/04-issue-lifecycle.md)
- mention 的副作用与防循环：[`multica-handbook/05-comments-and-mentions.md`](multica-handbook/05-comments-and-mentions.md)
- 完整 CLI flag：[`multica-handbook/03-cli-cheatsheet.md`](multica-handbook/03-cli-cheatsheet.md)

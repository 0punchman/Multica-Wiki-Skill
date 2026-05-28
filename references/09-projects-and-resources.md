# 08 · Project 与项目资源

## Project（项目）

把 issue 组织成 sprint / epic / 工作流的容器：

- 一条 issue **最多属于 1 个 project**（也可以不属于任何 project）
- Project 有自己的 `lead`（负责人，可以是人或 Agent）
- 删除 project **不删 issue**（issue 脱离 project 但留在 workspace）

```bash
multica project list [--status X]
multica project create --title "2026 W22 Sprint" --icon "🏃" --lead alice
multica project update <id> --title X --status in_progress
multica project status <id> in_progress    # planned/in_progress/paused/completed/cancelled
multica project delete <id>
```

## ProjectResource（项目资源）

挂在 project 上的**有类型的指针**——告诉所有该 project 下 issue 的 Agent："你应该 checkout 哪个仓库 / 在哪个目录里干活"。

每个资源是 `(resource_type, resource_ref)` 的对子：

- `resource_type`：`github_repo` 或 `local_directory`（未来还会加）
- `resource_ref`：按类型定型的 JSON payload

任务启动时 daemon 会把项目的资源列表写到 Agent 工作目录的两个位置：

1. **`.multica/project/resources.json`** —— 结构化数据，给 skill / 脚本 / Agent 直接 parse
2. Agent 元 skill 的 **`## Project Context`** 段落——给 Agent prompt 的人类可读摘要

资源拉取**best-effort**：API 调失败 → 段落省掉、文件不写，但任务照常启动。

## 资源类型 1：`github_repo`

默认类型——每次任务都在隔离 worktree 里 checkout：

```json
{
  "resource_type": "github_repo",
  "resource_ref": {
    "url": "https://github.com/owner/repo",
    "default_branch_hint": "main"
  }
}
```

`default_branch_hint` 可选，写了之后 Agent 会知道在哪个分支干活。

```bash
multica project resource add <project-id> --type github_repo --url https://github.com/owner/repo
```

Agent 在工作目录里执行 `multica repo checkout <url>` 就能拉到代码——daemon 会自动建一个独立 worktree，在一个独立分支上。

## 资源类型 2：`local_directory`

不适合每次重新 clone 的仓库（几十 GB 游戏项目、巨型 monorepo），改为指向**某台 daemon 机器上一个已有的目录**。Agent **直接在这个目录里运行**，不 clone、不复制。

```json
{
  "resource_type": "local_directory",
  "resource_ref": {
    "local_path": "/Users/me/code/big-game",
    "daemon_id": "0001234e-...",
    "label": "主开发目录"
  }
}
```

```bash
multica project resource add <project-id> \
  --type local_directory \
  --local-path /Users/me/code/big-game \
  --daemon-id <daemon-uuid> \
  --ref-label "主开发目录"
```

`--daemon-id` 用 `multica daemon list` 拿。

### 取舍对比

| 关注点 | `github_repo`（worktree） | `local_directory` |
|---|---|---|
| 每次任务 checkout 成本 | 重新 clone + worktree | 0 |
| 同仓库并发 | 多任务并行 | **同目录串行**（一次 1 条） |
| 分支 / 脏改动 | 每次干净分支 | 用目录当前状态 |
| 哪台机器跑 | 任意 daemon | 仅绑定那台 |
| 磁盘占用 | 每条任务一份 worktree | 0 |

什么时候选 `local_directory`：

1. Clone 成本盖过实际工作（巨型仓库 / LFS）
2. 高频在本地 IDE 里 review 改动，不想反复切 worktree

代价是**没有文件级写入锁**——同目录串行执行是唯一的并发保护。要真并发请继续用 `github_repo`。

### 路径规则（添加 `local_directory` 时校验）

- 必须是**绝对路径**
- 必须**存在**且是**目录**（不能是普通文件 / 设备 / 指向文件的 symlink）
- daemon 进程必须能读 + 写
- **不能**是系统根 / 用户根 / 关键系统目录：`/`、`/Users`、`/home`、`/root`、`/etc`、`/tmp`、`/var`、`/usr`、`/opt`、`/Users/Shared`、`$HOME`、Windows 盘根、`C:\Users`、`C:\Windows` 等
- 指向上述路径的 symlink 也会被拒
- macOS 上 `/private/tmp` 与 `/tmp` 同等被拒
- **每台 daemon 每个项目最多 1 条 `local_directory`**——同机重复 add 会被 API 用 409 拒

### Multica 会写、不会动什么

- **会写入** `CLAUDE.md` / `AGENTS.md`（按 provider）和 `.multica/project/resources.json` 到目录根。如果不想 commit 进 git，加进 `.gitignore`。
- **会写入**所有 Agent 决定的代码改动（和你自己跑这个 Agent 没区别）
- **绝不物理删除**你的目录或里面任何文件——Multica 的 GC 只清自己 `workspacesRoot` 下的 `output/` 与 `logs/`，**完全不碰**你的目录

### `local_directory` v1 阶段的限制

- **不会自动切分支**——Agent 跑在你目录当前的分支上。任务前自己切好。
- **不保护脏工作区，也不自动 commit**——脏改动 Agent 看得到、可能就地修改。重要的运行前自己 commit。
- **不自动开 PR**——任务结束后改动停在它做出来的分支上。需要 push / PR 自己来。
- **`waiting_local_directory` 只显示状态，不显示是谁占着**

## 工作区仓库 vs project 仓库的优先级

Agent 看到的仓库列表（`CLAUDE.md` / `AGENTS.md` 里的 `## Repositories` 段）：

| 情况 | 显示什么 |
|---|---|
| project 挂了 ≥1 条 `github_repo` | **只**显示项目挂的仓库（工作区绑定的仓库被隐藏，避免猜哪个属于这条 issue） |
| project 没挂 `github_repo`（或 issue 不在任何 project） | 回落到工作区的仓库列表 |

`.multica/project/resources.json` **始终带完整列表**——需要看全部资源的 skill 仍能拿到。

`multica repo checkout <url>` 能处理任意 URL（包括只挂在 project 上、没绑到 workspace 的仓库）——因为 daemon 在领任务时会把项目级 URL 合并进白名单。

## 当前**不**做的事

- **跨项目共享**——每条资源只能挂在一个 project 上
- **按 skill 划定资源可见性**——所有资源对本次 run 的所有 skill 可见
- **缓存 / 同步**——`github_repo` 仍按需 `multica repo checkout`，没有预缓存

下一步：[10-skills.md](10-skills.md) — Skill 是什么、放在哪、第三方 Skill 安全。

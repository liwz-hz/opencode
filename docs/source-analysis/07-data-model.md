# OpenCode 核心数据模型分析

本文档分析 OpenCode 的核心数据模型：Project、Session、Message、Part 的层级关系、数据结构、存储机制及设计问题。

## 目录

1. [概述](#概述)
2. [层级关系](#层级关系)
3. [数据模型定义](#数据模型定义)
4. [数据库存储](#数据库存储)
5. [数据查询流程](#数据查询流程)
6. [Project 创建场景](#project-创建场景)
7. [Session 创建流程](#session-创建流程)
8. [Instance 与 Project 的区别](#instance-与-project-的区别)
9. [设计问题分析](#设计问题分析)
10. [参考文件](#参考文件)

---

## 概述

OpenCode 采用**全局数据库 + 按项目过滤**的数据存储架构：

- **全局数据库**：所有项目数据存储在同一个 SQLite 数据库文件
- **层级结构**：Project → Session → Message → Part
- **关联方式**：通过 `project_id`、`session_id`、`message_id` 字段关联
- **级联删除**：数据库层面配置了 `onDelete: "cascade"`

---

## 层级关系

### 四层层级结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Project (项目)                                                           │
│ - 代表一个 Git Worktree 或非 Git 目录                                     │
│ - ID 基于 Git 仓库首个 commit hash                                       │
│ - 一个 Project 可包含多个 Sandbox（工作目录）                              │
└─────────────────────────────────────────────────────────────────────────┘
                    │ 1:N (通过 project_id 关联)
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Session (会话)                                                           │
│ - 代表一次对话                                                            │
│ - 关联到 Project 和具体工作目录                                            │
│ - 可有 parentID 形成父子关系（子 Agent 会话）                              │
└─────────────────────────────────────────────────────────────────────────┘
                    │ 1:N (通过 session_id 关联)
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Message (消息)                                                           │
│ - Session 中的一条消息（user 或 assistant）                               │
│ - 包含时间戳、错误信息、模型信息等元数据                                    │
└─────────────────────────────────────────────────────────────────────────┘
                    │ 1:N (通过 message_id 关联)
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Part (内容片段)                                                           │
│ - Message 中的一个内容单元                                                 │
│ - 类型包括：text, file, tool, reasoning, snapshot, patch 等              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 数据模型定义

### Project 数据结构

```typescript
// src/project/project.ts
export const Info = z.object({
  id: ProjectID.zod, // 项目唯一标识（基于 git 首个 commit hash）
  worktree: z.string(), // Git Worktree 根目录
  vcs: z.literal("git").optional(), // 版本控制系统类型
  name: z.string().optional(), // 项目名称
  icon: z
    .object({
      // 项目图标
      url: z.string().optional(),
      color: z.string().optional(),
    })
    .optional(),
  commands: z
    .object({
      // 项目命令
      start: z.string().optional(), // 启动脚本
    })
    .optional(),
  time: z.object({
    created: z.number(),
    updated: z.number(),
    initialized: z.number().optional(),
  }),
  sandboxes: z.array(z.string()), // 属于此 Project 的所有工作目录列表
})
```

**关键字段说明**：

| 字段        | 说明                                         |
| ----------- | -------------------------------------------- |
| `id`        | 基于 Git 仓库首个 commit hash 生成，全局唯一 |
| `worktree`  | Git 仓库的主工作目录                         |
| `sandboxes` | 所有 Git Worktree 目录列表                   |
| `vcs`       | 版本控制系统类型，仅支持 "git"               |

### Session 数据结构

```typescript
// src/session/index.ts
export const Info = z.object({
  id: SessionID.zod,
  slug: z.string(), // URL 友好的短标识
  projectID: ProjectID.zod, // 关联 Project（关键关联）
  workspaceID: WorkspaceID.zod.optional(), // Workspace ID（可选）
  directory: z.string(), // 当前工作目录
  parentID: SessionID.zod.optional(), // 父 Session（子 Agent 用）
  title: z.string(), // 会话标题
  version: z.string(), // OpenCode 版本
  summary: z
    .object({
      // 代码变更摘要
      additions: z.number(),
      deletions: z.number(),
      files: z.number(),
      diffs: Snapshot.FileDiff.array().optional(),
    })
    .optional(),
  share: z.object({ url: z.string() }).optional(), // 分享链接
  revert: z
    .object({
      // 回滚信息
      messageID: MessageID.zod,
      partID: PartID.zod.optional(),
      snapshot: z.string().optional(),
      diff: z.string().optional(),
    })
    .optional(),
  permission: Permission.Ruleset.optional(), // 权限规则
  time: z.object({
    created: z.number(),
    updated: z.number(),
    compacting: z.number().optional(),
    archived: z.number().optional(),
  }),
})
```

**关键字段说明**：

| 字段        | 说明                                     |
| ----------- | ---------------------------------------- |
| `projectID` | 关联到 Project，是数据过滤的核心字段     |
| `directory` | 当前工作目录，区分不同 Worktree          |
| `parentID`  | 父 Session ID，用于子 Agent 会话层级关系 |

### Message 数据结构

```typescript
// src/session/message-v2.ts
export const User = z
  .object({
    id: MessageID.zod,
    sessionID: SessionID.zod, // 关联 Session（关键关联）
    role: z.literal("user"),
    error: z.any().optional(),
    time: z.object({ created: z.number() }),
  })
  .meta({ ref: "MessageUser" })

export const Assistant = z
  .object({
    id: MessageID.zod,
    sessionID: SessionID.zod, // 关联 Session
    role: z.literal("assistant"),
    parentID: MessageID.zod.optional(), // 关联的 user 消息
    modelID: ModelID.zod.optional(),
    providerID: ProviderID.zod.optional(),
    error: APIError.Schema.optional(),
    time: z.object({
      created: z.number(),
      updated: z.number().optional(),
    }),
  })
  .meta({ ref: "MessageAssistant" })
```

**关键字段说明**：

| 字段        | 说明                               |
| ----------- | ---------------------------------- |
| `sessionID` | 关联到 Session                     |
| `parentID`  | assistant 消息关联对应的 user 消息 |

### Part 数据结构

```typescript
// src/session/message-v2.ts
const PartBase = z.object({
  id: PartID.zod,
  sessionID: SessionID.zod, // 冗余存储，便于查询
  messageID: MessageID.zod, // 关联 Message（关键关联）
})

// Part 类型（部分示例）
export const TextPart = PartBase.extend({
  type: z.literal("text"),
  text: z.string(),
})

export const ToolPart = PartBase.extend({
  type: z.literal("tool"),
  callID: z.string(),
  tool: z.string(),
  state: ToolState, // pending | running | completed | error
})

export const FilePart = PartBase.extend({
  type: z.literal("file"),
  mime: z.string(),
  url: z.string(),
  filename: z.string().optional(),
})

export const ReasoningPart = PartBase.extend({
  type: z.literal("reasoning"),
  text: z.string(),
})

// ... 还有 snapshot, patch, agent, compaction 等类型
```

**Part 类型列表**：

| 类型         | 说明                     |
| ------------ | ------------------------ |
| `text`       | 文本内容                 |
| `tool`       | 工具调用及结果           |
| `file`       | 文件引用（图片、PDF 等） |
| `reasoning`  | 推理过程                 |
| `snapshot`   | 代码快照                 |
| `patch`      | 代码补丁                 |
| `agent`      | Agent 引用               |
| `compaction` | 压缩标记                 |

---

## 数据库存储

### 全局数据库位置

```typescript
// src/global/index.ts
const data = path.join(xdgData!, "opencode")

// src/storage/db.ts
return path.join(Global.Path.data, "opencode.db")
```

**实际路径**（Linux）：

| 平台    | 路径                                                 |
| ------- | ---------------------------------------------------- |
| Linux   | `~/.local/share/opencode/opencode.db`                |
| macOS   | `~/Library/Application Support/opencode/opencode.db` |
| Windows | `%APPDATA%/opencode/opencode.db`                     |

### 数据库表结构

```typescript
// src/project/project.sql.ts
export const ProjectTable = sqliteTable("project", {
  id: text().$type<ProjectID>().primaryKey(),
  worktree: text().notNull(),
  vcs: text(),
  name: text(),
  icon_url: text(),
  icon_color: text(),
  time_created: integer(),
  time_updated: integer(),
  time_initialized: integer(),
  sandboxes: text({ mode: "json" }).notNull().$type<string[]>(),
  commands: text({ mode: "json" }).$type<{ start?: string }>(),
})

// src/session/session.sql.ts
export const SessionTable = sqliteTable(
  "session",
  {
    id: text().$type<SessionID>().primaryKey(),
    project_id: text()
      .$type<ProjectID>()
      .notNull()
      .references(() => ProjectTable.id, { onDelete: "cascade" }), // 级联删除
    workspace_id: text().$type<WorkspaceID>(),
    parent_id: text().$type<SessionID>(),
    slug: text().notNull(),
    directory: text().notNull(),
    title: text().notNull(),
    version: text().notNull(),
    // ... 其他字段
  },
  (table) => [
    index("session_project_idx").on(table.project_id), // 项目索引
    index("session_parent_idx").on(table.parent_id), // 父会话索引
  ],
)

export const MessageTable = sqliteTable(
  "message",
  {
    id: text().$type<MessageID>().primaryKey(),
    session_id: text()
      .$type<SessionID>()
      .notNull()
      .references(() => SessionTable.id, { onDelete: "cascade" }), // 级联删除
    time_created: integer(),
    time_updated: integer(),
    data: text({ mode: "json" }).notNull().$type<InfoData>(),
  },
  (table) => [index("message_session_time_created_id_idx").on(table.session_id, table.time_created, table.id)],
)

export const PartTable = sqliteTable(
  "part",
  {
    id: text().$type<PartID>().primaryKey(),
    message_id: text()
      .$type<MessageID>()
      .notNull()
      .references(() => MessageTable.id, { onDelete: "cascade" }), // 级联删除
    session_id: text().$type<SessionID>().notNull(), // 冗余存储
    time_created: integer(),
    time_updated: integer(),
    data: text({ mode: "json" }).notNull().$type<PartData>(),
  },
  (table) => [
    index("part_message_id_id_idx").on(table.message_id, table.id),
    index("part_session_idx").on(table.session_id),
  ],
)
```

### 数据库存储示意

```
opencode.db（全局唯一）
│
├── project 表
│     ├── id: "abc123", worktree: "~/projects/A"  ← Project A
│     ├── id: "def456", worktree: "~/projects/B"  ← Project B
│     └── id: "global", worktree: "/"             ← 全局 Project
│
├── session 表
│     ├── id: "sess_001", project_id: "abc123"    ← 属于 Project A
│     ├── id: "sess_002", project_id: "abc123"    ← 属于 Project A
│     ├── id: "sess_003", project_id: "def456"    ← 属于 Project B
│     └── id: "sess_004", project_id: "global"    ← 属于全局 Project
│
├── message 表
│     ├── id: "msg_001", session_id: "sess_001"   ← 属于 Session 001
│     └── id: "msg_002", session_id: "sess_001"   ← 属于 Session 001
│
└── part 表
│     ├── id: "part_001", message_id: "msg_001"   ← 属于 Message 001
│     └── id: "part_002", message_id: "msg_001"   ← 属于 Message 001
```

---

## 数据查询流程

### 打开 OpenCode 时的数据查询

```
用户在目录 A 执行 opencode
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 读取全局数据库                                            │
│    - 打开 ~/.local/share/opencode/opencode.db                │
│    - 所有 Project/Session/Message/Part 都在这里               │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Project.fromDirectory(A)                                  │
│    - 检测 Git 仓库                                            │
│    - 生成/读取 ProjectID                                      │
│    - 查询 project 表：SELECT * FROM project WHERE id = ?      │
│    - Upsert（不存在则 INSERT，存在则 UPDATE）                  │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Session.list()                                            │
│    - 从 ALS 上下文获取当前 Project                            │
│    - 查询当前 Project 的所有 Session                          │
│                                                              │
│    SELECT * FROM session                                     │
│    WHERE project_id = 'abc123'   ← 关键过滤                   │
│    ORDER BY time_updated DESC                                │
│                                                              │
│    只返回属于这个 Project 的 Session                          │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Message.messages(sessionID)                               │
│    - 查询某个 Session 的所有 Message                          │
│                                                              │
│    SELECT * FROM message                                     │
│    WHERE session_id = 'sess_001'                             │
│                                                              │
│    再查询 Part：                                              │
│    SELECT * FROM part WHERE message_id = 'msg_001'           │
└─────────────────────────────────────────────────────────────┘
```

### 关键查询代码

```typescript
// src/session/index.ts 第 702-703 行
export function* list(input?) {
  const project = Instance.project // 从 ALS 上下文获取当前 Project
  const conditions = [eq(SessionTable.project_id, project.id)] // ← 关键过滤
  // ...
  // 只返回当前 Project 的 Session
}
```

---

## Project 创建场景

### 场景一：Git Worktree

**核心概念**：Git Worktree 允许一个 Git 仓库在多个目录同时检出不同分支。

```
Git Repository (共享 .git 目录)
    │
    ├── worktree/           ← 主工作目录（Project.worktree）
    │   └── .git            ← Git 目录
    │       └── opencode    ← 缓存 ProjectID
    │
    ├── sandbox-1/          ← Git Worktree 1（Project.sandboxes[0]）
    │   └── .git (文件)     ← 指向 ../worktree/.git
    │
    └── sandbox-2/          ← Git Worktree 2（Project.sandboxes[1]）
        └── .git (文件)     ← 指向 ../worktree/.git
```

**关键点**：

| 要点           | 说明                                                |
| -------------- | --------------------------------------------------- |
| ProjectID 共享 | 同一个 Git 仓库的所有 Worktree 共享同一个 ProjectID |
| ProjectID 生成 | 基于 Git 仓库首个 commit hash                       |
| ProjectID 缓存 | 存储在 `.git/opencode` 文件中                       |
| sandboxes 字段 | 记录所有属于此 Project 的工作目录                   |

**代码实现**：

```typescript
// src/project/project.ts
const revList = yield * git(["rev-list", "--max-parents=0", "HEAD"], { cwd: sandbox })
const roots = revList.text
  .split("\n")
  .filter(Boolean)
  .map((x) => x.trim())
  .toSorted()
id = roots[0] ? ProjectID.make(roots[0]) : undefined

// 缓存 ProjectID
if (id) {
  yield * fs.writeFileString(path.join(worktree, ".git", "opencode"), id)
}
```

### 场景二：多个独立 Git 仓库

每个独立的 Git 仓库有独立的历史，生成不同的 ProjectID：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Project A       │     │ Project B       │     │ Project C       │
│ (repo-a.git)    │     │ (repo-b.git)    │     │ (repo-c.git)    │
│ ID: abc123      │     │ ID: def456      │     │ ID: ghi789      │
│                 │     │                 │     │                 │
│ ├── worktree/   │     │ ├── worktree/   │     │ ├── worktree/   │
│ └── sandbox/    │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
  Session 1...N           Session 1...M           Session 1...K
```

**触发方式**：用户在不同 Git 仓库目录启动 OpenCode

### 场景三：非 Git 目录（全局 Project）

当目录不在 Git 仓库内时，使用"全局 Project"：

```typescript
// src/project/project.ts
if (!dotgit) {
  return {
    id: ProjectID.global, // 全局 Project ID
    worktree: "/",
    sandbox: "/",
    vcs: fakeVcs,
  }
}
```

**特点**：

| 特点                | 说明                                  |
| ------------------- | ------------------------------------- |
| 统一 ProjectID      | 所有非 Git 目录使用同一个全局 Project |
| worktree 和 sandbox | 都设置为 "/"                          |
| 不支持 Worktree     | 无 Git 仓库，无法区分工作目录         |

---

## Session 创建流程

```
用户打开目录 /path/to/project
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Instance.provide({ directory })                           │
│    - 创建/获取 Instance 上下文                                │
│    - 通过 ALS (AsyncLocalStorage) 管理                       │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Project.fromDirectory(directory)                          │
│    - 查找 .git 目录                                           │
│    - 解析 Git Worktree 信息                                   │
│    - 生成/缓存 ProjectID                                      │
│    - Upsert Project 到数据库                                  │
│    - 返回 { project, sandbox }                                │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Session.create()                                          │
│    - 创建新 Session                                           │
│    - 关联 projectID 和 directory                              │
│    - 设置默认标题                                              │
│    - 存储到数据库                                              │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Message 创建（用户输入）                                    │
│    - 创建 User Message                                        │
│    - 添加 TextPart/FilePart 等                                │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Assistant Message 创建（LLM 响应）                         │
│    - 创建 Assistant Message                                   │
│    - 流式添加 TextPart, ToolPart, ReasoningPart 等            │
└─────────────────────────────────────────────────────────────┘
```

---

## Instance 与 Project 的区别

### Instance（运行时概念）

```typescript
// src/project/instance.ts
export interface InstanceContext {
  directory: string // 当前工作目录
  worktree: string // Git Worktree 根目录
  project: Project.Info // 关联的 Project 信息
}
```

**特点**：

| 特点         | 说明                              |
| ------------ | --------------------------------- |
| 运行时概念   | 通过 AsyncLocalStorage (ALS) 管理 |
| 每次打开创建 | 打开一个目录就创建一个 Instance   |
| 上下文提供   | 让代码可以访问当前目录和 Project  |
| 生命周期     | 随 OpenCode 进程结束而销毁        |

### Project（持久化概念）

**特点**：

| 特点             | 说明                              |
| ---------------- | --------------------------------- |
| 持久化概念       | 存储在数据库中                    |
| Git 仓库绑定     | 与 Git 仓库生命周期绑定           |
| 跨进程持久       | 数据永久保存                      |
| 多 Worktree 共享 | 同一 Git 仓库的所有 Worktree 共享 |

### 关系图

```
┌─────────────────────────────────────────┐
│ Instance (运行时 ALS 上下文)              │
│                                         │
│  directory ──────► 当前工作目录           │
│  worktree  ──────► Git Worktree 根目录   │
│  project   ──────► Project.Info (持久化) │
└─────────────────────────────────────────┘

打开目录 ~/projects/A/worktree：
  → Instance { directory: "~/projects/A/worktree", project: Project A }

打开目录 ~/projects/A/sandbox-1：
  → Instance { directory: "~/projects/A/sandbox-1", project: Project A }（同一 Project！）

打开目录 ~/projects/B：
  → Instance { directory: "~/projects/B", project: Project B }（不同 Project）
```

---

## 设计问题分析

### 问题一：全局数据库导致数据残留

**问题描述**：

当用户删除项目目录后，数据库中的数据不会自动清理。

```
用户场景：

~/projects/A/     ← Git 仓库 A
    │
    │  用户删除了这个目录！
    ▼
rm -rf ~/projects/A

数据库中的残留：
opencode.db
├── project: { id: "abc123", worktree: "~/projects/A" }  ← 还在！
├── session: { project_id: "abc123", ... }              ← 还在！
├── message: ...                                         ← 还在！
└── part: ...                                            ← 还在！

这些数据永远不会被自动清理！
```

**根本原因**：

| 原因       | 说明                               |
| ---------- | ---------------------------------- |
| 全局数据库 | 所有项目数据混在一起               |
| 无自动清理 | 没有检测项目目录是否存在的机制     |
| 无删除命令 | CLI 没有提供 `project delete` 命令 |

### 问题二：缺少 Project 管理命令

**现状**：

```bash
# 可用的 CLI 命令
opencode session list              ← 列出 Session
opencode session delete <id>       ← 删除单个 Session
opencode db                        ← 直接操作数据库

# 不存在的命令
opencode project list              ← ❌ 不存在
opencode project delete <id>       ← ❌ 不存在
opencode project clean             ← ❌ 不存在
```

**级联删除机制已配置，但没有入口**：

```typescript
// src/session/session.sql.ts
project_id: text().references(() => ProjectTable.id, { onDelete: "cascade" })

// 理论上：删除 Project → 级联删除 Session → Message → Part
// 但没有 project delete 命令来触发这个级联
```

### 问题三：数据库膨胀

**问题描述**：

随着项目增多，数据库会持续膨胀：

| 问题             | 影响                       |
| ---------------- | -------------------------- |
| 项目越多数据越多 | 所有项目的数据都在一起     |
| 废弃项目数据残留 | 删除的项目目录数据不会清理 |
| 无数据归档机制   | 没有数据归档或压缩策略     |

### 当前解决方案

用户只能手动操作数据库：

```bash
# 查看数据库位置
opencode db path
# 输出: ~/.local/share/opencode/opencode.db

# 手动查询项目列表
opencode db "SELECT id, worktree FROM project"

# 手动删除 Project（会级联删除所有相关数据）
opencode db "DELETE FROM project WHERE id = 'abc123'"

# 或直接使用 sqlite3
sqlite3 ~/.local/share/opencode/opencode.db
> DELETE FROM project WHERE worktree LIKE '%/已删除的项目';
```

### 建议改进

| 改进                       | 说明                           |
| -------------------------- | ------------------------------ |
| 添加 `project list` 命令   | 列出所有 Project               |
| 添加 `project delete` 命令 | 删除 Project 及级联数据        |
| 添加 `project clean` 命令  | 自动清理目录已不存在的 Project |
| 启动时检测废弃项目         | 检测 worktree 目录是否存在     |
| 数据归档机制               | 定期归档旧数据                 |

---

## 参考文件

| 文件                         | 说明                 |
| ---------------------------- | -------------------- |
| `src/global/index.ts`        | 全局路径配置         |
| `src/storage/db.ts`          | 数据库连接和管理     |
| `src/project/project.ts`     | Project 定义和服务   |
| `src/project/project.sql.ts` | Project 数据库表     |
| `src/project/instance.ts`    | Instance 上下文管理  |
| `src/session/index.ts`       | Session 定义和服务   |
| `src/session/session.sql.ts` | Session 数据库表     |
| `src/session/message-v2.ts`  | Message 和 Part 定义 |
| `src/cli/cmd/session.ts`     | Session CLI 命令     |
| `src/cli/cmd/db.ts`          | 数据库 CLI 命令      |

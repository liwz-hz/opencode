# OpenCode 核心数据模型与同步架构分析

本文档从系统架构角度，深入分析 OpenCode 的数据模型、数据库管理、事件同步机制，以及 SSE 实时推送的完整流程。

## 目录

1. [概述](#概述)
2. [系统架构全景](#系统架构全景)
3. [数据库管理机制](#数据库管理机制)
4. [事件系统与 SSE 实时推送](#事件系统与-sse-实时推送)
5. [层级关系与数据模型](#层级关系与数据模型)
6. [Project 创建触发机制](#project-创建触发机制)
7. [Session 层级机制](#session-层级机制)
8. [Message 创建流程](#message-创建流程)
9. [Part 流式生成机制](#part-流式生成机制)
10. [完整示例流程](#完整示例流程)
11. [设计问题分析](#设计问题分析)
12. [参考文件](#参考文件)

---

## 概述

OpenCode 采用**全局数据库 + 事件同步 + SSE 实时推送**的架构：

- **全局数据库**：所有项目数据存储在同一个 SQLite 数据库文件
- **双层事件系统**：SyncEvent（持久化）+ BusEvent（瞬态）
- **层级结构**：Project → Session → Message → Part
- **实时同步**：数据库变更 → 事件发布 → SSE 推送 → 前端接收

---

## 系统架构全景

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户交互层                                       │
│                                                                             │
│   CLI (opencode)     Web UI (packages/app)     Desktop (Tauri)              │
│          │                   │                      │                       │
│          └───────────────────┼──────────────────────│                       │
│                              │                                              │
│                              ▼                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               │ HTTP/SSE
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              服务层                                          │
│                                                                             │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                       │
│   │ REST Routes │   │ SSE Routes  │   │ LLM Stream  │                       │
│   │ (Hono)      │   │ (streamSSE) │   │ (AI SDK)    │                       │
│   └─────────────┘   └─────────────┘   └─────────────┘                       │
│          │                 │                 │                              │
│          └─────────────────┼─────────────────┘                              │
│                            │                                                │
│                            ▼                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              事件同步层                                       │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────┐             │
│   │                    SyncEvent 系统                          │             │
│   │                                                          │             │
│   │  SyncEvent.run(Event, data)                              │             │
│   │       │                                                  │             │
│   │       ├─► Projector(db, data) → INSERT/UPDATE/DELETE     │             │
│   │       ├─► EventTable INSERT (持久化事件)                   │             │
│   │       ├─► Bus.emit("event")                              │             │
│   │       └─► Bus.publish(Event, data) → SSE                  │             │
│   │                                                          │             │
│   └───────────────────────────────────────────────────────────┘             │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────┐             │
│   │                    BusEvent 系统                          │             │
│   │                                                          │             │
│   │  Bus.publish(Event, data)                                │             │
│   │       │                                                  │             │
│   │       ├─► PubSub.publish(typed)                          │             │
│   │       ├─► PubSub.publish(wildcard)                        │             │
│   │       └─► GlobalBus.emit() → 跨实例广播                   │             │
│   │                                                          │             │
│   └───────────────────────────────────────────────────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据存储层                                       │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────┐             │
│   │              SQLite 数据库 (全局唯一)                      │             │
│   │                                                          │             │
│   │  ~/.local/share/opencode/opencode.db                     │             │
│   │                                                          │             │
│   │  ├── project 表                                          │             │
│   │  ├── session 表                                          │             │
│   │  ├── message 表                                          │             │
│   │  ├── part 表                                             │             │
│   │  ├── event 表 (事件日志)                                  │             │
│   │  └── event_sequence 表 (序列号)                          │             │
│   │                                                          │             │
│   └───────────────────────────────────────────────────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 数据流向

```
用户输入 → SessionPrompt → Message + Part 创建
                              │
                              ▼
                    SyncEvent.run(Event, data)
                              │
                              ├─► Projector → 数据库 INSERT
                              │
                              ├─► EventTable 持久化
                              │
                              └─► Bus.publish → PubSub
                                        │
                                        ▼
                              GlobalBus.emit
                                        │
                                        ▼
                              SSE stream.writeSSE
                                        │
                                        ▼
                              前端实时接收
```

---

## 数据库管理机制

### 数据库初始化

**文件**: `src/storage/db.ts`

```typescript
export const Client = lazy(() => {
  log.info("opening database", { path: Path })

  const db = init(Path) // Bun/Node SQLite 初始化

  // SQLite 性能优化配置
  db.run("PRAGMA journal_mode = WAL") // 写前日志模式
  db.run("PRAGMA synchronous = NORMAL") // 同步级别
  db.run("PRAGMA busy_timeout = 5000") // 超时
  db.run("PRAGMA cache_size = -64000") // 缓存大小 (64MB)
  db.run("PRAGMA foreign_keys = ON") // 启用外键约束
  db.run("PRAGMA wal_checkpoint(PASSIVE)") // WAL checkpoint

  // 应用 Drizzle 迁移
  const entries = migrations(path.join(import.meta.dirname, "../../migration"))
  migrate(db, entries)

  return db
})
```

**数据库路径**：

| 平台    | 默认路径                                             |
| ------- | ---------------------------------------------------- |
| Linux   | `~/.local/share/opencode/opencode.db`                |
| macOS   | `~/Library/Application Support/opencode/opencode.db` |
| Windows | `%APPDATA%/opencode/opencode.db`                     |

### 连接管理与事务

**文件**: `src/storage/db.ts`

```typescript
// 事务管理 - 使用 ALS (AsyncLocalStorage) 实现上下文传递
const ctx = Context.create<{
  tx: TxOrDb
  effects: (() => void | Promise<void>)[] // 延迟执行的副作用
}>("database")

// 在事务中执行
export function transaction<T>(
  callback: (tx: TxOrDb) => T,
  options?: { behavior: "deferred" | "immediate" | "exclusive" },
): T {
  // 使用 "immediate" 事务确保读写一致性
  return Client().transaction(callback, { behavior: options?.behavior })
}

// 延迟副作用 - 事务提交后执行
export function effect(fn: () => any) {
  ctx.use().effects.push(InstanceState.bind(fn))
}
```

**关键设计**：

- 延迟副作用：`Bus.publish` 等操作在事务提交后才执行
- 事务嵌套：通过 ALS 实现上下文传递，避免显式传参

### 迁移系统

**两种迁移机制**：

#### 1. Schema 迁移 (Drizzle ORM)

```bash
# 生成迁移
bun run db generate --name <slug>

# 输出
migration/<timestamp>_<slug>/migration.sql
```

**Schema 定义** (`src/**/*.sql.ts`)：

```typescript
// src/session/session.sql.ts
export const SessionTable = sqliteTable("session", {
  id: text().$type<SessionID>().primaryKey(),
  project_id: text()
    .$type<ProjectID>()
    .notNull()
    .references(() => ProjectTable.id, { onDelete: "cascade" }),
  // ... 其他字段
})

// 迁移应用 - 启动时自动执行
migrate(db, entries)
```

#### 2. JSON → SQLite 迁移 (一次性)

**文件**: `src/storage/json-migration.ts`

将旧版 JSON 文件存储迁移到 SQLite：

```typescript
export async function run(db, options?: { progress? }) {
  // 批量导入
  const stats = {
    projects: 0,
    sessions: 0,
    messages: 0,
    // ...
  }

  // 迁移各类数据
  for (const [key, value] of Object.entries(storageData)) {
    // INSERT INTO ...
  }

  return stats
}
```

### 表结构概览

**所有表共享时间戳** (`src/storage/schema.sql.ts`)：

```typescript
export const Timestamps = {
  time_created: integer()
    .notNull()
    .$default(() => Date.now()),
  time_updated: integer()
    .notNull()
    .$onUpdate(() => Date.now()),
}
```

**核心表**：

| 表名             | 说明     | 关键索引                           |
| ---------------- | -------- | ---------------------------------- |
| `project`        | 项目信息 | `id (PK)`                          |
| `session`        | 会话     | `project_id_idx`, `parent_id_idx`  |
| `message`        | 消息     | `session_id_time_created_id_idx`   |
| `part`           | 内容片段 | `message_id_id_idx`, `session_idx` |
| `event`          | 事件日志 | `aggregate_id (FK)`                |
| `event_sequence` | 序列号   | `aggregate_id (PK)`                |

---

## 事件系统与 SSE 实时推送

### 双层事件系统

OpenCode 使用两层事件系统：

#### 1. SyncEvent (持久化事件)

**文件**: `src/sync/index.ts`

用于**事件溯源**，支持会话重放：

```typescript
// 定义事件
const Created = SyncEvent.define({
  type: "session.created",
  version: 1,
  aggregate: "sessionID",  // 聚合根字段
  schema: z.object({
    sessionID: SessionID.zod,
    info: Info,
  }),
})

// 执行事件
SyncEvent.run(Created, { sessionID: id, info: {...} })
```

**执行流程** (`SyncEvent.run`)：

```typescript
export function run<Def extends Definition>(def: Def, data: Event<Def>["data"]) {
  // 立即事务 - 确保读写一致性
  Database.transaction(
    (tx) => {
      // 生成序列号
      const seq = row?.seq != null ? row.seq + 1 : 0
      const event = { id, seq, aggregateID: agg, data }

      // 处理事件
      process(def, event, { publish: true })
    },
    { behavior: "immediate" },
  )
}
```

**process() 内部**：

```typescript
function process(def, event, options) {
  Database.transaction((tx) => {
    // 1. 执行 Projector - 数据库变更
    projector(tx, event.data)

    // 2. 存储事件（可选，特性开关）
    if (Flag.OPENCODE_EXPERIMENTAL_WORKSPACES) {
      tx.insert(EventSequenceTable).values({ aggregate_id, seq }).run()
      tx.insert(EventTable).values({ id, type, data }).run()
    }

    // 3. 延迟副作用 - 事务提交后执行
    Database.effect(() => {
      Bus.emit("event", { def, event })
      if (options.publish) {
        ProjectBus.publish({ type: def.type, properties: def.schema }, data)
      }
    })
  })
}
```

#### 2. BusEvent (瞬态事件)

**文件**: `src/bus/bus-event.ts`, `src/bus/index.ts`

用于**实时通知**，不持久化：

```typescript
// 定义事件
const PartDelta = BusEvent.define(
  "message.part.delta",
  z.object({
    sessionID: SessionID.zod,
    messageID: MessageID.zod,
    partID: PartID.zod,
    field: z.string(),
    delta: z.string(),  // 流式文本增量
  }),
)

// 发布事件
Bus.publish(PartDelta, { ... })
```

**发布流程** (`Bus.publish`)：

```typescript
function publish(def, properties) {
  const s = yield * InstanceState.get(state)
  const payload = { type: def.type, properties }

  // 发布到类型订阅者
  if (s.typed.get(def.type)) {
    yield * PubSub.publish(ps, payload)
  }

  // 发布到通配订阅者
  yield * PubSub.publish(s.wildcard, payload)

  // 跨实例广播
  GlobalBus.emit("event", { directory: dir, payload })
}
```

### SSE 实时推送

#### SSE 端点

**文件**: `src/server/routes/event.ts`, `src/server/routes/global.ts`

| 端点                 | 说明         | 作用域     |
| -------------------- | ------------ | ---------- |
| `/event`             | 项目实例 SSE | 单项目目录 |
| `/global/event`      | 全局事件 SSE | 所有项目   |
| `/global/sync-event` | 同步事件 SSE | 原始事件流 |

#### SSE 实现

```typescript
// src/server/routes/event.ts
;async (c) => {
  return streamSSE(c, async (stream) => {
    const q = new AsyncQueue<string | null>() // 异步队列缓冲

    // 发送连接事件
    q.push(
      JSON.stringify({
        type: "server.connected",
        properties: {},
      }),
    )

    // 心跳 - 每 10 秒
    const heartbeat = setInterval(() => {
      q.push(
        JSON.stringify({
          type: "server.heartbeat",
          properties: {},
        }),
      )
    }, 10_000)

    // 订阅所有事件
    const unsub = Bus.subscribeAll((event) => {
      q.push(JSON.stringify(event))
      if (event.type === Bus.InstanceDisposed.type) {
        stop() // 实例销毁时停止
      }
    })

    // 写入 SSE 流
    for await (const data of q) {
      if (data === null) return
      await stream.writeSSE({ data })
    }
  })
}
```

### 数据库变更 → SSE 完整流程

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. 数据变更请求                                                    │
│    例如: Session.create(), Message.update(), Part.update()         │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│ 2. SyncEvent.run(Event, data)                                      │
│    - 进入立即事务                                                    │
│    - 生成事件 ID 和序列号                                            │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│ 3. Projector 执行                                                   │
│    - INSERT/UPDATE/DELETE 数据库                                    │
│    - 例如: db.insert(SessionTable).values(...).run()               │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│ 4. 事件持久化 (可选)                                                │
│    - EventSequenceTable: { aggregate_id, seq }                     │
│    - EventTable: { id, seq, type, data }                           │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│ 5. 延迟副作用 (事务提交后执行)                                       │
│    Database.effect(() => {                                          │
│      Bus.emit("event", { def, event })                              │
│      Bus.publish(def, data)                                         │
│    })                                                               │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│ 6. Bus.publish                                                      │
│    - PubSub.publish(typed) → 类型订阅者                             │
│    - PubSub.publish(wildcard) → 通配订阅者                          │
│    - GlobalBus.emit() → 跨实例广播                                  │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│ 7. SSE 路由接收                                                     │
│    Bus.subscribeAll((event) => {                                    │
│      q.push(JSON.stringify(event))                                  │
│    })                                                               │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│ 8. 写入 SSE 流                                                      │
│    await stream.writeSSE({ data })                                  │
│    - 前端实时接收事件                                                 │
│    - 更新 UI 状态                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 层级关系与数据模型

### 四层层级结构

```
Project (项目)
    │ 1:N (通过 project_id)
    ▼
Session (会话)
    │ 1:N (通过 session_id)
    ▼
Message (消息)
    │ 1:N (通过 message_id)
    ▼
Part (内容片段)
```

### 数据模型定义

#### Project

```typescript
// src/project/project.ts
export const Info = z.object({
  id: ProjectID.zod, // Git 首个 commit hash
  worktree: z.string(), // Git Worktree 根目录
  vcs: z.literal("git").optional(),
  name: z.string().optional(),
  sandboxes: z.array(z.string()), // 所有工作目录
  time: { created, updated, initialized },
})
```

#### Session

```typescript
// src/session/index.ts
export const Info = z.object({
  id: SessionID.zod,
  projectID: ProjectID.zod, // 关联 Project
  directory: z.string(), // 当前工作目录
  parentID: SessionID.zod.optional(), // 父 Session
  title: z.string(),
  permission: Permission.Ruleset.optional(),
  time: { created, updated, compacting, archived },
})
```

#### Message

```typescript
// src/session/message-v2.ts
export const User = z.object({
  id: MessageID.zod,
  sessionID: SessionID.zod,
  role: z.literal("user"),
  agent: z.string().optional(),
  model: { providerID, modelID }.optional(),
  time: { created },
})

export const Assistant = z.object({
  id: MessageID.zod,
  sessionID: SessionID.zod,
  role: z.literal("assistant"),
  parentID: MessageID.zod.optional(),
  modelID,
  providerID,
  tokens: { input, output, reasoning, cache },
  time: { created, updated, completed },
})
```

#### Part 类型

```typescript
// src/session/message-v2.ts
const PartBase = z.object({
  id: PartID.zod,
  sessionID: SessionID.zod,
  messageID: MessageID.zod,
})

// Part 类型
TextPart = PartBase.extend({ type: "text", text })
ReasoningPart = PartBase.extend({ type: "reasoning", text })
ToolPart = PartBase.extend({ type: "tool", callID, tool, state })
FilePart = PartBase.extend({ type: "file", mime, url })
SnapshotPart = PartBase.extend({ type: "snapshot", snapshot })
```

---

## Project 创建触发机制

### 触发时机

Project 在以下时机自动创建/更新：

| 触发点        | 说明                              |
| ------------- | --------------------------------- |
| 启动 OpenCode | `Instance.provide({ directory })` |
| 切换目录      | 用户切换到新目录工作              |
| Git Worktree  | 打开同一仓库的不同 Worktree       |

### 创建流程

**文件**: `src/project/project.ts`

```typescript
const fromDirectory = Effect.fn("Project.fromDirectory")(function* (directory) {
  // Phase 1: Git 信息检测
  const dotgit = yield* fs.up({ targets: [".git"], start: directory })

  if (!dotgit) {
    // 非 Git 目录 → 全局 Project
    return { id: ProjectID.global, worktree: "/", sandbox: "/" }
  }

  // 解析 Git Worktree
  const commonDir = yield* git(["rev-parse", "--git-common-dir"])
  const worktree = resolveGitPath(sandbox, commonDir)

  // ProjectID = Git 首个 commit hash
  const revList = yield* git(["rev-list", "--max-parents=0", "HEAD"])
  const roots = revList.text.split("\n").filter(Boolean).toSorted()
  id = roots[0] ? ProjectID.make(roots[0]) : undefined

  // 缓存 ProjectID 到 .git/opencode
  yield* fs.writeFileString(path.join(worktree, ".git", "opencode"), id)

  // Phase 2: Upsert 数据库
  yield* db((d) =>
    d.insert(ProjectTable).values(result)
      .onConflictDoUpdate({ target: ProjectTable.id, set: {...} })
      .run()
  )

  return { project: result, sandbox }
})
```

### ProjectID 缓存机制

```
.git/opencode 文件内容示例:

abc123def456...  ← Git 首个 commit hash

作用:
- 避免每次启动都执行 git rev-list
- 同一 Git 仓库的所有 Worktree 共享 ID
```

---

## Session 层级机制

### Session 创建

**文件**: `src/session/index.ts`

```typescript
const create = Effect.fn("Session.create")(function* (input?) {
  const directory = yield* InstanceState.directory

  return yield* createNext({
    parentID: input?.parentID,
    directory,
    title: input?.title ?? createDefaultTitle(),
    permission: input?.permission,
  })
})

const createNext = Effect.fn("Session.createNext")(function* (input) {
  const ctx = yield* InstanceState.context

  const result: Info = {
    id: SessionID.descending(input.id),
    slug: Slug.create(),
    version: Installation.VERSION,
    projectID: ctx.project.id, // 从上下文获取 Project
    directory: input.directory,
    parentID: input.parentID,
    title: input.title,
    permission: input.permission,
    time: { created: Date.now(), updated: Date.now() },
  }

  // 触发事件
  yield* Effect.sync(() => SyncEvent.run(Event.Created, { sessionID: result.id, info: result }))

  return result
})
```

### Session 事件定义

```typescript
// src/session/index.ts
export const Event = {
  Created: SyncEvent.define({
    type: "session.created",
    version: 1,
    aggregate: "sessionID",
    schema: z.object({ sessionID, info }),
  }),

  Updated: SyncEvent.define({
    type: "session.updated",
    version: 1,
    aggregate: "sessionID",
    schema: z.object({ sessionID, info: partialSchema(Info) }),
  }),

  Deleted: SyncEvent.define({
    type: "session.deleted",
    version: 1,
    aggregate: "sessionID",
    schema: z.object({ sessionID, info }),
  }),
}
```

### Session Projector

**文件**: `src/session/projectors.ts`

```typescript
// Projector: 事件 → 数据库变更
SyncEvent.project(Session.Event.Created, (db, data) => {
  db.insert(SessionTable).values(Session.toRow(data.info)).run()
})

SyncEvent.project(Session.Event.Updated, (db, data) => {
  db.update(SessionTable).set(toPartialRow(data.info)).where(eq(SessionTable.id, data.sessionID)).run()
})

SyncEvent.project(Session.Event.Deleted, (db, data) => {
  db.delete(SessionTable).where(eq(SessionTable.id, data.sessionID)).run()
})
```

---

## Message 创建流程

### User Message 创建

**文件**: `src/session/prompt.ts`

```typescript
const createUserMessage = Effect.fn("SessionPrompt.createUserMessage")(function* (input) {
  // 1. 创建 User Message
  const userMsg: MessageV2.User = {
    id: input.messageID ?? MessageID.ascending(),
    sessionID: input.sessionID,
    role: "user",
    agent: input.agent,
    model: { providerID, modelID },
    time: { created: Date.now() },
  }
  yield* sessions.updateMessage(userMsg) // → SyncEvent.run

  // 2. 创建 Parts
  for (const part of input.parts) {
    yield* sessions.updatePart({
      id: PartID.ascending(),
      messageID: userMsg.id,
      sessionID: input.sessionID,
      ...part,
    }) // → SyncEvent.run(PartUpdated)
  }

  return userMsg
})
```

### Assistant Message 创建

**文件**: `src/session/prompt.ts`, `src/session/processor.ts`

```typescript
const runLoop = Effect.fn("SessionPrompt.runLoop")(function* (input) {
  // 1. 创建 Assistant Message Shell
  const assistantMessage: MessageV2.Assistant = {
    id: MessageID.ascending(),
    sessionID,
    parentID: userMsg.id,
    role: "assistant",
    modelID, providerID,
    tokens: { input: 0, output: 0, ... },
    time: { created: Date.now() },
  }
  yield* sessions.updateMessage(assistantMessage)

  // 2. 创建 Processor 处理 LLM Stream
  const handle = yield* processor.create({
    assistantMessage,
    sessionID,
    model,
  })

  // 3. 处理 LLM Stream → 创建 Parts
  const result = yield* handle.process({
    user: systemPrompt,
    agent: agentPrompt,
    messages: history,
    tools: toolDefinitions,
  })

  // 4. 最终更新 Message
  assistantMessage.time.completed = Date.now()
  assistantMessage.tokens = result.tokens
  yield* sessions.updateMessage(assistantMessage)

  return { info: assistantMessage, parts: handle.parts }
})
```

---

## Part 流式生成机制

### Part 事件类型

```typescript
// src/session/message-v2.ts
export const Event = {
  PartUpdated: SyncEvent.define({
    type: "message.part.updated",
    version: 1,
    aggregate: "sessionID",
    schema: z.object({ sessionID, part, time }),
  }),

  PartDelta: BusEvent.define(
    // 瞬态事件
    "message.part.delta",
    z.object({ sessionID, messageID, partID, field, delta }),
  ),
}
```

### LLM Stream → Part 创建

**文件**: `src/session/processor.ts`

```typescript
// 处理 LLM Stream 事件
const handleEvent = (ctx, event) => {
  switch (event.type) {
    case "text-start":
      // 创建 TextPart
      ctx.currentText = {
        id: PartID.ascending(),
        messageID: ctx.assistantMessage.id,
        sessionID: ctx.sessionID,
        type: "text",
        text: "",
        time: { start: Date.now() },
      }
      sessions.updatePart(ctx.currentText)
      break

    case "text-delta":
      // 流式更新 - 使用 PartDelta (BusEvent)
      sessions.updatePartDelta({
        sessionID: ctx.currentText.sessionID,
        messageID: ctx.currentText.messageID,
        partID: ctx.currentText.id,
        field: "text",
        delta: event.value.text,
      })
      // 同时更新 Part 本体
      ctx.currentText.text += event.value.text
      break

    case "text-end":
      // 完成更新
      ctx.currentText.time.end = Date.now()
      sessions.updatePart(ctx.currentText)
      break

    case "reasoning-start":
      // 创建 ReasoningPart
      ctx.reasoningMap[event.id] = {
        id: PartID.ascending(),
        type: "reasoning",
        text: "",
        time: { start: Date.now() },
      }
      sessions.updatePart(ctx.reasoningMap[event.id])
      break

    case "tool-input-start":
      // 创建 ToolPart (pending)
      const toolPart = {
        id: PartID.ascending(),
        type: "tool",
        callID: event.id,
        tool: event.toolName,
        state: { status: "pending", time: { start: Date.now() } },
      }
      sessions.updatePart(toolPart)
      ctx.toolcalls[event.id] = { partID: toolPart.id, ... }
      break

    case "tool-call":
      // ToolPart 状态 → running
      const call = ctx.toolcalls[event.id]
      sessions.updatePart({
        ...call,
        state: { status: "running", input: event.args },
      })
      break

    case "tool-result":
      // ToolPart 状态 → completed
      sessions.updatePart({
        ...call,
        state: {
          status: "completed",
          output: event.result,
          time: { end: Date.now() },
        },
      })
      break
  }
}
```

### Part 流式推送流程

```
LLM Stream Event
      │
      ├─► text-delta
      │       │
      │       ▼
      │   PartDelta (BusEvent)
      │       │
      │       ▼
      │   Bus.publish → SSE
      │       │
      │       ▼
      │   前端实时显示文本
      │
      ├─► text-end / tool-result
      │       │
      │       ▼
      │   PartUpdated (SyncEvent)
      │       │
      │       ▼
      │   Projector → 数据库 UPDATE
      │       │
      │       ▼
      │   Bus.publish → SSE
```

---

## 完整示例流程

### 场景：用户发送一条消息

```
┌─────────────────────────────────────────────────────────────────────┐
│ 用户输入: "分析 src/auth 模块的认证流程"                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. 前端发送请求                                                      │
│    POST /session/:sessionID/prompt                                   │
│    Body: { text: "分析 src/auth 模块的认证流程" }                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. SessionPrompt.prompt(input)                                       │
│                                                                      │
│    a. createUserMessage(input)                                       │
│       - sessions.updateMessage(User)                                 │
│         → SyncEvent.run(Message.Updated)                             │
│         → Projector INSERT message                                   │
│         → Bus.publish → SSE                                          │
│                                                                      │
│       - sessions.updatePart(TextPart)                                │
│         → SyncEvent.run(PartUpdated)                                 │
│         → Projector INSERT part                                      │
│         → Bus.publish → SSE                                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. loop({ sessionID })                                               │
│                                                                      │
│    a. 创建 Assistant Message                                         │
│       - sessions.updateMessage(Assistant)                            │
│         → SyncEvent.run(Message.Updated)                             │
│         → SSE 推送                                                    │
│                                                                      │
│    b. processor.create() → 处理器初始化                               │
│                                                                      │
│    c. handle.process() → LLM Stream                                  │
│       ┌─────────────────────────────────────────┐                    │
│       │ LLM Stream Events:                       │                    │
│       │                                          │                    │
│       │ text-start → 创建 TextPart               │                    │
│       │ text-delta → PartDelta → SSE 实时推送    │                    │
│       │ text-end → PartUpdated → 数据库          │                    │
│       │                                          │                    │
│       │ reasoning-start → 创建 ReasoningPart     │                    │
│       │ reasoning-delta → PartDelta              │                    │
│       │ reasoning-end → PartUpdated              │                    │
│       │                                          │                    │
│       │ tool-call (task) → ToolPart              │                    │
│       │ tool-result → PartUpdated                │                    │
│       └─────────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. SSE 实时推送                                                       │
│                                                                      │
│    前端 SSE 连接:                                                     │
│    GET /event                                                        │
│                                                                      │
│    接收事件流:                                                        │
│    - message.updated → 更新消息列表                                   │
│    - message.part.updated → 更新 Part                                │
│    - message.part.delta → 实时显示文本流                              │
│    - server.heartbeat → 保持连接                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. 前端 UI 更新                                                       │
│                                                                      │
│    - 消息列表显示新消息                                               │
│    - 文本实时流式显示                                                 │
│    - 工具调用状态更新                                                 │
│    - Token 计数更新                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 数据库最终状态

```sql
-- message 表
INSERT INTO message VALUES (
  'msg_001',              -- id
  'sess_001',             -- session_id
  1712834400000,          -- time_created
  '{"role":"user",...}'   -- data
);

INSERT INTO message VALUES (
  'msg_002',              -- id
  'sess_001',             -- session_id
  1712834500000,          -- time_created
  '{"role":"assistant",...}'  -- data
);

-- part 表
INSERT INTO part VALUES (
  'part_001',             -- id
  'msg_001',              -- message_id
  'sess_001',             -- session_id
  '{"type":"text","text":"分析 src/auth..."}'  -- data
);

INSERT INTO part VALUES (
  'part_002',             -- id
  'msg_002',              -- message_id
  'sess_001',             -- session_id
  '{"type":"text","text":"我来分析认证流程..."}'  -- data
);

INSERT INTO part VALUES (
  'part_003',             -- id
  'msg_002',              -- message_id
  'sess_001',             -- session_id
  '{"type":"tool","tool":"task",...}'  -- data
);

-- event 表 (可选)
INSERT INTO event VALUES (
  'evt_001',              -- id
  'sess_001',             -- aggregate_id
  0,                      -- seq
  'session.created.1',    -- type
  '{"sessionID":"sess_001",...}'  -- data
);
```

---

## 设计问题分析

### 问题一：全局数据库导致数据残留

删除项目目录后，数据库数据不会自动清理。

**解决方案**（手动）：

```bash
# 查看数据库路径
opencode db path

# 手动删除废弃 Project
sqlite3 ~/.local/share/opencode/opencode.db
> DELETE FROM project WHERE worktree LIKE '%/已删除目录';
```

### 问题二：缺少 Project 管理命令

建议添加：

```bash
opencode project list
opencode project delete <id>
opencode project clean  # 自动清理废弃项目
```

### 问题三：事件存储可选

`EventTable` 存储受 `OPENCODE_EXPERIMENTAL_WORKSPACES` 特性开关控制。

---

## 参考文件

| 文件                          | 说明                           |
| ----------------------------- | ------------------------------ |
| `src/storage/db.ts`           | 数据库初始化、事务管理         |
| `src/storage/db.bun.ts`       | Bun SQLite 实现                |
| `src/storage/db.node.ts`      | Node SQLite 实现               |
| `src/sync/index.ts`           | SyncEvent 事件溯源系统         |
| `src/sync/event.sql.ts`       | 事件表定义                     |
| `src/bus/index.ts`            | BusEvent pub/sub 服务          |
| `src/bus/global.ts`           | GlobalBus 跨实例广播           |
| `src/server/routes/event.ts`  | SSE 端点实现                   |
| `src/server/routes/global.ts` | 全局 SSE 端点                  |
| `src/session/projectors.ts`   | Session/Message/Part Projector |
| `src/session/index.ts`        | Session 服务                   |
| `src/session/message-v2.ts`   | Message/Part 定义和事件        |
| `src/session/prompt.ts`       | SessionPrompt LLM 交互         |
| `src/session/processor.ts`    | LLM Stream 处理器              |
| `src/project/project.ts`      | Project 服务                   |

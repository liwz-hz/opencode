# OpenCode 调试系统

本文档分析 OpenCode 的调试机制，包括 CLI debug 命令、日志系统、错误处理模式和 Effect 运行时调试。

## 概述

OpenCode 提供了完整的调试工具链：

| 调试层次          | 工具                              | 用途             |
| ----------------- | --------------------------------- | ---------------- |
| **CLI 调试命令**  | `opencode debug <subcmd>`         | 直接诊断各子系统 |
| **日志系统**      | `Log.create()`                    | 结构化日志输出   |
| **错误处理**      | `NamedError` / `TaggedErrorClass` | 类型化错误定义   |
| **Effect 运行时** | `makeRuntime` / `InstanceState`   | Effect 服务调试  |
| **会话调试**      | `session list/read/search`        | 会话历史分析     |

## CLI Debug 命令

### 命令结构

```bash
opencode debug <subcommand>
```

**子命令列表**：

| 子命令     | 功能                                       | 示例                                     |
| ---------- | ------------------------------------------ | ---------------------------------------- |
| `config`   | 显示解析后的完整配置                       | `opencode debug config`                  |
| `paths`    | 显示全局路径（data, cache, config, state） | `opencode debug paths`                   |
| `file`     | 文件系统调试                               | `opencode debug file tree`               |
| `rg`       | ripgrep 调试                               | `opencode debug rg search "pattern"`     |
| `lsp`      | LSP 调试                                   | `opencode debug lsp diagnostics file.ts` |
| `agent`    | Agent 配置调试                             | `opencode debug agent build`             |
| `skill`    | Skill 调试                                 | `opencode debug skill list`              |
| `snapshot` | 快照调试                                   | `opencode debug snapshot list`           |
| `scrap`    | Scrap 调试                                 | `opencode debug scrap list`              |
| `wait`     | 无限等待（用于调试挂起问题）               | `opencode debug wait`                    |

### Debug 命令实现

```typescript
// src/cli/cmd/debug/index.ts
export const DebugCommand = cmd({
  command: "debug",
  describe: "debugging and troubleshooting tools",
  builder: (yargs) =>
    yargs
      .command(ConfigCommand)
      .command(LSPCommand)
      .command(RipgrepCommand)
      .command(FileCommand)
      .command(ScrapCommand)
      .command(SkillCommand)
      .command(SnapshotCommand)
      .command(AgentCommand)
      .command(PathsCommand)
      .command({
        command: "wait",
        describe: "wait indefinitely (for debugging)",
        async handler() {
          await bootstrap(process.cwd(), async () => {
            await new Promise((resolve) => setTimeout(resolve, 1_000 * 60 * 60 * 24))
          })
        },
      })
      .demandCommand(),
  async handler() {},
})
```

### Paths 命令

```typescript
// src/cli/cmd/debug/index.ts (lines 40-48)
const PathsCommand = cmd({
  command: "paths",
  describe: "show global paths (data, config, cache, state)",
  handler() {
    for (const [key, value] of Object.entries(Global.Path)) {
      console.log(key.padEnd(10), value)
    }
  },
})
```

**输出示例**：

```
home       /home/user
data       /home/user/.local/share/opencode
bin        /home/user/.cache/opencode/bin
log        /home/user/.local/share/opencode/log
cache      /home/user/.cache/opencode
config     /home/user/.config/opencode
state      /home/user/.local/state/opencode
```

### Config 命令

```typescript
// src/cli/cmd/debug/config.ts
export const ConfigCommand = cmd({
  command: "config",
  describe: "show resolved configuration",
  async handler() {
    await bootstrap(process.cwd(), async () => {
      const config = await Config.get()
      process.stdout.write(JSON.stringify(config, null, 2) + EOL)
    })
  },
})
```

**用途**：查看配置合并后的最终结果，包括 MCP、Agent、Provider 等所有配置。

### Agent 命令

```typescript
// src/cli/cmd/debug/agent.ts
export const AgentCommand = cmd({
  command: "agent <name>",
  describe: "show agent configuration details",
  builder: (yargs) =>
    yargs
      .positional("name", { type: "string", demandOption: true })
      .option("tool", { type: "string", description: "Tool id to execute" })
      .option("params", { type: "string", description: "Tool params as JSON" }),
  async handler(args) {
    // 显示 agent 配置 + 工具权限状态
    // 或执行指定工具（带 --tool 参数）
  },
})
```

**功能**：

1. 显示 Agent 配置详情
2. 显示工具启用/禁用状态
3. 可直接执行工具测试（`--tool` + `--params`）

**示例**：

```bash
# 查看 build agent 配置
opencode debug agent build

# 测试执行工具
opencode debug agent build --tool read --params '{"filePath": "/src/index.ts"}'
```

### LSP 命令

```typescript
// src/cli/cmd/debug/lsp.ts
export const LSPCommand = cmd({
  command: "lsp",
  describe: "LSP debugging utilities",
  builder: (yargs) =>
    yargs.command(DiagnosticsCommand).command(SymbolsCommand).command(DocumentSymbolsCommand).demandCommand(),
})

// 子命令示例
const DiagnosticsCommand = cmd({
  command: "diagnostics <file>",
  describe: "get diagnostics for a file",
  async handler(args) {
    await bootstrap(process.cwd(), async () => {
      await LSP.touchFile(args.file, true)
      await sleep(1000) // 等待 LSP 初始化
      process.stdout.write(JSON.stringify(await LSP.diagnostics(), null, 2))
    })
  },
})
```

**子命令**：
| 命令 | 功能 |
|------|------|
| `lsp diagnostics <file>` | 获取文件诊断（错误/警告） |
| `lsp symbols <query>` | 工作区符号搜索 |
| `lsp document-symbols <uri>` | 文档符号列表 |

### File 命令

```typescript
// src/cli/cmd/debug/file.ts
export const FileCommand = cmd({
  command: "file",
  describe: "file system debugging utilities",
  builder: (yargs) =>
    yargs
      .command(FileReadCommand)
      .command(FileStatusCommand)
      .command(FileListCommand)
      .command(FileSearchCommand)
      .command(FileTreeCommand)
      .demandCommand(),
})
```

**子命令**：
| 命令 | 功能 |
|------|------|
| `file read <path>` | 读取文件内容（JSON 格式） |
| `file status` | 文件状态信息 |
| `file list <path>` | 目录文件列表 |
| `file search <query>` | 文件搜索 |
| `file tree [dir]` | 目录树结构 |

### Ripgrep 命令

```typescript
// src/cli/cmd/debug/ripgrep.ts
export const RipgrepCommand = cmd({
  command: "rg",
  describe: "ripgrep debugging utilities",
  builder: (yargs) => yargs.command(TreeCommand).command(FilesCommand).command(SearchCommand).demandCommand(),
})
```

**子命令**：
| 命令 | 功能 |
|------|------|
| `rg tree` | 文件树 |
| `rg files [--glob] [--limit]` | 文件列表 |
| `rg search <pattern>` | 内容搜索 |

## 日志系统

### Log 模块架构

```
┌─────────────────────────────────────────────────────────────┐
│                    日志系统架构                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Log.create({ service: "xxx" })                             │
│      │                                                      │
│      ├── 日志级别过滤                                        │
│      │      DEBUG < INFO < WARN < ERROR                     │
│      │                                                      │
│      ├── 输出目标                                            │
│      │      ├─ 开发模式: stderr                              │
│      │      └─ 生产模式: ~/.local/share/opencode/log/        │
│      │                                                      │
│      ├── 日志格式                                            │
│      │      TIMESTAMP +diff service=xxx key=value message   │
│      │                                                      │
│      └── Logger 缓存                                         │
│          同名 service 只创建一次                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Log 实现

```typescript
// src/util/log.ts
export namespace Log {
  export const Level = z.enum(["DEBUG", "INFO", "WARN", "ERROR"])

  const levelPriority: Record<Level, number> = {
    DEBUG: 0,
    INFO: 1,
    WARN: 2,
    ERROR: 3,
  }

  let level: Level = "INFO"

  function shouldLog(input: Level): boolean {
    return levelPriority[input] >= levelPriority[level]
  }

  export interface Options {
    print: boolean // 打印到 stderr 而非文件
    dev?: boolean // 开发模式
    level?: Level // 日志级别
  }

  export async function init(options: Options) {
    if (options.level) level = options.level
    cleanup(Global.Path.log) // 清理旧日志
    if (options.print) return

    logpath = path.join(
      Global.Path.log,
      options.dev ? "dev.log" : new Date().toISOString().split(".")[0].replace(/:/g, "") + ".log",
    )
  }
}
```

### Logger 接口

```typescript
// src/util/log.ts (lines 25-39)
export type Logger = {
  debug(message?: any, extra?: Record<string, any>): void
  info(message?: any, extra?: Record<string, any>): void
  error(message?: any, extra?: Record<string, any>): void
  warn(message?: any, extra?: Record<string, any>): void
  tag(key: string, value: string): Logger
  clone(): Logger
  time(
    message: string,
    extra?: Record<string, any>,
  ): {
    stop(): void
    [Symbol.dispose](): void // 支持 using 语法
  }
}
```

### 创建 Logger

```typescript
// src/util/log.ts (lines 100-181)
export function create(tags?: Record<string, any>) {
  const service = tags?.["service"]

  // 缓存同名 Logger
  if (service && typeof service === "string") {
    const cached = loggers.get(service)
    if (cached) return cached
  }

  function build(message: any, extra?: Record<string, any>) {
    // 格式化日志行
    const prefix = Object.entries({ ...tags, ...extra })
      .map(([key, value]) => `${key}=${formatValue(value)}`)
      .join(" ")
    const next = new Date()
    const diff = next.getTime() - last
    return [next.toISOString().split(".")[0], "+" + diff + "ms", prefix, message]
      .filter(Boolean)
      .join(" ") + "\n"
  }

  const result: Logger = {
    debug(message, extra) {
      if (shouldLog("DEBUG")) write("DEBUG " + build(message, extra))
    },
    info(message, extra) { ... },
    error(message, extra) { ... },
    warn(message, extra) { ... },
    tag(key, value) { ... },
    clone() { ... },
    time(message, extra) { ... },
  }

  // 缓存
  if (service) loggers.set(service, result)
  return result
}
```

### 使用示例

```typescript
// 创建 Logger
const log = Log.create({ service: "ripgrep" })

// 基本日志
log.info("search started", { pattern: "function", cwd: "/src" })
log.debug("found match", { file: "index.ts", line: 42 })
log.error("search failed", { error: err })
log.warn("timeout exceeded", { timeout: 30000 })

// 时间追踪（支持 using）
using timer = log.time("LSP initialization")
// ... 执行操作
// timer.stop() 自动调用，或 using 结束时自动调用

// 添加标签
const tagged = log.tag("session", "abc123")
tagged.info("session event", { type: "message" })

// 克隆 Logger
const cloned = log.clone()
```

### 日志输出示例

```
INFO  2025-03-31T10:00:00 +5ms service=ripgrep pattern=function cwd=/src search started
DEBUG 2025-03-31T10:00:01 +15ms service=ripgrep file=index.ts line=42 found match
ERROR 2025-03-31T10:00:02 +10ms service=ripgrep error="ENOENT: no such file" search failed
INFO  2025-03-31T10:00:05 +500ms service=lsp status=started LSP initialization
INFO  2025-03-31T10:00:10 +5000ms service=lsp status=completed duration=5000 LSP initialization
```

### 日志清理

```typescript
// src/util/log.ts (lines 80-90)
async function cleanup(dir: string) {
  const files = await Glob.scan("????-??-??T??????.log", {
    cwd: dir,
    absolute: true,
    include: "file",
  })
  if (files.length <= 5) return

  // 保留最近 10 个日志文件
  const filesToDelete = files.slice(0, -10)
  await Promise.all(filesToDelete.map((file) => fs.unlink(file).catch(() => {})))
}
```

## 错误处理

### NamedError 模式

OpenCode 使用 `NamedError` 创建类型化的自定义错误：

```typescript
// packages/util/src/error.ts
export abstract class NamedError extends Error {
  abstract schema(): z.core.$ZodType
  abstract toObject(): { name: string; data: any }

  static create<Name extends string, Data extends z.core.$ZodType>(name: Name, data: Data) {
    const schema = z
      .object({
        name: z.literal(name),
        data,
      })
      .meta({ ref: name })

    const result = class extends NamedError {
      public static readonly Schema = schema
      public override readonly name = name as Name

      constructor(
        public readonly data: z.input<Data>,
        options?: ErrorOptions,
      ) {
        super(name, options)
        this.name = name
      }

      static isInstance(input: any): input is InstanceType<typeof result> {
        return typeof input === "object" && "name" in input && input.name === name
      }

      schema() {
        return schema
      }
      toObject() {
        return { name, data: this.data }
      }
    }
    return result
  }
}
```

### NamedError 使用示例

```typescript
// 定义错误
export const ExtractionFailedError = NamedError.create(
  "RipgrepExtractionFailedError",
  z.object({
    filepath: z.string(),
    stderr: z.string(),
  }),
)

export const DownloadFailedError = NamedError.create(
  "RipgrepDownloadFailedError",
  z.object({
    url: z.string(),
    status: z.number(),
  }),
)

// 使用错误
if (response.status !== 200) {
  throw new DownloadFailedError({ url, status: response.status })
}

// 检查错误类型
catch (err) {
  if (DownloadFailedError.isInstance(err)) {
    console.log(`Download failed from ${err.data.url}, status: ${err.data.status}`)
  }
}
```

### 项目中的 NamedError 定义

| 模块         | 错误名                     | 数据字段                              |
| ------------ | -------------------------- | ------------------------------------- |
| **ripgrep**  | `ExtractionFailedError`    | `filepath`, `stderr`                  |
| **ripgrep**  | `UnsupportedPlatformError` | `platform`                            |
| **ripgrep**  | `DownloadFailedError`      | `url`, `status`                       |
| **provider** | `ModelNotFoundError`       | `providerID`, `modelID`               |
| **provider** | `InitError`                | `providerID`                          |
| **worktree** | `NotGitError`              | `directory`                           |
| **worktree** | `CreateFailedError`        | `worktree`                            |
| **lsp**      | `InitializeError`          | `language`, `error`                   |
| **session**  | `OutputLengthError`        | -                                     |
| **session**  | `AuthError`                | `providerID`                          |
| **session**  | `APIError`                 | `providerID`, `statusCode`, `message` |
| **storage**  | `NotFoundError`            | `key`                                 |
| **mcp**      | `Failed`                   | `server`, `error`                     |

### TaggedErrorClass 模式

Effect 生态中使用 `Schema.TaggedErrorClass`：

```typescript
// src/permission/index.ts
export class RejectedError extends Schema.TaggedErrorClass<RejectedError>()("PermissionRejectedError", {}) {
  override get message() {
    return "The user rejected permission to use this specific tool call."
  }
}

export class DeniedError extends Schema.TaggedErrorClass<DeniedError>()("PermissionDeniedError", {
  ruleset: Schema.Any,
}) {
  override get message() {
    return "Permission denied by ruleset"
  }
}
```

**区别**：
| 特性 | NamedError | TaggedErrorClass |
|------|-----------|-----------------|
| **框架** | Zod | Effect Schema |
| **用途** | 通用错误 | Effect 错误处理 |
| **catch** | 手动检查 | `Effect.catch` |
| **序列化** | `toObject()` | 自动 Schema 编码 |

### 错误格式化

```typescript
// src/util/error.ts
export function errorFormat(error: unknown): string {
  if (error instanceof Error) {
    return error.stack ?? `${error.name}: ${error.message}`
  }
  if (typeof error === "object" && error !== null) {
    return JSON.stringify(error, null, 2)
  }
  return String(error)
}

export function errorMessage(error: unknown): string {
  if (error instanceof Error) {
    if (error.message) return error.message
    if (error.name) return error.name
  }
  if (isRecord(error) && typeof error.message === "string") {
    return error.message
  }
  return String(error)
}

export function errorData(error: unknown) {
  // 提取错误结构化数据
  // 返回 { type, message, stack, cause, formatted }
}
```

## Effect 运行时调试

### makeRuntime 模式

```typescript
// src/effect/run-service.ts
import { Effect, Layer, ManagedRuntime } from "effect"
import * as ServiceMap from "effect/ServiceMap"

export const memoMap = Layer.makeMemoMapUnsafe()

export function makeRuntime<I, S, E>(service: ServiceMap.Service<I, S>, layer: Layer.Layer<I, E>) {
  let rt: ManagedRuntime.ManagedRuntime<I, E> | undefined
  const getRuntime = () => (rt ??= ManagedRuntime.make(layer, { memoMap }))

  return {
    runSync: <A, Err>(fn: (svc: S) => Effect.Effect<A, Err, I>) => getRuntime().runSync(service.use(fn)),
    runPromiseExit: <A, Err>(fn: (svc: S) => Effect.Effect<A, Err, I>, options?) =>
      getRuntime().runPromiseExit(service.use(fn), options),
    runPromise: <A, Err>(fn: (svc: S) => Effect.Effect<A, Err, I>, options?) =>
      getRuntime().runPromise(service.use(fn), options),
    runFork: <A, Err>(fn: (svc: S) => Effect.Effect<A, Err, I>) => getRuntime().runFork(service.use(fn)),
    runCallback: <A, Err>(fn: (svc: S) => Effect.Effect<A, Err, I>) => getRuntime().runCallback(service.use(fn)),
  }
}
```

**关键特性**：

- **memoMap**: Layer 共享，避免重复构建
- **懒加载**: Runtime 首次使用时创建
- **多种执行方式**: Sync / Promise / Fork / Callback

### InstanceState 模式

```typescript
// src/effect/instance-state.ts
import { Effect, ScopedCache, Scope } from "effect"

export interface InstanceState<A, E = never, R = never> {
  readonly cache: ScopedCache.ScopedCache<string, A, E, R>
}

export namespace InstanceState {
  export const make = <A, E, R>(
    init: (ctx: InstanceContext) => Effect.Effect<A, E, R | Scope.Scope>,
  ): Effect.Effect<InstanceState<A, E, Exclude<R, Scope.Scope>>, never, R | Scope.Scope> =>
    Effect.gen(function* () {
      const cache = yield* ScopedCache.make<string, A, E, R>({
        capacity: Number.POSITIVE_INFINITY,
        lookup: () => init(Instance.current), // 按目录初始化
      })

      // 注册清理器
      const off = registerDisposer((directory) => Effect.runPromise(ScopedCache.invalidate(cache, directory)))
      yield* Effect.addFinalizer(() => Effect.sync(off))

      return { cache }
    })

  export const get = <A, E, R>(self: InstanceState<A, E, R>) =>
    Effect.suspend(() => ScopedCache.get(self.cache, Instance.directory))

  export const invalidate = <A, E, R>(self: InstanceState<A, E, R>) =>
    Effect.suspend(() => ScopedCache.invalidate(self.cache, Instance.directory))
}
```

**用途**：

- 每个项目目录独立状态
- 自动清理（目录关闭时）
- 避免跨目录污染

### Effect 调试技巧

```typescript
// 1. 添加日志追踪
Effect.gen(function* () {
  const log = yield* Log.create({ service: "session" })
  log.info("starting effect", { step: 1 })
  // ...
})

// 2. 使用 Effect.fn 命名追踪
const processMessage = Effect.fn("Session.processMessage", function* (msg) {
  // Effect 会自动追踪函数名
})

// 3. 错误捕获
Effect.gen(function* () {
  yield* someEffect.pipe(
    Effect.catch("PermissionDeniedError", (err) => Effect.succeed("denied")),
    Effect.catch("APIError", (err) => Effect.fail(err)),
  )
})

// 4. 运行时调试
const runtime = makeRuntime(Service, Layer)
const exit = await runtime.runPromiseExit((svc) => svc.doSomething())
if (Exit.isFailure(exit)) {
  console.log("Error:", Cause.pretty(exit.cause))
}

// 5. Fork 调试
const fiber = runtime.runFork((svc) => svc.backgroundTask())
// fiber 可检查状态、中断等

// 6. 使用 using 确保清理
using scope = yield * Scope.make()
// ... scope 结束时自动清理
```

## Session 调试工具

### session 命令

OpenCode 提供会话调试命令：

```bash
# 列出会话
opencode session list

# 读取会话内容
opencode session read <session-id>

# 搜索会话内容
opencode session search "error"

# 会话详情
opencode session info <session-id>
```

### Session 工具 API

在运行时，可通过工具函数调试会话：

```typescript
// 列出会话
const sessions = await session_list({ limit: 10 })

// 读取会话
const content = await session_read({
  session_id: "ses_abc123",
  include_todos: true,
  include_transcript: true,
})

// 搜索会话
const results = await session_search({
  query: "MCP",
  session_id: "ses_abc123", // 可选，限定范围
})

// 会话信息
const info = await session_info({ session_id: "ses_abc123" })
```

## 调试流程示例

### 诊断 LSP 问题

```bash
# 1. 检查 LSP 状态
opencode debug lsp diagnostics src/index.ts

# 2. 查看符号
opencode debug lsp symbols "function"

# 3. 查看日志
cat ~/.local/share/opencode/log/dev.log | grep "lsp"
```

### 诊断配置问题

```bash
# 1. 查看完整配置
opencode debug config

# 2. 查看路径
opencode debug paths

# 3. 检查 agent 配置
opencode debug agent build
```

### 诊断权限问题

```bash
# 1. 查看 agent 工具权限
opencode debug agent build

# 2. 查看配置中的 permission ruleset
opencode debug config | grep -A 20 "permission"
```

### 诊断 MCP 连接问题

```bash
# 1. 查看 MCP 状态
opencode mcp list

# 2. 调试连接
opencode mcp debug <server-name>

# 3. 查看日志
cat ~/.local/share/opencode/log/*.log | grep "mcp"
```

### 诊断 Session 问题

```bash
# 1. 列出最近会话
opencode session list --limit 5

# 2. 搜索错误
opencode session search "error" --limit 10

# 3. 读取具体会话
opencode session read ses_abc123 --include-todos
```

## 日志文件位置

| 系统        | 日志路径                                      |
| ----------- | --------------------------------------------- |
| **Linux**   | `~/.local/share/opencode/log/`                |
| **macOS**   | `~/Library/Application Support/opencode/log/` |
| **Windows** | `%APPDATA%\opencode\log\`                     |

**日志文件命名**：

- 生产模式: `20250331T100000.log` (时间戳)
- 开发模式: `dev.log`

## 总结

| 调试层次    | 工具                              | 用途           |
| ----------- | --------------------------------- | -------------- |
| **CLI**     | `debug <cmd>`                     | 快速诊断子系统 |
| **日志**    | `Log.create()`                    | 结构化追踪     |
| **错误**    | `NamedError` / `TaggedErrorClass` | 类型化错误     |
| **Effect**  | `makeRuntime` / `InstanceState`   | 运行时调试     |
| **Session** | `session list/read/search`        | 会话分析       |

OpenCode 的调试设计体现了"**可观测性优先**"的理念——每个子系统都有清晰的调试入口，日志结构化便于分析，错误类型化便于定位。

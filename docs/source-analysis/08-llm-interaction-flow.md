# OpenCode LLM 交互流程详解

本文档深入分析 OpenCode 的 LLM 交互核心逻辑，包括背景知识、代码实现、完整流程，彻底讲清楚 `handle.process()` 的调用机制。

## 目录

1. [背景知识](#背景知识)
2. [架构概览](#架构概览)
3. [LLM 服务抽象层](#llm-服务抽象层)
4. [SessionProcessor 核心逻辑](#sessionprocessor-核心逻辑)
5. [SessionPrompt 编排层](#sessionprompt-编排层)
6. [事件类型详解](#事件类型详解)
7. [Part 创建流程](#part-创建流程)
8. [完整调用流程](#完整调用流程)
9. [关键设计细节](#关键设计细节)
10. [参考文件](#参考文件)

---

## 背景知识

### Vercel AI SDK

OpenCode 使用 **Vercel AI SDK** 作为 LLM 交互的基础框架。核心概念：

| 概念           | 说明                                                        |
| -------------- | ----------------------------------------------------------- |
| `streamText`   | 流式文本生成函数，返回包含 `fullStream` 的结果              |
| `fullStream`   | 异步迭代器，按顺序发出所有事件（文本、工具调用、步骤等）    |
| `ModelMessage` | AI SDK 的消息格式，包含 role、content 等字段                |
| `Tool`         | 工具定义，包含 description、parameters schema、execute 函数 |

### 流式事件模式

LLM 流式输出采用**分段事件**模式：

```
streamText() 返回
    │
    ├── fullStream (AsyncIterable)
    │       │
    │       ├── start               → 流开始
    │       │
    │       ├── text-start          → 开始文本段
    │       ├── text-delta (多次)   → 文本增量
    │       ├── text-end            → 文本段结束
    │       │
    │       ├── reasoning-start     → 开始推理段
    │       ├── reasoning-delta     → 推理增量
    │       ├── reasoning-end       → 推理结束
    │       │
    │       ├── tool-input-start    → 工具调用开始
    │       ├── tool-call           → 工具执行（输入确定）
    │       ├── tool-result         → 工具结果返回
    │       │
    │       ├── start-step          → 步骤开始（多轮循环）
    │       ├── finish-step         → 步骤结束
    │       │
    │       ├── error               → 流错误
    │       ├── finish              → 流结束
    │
    └── text, usage, finishReason 等最终结果
```

### Effect 框架集成

OpenCode 使用 **Effect** 框架管理异步流和副作用：

| 概念                       | 说明                           |
| -------------------------- | ------------------------------ |
| `Effect.gen`               | 生成器语法组合 Effect          |
| `Stream.fromAsyncIterable` | 将异步迭代器转为 Effect Stream |
| `Stream.tap`               | 对每个事件执行副作用           |
| `Stream.runDrain`          | 执行流直到完成                 |

---

## 架构概览

### 三层架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    编排层 (SessionPrompt)                                     │
│                                                                             │
│  prompt(input)                                                              │
│      │                                                                      │
│      ├─► createUserMessage() → User Message + Parts                         │
│      │                                                                      │
│      └─► loop() → runLoop()                                                 │
│              │                                                              │
│              ├─► MessageV2.filterCompactedEffect() → 加载历史消息            │
│              ├─► sessions.updateMessage() → Assistant Message Shell         │
│              ├─► buildTools() → 构建工具定义                                  │
│              ├─► processor.create() → 创建处理器                             │
│              │                                                              │
│              └─► handle.process(streamInput)                                │
│                      │                                                      │
│                      ▼                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    处理层 (SessionProcessor)                                  │
│                                                                             │
│  create(input) → Handle                                                     │
│      │                                                                      │
│      ├─► 初始化 ProcessorContext                                            │
│      ├─► 定义 handleEvent() 处理每种事件                                     │
│      │                                                                      │
│      └─► 返回 Handle                                                        │
│              │                                                              │
│              ├─► message: Assistant Message                                 │
│              ├─► updateToolCall()                                           │
│              ├─► completeToolCall()                                         │
│              │                                                              │
│              └─► process(streamInput)                                       │
│                      │                                                      │
│                      ├─► llm.stream(streamInput)                            │
│                      │       │                                              │
│                      │       ▼                                              │
│                      │   fullStream                                          │
│                      │       │                                              │
│                      │       ├─► text-start → TextPart                      │
│                      │       ├─► text-delta → PartDelta                     │
│                      │       ├─► tool-call → ToolPart                       │
│                      │       ├─► finish-step → StepFinishPart               │
│                      │       │                                              │
│                      │       └─► ...                                         │
│                      │                                                      │
│                      ├─► Stream.tap(handleEvent)                            │
│                      ├─► cleanup()                                          │
│                      │                                                      │
│                      └─► 返回 "compact" | "stop" | "continue"               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    抽象层 (LLM Service)                                       │
│                                                                             │
│  stream(input: StreamInput) → Stream.Stream<Event>                          │
│      │                                                                      │
│      ├─► Provider.getLanguage(model) → LanguageModel                       │
│      ├─► Config.get() → 配置                                                │
│      ├─► Plugin.trigger() → 系统提示转换                                     │
│      ├─► resolveTools() → 过滤禁用工具                                       │
│      │                                                                      │
│      └─► streamText({                                                       │
│              model,                                                         │
│              messages,                                                      │
│              tools,                                                         │
│              temperature,                                                   │
│              ...                                                            │
│          })                                                                 │
│              │                                                              │
│              └─► { fullStream, text, usage, ... }                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## LLM 服务抽象层

### 文件位置

`src/session/llm.ts`

### StreamInput 定义

```typescript
// src/session/llm.ts 第 29-42 行
export type StreamInput = {
  user: MessageV2.User // 用户消息信息
  sessionID: string // 会话 ID
  parentSessionID?: string // 父会话 ID（子 Agent）
  model: Provider.Model // 模型配置（providerID, modelID, capabilities）
  agent: Agent.Info // Agent 配置（name, mode, permission, temperature）
  permission?: Permission.Ruleset // 权限规则覆盖
  system: string[] // 系统提示词列表
  messages: ModelMessage[] // 对话历史（AI SDK 格式）
  small?: boolean // 使用小模型优化（用于 compaction）
  tools: Record<string, Tool> // 可用工具（AI SDK Tool 定义）
  retries?: number // 失败重试次数
  toolChoice?: "auto" | "required" | "none" // 工具选择策略
}
```

### Event 类型定义

Event 类型从 AI SDK 的 `fullStream` 推导：

```typescript
// src/session/llm.ts 第 48 行
export type Event = Awaited<ReturnType<typeof stream>>["fullStream"]
  extends AsyncIterable<infer T>
  ? T
  : never
```

**事件类型列表**：

| 类型               | 说明           | 数据字段                                              |
| ------------------ | -------------- | ----------------------------------------------------- |
| `start`            | 流开始         | 无                                                    |
| `text-start`       | 文本段开始     | `id`, `providerMetadata`                              |
| `text-delta`       | 文本增量       | `id`, `text`, `providerMetadata`                      |
| `text-end`         | 文本段结束     | `id`, `providerMetadata`                              |
| `reasoning-start`  | 推理段开始     | `id`, `providerMetadata`                              |
| `reasoning-delta`  | 推理增量       | `id`, `text`, `providerMetadata`                      |
| `reasoning-end`    | 推理段结束     | `id`, `providerMetadata`                              |
| `tool-input-start` | 工具调用初始化 | `id`, `toolName`, `providerExecuted`                  |
| `tool-input-delta` | 工具输入增量   | `id`, `text`                                          |
| `tool-input-end`   | 工具输入结束   | `id`                                                  |
| `tool-call`        | 工具执行开始   | `toolCallId`, `toolName`, `input`, `providerMetadata` |
| `tool-result`      | 工具结果返回   | `toolCallId`, `output`                                |
| `tool-error`       | 工具执行错误   | `toolCallId`, `error`                                 |
| `start-step`       | 步骤开始       | 无                                                    |
| `finish-step`      | 步骤结束       | `finishReason`, `usage`, `providerMetadata`           |
| `error`            | 流错误         | `error`                                               |
| `finish`           | 流结束         | `finishReason`, `usage`                               |

### streamText 核心调用

```typescript
// src/session/llm.ts 第 318-394 行
return streamText({
  // 错误处理
  onError(error) {
    log.error("stream error", { error })
  },

  // 工具调用修复（修复小写工具名）
  async experimental_repairToolCall(failed) {
    const lower = failed.toolCall.toolName.toLowerCase()
    if (lower !== failed.toolCall.toolName && tools[lower]) {
      return { ...failed.toolCall, toolName: lower }
    }
    return { ...failed.toolCall, input: JSON.stringify({ error }), toolName: "invalid" }
  },

  // 生成参数
  temperature: params.temperature,
  topP: params.topP,
  topK: params.topK,
  maxOutputTokens: params.maxOutputTokens,

  // 工具配置
  activeTools: Object.keys(tools).filter((x) => x !== "invalid"),
  tools,
  toolChoice: input.toolChoice,

  // 控制参数
  abortSignal: input.abort,
  maxRetries: input.retries ?? 0,

  // 消息
  messages, // 包含 system prompt + 对话历史

  // 模型（带中间件）
  model: wrapLanguageModel({
    model: language,
    middleware: [
      {
        specificationVersion: "v3",
        async transformParams(args) {
          // 消息转换中间件
          args.params.prompt = ProviderTransform.message(args.params.prompt, input.model, options)
          return args.params
        },
      },
    ],
  }),

  // Headers
  headers: {
    "x-session-affinity": input.sessionID,
    "User-Agent": `opencode/${Installation.VERSION}`,
    ...input.model.headers,
    ...headers,
  },

  // OpenTelemetry
  experimental_telemetry: {
    isEnabled: cfg.experimental?.openTelemetry,
    metadata: { userId, sessionId },
  },
})
```

### Effect 服务封装

```typescript
// src/session/llm.ts 第 56-80 行
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    return Service.of({
      // stream 方法：将 AI SDK 流转为 Effect Stream
      stream(input) {
        return Stream.scoped(
          Stream.unwrap(
            Effect.gen(function* () {
              // 创建 AbortController，自动清理
              const ctrl = yield* Effect.acquireRelease(
                Effect.sync(() => new AbortController()),
                (ctrl) => Effect.sync(() => ctrl.abort()),
              )

              // 调用 streamText
              const result = yield* Effect.promise(() => LLM.stream({ ...input, abort: ctrl.signal }))

              // 将 fullStream 转为 Effect Stream
              return Stream.fromAsyncIterable(result.fullStream, (e) => (e instanceof Error ? e : new Error(String(e))))
            }),
          ),
        )
      },
    })
  }),
)
```

---

## SessionProcessor 核心逻辑

### 文件位置

`src/session/processor.ts`

### Handle 接口定义

```typescript
// src/session/processor.ts 第 32-48 行
export interface Handle {
  // 当前 Assistant Message
  readonly message: MessageV2.Assistant

  // 更新工具调用状态
  readonly updateToolCall: (
    toolCallID: string,
    update: (part: MessageV2.ToolPart) => MessageV2.ToolPart,
  ) => Effect.Effect<MessageV2.ToolPart | undefined>

  // 完成工具调用
  readonly completeToolCall: (
    toolCallID: string,
    output: {
      title: string
      metadata: Record<string, any>
      output: string
      attachments?: MessageV2.FilePart[]
    },
  ) => Effect.Effect<void>

  // 核心：处理 LLM Stream
  readonly process: (streamInput: LLM.StreamInput) => Effect.Effect<Result>
}
```

### 返回结果类型

```typescript
// src/session/processor.ts 第 28 行
export type Result = "compact" | "stop" | "continue"
```

| 结果       | 说明                             |
| ---------- | -------------------------------- |
| `compact`  | Token 超限，需要压缩历史         |
| `stop`     | 停止循环（权限拒绝、错误、完成） |
| `continue` | 继续循环（工具调用后继续）       |

### ProcessorContext 状态

```typescript
// src/session/processor.ts 第 67-75 行
interface ProcessorContext extends Input {
  toolcalls: Record<string, ToolCall> // 工具调用映射
  shouldBreak: boolean // 权限拒绝时是否停止
  snapshot: string | undefined // 当前快照
  blocked: boolean // 是否被阻塞（权限拒绝）
  needsCompaction: boolean // 是否需要压缩
  currentText: MessageV2.TextPart | undefined // 当前文本 Part
  reasoningMap: Record<string, MessageV2.ReasoningPart> // 推理 Part 映射
}
```

### create() 方法

```typescript
// src/session/processor.ts 第 106-122 行
const create = Effect.fn("SessionProcessor.create")(function* (input: Input) {
  // 预捕获快照（AI SDK 可能提前执行工具）
  const initialSnapshot = yield* snapshot.track()

  const ctx: ProcessorContext = {
    assistantMessage: input.assistantMessage,
    sessionID: input.sessionID,
    model: input.model,
    toolcalls: {},
    shouldBreak: false,
    snapshot: initialSnapshot,
    blocked: false,
    needsCompaction: false,
    currentText: undefined,
    reasoningMap: {},
  }

  let aborted = false
  // ... 定义 handleEvent、cleanup、halt 等方法

  return {
    get message() {
      return ctx.assistantMessage
    },
    updateToolCall,
    completeToolCall,
    process,
  } satisfies Handle
})
```

### process() 方法核心

```typescript
// src/session/processor.ts 第 532-581 行
const process = Effect.fn("SessionProcessor.process")(function* (streamInput: LLM.StreamInput) {
  log.info("process")

  // 重置状态
  ctx.needsCompaction = false
  ctx.shouldBreak = (yield* config.get()).experimental?.continue_loop_on_deny !== true

  return yield* Effect.gen(function* () {
    yield* Effect.gen(function* () {
      ctx.currentText = undefined
      ctx.reasoningMap = {}

      // 获取 LLM Stream
      const stream = llm.stream(streamInput)

      // 处理流事件
      yield* stream.pipe(
        Stream.tap((event) => handleEvent(event)), // 处理每个事件
        Stream.takeUntil(() => ctx.needsCompaction), // 压缩时提前结束
        Stream.runDrain, // 执行流
      )
    }).pipe(
      // 中断处理
      Effect.onInterrupt(() =>
        Effect.gen(function* () {
          aborted = true
          if (!ctx.assistantMessage.error) {
            yield* halt(new DOMException("Aborted", "AbortError"))
          }
        }),
      ),

      // 重试策略
      Effect.retry(SessionRetry.policy({ parse, set })),

      // 错误处理
      Effect.catch(halt),

      // 确保清理
      Effect.ensuring(cleanup()),
    )

    // 返回结果
    if (ctx.needsCompaction) return "compact"
    if (ctx.blocked || ctx.assistantMessage.error) return "stop"
    return "continue"
  })
})
```

---

## 事件类型详解

### handleEvent() 方法结构

```typescript
// src/session/processor.ts 第 213-454 行
const handleEvent = Effect.fn("SessionProcessor.handleEvent")(function* (value: StreamEvent) {
  switch (value.type) {
    case "start": { ... }
    case "reasoning-start": { ... }
    case "reasoning-delta": { ... }
    case "reasoning-end": { ... }
    case "tool-input-start": { ... }
    case "tool-call": { ... }
    case "tool-result": { ... }
    case "tool-error": { ... }
    case "start-step": { ... }
    case "finish-step": { ... }
    case "text-start": { ... }
    case "text-delta": { ... }
    case "text-end": { ... }
    case "error": { ... }
    case "finish": { ... }
    default:
      log.info("unhandled", { ...value })
  }
})
```

### 1. start 事件

```typescript
case "start":
  yield* status.set(ctx.sessionID, { type: "busy" })
  return
```

设置会话状态为 "busy"。

### 2. text 事件系列

#### text-start

```typescript
case "text-start":
  ctx.currentText = {
    id: PartID.ascending(),
    messageID: ctx.assistantMessage.id,
    sessionID: ctx.assistantMessage.sessionID,
    type: "text",
    text: "",
    time: { start: Date.now() },
    metadata: value.providerMetadata,
  }
  yield* session.updatePart(ctx.currentText)
  return
```

**作用**：创建空的 TextPart，持久化到数据库。

#### text-delta

```typescript
case "text-delta":
  if (!ctx.currentText) return

  // 本地累积文本
  ctx.currentText.text += value.text
  if (value.providerMetadata) ctx.currentText.metadata = value.providerMetadata

  // 发布增量事件（实时推送）
  yield* session.updatePartDelta({
    sessionID: ctx.currentText.sessionID,
    messageID: ctx.currentText.messageID,
    partID: ctx.currentText.id,
    field: "text",
    delta: value.text,
  })
  return
```

**关键点**：

- 本地维护 `ctx.currentText.text` 状态
- `updatePartDelta` 发布 `BusEvent.PartDelta` → SSE 实时推送
- **不调用 SyncEvent**，不持久化增量

#### text-end

```typescript
case "text-end":
  if (!ctx.currentText) return

  // Trim 末尾空白
  ctx.currentText.text = ctx.currentText.text.trimEnd()

  // 插件钩子
  ctx.currentText.text = (yield* plugin.trigger(
    "experimental.text.complete",
    { sessionID, messageID, partID },
    { text: ctx.currentText.text }
  )).text

  // 设置结束时间
  const end = Date.now()
  ctx.currentText.time = { start: ctx.currentText.time?.start ?? end, end }

  // 持久化最终状态
  yield* session.updatePart(ctx.currentText)

  ctx.currentText = undefined
  return
```

**关键点**：

- 调用 `session.updatePart` → `SyncEvent.run(PartUpdated)` → 持久化
- 清空 `ctx.currentText`

### 3. reasoning 事件系列

#### reasoning-start

```typescript
case "reasoning-start":
  if (value.id in ctx.reasoningMap) return

  ctx.reasoningMap[value.id] = {
    id: PartID.ascending(),
    messageID: ctx.assistantMessage.id,
    sessionID: ctx.assistantMessage.sessionID,
    type: "reasoning",
    text: "",
    time: { start: Date.now() },
    metadata: value.providerMetadata,
  }
  yield* session.updatePart(ctx.reasoningMap[value.id])
  return
```

#### reasoning-delta

```typescript
case "reasoning-delta":
  if (!(value.id in ctx.reasoningMap)) return

  ctx.reasoningMap[value.id].text += value.text
  if (value.providerMetadata) ctx.reasoningMap[value.id].metadata = value.providerMetadata

  yield* session.updatePartDelta({
    sessionID: ctx.reasoningMap[value.id].sessionID,
    messageID: ctx.reasoningMap[value.id].messageID,
    partID: ctx.reasoningMap[value.id].id,
    field: "text",
    delta: value.text,
  })
  return
```

#### reasoning-end

```typescript
case "reasoning-end":
  if (!(value.id in ctx.reasoningMap)) return

  ctx.reasoningMap[value.id].text = ctx.reasoningMap[value.id].text.trimEnd()
  ctx.reasoningMap[value.id].time = { ...ctx.reasoningMap[value.id].time, end: Date.now() }
  if (value.providerMetadata) ctx.reasoningMap[value.id].metadata = value.providerMetadata

  yield* session.updatePart(ctx.reasoningMap[value.id])
  delete ctx.reasoningMap[value.id]
  return
```

### 4. tool 事件系列

#### tool-input-start

```typescript
case "tool-input-start":
  // 禁止在 summary 模式下调用工具
  if (ctx.assistantMessage.summary) {
    throw new Error(`Tool call not allowed while generating summary: ${value.toolName}`)
  }

  const part = yield* session.updatePart({
    id: ctx.toolcalls[value.id]?.partID ?? PartID.ascending(),
    messageID: ctx.assistantMessage.id,
    sessionID: ctx.assistantMessage.sessionID,
    type: "tool",
    tool: value.toolName,
    callID: value.id,
    state: { status: "pending", input: {}, raw: "" },
    metadata: value.providerExecuted ? { providerExecuted: true } : undefined,
  })

  ctx.toolcalls[value.id] = {
    done: yield* Deferred.make<void>(),
    partID: part.id,
    messageID: part.messageID,
    sessionID: part.sessionID,
  }
  return
```

**关键点**：

- 创建 ToolPart，状态为 `pending`
- 存储 `Deferred` 用于工具完成同步

#### tool-call

```typescript
case "tool-call": {
  if (ctx.assistantMessage.summary) {
    throw new Error(`Tool call not allowed while generating summary: ${value.toolName}`)
  }

  yield* updateToolCall(value.toolCallId, (match) => ({
    ...match,
    tool: value.toolName,
    state: {
      ...match.state,
      status: "running",
      input: value.input,
      time: { start: Date.now() },
    },
    metadata: match.metadata?.providerExecuted
      ? { ...value.providerMetadata, providerExecuted: true }
      : value.providerMetadata
  }))

  // Doom Loop 检测（连续相同工具调用）
  const parts = MessageV2.parts(ctx.assistantMessage.id)
  const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)  // 最近 3 个

  if (
    recentParts.length === DOOM_LOOP_THRESHOLD &&
    recentParts.every((part) =>
      part.type === "tool" &&
      part.tool === value.toolName &&
      part.state.status !== "pending" &&
      JSON.stringify(part.state.input) === JSON.stringify(value.input)
    )
  ) {
    // 触发权限询问
    yield* permission.ask({
      permission: "doom_loop",
      patterns: [value.toolName],
      sessionID: ctx.sessionID,
      metadata: { tool: value.toolName, input: value.input },
      always: [value.toolName],
      ruleset: agent.permission,
    })
  }
  return
}
```

**关键设计**：

- **Doom Loop 检测**：连续 3 次相同工具调用触发权限询问
- 防止 LLM 无限循环调用同一工具

#### tool-result

```typescript
case "tool-result": {
  yield* completeToolCall(value.toolCallId, value.output)
  return
}
```

调用 `completeToolCall` 更新状态为 `completed`：

```typescript
const completeToolCall = Effect.fn("SessionProcessor.completeToolCall")(function* (
  toolCallID: string,
  output: { title; metadata; output; attachments },
) {
  const match = yield* readToolCall(toolCallID)
  if (!match || match.part.state.status !== "running") return

  yield* session.updatePart({
    ...match.part,
    state: {
      status: "completed",
      input: match.part.state.input,
      output: output.output,
      metadata: output.metadata,
      title: output.title,
      time: { start: match.part.state.time.start, end: Date.now() },
      attachments: output.attachments,
    },
  })

  yield* settleToolCall(toolCallID) // 完成 Deferred
})
```

#### tool-error

```typescript
case "tool-error": {
  yield* failToolCall(value.toolCallId, value.error)
  return
}
```

调用 `failToolCall` 更新状态为 `error`：

```typescript
const failToolCall = Effect.fn("SessionProcessor.failToolCall")(function* (toolCallID: string, error: unknown) {
  const match = yield* readToolCall(toolCallID)
  if (!match || match.part.state.status !== "running") return false

  yield* session.updatePart({
    ...match.part,
    state: {
      status: "error",
      input: match.part.state.input,
      error: errorMessage(error),
      time: { start: match.part.state.time.start, end: Date.now() },
    },
  })

  // 权限拒绝或问题拒绝时标记 blocked
  if (error instanceof Permission.RejectedError || error instanceof Question.RejectedError) {
    ctx.blocked = ctx.shouldBreak
  }

  yield* settleToolCall(toolCallID)
  return true
})
```

### 5. step 事件系列

#### start-step

```typescript
case "start-step":
  if (!ctx.snapshot) ctx.snapshot = yield* snapshot.track()

  yield* session.updatePart({
    id: PartID.ascending(),
    messageID: ctx.assistantMessage.id,
    sessionID: ctx.sessionID,
    snapshot: ctx.snapshot,
    type: "step-start",
  })
  return
```

**作用**：记录步骤开始时的代码快照。

#### finish-step

```typescript
case "finish-step": {
  // 计算 Token 使用和成本
  const usage = Session.getUsage({
    model: ctx.model,
    usage: value.usage,
    metadata: value.providerMetadata,
  })

  // 更新 Message 统计
  ctx.assistantMessage.finish = value.finishReason
  ctx.assistantMessage.cost += usage.cost
  ctx.assistantMessage.tokens = usage.tokens

  // 创建 StepFinish Part
  yield* session.updatePart({
    id: PartID.ascending(),
    reason: value.finishReason,
    snapshot: yield* snapshot.track(),
    messageID: ctx.assistantMessage.id,
    sessionID: ctx.sessionID,
    type: "step-finish",
    tokens: usage.tokens,
    cost: usage.cost,
  })

  // 更新 Message
  yield* session.updateMessage(ctx.assistantMessage)

  // 生成 Patch（如果有文件变更）
  if (ctx.snapshot) {
    const patch = yield* snapshot.patch(ctx.snapshot)
    if (patch.files.length) {
      yield* session.updatePart({
        id: PartID.ascending(),
        messageID: ctx.assistantMessage.id,
        sessionID: ctx.sessionID,
        type: "patch",
        hash: patch.hash,
        files: patch.files,
      })
    }
    ctx.snapshot = undefined
  }

  // 触发摘要生成
  SessionSummary.summarize({
    sessionID: ctx.sessionID,
    messageID: ctx.assistantMessage.parentID,
  })

  // 检查是否需要压缩
  if (
    !ctx.assistantMessage.summary &&
    isOverflow({ cfg: yield* config.get(), tokens: usage.tokens, model: ctx.model })
  ) {
    ctx.needsCompaction = true
  }
  return
}
```

**关键设计**：

- 记录 Token 使用和成本
- 自动生成代码变更 Patch
- 检测 Token 超限触发压缩

### 6. error 和 finish

```typescript
case "error":
  throw value.error

case "finish":
  return
```

---

## Part 创建流程

### Part 类型映射

| 事件类型                                         | Part 类型      | 状态变化                      |
| ------------------------------------------------ | -------------- | ----------------------------- |
| `text-start` → `text-end`                        | TextPart       | 创建 → 增量 → 完成            |
| `reasoning-start` → `reasoning-end`              | ReasoningPart  | 创建 → 增量 → 完成            |
| `tool-input-start` → `tool-call` → `tool-result` | ToolPart       | pending → running → completed |
| `tool-input-start` → `tool-call` → `tool-error`  | ToolPart       | pending → running → error     |
| `start-step`                                     | StepStartPart  | 包含快照                      |
| `finish-step`                                    | StepFinishPart | 包含 tokens, cost, patch      |

### Part 状态流程图

```
TextPart 流程:
  text-start → 创建 Part (text="")
      │
      │ text-delta (多次)
      │     → updatePartDelta (BusEvent → SSE)
      │     → 本地累积: text += delta
      │
      ▼
  text-end → updatePart (SyncEvent → DB)
      → Part 持久化完成

ToolPart 流程:
  tool-input-start → 创建 Part (status="pending")
      │
      ▼
  tool-call → updatePart (status="running", input 确定)
      │
      ├─► tool-result → updatePart (status="completed", output)
      │
      └─► tool-error → updatePart (status="error", error message)
```

### 数据库持久化时机

| 操作                      | SyncEvent               | 说明           |
| ------------------------- | ----------------------- | -------------- |
| `session.updatePart`      | ✅ PartUpdated          | 持久化到数据库 |
| `session.updatePartDelta` | ❌ PartDelta (BusEvent) | 仅 SSE 推送    |

---

## 完整调用流程

### 用户发送消息的完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. 用户输入                                                                  │
│    POST /session/:sessionID/prompt                                           │
│    Body: { text: "分析 src/auth 模块" }                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. SessionPrompt.prompt(input)                                               │
│                                                                              │
│    a. createUserMessage(input)                                               │
│       - 创建 User Message                                                    │
│       - 创建 TextPart                                                        │
│       - SyncEvent.run(MessageUpdated + PartUpdated)                          │
│                                                                              │
│    b. sessions.touch(sessionID)                                              │
│       - 更新 session.time.updated                                            │
│                                                                              │
│    c. loop({ sessionID })                                                    │
│       - 进入对话循环                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. SessionPrompt.loop() → runLoop()                                          │
│                                                                              │
│    a. MessageV2.filterCompactedEffect(sessionID)                             │
│       - 加载历史消息（排除已压缩）                                             │
│                                                                              │
│    b. sessions.updateMessage(assistantMessage)                               │
│       - 创建 Assistant Message Shell                                         │
│       - SyncEvent.run(MessageUpdated)                                        │
│                                                                              │
│    c. buildTools()                                                           │
│       - 从 ToolRegistry 获取工具定义                                          │
│       - 过滤禁用工具                                                          │
│                                                                              │
│    d. processor.create({ assistantMessage, sessionID, model })               │
│       - 返回 Handle                                                          │
│                                                                              │
│    e. handle.process(streamInput)                                            │
│       - 调用 LLM Stream                                                      │
│       - 处理所有事件                                                          │
│       - 返回 "compact" | "stop" | "continue"                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. SessionProcessor.process(streamInput)                                     │
│                                                                              │
│    a. llm.stream(streamInput)                                                │
│       - 调用 AI SDK streamText                                               │
│       - 返回 Effect Stream                                                    │
│                                                                              │
│    b. Stream.tap(handleEvent)                                                │
│       - 处理每个事件                                                          │
│       - 创建/更新 Parts                                                       │
│                                                                              │
│    c. Stream.runDrain                                                        │
│       - 执行直到流结束                                                        │
│                                                                              │
│    d. cleanup()                                                              │
│       - 完成未完成的 Parts                                                    │
│       - 更新 Message 统计                                                     │
│                                                                              │
│    e. 返回结果                                                                │
│       - "compact": Token 超限                                                │
│       - "stop": 完成/错误                                                     │
│       - "continue": 工具调用后继续                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. 循环控制                                                                   │
│                                                                              │
│    if result === "continue":                                                 │
│       - 继续 runLoop()                                                        │
│       - 重新调用 processor.process()                                         │
│                                                                              │
│    if result === "compact":                                                  │
│       - 触发 SessionCompaction.compact()                                     │
│       - 压缩历史消息                                                          │
│                                                                              │
│    if result === "stop":                                                     │
│       - 循环结束                                                              │
│       - 返回 Assistant Message                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. SSE 实时推送                                                               │
│                                                                              │
│    前端连接: GET /event                                                       │
│                                                                              │
│    接收事件:                                                                  │
│    - message.updated → 更新消息列表                                           │
│    - message.part.updated → 更新 Part                                        │
│    - message.part.delta → 实时显示文本                                        │
│    - session.error → 显示错误                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计细节

### 1. Doom Loop 检测

```typescript
// src/session/processor.ts 第 301-327 行
const DOOM_LOOP_THRESHOLD = 3

// 检测连续相同工具调用
const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)

if (
  recentParts.length === DOOM_LOOP_THRESHOLD &&
  recentParts.every((part) =>
    part.type === "tool" &&
    part.tool === value.toolName &&
    part.state.status !== "pending" &&
    JSON.stringify(part.state.input) === JSON.stringify(value.input)
  )
) {
  yield* permission.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    ...
  })
}
```

**目的**：防止 LLM 无限循环调用同一工具，触发用户确认。

### 2. Token 溢出检测

```typescript
// src/session/processor.ts 第 391-396 行
if (
  !ctx.assistantMessage.summary &&
  isOverflow({ cfg: yield * config.get(), tokens: usage.tokens, model: ctx.model })
) {
  ctx.needsCompaction = true
}
```

**触发**：Token 超限 → 返回 "compact" → 触发历史压缩。

### 3. 重试策略

```typescript
// src/session/processor.ts 第 561-573 行
Effect.retry(
  SessionRetry.policy({
    parse,
    set: (info) =>
      status.set(ctx.sessionID, {
        type: "retry",
        attempt: info.attempt,
        message: info.message,
        next: info.next,
      }),
  }),
)
```

**作用**：LLM 错误时自动重试，显示重试状态。

### 4. 快照追踪

```typescript
// src/session/processor.ts 第 343, 353-386 行
// 步骤开始时捕获快照
if (!ctx.snapshot) ctx.snapshot = yield * snapshot.track()

// 步骤结束时生成 Patch
const patch = yield * snapshot.patch(ctx.snapshot)
if (patch.files.length) {
  yield *
    session.updatePart({
      type: "patch",
      hash: patch.hash,
      files: patch.files,
    })
}
```

**目的**：自动记录 LLM 执行期间的代码变更。

### 5. 中断处理

```typescript
// src/session/processor.ts 第 549-556 行
Effect.onInterrupt(() =>
  Effect.gen(function* () {
    aborted = true
    if (!ctx.assistantMessage.error) {
      yield* halt(new DOMException("Aborted", "AbortError"))
    }
  }),
)
```

**作用**：用户取消时正确清理状态。

---

## 参考文件

| 文件                        | 说明                                          |
| --------------------------- | --------------------------------------------- |
| `src/session/llm.ts`        | LLM 服务抽象、StreamInput、Event 类型         |
| `src/session/processor.ts`  | SessionProcessor 核心逻辑、handleEvent        |
| `src/session/prompt.ts`     | SessionPrompt 编排层、loop、runLoop           |
| `src/session/message-v2.ts` | Message/Part 定义、updatePart/updatePartDelta |
| `src/session/index.ts`      | Session 服务、updateMessage、updatePart       |
| `src/session/retry.ts`      | 重试策略                                      |
| `src/session/overflow.ts`   | Token 溢出检测                                |
| `src/snapshot/index.ts`     | 快照追踪、Patch 生成                          |
| `src/tool/registry.ts`      | 工具注册中心                                  |
| `src/permission/index.ts`   | 权限系统                                      |

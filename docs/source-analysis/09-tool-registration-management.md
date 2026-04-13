# OpenCode 工具注册与管理机制详解

本文档端到端分析 OpenCode 的工具注册与管理机制，从工具定义、注册流程、权限控制到执行流程，彻底讲清楚整个工具系统。

## 目录

1. [概述](#概述)
2. [架构全景](#架构全景)
3. [Tool 定义与接口](#tool-定义与接口)
4. [ToolRegistry 注册中心](#toolregistry-注册中心)
5. [内置工具列表](#内置工具列表)
6. [自定义工具扩展](#自定义工具扩展)
7. [工具执行流程](#工具执行流程)
8. [权限控制集成](#权限控制集成)
9. [完整调用流程](#完整调用流程)
10. [参考文件](#参考文件)

---

## 概述

OpenCode 的工具系统是 LLM 与外部世界交互的桥梁，采用**注册中心 + Effect 服务**的架构：

- **ToolRegistry**：工具注册中心，管理所有内置和自定义工具
- **Tool.Def**：工具定义接口，描述工具参数和执行逻辑
- **Tool.Context**：工具执行上下文，包含会话信息和权限检查
- **动态描述**：部分工具（task、skill）描述根据 Agent 动态生成

---

## 架构全景

### 工具系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    工具注册中心 (ToolRegistry)                                 │
│                                                                             │
│  ToolRegistry.Service                                                       │
│      │                                                                      │
│      ├─► state: InstanceState<State>                                        │
│      │     ├── builtin: Tool.Def[]   ← 内置工具列表                         │
│      │     ├── custom: Tool.Def[]    ← 自定义工具列表                       │
│      │     ├── task: TaskDef         ← task 工具引用                        │
│      │     └── read: ReadDef         ← read 工具引用                        │
│      │                                                                      │
│      ├─► ids(): Effect<string[]>           ← 获取所有工具 ID                │
│      ├─► all(): Effect<Tool.Def[]>         ← 获取所有工具定义               │
│      ├─► named(): Effect<{task, read}>     ← 获取特定工具                   │
│      └─► tools(model, agent): Effect<[]>   ← 按模型/Agent 过滤工具          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               │ 初始化
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    内置工具定义                                               │
│                                                                             │
│  Tool.init() 将 Info 转为 Def                                                │
│      │                                                                      │
│      ├── BashTool     ← bash 命令执行                                       │
│      ├── ReadTool     ← 文件读取                                            │
│      ├── WriteTool    ← 文件写入                                            │
│      ├── EditTool     ← 文件编辑                                            │
│      ├── GlobTool     ← 文件模式匹配                                        │
│      ├── GrepTool     ← 内容搜索                                            │
│      ├── TaskTool     ← 子 Agent 调用                                       │
│      ├── WebFetchTool ← 网页抓取                                            │
│      ├── WebSearchTool← 网页搜索                                            │
│      ├── CodeSearchTool← 代码搜索                                           │
│      ├── TodoWriteTool← Todo 管理                                           │
│      ├── QuestionTool ← 问题询问                                            │
│      ├── SkillTool    ← 技能加载                                            │
│      ├── ApplyPatchTool← Patch 应用                                         │
│      ├── LspTool      ← LSP 工具                                            │
│      └── PlanExitTool ← 规划模式退出                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               │ 自定义扩展
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    自定义工具来源                                             │
│                                                                             │
│  1. 项目目录: .opencode/{tool,tools}/*.ts                                   │
│     → 扫描并 import，命名空间: filename_toolid                               │
│                                                                             │
│  2. Plugin 扩展: plugin.tool = { id: def }                                  │
│     → 从 plugin.list() 获取                                                 │
│                                                                             │
│  自定义工具定义格式:                                                          │
│  export default {                                                           │
│    description: "...",                                                      │
│    args: { param: z.string() },                                             │
│    execute: async (args, ctx) => { ... }                                    │
│  }                                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               │ LLM 调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AI SDK 工具集成                                            │
│                                                                             │
│  SessionPrompt.buildTools()                                                 │
│      │                                                                      │
│      ├─► registry.tools(model, agent)                                       │
│      │     └─► 过滤可用工具                                                  │
│      │                                                                      │
│      ├─► 转换为 AI SDK Tool 格式                                             │
│      │     tool({                                                           │
│      │       description: tool.description,                                 │
│      │       parameters: jsonSchema(tool.parameters),                       │
│      │       execute: async (args, ctx) => {                                │
│      │         // 调用 OpenCode tool.execute(args, toolCtx)                 │
│      │         // Truncate 输出                                             │
│      │         return { output, metadata, title }                           │
│      │       }                                                              │
│      │     })                                                               │
│      │                                                                      │
│      └─► 返回 Record<string, Tool>                                          │
│                                                                             │
│  AI SDK 调用: streamText({ tools, ... })                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               │ 执行
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    工具执行流程                                               │
│                                                                             │
│  LLM tool-call event                                                        │
│      │                                                                      │
│      ├─► Processor.handleEvent("tool-call")                                 │
│      │     └─► 创建 ToolPart (status: running)                              │
│      │                                                                      │
│      ├─► AI SDK 调用 tool.execute(args, ctx)                                │
│      │     │                                                                │
│      │     ├─► ctx.ask() → Permission 检查                                  │
│      │     ├─► 执行工具逻辑                                                  │
│      │     ├─► ctx.metadata() → 实时更新 ToolPart                           │
│      │     │                                                                │
│      │     └─► 返回 { output, metadata, title }                             │
│      │                                                                      │
│      ├─► Processor.completeToolCall(toolCallId, output)                     │
│      │     └─► 更新 ToolPart (status: completed)                            │
│      │                                                                      │
│      └─► 结果返回给 LLM                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Tool 定义与接口

### 文件位置

`src/tool/tool.ts`

### Tool.Def 接口

```typescript
// src/tool/tool.ts 第 29-43 行
export interface Def<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
  id: string // 工具唯一标识
  description: string // 工具描述（给 LLM 看）
  parameters: Parameters // 参数 Schema（Zod）
  execute( // 执行函数
    args: z.infer<Parameters>, // 解析后的参数
    ctx: Context, // 执行上下文
  ): Promise<{
    title: string // 结果标题
    metadata: M // 元数据（显示在 UI）
    output: string // 输出内容（返回给 LLM）
    attachments?: FilePart[] // 可选附件（图片等）
  }>
  formatValidationError?(error: ZodError): string // 自定义参数错误格式化
}
```

### Tool.Context 上下文

```typescript
// src/tool/tool.ts 第 17-27 行
export type Context<M extends Metadata = Metadata> = {
  sessionID: SessionID // 当前会话 ID
  messageID: MessageID // 当前消息 ID
  agent: string // 当前 Agent 名称
  abort: AbortSignal // 中断信号
  callID?: string // 工具调用 ID
  extra?: { [key: string]: any } // 额外参数
  messages: MessageV2.WithParts[] // 消息历史

  // 方法
  metadata(input: { title?: string; metadata?: M }): void // 更新 ToolPart 状态
  ask(input: Permission.Request): Promise<void> // 权限检查
}
```

### Tool.Info 接口

```typescript
// src/tool/tool.ts 第 49-52 行
export interface Info<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
  id: string
  init: () => Promise<DefWithoutID<Parameters, M>> // 初始化函数
}
```

### 工具定义方法

#### Tool.define（同步定义）

```typescript
// src/tool/tool.ts 第 108-116 行
export function define<Parameters extends z.ZodType, Result extends Metadata, ID extends string = string>(
  id: ID,
  init: (() => Promise<DefWithoutID<Parameters, Result>>) | DefWithoutID<Parameters, Result>,
): Info<Parameters, Result> & { id: ID } {
  return {
    id,
    init: wrap(id, init), // wrap 添加参数验证和输出截断
  }
}
```

#### Tool.defineEffect（Effect 定义）

```typescript
// src/tool/tool.ts 第 118-126 行
export function defineEffect<Parameters extends z.ZodType, Result extends Metadata, R, ID extends string = string>(
  id: ID,
  init: Effect.Effect<(() => Promise<DefWithoutID<Parameters, Result>>) | DefWithoutID<Parameters, Result>, never, R>,
): Effect.Effect<Info<Parameters, Result>, never, R> & { id: ID } {
  return Object.assign(
    Effect.map(init, (next) => ({ id, init: wrap(id, next) })),
    { id },
  )
}
```

### wrap 包装函数

```typescript
// src/tool/tool.ts 第 70-106 行
function wrap<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: (() => Promise<DefWithoutID<Parameters, Result>>) | DefWithoutID<Parameters, Result>,
) {
  return async () => {
    const toolInfo = init instanceof Function ? await init() : { ...init }
    const execute = toolInfo.execute

    // 包装 execute 函数
    toolInfo.execute = async (args, ctx) => {
      // 1. 参数验证
      try {
        toolInfo.parameters.parse(args)
      } catch (error) {
        if (error instanceof z.ZodError && toolInfo.formatValidationError) {
          throw new Error(toolInfo.formatValidationError(error), { cause: error })
        }
        throw new Error(`The ${id} tool was called with invalid arguments: ${error}.`, { cause: error })
      }

      // 2. 执行原函数
      const result = await execute(args, ctx)

      // 3. 输出截断（如果未截断）
      if (result.metadata.truncated !== undefined) {
        return result
      }
      const truncated = await Truncate.output(result.output, {}, await Agent.get(ctx.agent))
      return {
        ...result,
        output: truncated.content,
        metadata: {
          ...result.metadata,
          truncated: truncated.truncated,
          ...(truncated.truncated && { outputPath: truncated.outputPath }),
        },
      }
    }
    return toolInfo
  }
}
```

**wrap 函数关键功能**：

1. 参数 Schema 验证
2. 自动输出截断（防止输出过长）
3. 自定义错误格式化

---

## ToolRegistry 注册中心

### 文件位置

`src/tool/registry.ts`

### State 状态定义

```typescript
// src/tool/registry.ts 第 50-55 行
type State = {
  custom: Tool.Def[] // 自定义工具列表
  builtin: Tool.Def[] // 内置工具列表
  task: TaskDef // task 工具引用（用于子 Agent）
  read: ReadDef // read 工具引用（用于特定场景）
}
```

### Interface 接口定义

```typescript
// src/tool/registry.ts 第 57-66 行
export interface Interface {
  // 获取所有工具 ID
  readonly ids: () => Effect.Effect<string[]>

  // 获取所有工具定义
  readonly all: () => Effect.Effect<Tool.Def[]>

  // 获取特定工具（task、read）
  readonly named: () => Effect.Effect<{ task: TaskDef; read: ReadDef }>

  // 按模型/Agent 过滤工具
  readonly tools: (model: { providerID: ProviderID; modelID: ModelID; agent: Agent.Info }) => Effect.Effect<Tool.Def[]>
}
```

### 初始化流程

```typescript
// src/tool/registry.ts 第 96-195 行
const state =
  yield *
  InstanceState.make<State>(
    Effect.fn("ToolRegistry.state")(function* (ctx) {
      const custom: Tool.Def[] = []

      // 1. 加载项目目录工具
      const dirs = yield* config.directories()
      const matches = dirs.flatMap((dir) =>
        Glob.scanSync("{tool,tools}/*.{js,ts}", { cwd: dir, absolute: true, dot: true, symlink: true }),
      )

      if (matches.length) yield* config.waitForDependencies()

      for (const match of matches) {
        const namespace = path.basename(match, path.extname(match))
        const mod = yield* Effect.promise(
          () => import(process.platform === "win32" ? match : pathToFileURL(match).href),
        )
        for (const [id, def] of Object.entries<ToolDefinition>(mod)) {
          custom.push(fromPlugin(id === "default" ? namespace : `${namespace}_${id}`, def))
        }
      }

      // 2. 加载 Plugin 工具
      const plugins = yield* plugin.list()
      for (const p of plugins) {
        for (const [id, def] of Object.entries(p.tool ?? {})) {
          custom.push(fromPlugin(id, def))
        }
      }

      // 3. 初始化内置工具
      const tool = yield* Effect.all({
        invalid: Tool.init(InvalidTool),
        bash: Tool.init(BashTool),
        read: Tool.init(ReadTool),
        glob: Tool.init(GlobTool),
        grep: Tool.init(GrepTool),
        edit: Tool.init(EditTool),
        write: Tool.init(WriteTool),
        task: Tool.init(TaskTool),
        fetch: Tool.init(WebFetchTool),
        todo: Tool.init(TodoWriteTool),
        search: Tool.init(WebSearchTool),
        code: Tool.init(CodeSearchTool),
        skill: Tool.init(SkillTool),
        patch: Tool.init(ApplyPatchTool),
        question: Tool.init(QuestionTool),
        lsp: Tool.init(LspTool),
        plan: Tool.init(PlanExitTool),
      })

      // 4. 组装 State
      return {
        custom,
        builtin: [
          tool.invalid,
          ...(questionEnabled ? [tool.question] : []),
          tool.bash,
          tool.read,
          tool.glob,
          tool.grep,
          tool.edit,
          tool.write,
          tool.task,
          tool.fetch,
          tool.todo,
          tool.search,
          tool.code,
          tool.skill,
          tool.patch,
          ...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [tool.lsp] : []),
          ...(Flag.OPENCODE_EXPERIMENTAL_PLAN_MODE ? [tool.plan] : []),
        ],
        task: tool.task,
        read: tool.read,
      }
    }),
  )
```

### fromPlugin 转换函数

```typescript
// src/tool/registry.ts 第 100-123 行
function fromPlugin(id: string, def: ToolDefinition): Tool.Def {
  return {
    id,
    parameters: z.object(def.args),
    description: def.description,
    execute: async (args, toolCtx) => {
      const pluginCtx: PluginToolContext = {
        ...toolCtx,
        directory: ctx.directory,
        worktree: ctx.worktree,
      }
      const result = await def.execute(args as any, pluginCtx)

      // 输出截断
      const out = await Truncate.output(result, {}, await Agent.get(toolCtx.agent))
      return {
        title: "",
        output: out.truncated ? out.content : result,
        metadata: {
          truncated: out.truncated,
          outputPath: out.truncated ? out.outputPath : undefined,
        },
      }
    },
  }
}
```

### tools() 方法（按条件过滤）

```typescript
// src/tool/registry.ts 第 241-281 行
const tools: Interface["tools"] = Effect.fn("ToolRegistry.tools")(function* (input) {
  const filtered = (yield* all()).filter((tool) => {
    // 过滤 Exa 搜索工具（仅 opencode provider 可用）
    if (tool.id === CodeSearchTool.id || tool.id === WebSearchTool.id) {
      return input.providerID === ProviderID.opencode || Flag.OPENCODE_ENABLE_EXA
    }

    // 过滤 Patch 工具（GPT-4o 使用 patch，其他使用 edit/write）
    const usePatch =
      !!Env.get("OPENCODE_E2E_LLM_URL") ||
      (input.modelID.includes("gpt-") && !input.modelID.includes("oss") && !input.modelID.includes("gpt-4"))
    if (tool.id === ApplyPatchTool.id) return usePatch
    if (tool.id === EditTool.id || tool.id === WriteTool.id) return !usePatch

    return true
  })

  // 为每个工具生成描述
  return yield* Effect.forEach(
    filtered,
    Effect.fnUntraced(function* (tool) {
      const output = {
        description: tool.description,
        parameters: tool.parameters,
      }

      // 插件钩子
      yield* plugin.trigger("tool.definition", { toolID: tool.id }, output)

      return {
        id: tool.id,
        description: [
          output.description,
          // 动态描述：task 工具添加可用 Agent 列表
          tool.id === TaskTool.id ? yield* describeTask(input.agent) : undefined,
          // 动态描述：skill 工具添加可用技能列表
          tool.id === SkillTool.id ? yield* describeSkill(input.agent) : undefined,
        ]
          .filter(Boolean)
          .join("\n"),
        parameters: output.parameters,
        execute: tool.execute,
        formatValidationError: tool.formatValidationError,
      }
    }),
    { concurrency: "unbounded" },
  )
})
```

### 动态描述生成

#### describeTask

```typescript
// src/tool/registry.ts 第 226-239 行
const describeTask = Effect.fn("ToolRegistry.describeTask")(function* (agent: Agent.Info) {
  // 获取所有 subagent
  const items = (yield* agents.list()).filter((item) => item.mode !== "primary")

  // 过滤权限拒绝的 Agent
  const filtered = items.filter((item) => Permission.evaluate("task", item.name, agent.permission).action !== "deny")

  const list = filtered.toSorted((a, b) => a.name.localeCompare(b.name))
  const description = list.map((item) => `- ${item.name}: ${item.description ?? "..."}`).join("\n")

  return ["Available agent types and the tools they have access to:", description].join("\n")
})
```

#### describeSkill

```typescript
// src/tool/registry.ts 第 207-224 行
const describeSkill = Effect.fn("ToolRegistry.describeSkill")(function* (agent: Agent.Info) {
  const list = yield* skill.available(agent)
  if (list.length === 0) return "No skills are currently available."

  return [
    "Load a specialized skill that provides domain-specific instructions and workflows.",
    "",
    "When you recognize that a task matches one of the available skills listed below,",
    "use this tool to load the full skill instructions.",
    "",
    "The following skills provide specialized sets of instructions:",
    Skill.fmt(list, { verbose: false }),
  ].join("\n")
})
```

---

## 内置工具列表

### 工具分类

| 类别         | 工具                          | 说明                      |
| ------------ | ----------------------------- | ------------------------- |
| **文件操作** | read, write, edit, glob, grep | 文件读写、搜索            |
| **命令执行** | bash                          | Shell 命令执行            |
| **网络**     | fetch, search, code           | 网页抓取、搜索、代码搜索  |
| **Agent**    | task                          | 子 Agent 调用             |
| **管理**     | todo, question, skill         | Todo、问题、技能管理      |
| **特殊**     | patch, lsp, plan              | Patch 应用、LSP、规划退出 |
| **占位**     | invalid                       | 无效工具调用占位          |

### 工具详细说明

#### read（文件读取）

```typescript
// src/tool/read.ts
export const ReadTool = Tool.defineEffect(
  "read",
  Effect.gen(function* () {
    const fs = yield* AppFileSystem.Service
    const lsp = yield* LSP.Service
    const time = yield* FileTime.Service

    const parameters = z.object({
      filePath: z.string().describe("The absolute path to the file or directory"),
      offset: z.number().describe("Line number to start reading from (1-indexed)").optional(),
      limit: z.number().describe("Maximum number of lines to read (default 2000)").optional(),
    })

    return {
      description: DESCRIPTION,
      parameters,
      async execute(params, ctx) {
        // 1. 权限检查
        await ctx.ask({
          permission: "read",
          patterns: [filepath],
          always: ["*"],
          metadata: {},
        })

        // 2. 读取文件/目录
        // 3. 处理图片/PDF（返回 attachments）
        // 4. 截断长文件

        return {
          title: relativePath,
          output: contentXml,
          metadata: { preview, truncated, loaded },
          attachments: images ? [{ type: "file", mime, url }] : undefined,
        }
      },
    }
  }),
)
```

**特点**：

- 支持读取文件和目录
- 支持分页（offset、limit）
- 图片/PDF 返回 attachments
- 自动截断（MAX_BYTES = 50KB）

#### bash（命令执行）

```typescript
// src/tool/bash.ts
export const BashTool = Tool.define("bash", async () => {
  const shell = Shell.acceptable()
  const name = Shell.name(shell)

  const parameters = z.object({
    command: z.string().describe("The command to execute"),
    timeout: z.number().describe("Optional timeout in milliseconds").optional(),
    workdir: z.string().describe("The working directory").optional(),
    description: z.string().describe("Clear description of what this command does"),
  })

  return {
    description: DESCRIPTION,
    parameters,
    async execute(params, ctx) {
      // 1. 解析命令（Tree-sitter）
      const root = await parse(params.command, ps)
      const scan = await collect(root, cwd, ps, shell)

      // 2. 权限检查（外部目录、危险命令）
      await ask(ctx, scan)

      // 3. 执行命令
      // 4. 实时输出更新
      // 5. 超时处理

      return {
        title: params.description,
        output: commandOutput,
        metadata: { output: preview, exit: code, description },
      }
    },
  }
})
```

**特点**：

- Tree-sitter 解析命令
- 自动检测外部目录操作
- 实时输出更新（ctx.metadata）
- 超时中断（DEFAULT_TIMEOUT = 2min）
- 跨平台 Shell 支持

#### task（子 Agent 调用）

```typescript
// src/tool/task.ts
export const TaskTool = Tool.defineEffect("task", Effect.gen(function* () {
  const agent = yield* Agent.Service
  const config = yield* Config.Service

  const parameters = z.object({
    description: z.string().describe("A short (3-5 words) description"),
    prompt: z.string().describe("The task for the agent to perform"),
    subagent_type: z.string().describe("The type of specialized agent"),
    task_id: z.string().describe("Resume previous task if set").optional(),
    command: z.string().describe("The command that triggered this task").optional(),
  })

  return {
    description: DESCRIPTION,
    parameters,
    async execute(params, ctx) {
      // 1. 权限检查
      await ctx.ask({
        permission: "task",
        patterns: [params.subagent_type],
        always: ["*"],
        metadata: { description, subagent_type },
      })

      // 2. 获取 Agent
      const next = await Agent.get(params.subagent_type)

      // 3. 创建/恢复 Session
      const session = params.task_id
        ? await Session.get(params.task_id)
        : await Session.create({ parentID: ctx.sessionID, permission: [...] })

      // 4. 调用 SessionPrompt.prompt
      const result = await SessionPrompt.prompt({
        sessionID: session.id,
        agent: next.name,
        parts: [params.prompt],
        tools: { ...(canTodo ? {} : { todowrite: false }) },
      })

      return {
        title: params.description,
        metadata: { sessionId: session.id, model },
        output: [
          `task_id: ${session.id} (for resuming...)`,
          "<task_result>",
          result.text,
          "</task_result>",
        ].join("\n"),
      }
    }
  }
}))
```

**特点**：

- 创建子 Session（parentID 关联）
- 权限继承和限制
- 可恢复（task_id）
- 返回 task_id 供后续恢复

---

## 自定义工具扩展

### 方式一：项目目录工具

**位置**：`.opencode/{tool,tools}/*.ts`

**示例**：

```typescript
// .opencode/tools/custom.ts
import z from "zod"

export default {
  description: "Check code quality with custom rules",
  args: {
    path: z.string().describe("Path to check"),
    rules: z.array(z.string()).describe("Rules to apply").optional(),
  },
  execute: async (args, ctx) => {
    // args: { path: string, rules?: string[] }
    // ctx: ToolContext (sessionID, directory, worktree, ask, metadata)

    // 权限检查（可选）
    await ctx.ask({
      permission: "custom",
      patterns: [args.path],
      always: ["*"],
      metadata: {},
    })

    // 执行逻辑
    const result = await analyzeCode(args.path, args.rules)

    // 实时更新（可选）
    ctx.metadata({
      title: "Analyzing...",
      metadata: { progress: 50 },
    })

    return result // 返回字符串或 { output, title, metadata }
  },
}
```

**命名规则**：

- 文件名：`custom.ts`
- export default：工具 ID 为 `custom`
- export named：工具 ID 为 `custom_${id}`

### 方式二：Plugin 扩展

```typescript
// plugin 定义
const plugin = {
  id: "my-plugin",
  tool: {
    analyze: {
      description: "Analyze code patterns",
      args: {
        pattern: z.string(),
      },
      execute: async (args, ctx) => {
        // ctx: PluginToolContext (directory, worktree, sessionID, ...)
        return `Found ${count} matches for ${args.pattern}`
      },
    },
  },
}
```

### 自定义工具完整示例

```typescript
// .opencode/tools/security-check.ts
import z from "zod"

export const secrets = {
  description: "Check for potential secrets/credentials in files",
  args: {
    path: z.string().describe("Directory or file to scan"),
    severity: z.enum(["high", "medium", "low"]).describe("Minimum severity level").optional(),
  },
  execute: async (args, ctx) => {
    // 权限检查
    await ctx.ask({
      permission: "read",
      patterns: [args.path],
      always: ["*"],
      metadata: { tool: "security-check" },
    })

    ctx.metadata({
      title: "Scanning for secrets...",
      metadata: { scanned: 0 },
    })

    const findings = []
    const files = await scanDirectory(args.path)

    for (const file of files) {
      const content = await readFile(file)
      const matches = checkForSecrets(content, args.severity)
      findings.push(...matches)

      ctx.metadata({
        title: `Scanning ${file}`,
        metadata: { scanned: findings.length },
      })
    }

    const output =
      findings.length === 0
        ? "No secrets found"
        : `Found ${findings.length} potential secrets:\n${findings.map((f) => `- ${f.file}: ${f.type}`).join("\n")}`

    return {
      title: "Security check complete",
      metadata: { findings: findings.length, severity: args.severity },
      output,
    }
  },
}

export default {
  description: "Full security audit",
  args: {
    path: z.string(),
  },
  execute: async (args, ctx) => {
    // 运行所有检查
    const secretsResult = await secrets.execute(args, ctx)
    return secretsResult.output
  },
}

// 注册后的工具 ID：
// - security-check (default)
// - security-check_secrets (named export)
```

---

## 工具执行流程

### AI SDK 工具集成

```typescript
// src/session/prompt.ts (buildTools)
const tools: Record<string, Tool> = {}

const defs =
  yield *
  registry.tools({
    providerID: model.providerID,
    modelID: model.modelID,
    agent: input.agent,
  })

for (const def of defs) {
  tools[def.id] = tool({
    description: def.description,
    inputSchema: jsonSchema(def.parameters),
    execute: async (args, opts) => {
      // 构建 Tool.Context
      const toolCtx: Tool.Context = {
        sessionID: input.sessionID,
        messageID: assistantMessage.id,
        agent: input.agent.name,
        abort: opts.abortSignal,
        callID: opts.toolCallId,
        messages: input.messages,
        metadata(val) {
          // 实时更新 ToolPart
          void Effect.runFork(
            session.updatePart({
              ...part,
              state: { ...part.state, ...val },
            }),
          )
        },
        ask(req) {
          // 权限检查
          return Effect.runPromise(
            permission.ask({
              ...req,
              sessionID: input.sessionID,
              ruleset: Permission.merge(agent.permission, session.permission),
            }),
          )
        },
      }

      // 执行工具
      const result = await def.execute(args, toolCtx)

      // 输出格式
      return {
        title: result.title,
        metadata: result.metadata,
        output: result.output,
        attachments: result.attachments,
      }
    },
  })
}
```

### 执行时序图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. LLM 决定调用工具                                                          │
│    tool-call event: { toolName: "read", input: { filePath: "/src/index.ts" } }│
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. SessionProcessor.handleEvent("tool-call")                                 │
│    - 创建 ToolPart (status: running)                                         │
│    - 更新 UI 显示                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. AI SDK 调用 tool.execute(args, opts)                                      │
│    - 传入 args = { filePath: "/src/index.ts" }                               │
│    - 传入 opts = { abortSignal, toolCallId, messages }                       │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. OpenCode Tool.execute(args, ctx)                                          │
│                                                                              │
│    a. 参数验证                                                                │
│       parameters.parse(args) → 验证 filePath 是否存在                         │
│                                                                              │
│    b. 权限检查                                                                │
│       ctx.ask({                                                              │
│         permission: "read",                                                  │
│         patterns: [filePath],                                                │
│         always: ["*"],                                                       │
│       })                                                                     │
│       → 如果用户拒绝 → 抛出 Permission.RejectedError                          │
│                                                                              │
│    c. 执行核心逻辑                                                            │
│       - 读取文件                                                              │
│       - 处理内容                                                              │
│       - ctx.metadata() 实时更新 UI                                            │
│                                                                              │
│    d. 输出截断                                                                │
│       Truncate.output(result.output) → 截断过长输出                           │
│                                                                              │
│    e. 返回结果                                                                │
│       { title, metadata, output, attachments }                                │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. AI SDK 获取结果                                                           │
│    - 返回给 LLM 作为 tool-result                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. SessionProcessor.handleEvent("tool-result")                               │
│    - 更新 ToolPart (status: completed)                                       │
│    - completeToolCall(toolCallId, output)                                    │
│    - SSE 推送更新                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 7. LLM 继续处理                                                              │
│    - 看到工具结果                                                              │
│    - 继续推理或生成回复                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 权限控制集成

### ctx.ask() 方法

```typescript
// 在 Tool.Context 中定义
ask(input: Omit<Permission.Request, "id" | "sessionID" | "tool">): Promise<void>
```

**调用示例**：

```typescript
// read 工具
await ctx.ask({
  permission: "read",
  patterns: [filepath],
  always: ["*"], // 用户选择"总是允许"时的 pattern
  metadata: {},
})

// bash 工具（外部目录）
await ctx.ask({
  permission: "external_directory",
  patterns: [dir + "/*"],
  always: [dir + "/*"],
  metadata: {},
})

// bash 工具（命令）
await ctx.ask({
  permission: "bash",
  patterns: ["rm -rf *"],
  always: ["rm -rf /path/*"],
  metadata: {},
})

// task 工具
await ctx.ask({
  permission: "task",
  patterns: ["explore"],
  always: ["*"],
  metadata: { description, subagent_type },
})
```

### 权限检查流程

```
ctx.ask(request)
      │
      ▼
Permission.ask(request)
      │
      ├─► Permission.evaluate(permission, pattern, ruleset)
      │     │
      │     ├─► action = "allow" → 直接返回，不询问
      │     │
      │     ├─► action = "deny" → 抛出 Permission.RejectedError
      │     │
      │     └─► action = "ask" → 发布权限请求事件
      │             │
      │             ▼
      │         Bus.publish(Permission.Event.Requested)
      │             │
      │             ▼
      │         前端 UI 显示权限对话框
      │             │
      │             ├─► 用户点击"Allow" → Bus.publish(Permission.Event.Replied, { reply: "allow" })
      │             │
      │             ├─► 用户点击"Deny" → Bus.publish(Permission.Event.Replied, { reply: "deny" })
      │             │
      │             └─► 用户点击"Always Allow" →
      │                     Permission.store({ patterns, action: "allow" })
      │                     Bus.publish(Permission.Event.Replied, { reply: "allow" })
      │
      ▼
返回 void（或抛出 RejectedError）
```

### 权限拒绝处理

```typescript
// SessionProcessor.failToolCall
case "tool-error": {
  yield* failToolCall(value.toolCallId, value.error)
}

const failToolCall = Effect.fn("SessionProcessor.failToolCall")(function* (toolCallID, error) {
  // 更新 ToolPart 状态为 error
  yield* session.updatePart({
    ...part,
    state: {
      status: "error",
      error: errorMessage(error),
    }
  })

  // 如果是权限拒绝，标记 blocked
  if (error instanceof Permission.RejectedError || error instanceof Question.RejectedError) {
    ctx.blocked = ctx.shouldBreak
  }
})
```

---

## 完整调用流程

### 从 LLM tool-call 到结果返回的完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. 用户请求                                                                  │
│    "读取 src/index.ts 文件"                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. LLM 推理                                                                  │
│    - 决定调用 read 工具                                                       │
│    - 生成 tool-call event                                                    │
│    { type: "tool-call", toolName: "read", input: { filePath: "..." } }       │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. SessionProcessor.handleEvent("tool-call")                                 │
│    - 更新 ToolPart (status: running)                                         │
│    - SSE 推送 → 前端显示"正在读取..."                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. AI SDK 调用 tool.execute                                                  │
│    - tool 是在 buildTools() 中创建的 AI SDK Tool                             │
│    - execute 函数内部：                                                       │
│      a. 构建 Tool.Context                                                    │
│      b. 调用 ToolRegistry 中注册的 def.execute(args, ctx)                     │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. ReadTool.execute(args, ctx)                                               │
│                                                                              │
│    a. 参数验证                                                                │
│       parameters.parse({ filePath: "src/index.ts" })                         │
│                                                                              │
│    b. 权限检查                                                                │
│       ctx.ask({ permission: "read", patterns: [filePath] })                  │
│       → 如果用户未配置"总是允许"，前端弹出对话框                                │
│       → 用户选择后继续                                                        │
│                                                                              │
│    c. 执行读取                                                                │
│       - fs.stat(filePath)                                                    │
│       - lines(filePath, { limit: 2000 })                                     │
│                                                                              │
│    d. 实时更新                                                                │
│       ctx.metadata({ title: "读取中...", metadata: {} })                      │
│       → 更新 ToolPart                                                        │
│       → SSE 推送                                                              │
│                                                                              │
│    e. 输出截断                                                                │
│       Truncate.output(content)                                               │
│                                                                              │
│    f. 返回结果                                                                │
│       { title, output: content, metadata: { preview, truncated } }            │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. AI SDK tool.execute 返回                                                  │
│    - 返回 { title, metadata, output }                                        │
│    - AI SDK 将其转为 tool-result                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 7. SessionProcessor.handleEvent("tool-result")                               │
│    - completeToolCall(toolCallId, output)                                    │
│    - 更新 ToolPart (status: completed, output: "...")                        │
│    - SSE 推送 → 前端显示完整结果                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 8. LLM 继续推理                                                              │
│    - 看到 tool-result                                                        │
│    - 分析文件内容                                                              │
│    - 生成回复或继续调用工具                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 9. 最终回复                                                                  │
│    - text-start → text-delta → text-end                                      │
│    - "我已经读取了文件，内容如下..."                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 参考文件

| 文件                      | 说明                                      |
| ------------------------- | ----------------------------------------- |
| `src/tool/tool.ts`        | Tool 接口定义、define、defineEffect、wrap |
| `src/tool/registry.ts`    | ToolRegistry 注册中心、State、tools()     |
| `src/tool/read.ts`        | ReadTool 实现（文件读取）                 |
| `src/tool/bash.ts`        | BashTool 实现（命令执行）                 |
| `src/tool/task.ts`        | TaskTool 实现（子 Agent）                 |
| `src/tool/write.ts`       | WriteTool 实现（文件写入）                |
| `src/tool/edit.ts`        | EditTool 实现（文件编辑）                 |
| `src/tool/glob.ts`        | GlobTool 实现（文件匹配）                 |
| `src/tool/grep.ts`        | GrepTool 实现（内容搜索）                 |
| `src/tool/webfetch.ts`    | WebFetchTool 实现（网页抓取）             |
| `src/tool/websearch.ts`   | WebSearchTool 实现（网页搜索）            |
| `src/tool/codesearch.ts`  | CodeSearchTool 实现（代码搜索）           |
| `src/tool/todo.ts`        | TodoWriteTool 实现（Todo 管理）           |
| `src/tool/question.ts`    | QuestionTool 实现（问题询问）             |
| `src/tool/skill.ts`       | SkillTool 实现（技能加载）                |
| `src/tool/apply_patch.ts` | ApplyPatchTool 实现（Patch 应用）         |
| `src/tool/lsp.ts`         | LspTool 实现（LSP 工具）                  |
| `src/tool/plan.ts`        | PlanExitTool 实现（规划退出）             |
| `src/tool/truncate.ts`    | Truncate 输出截断                         |
| `src/session/prompt.ts`   | buildTools 构建 AI SDK 工具               |
| `src/permission/index.ts` | Permission 权限系统                       |

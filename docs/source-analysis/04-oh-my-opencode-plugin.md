# oh-my-opencode 插件核心机制分析

本文档深入分析 oh-my-opencode 插件的核心工作原理，解释其与 OpenCode 核心的关系和机制重叠情况。

## 一、插件定位与架构

### 1.1 插件定位

oh-my-opencode 是 OpenCode 的官方增强插件，其定位类似于 oh-my-zsh之于 zsh——提供更强大的智能体编排能力和自动化工作流。

```
OpenCode 核心          ←→    oh-my-opencode 插件
     │                              │
     ├─ 基础 Agent 系统             ├─ 高级编排 Agent（Sisyphus、Atlas）
     ├─ 基会话管理                 ├─ Todo Continuation（自动继续）
     ├─ 基础工具集                  ├─ 31 个生命周期钩子扩展
     ├─ MCP 客户端                  ├─ 内置 MCP 服务器（websearch、context7）
     └─ Provider 封装               ├─ 技能系统（Skills）
                                    └─ 后台智能体管理（BackgroundManager）
```

### 1.2 插件加载机制

oh-my-opencode 通过 OpenCode 的 `plugin` 配置字段加载：

```json
// opencode.json
{
  "plugin": {
    "path": "/path/to/oh-my-opencode",
    "config": {
      "disabled_hooks": ["keyword-detector"],
      "sisyphus_agent": {
        "disabled": false
      }
    }
  }
}
```

插件入口文件 `src/index.ts` 导出一个符合 `@opencode-ai/plugin` 接口的函数：

```typescript
// src/index.ts
const OhMyOpenCodePlugin: Plugin = async (ctx) => {
  // ctx 包含：
  // - directory: 工作目录路径
  // - client: OpenCode API 客户端
  // - session: 会话信息

  return {
    tool: { ... },           // 注册新工具
    "chat.message": async (input, output) => { ... },  // 钩子函数
    "event": async (input) => { ... },                 // 事件处理
    "tool.execute.before": async (input, output) => { ... },
    "tool.execute.after": async (input, output) => { ... },
    config: async (input, output) => { ... },          // 配置处理
  }
}
```

### 1.3 与 OpenCode 核心的关系

| 功能领域   | OpenCode 核心                    | oh-my-opencode                      | 关系                     |
| ---------- | -------------------------------- | ----------------------------------- | ------------------------ |
| Agent 系统 | build, plan, general, compaction | Sisyphus, Atlas, Prometheus, Oracle | **扩展**：高级编排智能体 |
| 会话控制   | 基础 prompt/response 循环        | Ralph Loop, Todo Continuation       | **增强**：自动继续机制   |
| 工具集     | read, write, edit, bash, grep 等 | delegate_task, skill, skill_mcp     | **扩展**：新工具类型     |
| MCP 系统   | 客户端实现                       | websearch, context7, grep_app       | **扩展**：内置服务器     |
| Hook 系统  | 无（核心无钩子）                 | 31 个生命周期钩子                   | **新增**：完全由插件提供 |

**关键结论**：oh-my-opencode 是 OpenCode 的**纯扩展层**，不覆盖核心功能，而是通过钩子和新工具增强其能力。

---

## 二、Ralph Loop 机制详解

### 2.1 核心概念

Ralph Loop 是一种**自指性开发循环**（Self-Referential Development Loop），其核心思想是：

> AI 智能体应该持续工作直到任务完全完成，而不是等待用户手动输入 "continue"。

这解决了 AI 智能体常见的问题：模型倾向于在部分完成时就停止，声称任务已完成但实际上还有很多遗漏。

### 2.2 工作流程

```
用户启动 /ralph-loop "任务描述"
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  1. 创建状态文件 .sisyphus/ralph-loop.local.md         │
│     - active: true                                      │
│     - iteration: 1                                      │
│     - max_iterations: 100                               │
│     - completion_promise: "DONE"                        │
│     - prompt: "任务描述"                                 │
│     - started_at: 时间戳                                │
│     - session_id: 当前会话 ID                           │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  2. 智能体开始执行任务                                  │
│     - 正常使用所有工具                                   │
│     - 创建 Todo 列表跟踪进度                             │
│     - 逐步完成各个子任务                                 │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  3. 智能体输出消息                                      │
│     - 如果完成，输出 <promise>DONE</promise>            │
│     - 如果未完成，不输出 promise 标签                    │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  4. 触发 session.idle 事件                              │
│     - Ralph Loop Hook 检测到会话空闲                    │
│     - 检查是否已输出 completion promise                 │
└─────────────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
已完成      未完成
    │         │
    ▼         ▼
清理状态   继续循环
    │         │
    ▼         ▼
结束       ┌─────────────────────────────────────────────┐
          │  5. 注入 Continuation Prompt                 │
          │     - 自动注入新的消息                       │
          │     - 内容包含进度信息                       │
          │     - iteration 计数器递增                   │
          │     - 检查是否达到 max_iterations            │
          └─────────────────────────────────────────────┘
                   │
                   ▼
              回到步骤 2
```

### 2.3 核心实现代码分析

**状态存储**（`storage.ts`）：

```typescript
// 状态文件使用 YAML frontmatter 格式
// 文件路径：.sisyphus/ralph-loop.local.md

export function writeState(directory: string, state: RalphLoopState): boolean {
  const content = `---
active: ${state.active}
iteration: ${state.iteration}
max_iterations: ${state.max_iterations}
completion_promise: "${state.completion_promise}"
started_at: "${state.started_at}"
session_id: "${state.session_id}"
---
${state.prompt}
`
  writeFileSync(filePath, content, "utf-8")
}

export function readState(directory: string): RalphLoopState | null {
  const content = readFileSync(filePath, "utf-8")
  const { data, body } = parseFrontmatter(content)
  return {
    active: data.active === true,
    iteration: Number(data.iteration),
    max_iterations: Number(data.max_iterations) || 100,
    completion_promise: data.completion_promise || "DONE",
    prompt: body.trim(),
    // ...
  }
}
```

**完成检测**（`index.ts`）：

```typescript
// 检测两种来源的完成标签

// 1. 通过 transcript 文件检测（本地缓存）
function detectCompletionPromise(transcriptPath: string, promise: string): boolean {
  const content = readFileSync(transcriptPath, "utf-8")
  const pattern = new RegExp(`<promise>\\s*${escapeRegex(promise)}\\s*</promise>`, "is")
  // 遍历 transcript 中的每行 JSON
  for (const line of lines) {
    const entry = JSON.parse(line)
    if (entry.type === "user") continue // 只检查 assistant 消息
    if (pattern.test(line)) return true
  }
  return false
}

// 2. 通过 Session Messages API 检测（远程）
async function detectCompletionInSessionMessages(sessionID: string, promise: string): Promise<boolean> {
  const response = await ctx.client.session.messages({ path: { id: sessionID } })
  const messages = response.data ?? []
  const assistantMessages = messages.filter((msg) => msg.info?.role === "assistant")
  const lastAssistant = assistantMessages[assistantMessages.length - 1]
  const responseText = lastAssistant.parts
    .filter((p) => p.type === "text")
    .map((p) => p.text ?? "")
    .join("\n")
  const pattern = new RegExp(`<promise>\\s*${escapeRegex(promise)}\\s*</promise>`, "is")
  return pattern.test(responseText)
}
```

**事件处理**（`index.ts`）：

```typescript
const event = async ({ event }): Promise<void> => {
  if (event.type === "session.idle") {
    const sessionID = props?.sessionID

    // 读取当前循环状态
    const state = readState(ctx.directory)

    // 检测完成标签
    const completionDetected = detectCompletionPromise(...) ||
                               await detectCompletionInSessionMessages(...)

    if (completionDetected) {
      // 任务完成，清理状态
      clearState(ctx.directory)
      await ctx.client.tui.showToast({ body: { title: "Ralph Loop Complete!" } })
      return
    }

    // 检查是否达到最大迭代
    if (state.iteration >= state.max_iterations) {
      clearState(ctx.directory)
      await ctx.client.tui.showToast({ body: { title: "Ralph Loop Stopped" } })
      return
    }

    // 递增迭代计数
    const newState = incrementIteration(ctx.directory)

    // 构造继续提示
    const continuationPrompt = CONTINUATION_PROMPT
      .replace("{{ITERATION}}", newState.iteration)
      .replace("{{MAX}}", newState.max_iterations)
      .replace("{{PROMISE}}", newState.completion_promise)
      .replace("{{PROMPT}}", newState.prompt)

    // 自动注入新消息
    await ctx.client.session.prompt({
      path: { id: sessionID },
      body: {
        agent: lastAgentName,           // 保持原智能体
        model: lastModel,               // 保持原模型
        parts: [{ type: "text", text: continuationPrompt }],
      },
    })
  }
}
```

### 2.4 启动方式

**方式一：Slash Command**

```
/ralph-loop "实现用户认证功能" --max-iterations=50 --completion-promise=AUTH_DONE
```

**方式二：Chat Message**

直接发送包含 Ralph Loop 模板的消息，Hook 会自动识别并启动循环。

### 2.5 Ultrawork 模式

Ultrawork 是 Ralph Loop 的一种增强模式：

```typescript
// /ulw-loop 命令
ralphLoop.startLoop(sessionID, prompt, {
  ultrawork: true,                    // 启用 ultrawork 模式
  maxIterations: ...,
  completionPromise: ...,
})

// 继续提示会加上 ultrawork 前缀
const finalPrompt = newState.ultrawork
  ? `ultrawork ${continuationPrompt}`
  : continuationPrompt
```

Ultrawork 模式会触发 Sisyphus 智能体的**超专注工作状态**，减少交互，提高执行效率。

---

## 三、Todo Continuation Enforcer 机制

### 3.1 核心概念

Todo Continuation Enforcer 是另一个自动继续机制，但它基于 **Todo 列表状态**而不是 completion promise 标签。

> 如果 Todo 列表中仍有未完成的任务，自动注入继续提示。

### 3.2 与 Ralph Loop 的区别

| 特性     | Ralph Loop                     | Todo Continuation Enforcer |
| -------- | ------------------------------ | -------------------------- |
| 检测方式 | `<promise>DONE</promise>` 标签 | Todo 列表未完成数量        |
| 触发条件 | 用户显式启动 `/ralph-loop`     | 任何会话空闲时自动检查     |
| 控制粒度 | 整个任务级别                   | 每个子任务级别             |
| 最大迭代 | 100（可配置）                  | 无限制（直到 todo 完成）   |
| 适用场景 | 大型复杂任务                   | 日常开发工作               |

### 3.3 工作流程

```
会话空闲（session.idle）
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  1. 检查是否为主会话                                    │
│     - 跳过子会话和后台任务会话                           │
│     - 跳过 recovery 状态                                │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  2. 检测最近消息是否被中止                              │
│     - MessageAbortedError → 跳过                        │
│     - AbortError → 跳过                                 │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  3. 获取 Todo 列表                                      │
│     - ctx.client.session.todo({ path: { id: sessionID } })│
│     - 统计未完成数量                                     │
└─────────────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
无未完成   有未完成
    │         │
    ▼         ▼
跳过      启动倒计时
    │         │
    ▼         ▼
结束       ┌─────────────────────────────────────────────┐
          │  4. 2秒倒计时                                │
          │     - 显示 Toast 提示                        │
          │     - 用户可输入取消                         │
          │     - 工具执行会取消                         │
          └─────────────────────────────────────────────┘
                   │
                   ▼
              ┌─────────────────────────────────────────────┐
              │  5. 注入 Continuation Prompt                │
              │     - 保持原 agent 和 model                 │
              │     - 包含进度状态                          │
              │     - "[Status: 3/10 completed, 7 remaining]"│
              └─────────────────────────────────────────────┘
                   │
                   ▼
              智能体继续工作
```

### 3.4 核心实现代码分析

**倒计时机制**：

```typescript
const COUNTDOWN_SECONDS = 2
const TOAST_DURATION_MS = 900

function startCountdown(sessionID: string, incompleteCount: number, total: number): void {
  const state = getState(sessionID)
  cancelCountdown(sessionID) // 清除之前的倒计时

  let secondsRemaining = COUNTDOWN_SECONDS
  showCountdownToast(secondsRemaining, incompleteCount)
  state.countdownStartedAt = Date.now()

  // 每秒显示倒计时 Toast
  state.countdownInterval = setInterval(() => {
    secondsRemaining--
    if (secondsRemaining > 0) {
      showCountdownToast(secondsRemaining, incompleteCount)
    }
  }, 1000)

  // 倒计时结束后注入继续提示
  state.countdownTimer = setTimeout(() => {
    cancelCountdown(sessionID)
    injectContinuation(sessionID, incompleteCount, total)
  }, COUNTDOWN_SECONDS * 1000)
}
```

**中止检测**：

```typescript
// 方式一：事件检测（主要）
if (event.type === "session.error") {
  const error = props?.error
  if (error?.name === "MessageAbortedError" || error?.name === "AbortError") {
    state.abortDetectedAt = Date.now()
    cancelCountdown(sessionID)
    return
  }
}

// 方式二：API 检测（备用）
function isLastAssistantMessageAborted(messages): boolean {
  const assistantMessages = messages.filter((m) => m.info?.role === "assistant")
  const lastAssistant = assistantMessages[assistantMessages.length - 1]
  const errorName = lastAssistant.info?.error?.name
  return errorName === "MessageAbortedError" || errorName === "AbortError"
}

// 在 session.idle 时检查
const messagesResp = await ctx.client.session.messages({ path: { id: sessionID } })
if (isLastAssistantMessageAborted(messagesResp.data)) {
  return // 用户手动中止，不继续
}
```

**取消条件**：

```typescript
// 用户输入新消息
if (event.type === "message.updated" && role === "user") {
  cancelCountdown(sessionID) // 立即取消倒计时
}

// 智能体开始响应
if (event.type === "message.updated" && role === "assistant") {
  cancelCountdown(sessionID)
}

// 工具执行开始/结束
if (event.type === "tool.execute.before" || event.type === "tool.execute.after") {
  cancelCountdown(sessionID)
}
```

---

## 四、生命周期钩子系统

### 4.1 钩子概览

oh-my-opencode 提供了 31 个生命周期钩子，用于拦截和修改智能体行为：

| 钩子类型                               | 数量 | 主要功能                         |
| -------------------------------------- | ---- | -------------------------------- |
| `chat.message`                         | 6    | 消息处理、关键词检测、命令解析   |
| `tool.execute.before`                  | 10   | 工具输入验证、参数修改、权限检查 |
| `tool.execute.after`                   | 8    | 输出处理、错误恢复、警告追加     |
| `event`                                | 7    | 会话事件、后台通知、状态恢复     |
| `experimental.chat.messages.transform` | 2    | 消息历史转换、thinking 验证      |

### 4.2 钩子执行顺序

**chat.message 流**：

```
用户发送消息
         │
         ▼
keywordDetector ────→ 检测关键词（ultrawork、search 等）
         │
         ▼
claudeCodeHooks ────→ 兼容 Claude Code settings.json
         │
         ▼
autoSlashCommand ───→ 自动识别 /command 模式
         │
         ▼
startWork ──────────→ 启动工作会话
         │
         ▼
ralphLoop ──────────→ 启动/取消 Ralph Loop
         │
         ▼
消息发送到模型
```

**tool.execute.before 流**：

```
工具即将执行
         │
         ▼
questionLabelTruncator → 截断问题标签（>30 字符）
         │
         ▼
claudeCodeHooks ────→ 工具兼容处理
         │
         ▼
nonInteractiveEnv ──→ 非 TTY 环境处理
         │
         ▼
commentChecker ─────→ 检查 AI slop 注释
         │
         ▼
directoryAgentsInjector → 注入 AGENTS.md 内容
         │
         ▼
directoryReadmeInjector → 注入 README.md 内容
         │
         ▼
rulesInjector ──────→ 注入条件规则
         │
         ▼
prometheusMdOnly ───→ 计划器只读模式检查
         │
         ▼
sisyphusJuniorNotepad → 记录工作笔记
         │
         ▼
atlasHook ──────────→ Atlas 编排处理
         │
         ▼
工具执行
```

**tool.execute.after 流**：

```
工具执行完成
         │
         ▼
claudeCodeHooks ────→ 后处理
         │
         ▼
toolOutputTruncator → 截断过长输出
         │
         ▼
contextWindowMonitor → 上下文窗口提醒
         │
         ▼
commentChecker ─────→ 检查输出中的 AI slop
         │
         ▼
editErrorRecovery ──→ 编辑错误恢复
         │
         ▼
delegateTaskRetry ──→ 委派任务重试
         │
         ▼
atlasHook ──────────→ Atlas 后处理
         │
         ▼
taskResumeInfo ─────→ 记录任务恢复信息
         │
         ▼
返回结果给智能体
```

### 4.3 典型钩子实现分析

**directory-agents-injector**（自动注入 AGENTS.md）：

```typescript
// src/hooks/directory-agents-injector/index.ts

export function createDirectoryAgentsInjectorHook(ctx: PluginInput) {
  return {
    "tool.execute.before": async (input, output) => {
      // 只处理 read/glob/grep 等文件探索工具
      if (!["read", "glob", "grep", "ast_grep_search"].includes(input.tool)) return

      // 查找目录中的 AGENTS.md
      const agentsPath = findAgentsFile(ctx.directory)
      if (!agentsPath) return

      // 读取内容
      const content = await Filesystem.read(agentsPath)

      // 在工具输出前注入提示
      output.output += `\n\n[AGENTS.md context]\n${content}`
    },
  }
}
```

**rules-injector**（条件规则注入）：

```typescript
// src/hooks/rules-injector/index.ts

export function createRulesInjectorHook(ctx: PluginInput) {
  const injectedHashes = new Set<string>() // 内容哈希去重

  return {
    "tool.execute.before": async (input, output) => {
      // 解析用户规则文件
      const rules = parseRulesFile(ctx.directory)

      // 根据工具类型匹配规则
      for (const rule of rules) {
        if (rule.tools.includes(input.tool)) {
          const hash = createHash(rule.content)
          if (injectedHashes.has(hash)) continue // 去重
          injectedHashes.add(hash)

          output.output += `\n\n[Rule: ${rule.name}]\n${rule.content}`
        }
      }
    },
  }
}
```

---

## 五、工具扩展系统

### 5.1 新增工具概览

oh-my-opencode 添加了以下新工具：

| 工具名称           | 功能描述           | 使用场景             |
| ------------------ | ------------------ | -------------------- |
| `delegate_task`    | 委派任务给子智能体 | 大型任务分解         |
| `skill`            | 加载技能/执行命令  | 领域专家指导         |
| `skill_mcp`        | 调用内置 MCP       | 文档搜索、GitHub搜索 |
| `slashcommand`     | 执行 slash 命令    | 快捷操作             |
| `interactive_bash` | TMUX 交互式 Bash   | TUI 应用交互         |
| `look_at`          | 多模态内容分析     | PDF、图片分析        |
| `call_omo_agent`   | 调用特定智能体     | Oracle、Librarian    |
| `background_task`  | 后台任务管理       | 并行探索             |

### 5.2 delegate_task 工具详解

`delegate_task` 是 oh-my-opencode 的核心工具，用于将复杂任务委派给专门的子智能体：

```typescript
// src/tools/delegate-task/index.ts

export const createDelegateTask = (options) => {
  return Tool.define("delegate_task", {
    description: "Spawn agent task with category-based or direct agent selection",
    parameters: z.object({
      category: z.enum(["visual-engineering", "ultrabrain", "deep", "quick", ...]),
      subagent_type: z.enum(["explore", "librarian", "oracle", "metis", "momus"]),
      prompt: z.string(),
      run_in_background: z.boolean(),
      session_id: z.string().optional(),  // 继续之前的会话
      load_skills: z.array(z.string()),   // 加载技能
    }),
    async execute(params, ctx) {
      // 1. 选择智能体
      const agent = params.subagent_type ?? getCategoryAgent(params.category)

      // 2. 创建后台任务
      const task = await manager.spawn({
        agent,
        prompt: params.prompt,
        parentSession: ctx.sessionID,
        skills: params.load_skills,
      })

      // 3. 返回任务 ID
      return {
        task_id: task.id,
        status: "running",
      }
    }
  })
}
```

**使用示例**：

```typescript
// 委派给 explore 智能体进行代码探索
await delegate_task({
  subagent_type: "explore",
  prompt: "Find all authentication implementations in src/",
  run_in_background: true,
  load_skills: [],
})

// 委派给 visual-engineering 类别进行 UI 工作
await delegate_task({
  category: "visual-engineering",
  prompt: "Redesign the sidebar layout",
  run_in_background: false,
  load_skills: ["frontend-ui-ux", "playwright"],
})
```

### 5.3 skill 工具详解

`skill` 工具用于加载领域专家技能或执行 slash 命令：

```typescript
// src/tools/skill/index.ts

export const createSkillTool = (options) => {
  return Tool.define("skill", {
    description: "Load a skill or execute a slash command",
    parameters: z.object({
      name: z.string(), // 技能名称（不带斜杠）
      user_message: z.string().optional(), // 命令参数
    }),
    async execute(params, ctx) {
      // 1. 查找技能
      const skill = mergedSkills.find((s) => s.name === params.name)
      if (!skill) throw new Error(`Skill not found: ${params.name}`)

      // 2. 加载技能内容
      const skillContent = await loadSkill(skill)

      // 3. 返回技能指导
      return {
        output: skillContent.instructions,
        metadata: { skill: skill.name },
      }
    },
  })
}
```

**内置技能列表**：

| 技能名称         | 领域         | 描述                       |
| ---------------- | ------------ | -------------------------- |
| `playwright`     | 浏览器自动化 | 必须用于任何浏览器相关任务 |
| `frontend-ui-ux` | UI/UX        | 无设计稿也能创建精美 UI    |
| `git-master`     | Git 操作     | 必须用于任何 git 操作      |
| `dev-browser`    | 浏览器自动化 | 持久页面状态的浏览器交互   |

### 5.4 skill_mcp 工具详解

`skill_mcp` 工具用于调用内置的 MCP 服务器：

```typescript
// src/tools/skill-mcp/index.ts

export const createSkillMcpTool = (options) => {
  return Tool.define("skill_mcp", {
    description: "Invoke MCP server operations from skill-embedded MCPs",
    parameters: z.object({
      mcp_name: z.enum(["websearch", "context7", "grep_app"]),
      tool_name: z.string().optional(),
      resource_name: z.string().optional(),
      arguments: z.record(z.unknown()).optional(),
    }),
    async execute(params, ctx) {
      // 1. 获取 MCP 连接
      const connection = await skillMcpManager.getConnection(params.mcp_name)

      // 2. 调用 MCP 工具
      if (params.tool_name) {
        const result = await connection.callTool(params.tool_name, params.arguments)
        return { output: result }
      }

      // 3. 或读取 MCP 资源
      if (params.resource_name) {
        const content = await connection.readResource(params.resource_name)
        return { output: content }
      }
    },
  })
}
```

---

## 六、Block Anchor Edit 机制（Hash-Anchored Edit）

### 6.1 核心概念

OpenCode 的 `edit` 工具实现了多种编辑算法，其中 **BlockAnchorReplacer** 是一种基于锚点的模糊匹配编辑方式：

> 不依赖精确的行号或字符位置，而是使用代码块的首尾两行作为锚点，中间内容通过相似度算法匹配。

这种方式比传统的行号编辑更健壮，能够处理：

- 代码重构导致的行号变化
- 缩进调整
- 空白字符差异
- 轻微的内容修改

### 6.2 Replacer 算法链

OpenCode 的 `edit.ts` 实现了 9 种 Replacer 算法，按优先级顺序执行：

```typescript
// packages/opencode/src/tool/edit.ts

export function replace(content: string, oldString: string, newString: string): string {
  for (const replacer of [
    SimpleReplacer, // 1. 精确匹配
    LineTrimmedReplacer, // 2. 行修剪匹配
    BlockAnchorReplacer, // 3. ⭐ 锚点匹配（Hash-Anchored）
    WhitespaceNormalizedReplacer, // 4. 空白规范化
    IndentationFlexibleReplacer, // 5. 缩进灵活
    EscapeNormalizedReplacer, // 6. 转义规范化
    TrimmedBoundaryReplacer, // 7. 边界修剪
    ContextAwareReplacer, // 8. 上下文感知
    MultiOccurrenceReplacer, // 9. 多次出现
  ]) {
    for (const search of replacer(content, oldString)) {
      // 找到匹配，执行替换
      return content.replace(search, newString)
    }
  }
  throw new Error("Could not find oldString in the file")
}
```

### 6.3 BlockAnchorReplacer 算法详解

```typescript
// packages/opencode/src/tool/edit.ts

export const BlockAnchorReplacer: Replacer = function* (content, find) {
  const originalLines = content.split("\n")
  const searchLines = find.split("\n")

  // 需要至少 3 行才能使用锚点匹配
  if (searchLines.length < 3) return

  // 提取首尾两行作为锚点
  const firstLineSearch = searchLines[0].trim() // 首行锚点
  const lastLineSearch = searchLines[searchLines.length - 1].trim() // 尾行锚点

  // 在文件中查找匹配首尾锚点的代码块
  const candidates: Array<{ startLine: number; endLine: number }> = []
  for (let i = 0; i < originalLines.length; i++) {
    if (originalLines[i].trim() !== firstLineSearch) continue

    // 找到匹配首锚点的行，然后查找尾锚点
    for (let j = i + 2; j < originalLines.length; j++) {
      if (originalLines[j].trim() === lastLineSearch) {
        candidates.push({ startLine: i, endLine: j })
        break
      }
    }
  }

  if (candidates.length === 0) return

  // 单候选：使用宽松阈值
  if (candidates.length === 1) {
    const similarity = calculateSimilarity(originalLines, searchLines, candidates[0])
    if (similarity >= 0.0) {
      // 只要有锚点匹配就接受
      yield extractBlock(content, candidates[0])
    }
    return
  }

  // 多候选：选择相似度最高的
  let bestMatch = null
  let maxSimilarity = -1

  for (const candidate of candidates) {
    const similarity = calculateSimilarity(originalLines, searchLines, candidate)
    if (similarity > maxSimilarity) {
      maxSimilarity = similarity
      bestMatch = candidate
    }
  }

  // 相似度阈值判断（0.3）
  if (maxSimilarity >= 0.3 && bestMatch) {
    yield extractBlock(content, bestMatch)
  }
}
```

**相似度计算（Levenshtein 距离）**：

```typescript
function levenshtein(a: string, b: string): number {
  // 编辑距离算法
  const matrix = Array.from({ length: a.length + 1 }, (_, i) =>
    Array.from({ length: b.length + 1 }, (_, j) => (i === 0 ? j : j === 0 ? i : 0)),
  )

  for (let i = 1; i <= a.length; i++) {
    for (let j = 1; j <= b.length; j++) {
      const cost = a[i - 1] === b[j - 1] ? 0 : 1
      matrix[i][j] = Math.min(
        matrix[i - 1][j] + 1, // 删除
        matrix[i][j - 1] + 1, // 插入
        matrix[i - 1][j - 1] + cost, // 替换
      )
    }
  }
  return matrix[a.length][b.length]
}

function calculateSimilarity(originalLines, searchLines, candidate): number {
  const { startLine, endLine } = candidate
  const actualBlockSize = endLine - startLine + 1
  const searchBlockSize = searchLines.length

  let similarity = 0
  const linesToCheck = Math.min(searchBlockSize - 2, actualBlockSize - 2)

  if (linesToCheck > 0) {
    for (let j = 1; j < searchBlockSize - 1; j++) {
      const originalLine = originalLines[startLine + j].trim()
      const searchLine = searchLines[j].trim()
      const maxLen = Math.max(originalLine.length, searchLine.length)
      if (maxLen === 0) continue

      const distance = levenshtein(originalLine, searchLine)
      similarity += (1 - distance / maxLen) / linesToCheck
    }
  } else {
    similarity = 1.0 // 只有锚点，无条件接受
  }

  return similarity
}
```

### 6.4 使用示例

**场景：函数签名重构**

```typescript
// 原代码（第 100-105 行）
async function fetchUser(id: string): Promise<User | null> {
  const response = await api.get(`/users/${id}`)
  return response.data
}

// AI 尝试编辑（假设文件内容已变化，行号不准确）
oldString: `async function fetchUser(id: string): Promise<User | null> {
  const response = await api.get(\`/users/\${id}\`)
  return response.data
}`

newString: `async function fetchUser(id: string): Promise<User | null> {
  const response = await api.get(\`/users/\${id}\`)
  if (!response.ok) return null
  return response.data
}`
```

**BlockAnchorReplacer 匹配过程**：

1. 提取锚点：
   - 首锚点：`async function fetchUser(id: string): Promise<User | null> {`
   - 尾锚点：`}`

2. 在文件中查找匹配锚点的候选块

3. 计算中间内容的相似度

4. 选择相似度最高的候选块（即使行号变化）

5. 执行替换

### 6.5 与其他 Replacer 的对比

| Replacer                     | 依赖              | 健壮性 | 适用场景   |
| ---------------------------- | ----------------- | ------ | ---------- |
| SimpleReplacer               | 精确字符串        | 低     | 简单修改   |
| LineTrimmedReplacer          | 行内容（trim）    | 中     | 空白差异   |
| BlockAnchorReplacer          | 首尾锚点 + 相似度 | **高** | 代码重构   |
| WhitespaceNormalizedReplacer | 空白规范化        | 中     | 格式差异   |
| IndentationFlexibleReplacer  | 缩进无关          | 中     | 缩进调整   |
| EscapeNormalizedReplacer     | 转义规范化        | 中     | 字符串转义 |
| ContextAwareReplacer         | 上下文匹配        | 高     | 大代码块   |

---

## 七、内置 MCP 服务器

### 7.1 MCP 概述

oh-my-opencode 内置了三个 MCP 服务器，通过 `skill_mcp` 工具调用：

| MCP 名称    | 提供商   | 功能               | 状态检测  |
| ----------- | -------- | ------------------ | --------- |
| `websearch` | Exa      | 网络搜索、信息检索 | Connected |
| `context7`  | Context7 | 官方文档搜索       | Connected |
| `grep_app`  | GitHub   | GitHub 代码搜索    | Connected |

### 7.2 MCP 加载机制

```typescript
// src/features/skill-mcp-manager/index.ts

export class SkillMcpManager {
  private connections = new Map<string, MCPConnection>()

  async getConnection(mcpName: string): Promise<MCPConnection> {
    if (this.connections.has(mcpName)) {
      return this.connections.get(mcpName)!
    }

    // 启动 MCP 服务器
    const connection = await this.startMcpServer(mcpName)
    this.connections.set(mcpName, connection)
    return connection
  }

  private async startMcpServer(mcpName: string): Promise<MCPConnection> {
    const serverConfig = BUILTIN_MCP_CONFIGS[mcpName]
    // serverConfig 包含：
    // - command: 启动命令
    // - args: 命令参数
    // - env: 环境变量

    // 例如 websearch MCP：
    // {
    //   command: "npx",
    //   args: ["-y", "@opencode-ai/mcp-server-exa"],
    //   env: { EXA_API_KEY: "..." }
    // }

    const process = spawn(serverConfig.command, serverConfig.args, {
      env: { ...process.env, ...serverConfig.env },
    })

    // 建立 stdio 通信
    const connection = new MCPConnection(process.stdin, process.stdout)
    await connection.initialize()
    return connection
  }
}
```

### 7.3 使用示例

**网络搜索（websearch）**：

```typescript
await skill_mcp({
  mcp_name: "websearch",
  tool_name: "web_search_exa",
  arguments: {
    query: "React Server Components best practices",
    numResults: 5,
  },
})
```

**官方文档搜索（context7）**：

```typescript
await skill_mcp({
  mcp_name: "context7",
  tool_name: "search_docs",
  arguments: {
    library: "react",
    topic: "hooks",
  },
})
```

**GitHub 代码搜索（grep_app）**：

```typescript
await skill_mcp({
  mcp_name: "grep_app",
  tool_name: "searchGitHub",
  arguments: {
    query: "useState(",
    language: ["TypeScript", "TSX"],
  },
})
```

---

## 八、后台智能体管理（BackgroundManager）

### 8.1 核心概念

BackgroundManager 管理后台运行的子智能体任务：

> 主智能体可以委派子任务给后台智能体，它们并行执行，完成后通知主智能体。

### 8.2 任务生命周期

```
主智能体调用 delegate_task
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  BackgroundManager.spawn()                              │
│     - 创建子会话                                         │
│     - 分配智能体类型                                     │
│     - 设置技能加载                                       │
│     - 返回 task_id                                       │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  子智能体在后台执行                                      │
│     - 访问受限工具集                                     │
│     - 独立的上下文窗口                                   │
│     - 不阻塞主会话                                       │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  任务完成                                                │
│     - 触发 background-notification Hook                 │
│     - 显示 Toast 通知                                    │
│     - 主智能体可通过 background_output 获取结果         │
└─────────────────────────────────────────────────────────┘
```

### 8.3 任务状态管理

```typescript
// src/features/background-agent/manager.ts

export class BackgroundManager {
  private tasks = new Map<string, BackgroundTask>()

  spawn(options: SpawnOptions): BackgroundTask {
    const task: BackgroundTask = {
      id: generateTaskId(),
      status: "pending",
      agent: options.agent,
      prompt: options.prompt,
      parentSession: options.parentSession,
      createdAt: Date.now(),
      sessionID: null,
    }

    this.tasks.set(task.id, task)

    // 异步启动子智能体
    this.runTask(task, options).catch((err) => {
      task.status = "failed"
      task.error = err.message
    })

    return task
  }

  getTask(taskId: string): BackgroundTask | undefined {
    return this.tasks.get(taskId)
  }

  getTasksByParentSession(sessionId: string): BackgroundTask[] {
    return Array.from(this.tasks.values()).filter((t) => t.parentSession === sessionId)
  }

  cancel(taskId: string): boolean {
    const task = this.tasks.get(taskId)
    if (!task || task.status !== "running") return false

    // 中止子会话
    if (task.sessionID) {
      ctx.client.session.abort({ path: { id: task.sessionID } })
    }

    task.status = "cancelled"
    return true
  }
}
```

---

## 九、与 OpenCode 核心的机制重叠分析

### 9.1 重叠情况总结

| 机制       | OpenCode 核心       | oh-my-opencode               | 重叠程度                         |
| ---------- | ------------------- | ---------------------------- | -------------------------------- |
| Agent 系统 | 基础 Agent          | 高级编排 Agent               | **无重叠**：插件扩展             |
| Edit 工具  | BlockAnchorReplacer | 无                           | **无重叠**：核心独有             |
| 会话管理   | 基础控制            | Ralph Loop/Todo Continuation | **无重叠**：插件增强             |
| MCP 客户端 | 客户端实现          | 内置服务器                   | **互补**：核心客户端，插件服务器 |
| Hook 系统  | 无                  | 31 个钩子                    | **无重叠**：插件新增             |
| 技能系统   | 无                  | Skills                       | **无重叠**：插件新增             |

### 9.2 BlockAnchorReplacer 的归属

用户提到的 "Hash-Anchored Edit Tool" 实际上是 **OpenCode 核心**的实现，位于 `packages/opencode/src/tool/edit.ts`。

**oh-my-opencode 并没有覆盖或修改这个工具**，而是：

- 通过 `edit-error-recovery` Hook 提供编辑失败的恢复机制
- 通过 `delegate-task-retry` Hook 提供委派任务的重试机制

### 9.3 协作关系图

```
┌────────────────────────────────────────────────────────────┐
│                      用户请求                               │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│              oh-my-opencode Hook 层                         │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ keywordDetector │→ │ autoSlashCommand│                 │
│  └─────────────────┘  └─────────────────┘                 │
│           ↓                      ↓                          │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ ralphLoop       │  │ rulesInjector   │                 │
│  └─────────────────┘  └─────────────────┘                 │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│               OpenCode 核心                                 │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ Session Manager │  │ Tool Registry   │                 │
│  └─────────────────┘  └─────────────────┘                 │
│           ↓                      ↓                          │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ Provider        │  │ Edit Tool       │ ← BlockAnchor   │
│  └─────────────────┘  └─────────────────┘                 │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│                      AI 模型                                │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│              oh-my-opencode Post-Hook 层                    │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ toolOutputTruncator│ │ editErrorRecovery│               │
│  └─────────────────┘  └─────────────────┘                 │
└────────────────────────────────────────────────────────────┘
```

---

## 十、总结

### 10.1 核心机制一览

| 机制                | 描述                             | 关键文件                                  |
| ------------------- | -------------------------------- | ----------------------------------------- |
| Ralph Loop          | 自指性开发循环，自动继续直到完成 | `src/hooks/ralph-loop/`                   |
| Todo Continuation   | 基于 Todo 列表的自动继续         | `src/hooks/todo-continuation-enforcer.ts` |
| 生命周期钩子        | 31 个拦截点扩展行为              | `src/hooks/`                              |
| delegate_task       | 任务委派给子智能体               | `src/tools/delegate-task/`                |
| skill               | 加载领域专家技能                 | `src/tools/skill/`                        |
| skill_mcp           | 调用内置 MCP 服务器              | `src/tools/skill-mcp/`                    |
| BackgroundManager   | 后台智能体任务管理               | `src/features/background-agent/`          |
| BlockAnchorReplacer | 锚点编辑算法（OpenCode 核心）    | `packages/opencode/src/tool/edit.ts`      |

### 10.2 与 OpenCode 的关系

oh-my-opencode 是 OpenCode 的**纯扩展层**，通过以下方式增强核心：

1. **钩子拦截**：在不修改核心代码的情况下，拦截和修改智能体行为
2. **工具扩展**：添加新工具类型，扩展智能体能力范围
3. **智能体编排**：提供高级编排智能体，实现复杂任务的自动化
4. **内置 MCP**：提供开箱即用的 MCP 服务器，无需用户配置

### 10.3 设计哲学

oh-my-opencode 的设计遵循以下原则：

1. **最小侵入**：通过钩子而非修改核心代码
2. **可配置性**：所有功能可通过配置文件禁用
3. **渐进式增强**：基础功能由 OpenCode 提供，高级功能由插件提供
4. **语义化设计**：Ralph Loop、Todo Continuation 等机制都基于明确的语义标记

---

## 附录：关键文件路径

```
oh-my-opencode/
├── src/
│   ├── index.ts                      # 插件入口
│   ├── hooks/
│   │   ├── ralph-loop/
│   │   │   ├── index.ts              # Ralph Loop 实现
│   │   │   ├── storage.ts            # 状态存储
│   │   │   ├── types.ts              # 类型定义
│   │   │   └── constants.ts          # 常量
│   │   ├── todo-continuation-enforcer.ts  # Todo 继续机制
│   │   ├── atlas/index.ts            # Atlas 编排
│   │   ├── directory-agents-injector/ # AGENTS.md 注入
│   │   ├── rules-injector/           # 规则注入
│   │   └── ...                       # 其他 31 个钩子
│   ├── tools/
│   │   ├── delegate-task/index.ts    # 任务委派
│   │   ├── skill/index.ts            # 技能加载
│   │   ├── skill-mcp/index.ts        # MCP 调用
│   │   ├── background-task/index.ts  # 后台任务工具
│   │   └── ...                       # 其他工具
│   ├── features/
│   │   ├── background-agent/
│   │   │   ├── manager.ts            # 后台管理器
│   │   │   └── index.ts              # 后台任务
│   │   ├── skill-mcp-manager/        # MCP 管理
│   │   ├── builtin-skills/           # 内置技能
│   │   └── builtin-commands/         # 内置命令
│   └── agents/
│       ├── sisyphus.ts               # Sisyphus 智能体
│       ├── atlas.ts                  # Atlas 智能体
│       └── ...                       # 其他智能体

OpenCode 核心：
packages/opencode/src/tool/
├── edit.ts                           # ⭐ BlockAnchorReplacer
├── read.ts                           # 文件读取
├── write.ts                          # 文件写入
├── grep.ts                           # 内容搜索
└── ...                               # 其他基础工具
```

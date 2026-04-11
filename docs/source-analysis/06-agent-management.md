# OpenCode Agent 管理机制技术分析

本文档分析 OpenCode 仓库核心架构子功能模块的技术实现，第一部分聚焦于 Agent 管理机制。

## 目录

1. [概述](#概述)
2. [Agent 类型系统](#agent-类型系统)
3. [内置 Agents](#内置-agents)
4. [自定义 Agents](#自定义-agents)
5. [Agent 自动生成功能](#agent-自动生成功能)
6. [开放接口](#开放接口)
7. [权限控制](#权限控制)
8. [CLI 命令](#cli-命令)
9. [配置方式](#配置方式)
10. [核心数据模型关系](#核心数据模型关系)

---

## 概述

Agent 是 OpenCode 的核心执行单元，每个 Agent 代表一个具有特定能力和权限限制的 AI 角色。Agent 系统的设计理念：

- **角色分离**：Primary Agent 直接与用户交互，Subagent 被 Primary Agent 调用
- **权限隔离**：每个 Agent 有独立的权限规则集，限制可使用的工具
- **可扩展性**：支持通过配置文件和 CLI 创建自定义 Agent
- **模型绑定**：每个 Agent 可绑定特定的模型和变体

---

## Agent 类型系统

### Agent.Info 数据结构

```typescript
// src/agent/agent.ts
export const Info = z.object({
  name: z.string(), // Agent 名称（唯一标识）
  description: z.string().optional(), // 描述文本
  mode: z.enum(["subagent", "primary", "all"]), // 运行模式
  native: z.boolean().optional(), // 是否内置 Agent
  hidden: z.boolean().optional(), // 是否隐藏（不在 UI 显示）
  topP: z.number().optional(), // 采样参数
  temperature: z.number().optional(), // 温度参数
  color: z.string().optional(), // UI 显示颜色
  permission: Permission.Ruleset, // 权限规则集
  model: z
    .object({
      // 绑定模型
      modelID: ModelID.zod,
      providerID: ProviderID.zod,
    })
    .optional(),
  variant: z.string().optional(), // 模型变体
  prompt: z.string().optional(), // 自定义 System Prompt
  options: z.record(z.string(), z.any()), // 扩展选项
  steps: z.number().int().positive().optional(), // 最大执行步数
})
```

### Agent 模式说明

| 模式       | 说明                                      | 使用场景              |
| ---------- | ----------------------------------------- | --------------------- |
| `primary`  | 主 Agent，直接与用户交互                  | build、plan           |
| `subagent` | 子 Agent，被其他 Agent 通过 task 工具调用 | explore、general      |
| `all`      | 可同时作为主 Agent 和子 Agent             | 自定义 Agent 默认模式 |

---

## 内置 Agents

### Primary Agents（主 Agent）

#### build

默认主 Agent，具有完整权限，用于常规开发工作。

```typescript
build: {
  name: "build",
  description: "The default agent. Executes tools based on configured permissions.",
  mode: "primary",
  native: true,
  permission: [
    // 默认权限：允许所有操作
    { permission: "*", pattern: "*", action: "allow" },
    // 特殊权限：询问确认
    { permission: "doom_loop", pattern: "*", action: "ask" },
    { permission: "external_directory", pattern: "*", action: "ask" },
    { permission: "read", pattern: "*.env", action: "ask" },
    { permission: "question", pattern: "*", action: "allow" },
    { permission: "plan_enter", pattern: "*", action: "allow" },
  ],
}
```

**特点**：

- 完整工具访问权限
- 支持问题询问（question tool）
- 支持进入规划模式（plan_enter）
- 可以编辑文件、执行 bash 命令

#### plan

规划模式 Agent，禁止编辑操作，专注于代码探索和方案规划。

```typescript
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  mode: "primary",
  native: true,
  permission: [
    // 禁止编辑
    { permission: "edit", pattern: "*", action: "deny" },
    // 允许编辑规划文件
    { permission: "edit", pattern: ".opencode/plans/*.md", action: "allow" },
    { permission: "edit", pattern: "$DATA/plans/*.md", action: "allow" },
    // 允许退出规划模式
    { permission: "plan_exit", pattern: "*", action: "allow" },
    { permission: "question", pattern: "*", action: "allow" },
  ],
}
```

**特点**：

- 禁止所有编辑工具（edit）
- 仅允许编辑 `.opencode/plans/*.md` 规划文件
- 允许退出规划模式（plan_exit）
- 只读探索，适合代码审查和方案设计

### Subagent Agents（子 Agent）

#### general

通用子 Agent，用于复杂研究和多步骤任务执行。

```typescript
general: {
  name: "general",
  description: "General-purpose agent for researching complex questions and executing multi-step tasks. Use this agent to execute multiple units of work in parallel.",
  mode: "subagent",
  native: true,
  permission: [
    // 禁止 todowrite
    { permission: "todowrite", pattern: "*", action: "deny" },
    // 其他权限继承默认配置
  ],
}
```

**特点**：

- 允许执行多步骤任务
- 禁止 todowrite（防止子 Agent 管理 todo）
- 可用于并行执行多个工作单元

#### explore

代码探索专用子 Agent，只允许只读工具，专注于文件搜索和代码分析。

```typescript
explore: {
  name: "explore",
  description: "Fast agent specialized for exploring codebases...",
  mode: "subagent",
  native: true,
  prompt: PROMPT_EXPLORE,  // 专用 System Prompt
  permission: [
    // 禁止大部分工具
    { permission: "*", pattern: "*", action: "deny" },
    // 仅允许只读工具
    { permission: "grep", pattern: "*", action: "allow" },
    { permission: "glob", pattern: "*", action: "allow" },
    { permission: "list", pattern: "*", action: "allow" },
    { permission: "bash", pattern: "*", action: "allow" },
    { permission: "webfetch", pattern: "*", action: "allow" },
    { permission: "websearch", pattern: "*", action: "allow" },
    { permission: "codesearch", pattern: "*", action: "allow" },
    { permission: "read", pattern: "*", action: "allow" },
  ],
}
```

**System Prompt**（`src/agent/prompt/explore.txt`）：

```
You are a file search specialist. You excel at thoroughly navigating and exploring codebases.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

Guidelines:
- Use Glob for broad file pattern matching
- Use Grep for searching file contents with regex
- Use Read when you know the specific file path you need to read
- Use Bash for file operations like copying, moving, or listing directory contents
- Adapt your search approach based on the thoroughness level specified by the caller
- Return file paths as absolute paths in your final response
- Do not create any files, or run bash commands that modify the user's system state
```

**特点**：

- 只允许只读工具（grep、glob、read、list 等）
- 禁止编辑和写入操作
- 支持三种搜索深度：quick、medium、very thorough
- 专注于代码探索和问题回答

### Hidden Agents（隐藏的系统 Agent）

#### compaction

历史压缩 Agent，用于压缩会话历史以节省 token。

```typescript
compaction: {
  name: "compaction",
  mode: "primary",
  native: true,
  hidden: true,
  prompt: PROMPT_COMPACTION,
  permission: [
    { permission: "*", pattern: "*", action: "deny" },
  ],
}
```

**特点**：

- 隐藏，不显示在 UI
- 禁止所有工具
- 用于自动压缩会话历史

#### title

标题生成 Agent，用于自动生成会话标题。

```typescript
title: {
  name: "title",
  mode: "primary",
  native: true,
  hidden: true,
  temperature: 0.5,
  prompt: PROMPT_TITLE,
  permission: [
    { permission: "*", pattern: "*", action: "deny" },
  ],
}
```

**特点**：

- 隐藏，不显示在 UI
- 禁止所有工具
- 低温度（0.5）保证标题一致性
- 用于自动生成会话标题

#### summary

摘要生成 Agent，用于生成会话摘要。

```typescript
summary: {
  name: "summary",
  mode: "primary",
  native: true,
  hidden: true,
  permission: [
    { permission: "*", pattern: "*", action: "deny" },
  ],
  prompt: PROMPT_SUMMARY,
}
```

**特点**：

- 隐藏，不显示在 UI
- 禁止所有工具
- 用于自动生成会话摘要

### 内置 Agent 总结

| Agent      | 模式     | 可见 | 权限特点                             | 用途         |
| ---------- | -------- | ---- | ------------------------------------ | ------------ |
| build      | primary  | ✅   | 完整权限 + question + plan_enter     | 默认主 Agent |
| plan       | primary  | ✅   | 禁止 edit，允许 plan_exit + question | 规划模式     |
| general    | subagent | ✅   | 禁止 todowrite                       | 通用任务执行 |
| explore    | subagent | ✅   | 仅允许只读工具                       | 代码探索     |
| compaction | primary  | ❌   | 禁止所有                             | 历史压缩     |
| title      | primary  | ❌   | 禁止所有                             | 标题生成     |
| summary    | primary  | ❌   | 禁止所有                             | 摘要生成     |

---

## 自定义 Agents

### 创建方式

#### 1. 通过 CLI 命令创建

```bash
# 交互式创建
opencode agent create

# 非交互式创建
opencode agent create \
  --description "Review code for security issues" \
  --mode subagent \
  --tools read,grep,glob \
  --path ./agents
```

#### 2. 通过配置文件创建

在 `.opencode/config.json` 或 `~/.config/opencode/config.json` 中配置：

```json
{
  "agent": {
    "security-reviewer": {
      "description": "Review code for security vulnerabilities",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4",
      "prompt": "You are a security expert...",
      "permission": {
        "edit": false,
        "write": false
      }
    }
  }
}
```

#### 3. 通过 Markdown 文件创建

在 `.opencode/agent/` 目录下创建 `.md` 文件：

```markdown
---
description: Use this agent when you need to review code for security issues
mode: subagent
tools:
  bash: false
  write: false
---

You are a security code reviewer. Your responsibilities:

- Identify potential security vulnerabilities
- Check for common attack patterns
- Suggest remediation strategies
```

### Agent 配置字段

```typescript
// src/config/config.ts Agent Schema
const Agent = z.object({
  disable: z.boolean().optional(), // 禁用 Agent
  description: z.string().optional(), // 描述
  mode: z.enum(["primary", "subagent", "all"]).optional(),
  name: z.string().optional(), // 重命名
  hidden: z.boolean().optional(), // 是否隐藏
  color: z.string().optional(), // UI 颜色
  model: z.string().optional(), // 绑定模型 (provider/model)
  variant: z.string().optional(), // 模型变体
  prompt: z.string().optional(), // System Prompt 文件路径
  temperature: z.number().optional(), // 温度参数
  top_p: z.number().optional(), // TopP 参数
  steps: z.number().int().positive().optional(), // 最大步数
  permission: PermissionConfig.optional(), // 权限配置
  options: z.record(z.any()).optional(), // 扩展选项
})
```

### Agent 自动生成功能

OpenCode 提供了 LLM 驱动的 Agent 自动生成功能，用户只需提供描述，系统会自动生成完整的 Agent 配置。

#### 使用方式一：CLI 命令

```bash
# 交互式创建（自动调用 LLM 生成）
opencode agent create

# 非交互式创建（指定描述，自动生成）
opencode agent create --description "Review code for security vulnerabilities"

# 完整参数示例
opencode agent create \
  --description "Analyze database schema and suggest optimizations" \
  --mode subagent \
  --tools read,grep,glob \
  --model anthropic/claude-sonnet-4 \
  --path ./custom-agents
```

**CLI 交互流程**：

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 用户输入描述                                              │
│    "Review code for security vulnerabilities"                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 系统调用 Agent.generate()                                 │
│    - 使用 generate.txt 作为 System Prompt                    │
│    - 调用默认模型或用户指定模型                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. LLM 返回生成结果                                          │
│    {                                                         │
│      identifier: "security-reviewer",                        │
│      whenToUse: "Use this agent when you need to...",        │
│      systemPrompt: "You are a security code reviewer..."     │
│    }                                                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 用户选择工具、模式                                         │
│    - 选择启用的工具（多选）                                   │
│    - 选择 Agent 模式（all/primary/subagent）                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. 生成 .md 文件                                             │
│    保存到 .opencode/agent/security-reviewer.md               │
│    或 ~/.config/opencode/agent/security-reviewer.md          │
└─────────────────────────────────────────────────────────────┘
```

#### 使用方式二：API 调用

```typescript
// src/agent/agent.ts
import { Agent } from "@/agent/agent"

// 基本调用
const result = await Agent.generate({
  description: "Review code for security vulnerabilities",
})

// 指定模型
const result = await Agent.generate({
  description: "Analyze database schema and suggest optimizations",
  model: {
    providerID: "anthropic",
    modelID: "claude-sonnet-4",
  },
})

// 返回结果
console.log(result)
// {
//   identifier: "security-reviewer",
//   whenToUse: "Use this agent when you need to review code for security vulnerabilities. Examples: ...",
//   systemPrompt: "You are a security code reviewer specializing in..."
// }
```

#### 生成输出格式

LLM 返回的 JSON 结构：

```typescript
{
  // Agent 唯一标识符
  // 规则：小写字母、数字、连字符，2-4 个单词
  identifier: "security-reviewer",

  // 使用场景描述（包含触发示例）
  // 格式：以 "Use this agent when..." 开头
  whenToUse: "Use this agent when you need to review code for security vulnerabilities.\n\nExamples:\n- When user asks 'Review this code for security issues'\n- After implementing authentication logic\n- When adding new API endpoints",

  // 完整的 System Prompt
  // 格式：第二人称（"You are..."）
  systemPrompt: "You are a security code reviewer specializing in identifying vulnerabilities..."
}
```

#### LLM 生成 Prompt 模板

`src/agent/generate.txt` 定义了 Agent 生成的指导规则：

```
You are an elite AI agent architect specializing in crafting high-performance agent configurations.

When a user describes what they want an agent to do, you will:

1. **Extract Core Intent**: Identify the fundamental purpose, key responsibilities,
   and success criteria for the agent.

2. **Design Expert Persona**: Create a compelling expert identity that embodies deep
   domain knowledge relevant to the task.

3. **Architect Comprehensive Instructions**: Develop a system prompt that:
   - Establishes clear behavioral boundaries and operational parameters
   - Provides specific methodologies and best practices for task execution
   - Anticipates edge cases and provides guidance for handling them
   - Incorporates any specific requirements or preferences mentioned by the user
   - Defines output format expectations when relevant

4. **Optimize for Performance**: Include:
   - Decision-making frameworks appropriate to the domain
   - Quality control mechanisms and self-verification steps
   - Efficient workflow patterns
   - Clear escalation or fallback strategies

5. **Create Identifier**: Design a concise, descriptive identifier that:
   - Uses lowercase letters, numbers, and hyphens only
   - Is typically 2-4 words joined by hyphens
   - Clearly indicates the agent's primary function
   - Is memorable and easy to type

6. **Example Usage**: Include examples of when this agent should be used:
   - Context: The scenario where this agent is useful
   - User message example
   - Assistant action example (calling the Task tool)

Your output must be a valid JSON object with exactly these fields:
{
  "identifier": "...",
  "whenToUse": "...",
  "systemPrompt": "..."
}
```

#### 生成的 Agent 文件示例

保存的 `.md` 文件格式：

```markdown
---
description: Use this agent when you need to review code for security vulnerabilities
mode: subagent
tools:
  bash: false
  write: false
  edit: false
model: anthropic/claude-sonnet-4
---

You are a security code reviewer specializing in identifying vulnerabilities.

Your responsibilities:

- Identify potential security vulnerabilities (SQL injection, XSS, CSRF, etc.)
- Check authentication and authorization logic
- Review input validation and sanitization
- Analyze data exposure risks
- Suggest remediation strategies with specific code examples

Approach:

1. First, identify the code's purpose and entry points
2. Map data flow from user input to sensitive operations
3. Check each entry point for common vulnerability patterns
4. Verify proper error handling doesn't expose sensitive info
5. Provide actionable recommendations with code examples

Output format:

- List identified issues with severity (Critical/High/Medium/Low)
- Provide specific code location references
- Suggest remediation code snippets
- Summarize overall security posture
```

#### 生成流程技术实现

```typescript
// src/agent/agent.ts Agent.generate 实现
generate: Effect.fn("Agent.generate")(function* (input) {
  const cfg = yield* config.get()
  const model = input.model ?? (yield* provider.defaultModel())
  const resolved = yield* provider.getModel(model.providerID, model.modelID)
  const language = yield* provider.getLanguage(resolved)

  // 构建 System Prompt
  const system = [PROMPT_GENERATE] // generate.txt

  // 获取现有 Agent 列表（避免标识符冲突）
  const existing = yield* InstanceState.useEffect(state, (s) => s.list())

  // 构建用户消息
  const userMessage = `Create an agent configuration based on this request: 
    "${input.description}".
    
    IMPORTANT: The following identifiers already exist and must NOT be used: 
    ${existing.map((i) => i.name).join(", ")}
    
    Return ONLY the JSON object, no other text.`

  // 调用 LLM
  const result = yield* Effect.promise(() =>
    generateObject({
      model: language,
      temperature: 0.3,
      messages: [
        { role: "system", content: system.join("\n") },
        { role: "user", content: userMessage },
      ],
      schema: z.object({
        identifier: z.string(),
        whenToUse: z.string(),
        systemPrompt: z.string(),
      }),
    }).then((r) => r.object),
  )

  return result
})
```

---

## 开放接口

### Agent.Service 接口定义

```typescript
// src/agent/agent.ts
export interface Interface {
  readonly get: (agent: string) => Effect.Effect<Agent.Info>
  readonly list: () => Effect.Effect<Agent.Info[]>
  readonly defaultAgent: () => Effect.Effect<string>
  readonly generate: (input: {
    description: string
    model?: { providerID: ProviderID; modelID: ModelID }
  }) => Effect.Effect<{
    identifier: string
    whenToUse: string
    systemPrompt: string
  }>
}
```

### API 方法

| 方法                    | 说明                | 返回类型                                           |
| ----------------------- | ------------------- | -------------------------------------------------- |
| `Agent.get(name)`       | 获取指定 Agent 信息 | `Promise<Agent.Info>`                              |
| `Agent.list()`          | 获取所有 Agent 列表 | `Promise<Agent.Info[]>`                            |
| `Agent.defaultAgent()`  | 获取默认 Agent 名称 | `Promise<string>`                                  |
| `Agent.generate(input)` | 自动生成 Agent 配置 | `Promise<{ identifier, whenToUse, systemPrompt }>` |

### Effect 服务架构

```typescript
// Agent 使用 Effect 服务模式
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

// Layer 定义
export const layer = Layer.effect(Service, Effect.gen(function* () {
  // 使用 InstanceState 管理每目录状态
  const state = yield* InstanceState.make<State>(...)

  return Service.of({
    get: Effect.fn("Agent.get")(function* (agent: string) { ... }),
    list: Effect.fn("Agent.list")(function* () { ... }),
    defaultAgent: Effect.fn("Agent.defaultAgent")(function* () { ... }),
    generate: Effect.fn("Agent.generate")(function* (input) { ... }),
  })
}))
```

---

## 权限控制

### 默认权限配置

```typescript
const defaults = Permission.fromConfig({
  "*": "allow", // 默认允许所有
  doom_loop: "ask", // 无限循环询问
  external_directory: { "*": "ask" }, // 外部目录询问
  question: "deny", // 禁止问题工具
  plan_enter: "deny", // 禁止进入规划模式
  plan_exit: "deny", // 禁止退出规划模式
  read: {
    "*": "allow",
    "*.env": "ask", // .env 文件询问
    "*.env.*": "ask", // .env.* 文件询问
    "*.env.example": "allow", // .env.example 允许
  },
})
```

### 权限合并规则

```typescript
// 权限合并顺序：defaults → specific → user
agent.permission = Permission.merge(
  defaults,                                // 默认权限
  Permission.fromConfig({ ... }),          // Agent 特定权限
  user,                                    // 用户配置权限
)
```

### 权限动作类型

| 动作    | 说明               |
| ------- | ------------------ |
| `allow` | 允许执行，无需确认 |
| `deny`  | 禁止执行           |
| `ask`   | 执行前询问用户确认 |

---

## CLI 命令

### opencode agent create

创建新 Agent。

```bash
opencode agent create [options]

Options:
  --path <dir>          Agent 文件存放目录
  --description <text>  Agent 描述
  --mode <mode>         Agent 模式 (all/primary/subagent)
  --tools <tools>       启用的工具列表（逗号分隔）
  --model <model>       绑定模型 (provider/model)
```

**可用工具列表**：

- `bash` - 执行命令
- `read` - 读取文件
- `write` - 写入文件
- `edit` - 编辑文件
- `list` - 列出目录
- `glob` - 文件模式匹配
- `grep` - 内容搜索
- `webfetch` - 网页抓取
- `task` - 调用子 Agent
- `todowrite` - Todo 管理

### opencode agent list

列出所有可用 Agent。

```bash
opencode agent list

# 输出格式：
build (primary)
  [权限规则...]
plan (primary)
  [权限规则...]
explore (subagent)
  [权限规则...]
```

---

## 配置方式

### 全局配置

文件位置：`~/.config/opencode/config.json`

```json
{
  "default_agent": "build",
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4",
      "temperature": 0.7
    },
    "custom-reviewer": {
      "description": "Review code changes",
      "mode": "subagent",
      "permission": {
        "edit": false,
        "write": false
      }
    }
  }
}
```

### 项目配置

文件位置：`项目根目录/.opencode/config.json`

```json
{
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4",
      "prompt": "./custom-build-prompt.txt"
    }
  }
}
```

### Agent 文件

文件位置：`.opencode/agent/*.md` 或 `~/.config/opencode/agent/*.md`

格式：Markdown + YAML Frontmatter

```markdown
---
description: Use this agent when you need to...
mode: subagent
tools:
  bash: false
  write: false
  edit: false
model: anthropic/claude-haiku-3.5
---

You are a specialized agent. Your instructions...
```

---

## 核心数据模型关系

OpenCode 的数据模型采用四层层级结构：Project → Session → Message → Part。

### 层级关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Project (项目)                                                           │
│ - 代表一个 Git Worktree 或非 Git 目录                                     │
│ - ID 基于 Git 仓库首个 commit hash                                       │
│ - 一个 Project 可包含多个 Sandbox（工作目录）                              │
└─────────────────────────────────────────────────────────────────────────┘
                    │ 1:N
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Session (会话)                                                           │
│ - 代表一次对话                                                            │
│ - 关联到 Project 和具体工作目录                                            │
│ - 可有 parentID 形成父子关系（子 Agent 会话）                              │
└─────────────────────────────────────────────────────────────────────────┘
                    │ 1:N
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Message (消息)                                                           │
│ - Session 中的一条消息（user 或 assistant）                               │
│ - 包含时间戳、错误信息、模型信息等元数据                                    │
└─────────────────────────────────────────────────────────────────────────┘
                    │ 1:N
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Part (内容片段)                                                           │
│ - Message 中的一个内容单元                                                 │
│ - 类型包括：text, file, tool, reasoning, snapshot, patch 等              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 数据模型定义

#### Project 数据结构

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

**数据库表**（`src/project/project.sql.ts`）：

```typescript
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
```

#### Session 数据结构

```typescript
// src/session/index.ts
export const Info = z.object({
  id: SessionID.zod,
  slug: z.string(), // URL 友好的短标识
  projectID: ProjectID.zod, // 关联 Project
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

**数据库表**（`src/session/session.sql.ts`）：

```typescript
export const SessionTable = sqliteTable("session", {
  id: text().$type<SessionID>().primaryKey(),
  project_id: text()
    .$type<ProjectID>()
    .notNull()
    .references(() => ProjectTable.id, { onDelete: "cascade" }),
  workspace_id: text().$type<WorkspaceID>(),
  parent_id: text().$type<SessionID>(),
  slug: text().notNull(),
  directory: text().notNull(),
  title: text().notNull(),
  version: text().notNull(),
  share_url: text(),
  summary_additions: integer(),
  summary_deletions: integer(),
  summary_files: integer(),
  permission: text({ mode: "json" }).$type<Permission.Ruleset>(),
  time_created: integer(),
  time_updated: integer(),
  time_compacting: integer(),
  time_archived: integer(),
})
```

#### Message 数据结构

```typescript
// src/session/message-v2.ts
export const User = z
  .object({
    id: MessageID.zod,
    sessionID: SessionID.zod,
    role: z.literal("user"),
    error: z.any().optional(),
    time: z.object({ created: z.number() }),
  })
  .meta({ ref: "MessageUser" })

export const Assistant = z
  .object({
    id: MessageID.zod,
    sessionID: SessionID.zod,
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

**数据库表**：

```typescript
export const MessageTable = sqliteTable("message", {
  id: text().$type<MessageID>().primaryKey(),
  session_id: text()
    .$type<SessionID>()
    .notNull()
    .references(() => SessionTable.id, { onDelete: "cascade" }),
  time_created: integer(),
  time_updated: integer(),
  data: text({ mode: "json" }).notNull().$type<InfoData>(),
})
```

#### Part 数据结构

```typescript
// src/session/message-v2.ts
const PartBase = z.object({
  id: PartID.zod,
  sessionID: SessionID.zod,
  messageID: MessageID.zod,
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

**数据库表**：

```typescript
export const PartTable = sqliteTable("part", {
  id: text().$type<PartID>().primaryKey(),
  message_id: text()
    .$type<MessageID>()
    .notNull()
    .references(() => MessageTable.id, { onDelete: "cascade" }),
  session_id: text().$type<SessionID>().notNull(),
  time_created: integer(),
  time_updated: integer(),
  data: text({ mode: "json" }).notNull().$type<PartData>(),
})
```

### Project 创建多个的场景

#### 场景一：Git Worktree

**核心概念**：Git Worktree 允许一个 Git 仓库在多个目录同时检出不同分支。

```
Git Repository (共享 .git 目录)
    │
    ├── worktree/           ← 主工作目录（Project.worktree）
    │   └── .git            ← Git 目录
    │
    ├── sandbox-1/          ← Git Worktree 1（Project.sandboxes[0]）
    │   └── .git (文件，指向主 .git)
    │
    └── sandbox-2/          ← Git Worktree 2（Project.sandboxes[1]）
        └── .git (文件，指向主 .git)
```

**Project ID 生成逻辑**（`src/project/project.ts`）：

```typescript
// ProjectID 基于 Git 仓库的首个 commit hash
const revList = yield * git(["rev-list", "--max-parents=0", "HEAD"], { cwd: sandbox })
const roots = revList.text
  .split("\n")
  .filter(Boolean)
  .map((x) => x.trim())
  .toSorted()
id = roots[0] ? ProjectID.make(roots[0]) : undefined

// 缓存 ProjectID 到 .git/opencode 文件
if (id) {
  yield * fs.writeFileString(path.join(worktree, ".git", "opencode"), id)
}
```

**关键点**：

- 同一个 Git 仓库的所有 Worktree 共享同一个 ProjectID
- Project.sandboxes 记录所有属于此 Project 的工作目录
- 打开不同 Worktree 目录时，Session 属于同一个 Project

#### 场景二：多个独立 Git 仓库

每个独立的 Git 仓库有独立的历史，因此会创建不同的 Project：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Project A       │     │ Project B       │     │ Project C       │
│ (repo-a.git)    │     │ (repo-b.git)    │     │ (repo-c.git)    │
│                 │     │                 │     │                 │
│ ├── worktree/   │     │ ├── worktree/   │     │ ├── worktree/   │
│ └── sandbox/    │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
  Session 1...N           Session 1...M           Session 1...K
```

**触发方式**：

- 用户在不同 Git 仓库目录启动 OpenCode
- 每个 Git 仓库生成独立的 ProjectID

#### 场景三：非 Git 目录（全局 Project）

当目录不在 Git 仓库内时，会创建一个"全局 Project"：

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

- 所有非 Git 目录的 Session 都关联到同一个全局 Project
- worktree 和 sandbox 都设置为 "/"
- 不支持 Worktree 功能

### Session 创建流程

当用户打开一个工程时的完整流程：

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
│    - 添加 TextPart/FilPart 等                                 │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Assistant Message 创建（LLM 响应）                         │
│    - 创建 Assistant Message                                   │
│    - 流式添加 TextPart, ToolPart, ReasoningPart 等            │
└─────────────────────────────────────────────────────────────┘
```

**关键理解**：

| 问题                            | 答案                                                               |
| ------------------------------- | ------------------------------------------------------------------ |
| 打开工程是否直接创建会话？      | 是，创建 Session 是用户交互的第一步，但 Session 必须关联到 Project |
| Project 何时创建？              | 打开目录时自动检测并创建/更新 Project                              |
| 一个 Project 对应多个 Session？ | 是，同一 Project 下可以有多个 Session                              |
| 一个 Project 对应多个目录？     | 是，Git Worktree 场景下，Project.sandboxes 记录多个目录            |
| 多个 Project 何时出现？         | 用户在不同 Git 仓库目录工作，每个仓库一个 Project                  |

### Instance 与 Project 的区别

**Instance**：运行时上下文，通过 AsyncLocalStorage 管理：

```typescript
// src/project/instance.ts
export interface InstanceContext {
  directory: string // 当前工作目录
  worktree: string // Git Worktree 根目录
  project: Project.Info // 关联的 Project 信息
}
```

**Instance 作用**：

- 提供 ALS 上下文，让代码可以访问当前目录和 Project
- 每个打开的目录对应一个 Instance
- Instance 是运行时概念，Project 是持久化概念

**关系图**：

```
┌─────────────────────────────────────────┐
│ Instance (运行时 ALS 上下文)              │
│                                         │
│  directory ──────► 当前工作目录           │
│  worktree  ──────► Git Worktree 根目录   │
│  project   ──────► Project.Info (持久化) │
└─────────────────────────────────────────┘
```

---

## 总结

OpenCode Agent 系统的核心设计：

1. **三层 Agent 分类**：Primary、Subagent、Hidden
2. **内置 7 个 Agent**：build、plan、general、explore、compaction、title、summary
3. **灵活的扩展机制**：CLI、配置文件、Markdown 文件
4. **细粒度权限控制**：基于 Permission.Ruleset 的工具访问限制
5. **Effect 服务架构**：使用 InstanceState 管理每目录状态
6. **LLM 驱动生成**：支持自动生成 Agent 配置

---

## 参考文件

- `packages/opencode/src/agent/agent.ts` - Agent 定义和服务
- `packages/opencode/src/agent/prompt/explore.txt` - Explore Agent Prompt
- `packages/opencode/src/agent/generate.txt` - Agent 生成 Prompt
- `packages/opencode/src/config/config.ts` - Agent 配置 Schema
- `packages/opencode/src/cli/cmd/agent.ts` - CLI 命令实现
- `packages/opencode/src/permission/index.ts` - 权限系统

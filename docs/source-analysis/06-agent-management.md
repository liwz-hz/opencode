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

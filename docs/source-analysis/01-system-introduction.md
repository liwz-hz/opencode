# OpenCode 系统架构总览

本文档从源码级别系统性介绍 OpenCode 的核心架构、关键流程和模型交互界面。

## 1. 整体架构概览

OpenCode 是一个基于 Bun 运行时的 TypeScript/Node.js 项目，采用 Effect 框架构建服务层，使用 AI SDK v5 (Vercel AI SDK) 作为 AI 模型交互的基础层。

### 1.1 核心模块划分

```
packages/opencode/src/
├── index.ts              # CLI 入口点 (yargs 命令解析)
├── agent/                # Agent 定义与管理
├── session/              # 会话生命周期管理
├── tool/                 # 工具系统
├── provider/             # AI 提供商抽象层
├── project/              # 项目与实例上下文
├── permission/           # 权限控制系统
├── config/               # 配置管理
├── storage/              # 数据持久化
├── sync/                 # 事件同步系统
└── acp/                  # Agent Client Protocol (外部集成)
```

### 1.2 技术栈

- **运行时**: Bun (TypeScript 直接执行)
- **数据库**: SQLite + Drizzle ORM (WAL 模式)
- **服务框架**: Effect 3.x (函数式响应式编程)
- **AI SDK**: Vercel AI SDK v5 (统一模型接口)
- **状态管理**: AsyncLocalStorage (目录级上下文隔离)
- **事件系统**: 自定义 SyncEvent + Projector (事件溯源)

---

## 2. Agent 系统

### 2.1 Agent 类型定义

Agent 是 OpenCode 中的核心概念，定义了不同类型的"智能体"及其行为模式。

**Agent 信息结构** (`src/agent/agent.ts`):

```typescript
Agent.Info = {
  name: string,              // Agent 名称
  description?: string,      // 描述
  mode: "subagent" | "primary" | "all",  // 使用模式
  native?: boolean,          // 是否内置
  hidden?: boolean,          // 是否隐藏 (不显示在 UI)

  // 模型参数
  topP?: number,
  temperature?: number,

  // 权限与工具
  permission: Permission.Ruleset,  // 工具访问控制规则

  // 模型配置
  model?: { providerID, modelID },  // 可覆盖默认模型
  variant?: string,                  // 模型变体 (如 "thinking")

  // 提示词
  prompt?: string,           // 自定义系统提示词

  // 其他
  options: Record<string, any>,  // 提供商特定选项
  steps?: number,              // 最大迭代次数
  color?: string,              // UI 显示颜色
}
```

### 2.2 内置 Agent 类型

| Agent          | 模式     | 隐藏 | 功能说明                                                      |
| -------------- | -------- | ---- | ------------------------------------------------------------- |
| **build**      | primary  | 否   | 默认 Agent，拥有完整工具权限，执行代码修改                    |
| **plan**       | primary  | 否   | 只读模式，禁止编辑类工具，用于规划与分析                      |
| **general**    | subagent | 否   | 多步骤研究任务，通过 task 工具调用                            |
| **explore**    | subagent | 否   | 快速代码库探索，只读工具集 (glob, grep, read, bash, webfetch) |
| **compaction** | subagent | 是   | 会话压缩/摘要生成                                             |
| **title**      | subagent | 是   | 会话标题生成                                                  |
| **summary**    | subagent | 是   | 消息摘要生成                                                  |

### 2.3 Agent 处理流程

**主循环** (`src/session/prompt.ts` - `SessionPrompt.loop`):

```
┌─────────────────────────────────────────────────────────────┐
│                     Agent 主循环                             │
├─────────────────────────────────────────────────────────────┤
│  1. 创建 AbortController (会话中断控制)                      │
│                                                             │
│  2. While 循环:                                             │
│     a. 从数据库流式加载消息                                  │
│     b. 找到最后用户消息和最后完成的助手消息                   │
│     c. 检查待处理子任务 → 执行 TaskTool                      │
│     d. 检查待处理压缩 → 执行 SessionCompaction               │
│     e. 检查上下文溢出 → 创建压缩任务                         │
│     f. 获取 Agent 配置                                      │
│     g. 解析可用工具 (权限过滤 + 模型兼容性)                  │
│     h. 构建系统提示词                                        │
│     i. 创建 SessionProcessor (流事件处理器)                  │
│     j. 处理 LLM 流                                          │
│     k. 检查完成原因 → 继续或终止                             │
│                                                             │
│  3. 返回最终助手消息                                         │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 子 Agent 系统

通过 `TaskTool` 调用子 Agent (`src/tool/task.ts`):

```typescript
TaskTool.execute({
  description: string,      // 3-5 词简短描述
  prompt: string,           // 子 Agent 任务内容
  subagent_type: string,    // Agent 名称 (explore, general 等)
  task_id?: string,         // 恢复现有任务
  command?: string,         // 触发此任务的命令
})
```

**子 Agent 流程**:

1. 权限检查 (task 工具 + agent 名称)
2. 获取 Agent 配置
3. 创建或加载子会话
4. 解析提示词内容
5. 调用 `SessionPrompt.prompt()` 执行
6. 返回结果及 `task_id` 用于后续恢复

---

## 3. Session (会话) 系统

### 3.1 数据模型层次

Session 采用三层数据结构:

```
Session (会话)
    └── Message (消息) [N]
           └── Part (部件) [N]
```

**核心数据库表** (`src/session/session.sql.ts`):

| 表名         | 主键                  | 核心字段                                                                         |
| ------------ | --------------------- | -------------------------------------------------------------------------------- |
| SessionTable | id (SessionID)        | project_id, workspace_id, parent_id, title, share_url, permission, status 时间戳 |
| MessageTable | id (MessageID)        | session_id (FK), data (JSON: role, time, agent, model 等)                        |
| PartTable    | id (PartID)           | message_id (FK), session_id, data (JSON: type + 内容)                            |
| TodoTable    | session_id + position | content, status, priority                                                        |

**ID 生成策略** (`src/session/schema.ts`):

- **SessionID**: `descending()` - 时间倒序 ULID，便于按时间排序
- **MessageID/PartID**: `ascending()` - 正序，保持消息内顺序

### 3.2 消息类型

**User 消息** (`src/session/message-v2.ts`):

```typescript
{
  role: "user",
  agent?: string,      // 使用的 Agent
  model?: string,      // 使用的模型
  tools?: string[],    // 可用工具列表
  format?: string,     // 消息格式
  variant?: string,    // 模型变体
  time: number,        // 时间戳
}
```

**Assistant 消息**:

```typescript
{
  role: "assistant",
  agent?: string,
  finish?: "stop" | "tool-use" | "length" | "error",
  error?: string,
  tokens?: { input, output, reasoning, cache },
  cost?: number,
  time: number,
}
```

### 3.3 Part (部件) 类型

| 类型                         | 说明                                             |
| ---------------------------- | ------------------------------------------------ |
| `text`                       | 文本内容                                         |
| `reasoning`                  | 思维链/推理过程                                  |
| `file`                       | 文件附件 (URL, MIME, 文件名)                     |
| `tool`                       | 工具调用 (状态: pending/running/completed/error) |
| `subtask`                    | 待处理子 Agent 任务                              |
| `compaction`                 | 压缩标记                                         |
| `step-start` / `step-finish` | 多步骤边界                                       |
| `patch`                      | 文件差异快照                                     |

### 3.4 会话生命周期

**创建流程** (`src/session/index.ts`):

```
Session.create({ parentID?, title?, permission?, workspaceID? })
    │
    ├─→ SessionID.descending() 生成 ID
    ├─→ Slug.create() 生成 URL slug
    ├─→ 设置默认标题 (时间格式)
    ├─→ 发送 Session.Event.Created 同步事件
    ├─→ Projector 插入数据库记录
    └─→ (可选) 自动分享
```

**Fork (分叉) 流程**:

```
Session.fork({ sessionID, messageID? })
    │
    ├─→ 获取原始会话
    ├─→ 生成分叉标题: "title (fork #N)"
    ├─→ 创建新会话
    ├─→ 克隆消息至指定 messageID
    └─→ 重映射消息/部件 ID (保持父子关系)
```

### 3.5 事件溯源架构

OpenCode 采用事件溯源模式处理所有会话变更:

**SyncEvent 系统** (`src/sync/index.ts` + `src/session/projectors.ts`):

```
所有变更操作
      │
      ↓ 发送 SyncEvent
  Event Bus
      │
      ↓ Projector 处理
  Database Mutation
      │
      ↓ 版本化存储
  Event Log (可回放)
```

**事件类型**:

- `session.created` → INSERT SessionTable
- `session.updated` → UPDATE SessionTable (部分字段)
- `session.deleted` → DELETE SessionTable (级联删除)

---

## 4. Provider (提供商) 系统

### 4.1 Provider 抽象层

Provider 系统统一封装了 20+ AI 提供商，基于 AI SDK v5 构建。

**Provider.Info 结构** (`src/provider/provider.ts`):

```typescript
{
  id: ProviderID,              // 提供商标识 (anthropic, openai, google 等)
  name: string,                // 显示名称
  source: "env" | "config" | "custom" | "api",
  env: string[],               // API Key 环境变量名
  key: string,                 // API Key
  options: Record<string, any>, // 提供商特定选项
  models: Record<string, Model>  // 可用模型列表
}
```

**Provider.Model 结构**:

```typescript
{
  id: ModelID,
  providerID: ProviderID,
  api: { id, url, npm },       // SDK npm 包名
  capabilities: {
    temperature, reasoning, attachment, toolcall,
    input, output, interleaved
  },
  cost: {                      // 每百万 token 成本
    input, output,
    cache: { read, write }
  },
  limit: {                     // Token 限制
    context, output
  },
  status: "alpha" | "beta" | "deprecated" | "active"
}
```

### 4.2 内置 SDK 支持

**bundled SDKs** (20+):

- Anthropic (`@ai-sdk/anthropic`)
- OpenAI (`@ai-sdk/openai`)
- Google Gemini (`@ai-sdk/google`)
- Amazon Bedrock (`@ai-sdk/amazon-bedrock`)
- Azure OpenAI (`@ai-sdk/azure`)
- OpenRouter (`@openrouter/ai-sdk-provider`)
- GitLab Workflow (`gitlab-ai-provider`)
- GitHub Copilot (`@ai-sdk/github-copilot`)
- 其他: groq, xai, mistral, deepseek, cohere, fireworks 等

### 4.3 请求流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Provider 请求流程                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 模型选择                                                 │
│     Provider.getModel(providerID, modelID)                  │
│                                                             │
│  2. SDK 初始化                                               │
│     getSDK(model) → 创建/缓存 SDK 实例                      │
│                                                             │
│  3. LanguageModel 创建                                       │
│     Provider.getLanguage(model) → LanguageModelV3           │
│                                                             │
│  4. 消息变换                                                 │
│     ProviderTransform.message() → 规范化格式                │
│                                                             │
│  5. 流式调用                                                 │
│     LLM.stream() → streamText() → SSE 流                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 流式响应处理

**两种流式 API 实现**:

**Chat API** (OpenAI 兼容):

- `doStream()` 使用 SSE EventSource
- 处理 text-delta, reasoning-delta, tool-call 等事件

**Responses API** (OpenAI 新格式):

- 支持 reasoning (思维链)
- 支持 code interpreter, web search, file search
- 加密的 reasoning 内容用于多轮对话

### 4.5 Token 使用与成本计算

**getUsage()** (`src/session/index.ts`):

```typescript
tokens = {
  input: adjustedInputTokens,   // 排除缓存
  output: outputTokens,
  reasoning: reasoningTokens,
  cache: { write, read }
}

cost = tokens.input * cost.input / 1M
     + tokens.output * cost.output / 1M
     + tokens.cache.read * cost.cache.read / 1M
     + tokens.cache.write * cost.cache.write / 1M
     + tokens.reasoning * cost.output / 1M
```

### 4.6 认证机制

**两种认证类型**:

| 类型    | 存储结构                                      | 适用提供商                   |
| ------- | --------------------------------------------- | ---------------------------- |
| API Key | `{ type: "api", key }`                        | Anthropic, OpenAI, Google 等 |
| OAuth   | `{ type: "oauth", access, refresh, expires }` | GitHub Copilot, GitLab       |

---

## 5. Tool (工具) 系统

### 5.1 工具定义接口

**Tool.define** (`src/tool/tool.ts`):

```typescript
Tool.define({
  name: string,               // 工具名称
  description: string,        // 工具描述
  parameters: z.ZodSchema,    // 参数 Schema (Zod)

  // 执行函数
  execute: (ctx: Tool.Context, args) => Promise<Tool.Result>,

  // 可选
  enabled?: (ctx) => boolean, // 启用条件
  truncation?: {              // 输出截断策略
    threshold: number,
    truncate: (result) => string
  }
})
```

**Tool.Context**:

```typescript
{
  sessionID,
  messageID,
  agent,
  model,
  permission,
  abort,       // AbortSignal
  cwd,         // 工作目录
}
```

### 5.2 工具注册表

**Registry** (`src/tool/registry.ts`):

- 注册内置工具
- 按模型/提供商过滤工具
- 加载自定义工具 (用户定义)
- 集成 MCP 工具 (Model Context Protocol)

### 5.3 核心工具列表

| 工具         | 功能                                          |
| ------------ | --------------------------------------------- |
| `bash`       | 执行 shell 命令                               |
| `read`       | 读取文件/目录                                 |
| `write`      | 写入文件                                      |
| `edit`       | 文件编辑 (字符串替换)                         |
| `glob`       | 文件模式匹配                                  |
| `grep`       | 内容搜索                                      |
| `webfetch`   | 获取网页内容                                  |
| `task`       | 调用子 Agent                                  |
| `lsp_*`      | LSP 工具集 (诊断、定义跳转、引用查找、重命名) |
| `ast_grep_*` | AST 模式搜索/替换                             |
| `session_*`  | 会话管理工具                                  |

---

## 6. Permission (权限) 系统

### 6.1 规则集结构

**Permission.Rule** (`src/permission/index.ts`):

```typescript
{
  permission: string,    // 工具名或权限类别 ("edit", "external_directory")
  pattern: string,       // 通配符匹配模式
  action: "allow" | "deny" | "ask"  // 动作
}
```

### 6.2 权限评估流程

```
权限检查请求
      │
      ↓ 合并规则集
  defaults → agent → session → user-approved
      │
      ↓ 模式匹配
  遍历规则，找首个匹配
      │
      ↓ 动作执行
  ├─ allow → 继续执行
  ├─ deny → 抛出 DeniedError
  └─ ask → 通过 Bus 询问用户
```

---

## 7. Project & Instance (项目与实例)

### 7.1 Project 定义

**Project.Info** (`src/project/project.sql.ts`):

```typescript
{
  id: ProjectID,
  directory: string,       // 项目目录
  worktree?: string,       // Git worktree 根目录
  sandbox?: boolean,       // 是否沙盒模式
  config?: Config,         // 项目配置
}
```

### 7.2 Instance 上下文

Instance 使用 **AsyncLocalStorage** 实现目录级上下文隔离:

**Instance.State** (`src/project/instance.ts`):

```typescript
{
  directory: string,       // 当前工作目录
  worktree: string,        // Git 根目录 (非 git 为 "/")
  project: Project.Info,   // 项目信息
}
```

**用途**:

- 多目录并发操作隔离
- 工具执行时的 cwd 自动绑定
- Provider 状态缓存

---

## 8. 完整交互流程

### 8.1 用户请求 → AI 响应流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                      用户请求完整流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户输入                                                            │
│      │                                                              │
│      ↓                                                              │
│  SessionPrompt.prompt()                                             │
│      │                                                              │
│      ├─→ 创建 User Message + Parts                                  │
│      │                                                              │
│      ↓                                                              │
│  SessionPrompt.loop()                                               │
│      │                                                              │
│      ├─→ 获取 Agent 配置                                            │
│      ├─→ 解析工具 (Registry + Permission)                           │
│      ├─→ 构建系统提示词 (system.ts)                                  │
│      │                                                              │
│      ↓                                                              │
│  LLM.stream()                                                       │
│      │                                                              │
│      ├─→ ProviderTransform.message()                                │
│      ├─→ streamText() (AI SDK)                                      │
│      │                                                              │
│      ↓                                                              │
│  SessionProcessor                                                   │
│      │                                                              │
│      ├─→ 处理流事件 (text, reasoning, tool)                          │
│      ├─→ 工具执行 → Permission 检查                                  │
│      ├─→ 发送 PartDelta 事件 (UI 更新)                               │
│      │                                                              │
│      ↓                                                              │
│  完成检查                                                            │
│      │                                                              │
│      ├─→ finish="stop" → 返回                                       │
│      ├─→ finish="tool-use" → 继续循环                               │
│      ├─→ finish="length" → 触发压缩                                 │
│      │                                                              │
│      ↓                                                              │
│  返回 Assistant Message                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 模型交互界面

**输入层**:

- System Prompt (provider prompt + agent prompt + skills + environment)
- User Messages (转换为 ModelMessage 格式)
- Tool Definitions (filtered by permission + model capabilities)

**输出层**:

- Text Delta (流式文本)
- Reasoning Delta (思维链)
- Tool Calls (函数调用)
- Finish Reason + Usage (token 统计)

---

## 9. 关键设计模式

### 9.1 Effect 服务架构

使用 Effect 框架的服务模式:

- `Layer.effect` 定义服务层
- `makeRuntime` 共享服务依赖注入
- `Effect.gen` 函数式组合
- `Schema.Class` 数据验证

### 9.2 事件溯源

所有数据变更通过 SyncEvent:

- 变更先发送事件
- Projector 处理数据库变更
- 事件版本化，支持回放
- 跨设备同步基础

### 9.3 异步本地存储

使用 AsyncLocalStorage:

- 无需显式传递上下文
- 多目录并发隔离
- 工具自动获取 cwd

### 9.4 级联删除

数据库设计:

- Session → Message → Part 三层级联
- 删除 Session 自动清理所有子数据
- FK 约束保证数据一致性

---

## 10. 目录结构索引

关键源文件路径:

| 模块       | 核心文件                                              |
| ---------- | ----------------------------------------------------- |
| 入口       | `src/index.ts`                                        |
| Agent      | `src/agent/agent.ts`                                  |
| Session    | `src/session/index.ts`, `session.sql.ts`, `schema.ts` |
| 消息       | `src/session/message-v2.ts`, `processor.ts`           |
| Prompt     | `src/session/prompt.ts`, `system.ts`, `llm.ts`        |
| Tool       | `src/tool/tool.ts`, `registry.ts`, `task.ts`          |
| Provider   | `src/provider/provider.ts`, `transform.ts`, `auth.ts` |
| Permission | `src/permission/index.ts`                             |
| Project    | `src/project/project.sql.ts`, `instance.ts`           |
| 存储       | `src/storage/`, `src/sync/`                           |

---

_本文档基于 OpenCode 源码分析生成，旨在提供系统性的架构理解。后续文档将深入各子系统的实现细节。_

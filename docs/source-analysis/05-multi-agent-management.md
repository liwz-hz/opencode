# OpenCode 多 Agent 管理机制详解

本文档深入解读 OpenCode 的多 Agent 管理机制，包括管理 Agent 与被管理 Agent 的关联方式、权限控制、Session 层级关系，以及 oh-my-opencode 的扩展机制。

## 目录

1. [概述](#概述)
2. [Agent 类型系统](#agent-类型系统)
3. [Session 层级关系](#session-层级关系)
4. [Task 工具：Agent 调用的核心](#task-工具agent-调用的核心)
5. [权限控制机制](#权限控制机制)
6. [oh-my-opencode 扩展机制](#oh-my-opencode-扩展机制)
7. [完整调用流程](#完整调用流程)
8. [最佳实践](#最佳实践)

---

## 概述

OpenCode 采用**主从式 Agent 架构**，通过 Task 工具实现 Agent 之间的委派和协作。核心机制包括：

- **Agent 模式区分**：`primary`（主 Agent）与 `subagent`（子 Agent）
- **Session 层级关联**：父子 Session 通过 `parentID` 建立关联
- **权限传递控制**：通过 Permission.Ruleset 限制子 Agent 能力
- **会话恢复机制**：通过 `task_id` 恢复子 Agent 会话上下文

---

## Agent 类型系统

### Agent 模式定义

Agent 通过 `mode` 字段区分角色：

```typescript
// src/agent/agent.ts
type AgentMode = "primary" | "subagent" | "all"

interface AgentInfo {
  name: string
  mode: AgentMode // 角色模式
  permission: Permission.Ruleset // 权限规则
  hidden?: boolean // 是否隐藏（不在 UI 显示）
  prompt?: string // 自定义 prompt
  model?: {
    // 可选的模型配置
    modelID: string
    providerID: string
  }
}
```

### 内置 Agent 类型

| Agent 名称   | 模式     | 职责                       | 权限特点       |
| ------------ | -------- | -------------------------- | -------------- |
| `build`      | primary  | 默认主 Agent，用户直接交互 | 完整权限       |
| `plan`       | primary  | 只读规划 Agent             | 禁止编辑工具   |
| `general`    | subagent | 通用子 Agent               | 禁止 todowrite |
| `explore`    | subagent | 代码探索子 Agent           | 只允许只读工具 |
| `compaction` | hidden   | 自动压缩历史               | 系统 Agent     |
| `title`      | hidden   | 自动生成标题               | 系统 Agent     |
| `summary`    | hidden   | 自动生成摘要               | 系统 Agent     |

### Agent 定义示例

```typescript
// 内置 Agent 定义
const AGENTS: Record<string, AgentInfo> = {
  build: {
    name: "build",
    mode: "primary",
    permission: [
      /* 完整权限 */
    ],
  },
  explore: {
    name: "explore",
    mode: "subagent",
    permission: [
      { permission: "read", pattern: "*", action: "allow" },
      { permission: "write", pattern: "*", action: "deny" },
      { permission: "bash", pattern: "*", action: "deny" },
      // ... 其他工具限制
    ],
  },
}
```

---

## Session 层级关系

### Session 创建与父子关联

Session 是 Agent 执行的上下文单元，通过 `parentID` 建立层级关系：

```typescript
// src/session/index.ts
export function create(input?: {
  parentID?: SessionID // 父会话 ID
  permission?: Permission.Ruleset // 权限规则
}) {
  const session = new Session({
    id: generateID(),
    parentID: input?.parentID,
    permission: input?.permission,
    // ...
  })
  return session
}

// 获取所有子会话
export function children(parentID: SessionID): Session[] {
  return SESSIONS.filter((s) => s.parentID === parentID)
}
```

### Session 层级结构示意

```
用户请求
    │
    ▼
┌─────────────────────┐
│  Primary Session    │  ← 主 Agent (build)
│  SessionID: ses_001 │
│  parentID: null     │
└──────────┬──────────┘
           │ task("explore", ...)
           ▼
    ┌─────────────────────┐
    │  Subagent Session   │  ← 子 Agent (explore)
    │  SessionID: ses_002 │
    │  parentID: ses_001  │  ← 关联父 Session
    └──────────┬──────────┘
               │ task("general", ...)
               ▼
        ┌─────────────────────┐
        │  Nested Session     │  ← 嵌套子 Agent
        │  SessionID: ses_003 │
        │  parentID: ses_002  │
        └─────────────────────┘
```

### Session 恢复机制

通过 `task_id` 恢复之前的子会话，保持上下文连续性：

```typescript
// 恢复子会话
const result = await task({
  session_id: "ses_002", // 恢复之前的子会话
  prompt: "继续分析...",
})
```

---

## Task 工具：Agent 调用的核心

### Task 工具参数

```typescript
interface TaskParams {
  // 必需参数（二选一）
  category?: string // 预定义类别（visual-engineering, ultrabrain, deep, quick 等）
  subagent_type?: string // 直接指定 Agent 类型（explore, librarian, oracle 等）

  // 任务描述
  description: string // 简短描述（3-5 词）
  prompt: string // 完整任务内容

  // 会话控制
  session_id?: string // 恢复现有会话
  run_in_background: boolean // 是否后台运行

  // 技能加载
  load_skills: string[] // 加载的技能列表
}
```

### Task 工具执行流程

```typescript
// src/tool/task.ts 核心逻辑
async function execute(params: TaskParams, ctx: Context) {
  // 1. 确定 Agent
  const agent = params.subagent_type ? Agent.get(params.subagent_type) : Agent.getByCategory(params.category)

  // 2. 创建子 Session
  const session = await Session.create({
    parentID: ctx.sessionID, // 关键：关联父 Session
    title: params.description + ` (@${agent.name} subagent)`,
    permission: agent.permission,
  })

  // 3. 权限控制：禁用未授权工具
  const hasTaskPermission = agent.permission.some((rule) => rule.permission === "task" && rule.action !== "deny")
  const hasTodoWritePermission = agent.permission.some(
    (rule) => rule.permission === "todowrite" && rule.action !== "deny",
  )

  const tools = {
    ...(hasTodoWritePermission ? {} : { todowrite: false }),
    ...(hasTaskPermission ? {} : { task: false }),
  }

  // 4. 调用 LLM 执行子 Agent 任务
  const result = await SessionPrompt.prompt({
    agent,
    sessionID: session.id,
    tools,
    prompt: params.prompt,
  })

  // 5. 返回结果和 task_id
  return {
    content: result.content,
    task_id: session.id, // 供后续恢复使用
  }
}
```

### 关键机制说明

1. **父子关联**：`parentID: ctx.sessionID` 建立层级
2. **权限继承**：子 Agent 使用自己的 `agent.permission`
3. **工具禁用**：根据权限动态禁用工具
4. **会话恢复**：返回 `task_id` 供后续恢复

---

## 权限控制机制

### Permission 规则定义

```typescript
// src/permission/index.ts
type Action = "allow" | "deny" | "ask"

interface PermissionRule {
  permission: string // 工具名称（"read", "write", "bash", "task" 等）
  pattern: string // 匹配模式（支持通配符）
  action: Action // 动作
}

type Ruleset = PermissionRule[]
```

### 权限评估函数

```typescript
export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): PermissionRule {
  // 按顺序检查规则集
  for (const ruleset of rulesets) {
    for (const rule of ruleset) {
      if (matchPattern(permission, rule.pattern)) {
        if (matchPattern(pattern, rule.pattern)) {
          return rule
        }
      }
    }
  }
  // 默认拒绝
  return { permission, pattern: "*", action: "deny" }
}
```

### 权限规则示例

```typescript
// explore Agent 的权限配置
const explorePermission: Ruleset = [
  { permission: "read", pattern: "*", action: "allow" },
  { permission: "glob", pattern: "*", action: "allow" },
  { permission: "grep", pattern: "*", action: "allow" },
  { permission: "lsp_*", pattern: "*", action: "allow" },

  // 禁止写操作
  { permission: "write", pattern: "*", action: "deny" },
  { permission: "edit", pattern: "*", action: "deny" },
  { permission: "bash", pattern: "*", action: "deny" },

  // 禁止委派
  { permission: "task", pattern: "*", action: "deny" },
]
```

### 权限传递流程

```
主 Agent 权限
    │
    ▼
Task 工具调用
    │
    ▼
读取子 Agent 定义的 permission
    │
    ▼
创建子 Session，应用权限规则
    │
    ▼
动态禁用未授权工具
    │
    ▼
子 Agent 执行（受限权限）
```

---

## oh-my-opencode 扩展机制

oh-my-opencode 通过插件系统扩展了多 Agent 管理能力。

### delegate_task 工具

增强版任务委派工具，提供更细粒度的控制：

```typescript
// oh-my-opencode 扩展的 delegate_task
interface DelegateTaskParams {
  // 目标 Agent
  agent_type: string

  // 任务内容
  prompt: string
  description: string

  // 权限控制
  inherit_permission: boolean // 是否继承父 Agent 权限
  custom_permission: Ruleset // 自定义权限规则

  // 资源限制
  max_tokens: number // 最大 token 数
  timeout: number // 超时时间

  // 回调机制
  on_complete?: (result) => void // 完成回调
  on_error?: (error) => void // 错误回调
}
```

### BackgroundManager

后台 Agent 任务管理器，支持：

- **任务队列**：管理多个后台任务
- **状态追踪**：实时追踪任务状态
- **结果收集**：异步收集任务结果
- **取消机制**：支持取消正在运行的任务

```typescript
// BackgroundManager 核心功能
class BackgroundManager {
  // 启动后台任务
  async start(agent: string, prompt: string): Promise<TaskID>

  // 获取任务状态
  status(taskID: TaskID): TaskStatus

  // 收集任务结果
  async collect(taskID: TaskID): Promise<TaskResult>

  // 取消任务
  cancel(taskID: TaskID): void

  // 获取所有后台任务
  list(): TaskInfo[]
}
```

### Ralph Loop 机制

自动继续机制，用于需要多轮交互的任务：

```typescript
// ralph-loop: 自动继续执行
interface RalphLoopConfig {
  // 触发条件
  trigger: "auto" | "on-complete" | "on-error"

  // 继续条件
  should_continue: (result) => boolean

  // 最大迭代次数
  max_iterations: number

  // 回调
  on_iteration: (iteration, result) => void
}
```

---

## 完整调用流程

### 场景：主 Agent 委派代码探索任务

```
┌─────────────────────────────────────────────────────────────┐
│ 用户请求: "分析 src/auth 模块的认证流程"                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 主 Agent (build) 接收请求                                  │
│    - Session: ses_001                                        │
│    - 权限: 完整权限                                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 主 Agent 调用 task 工具                                    │
│    task({                                                    │
│      subagent_type: "explore",                               │
│      description: "分析认证流程",                             │
│      prompt: "搜索 src/auth 目录，分析认证相关代码...",         │
│      run_in_background: true,                                │
│    })                                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Task 工具创建子 Session                                    │
│    - 获取 explore Agent 定义                                  │
│    - 创建 Session: ses_002                                   │
│    - 设置 parentID: ses_001                                  │
│    - 应用 explore 权限规则                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 权限控制生效                                                │
│    - 禁用 write 工具                                          │
│    - 禁用 edit 工具                                           │
│    - 禁用 bash 工具                                           │
│    - 禁用 task 工具（防止嵌套委派）                              │
│    - 保留 read, glob, grep, lsp_* 工具                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. 子 Agent (explore) 执行                                    │
│    - 使用只读工具搜索代码                                      │
│    - 分析认证流程                                              │
│    - 生成分析报告                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. 返回结果                                                   │
│    - 返回分析内容                                              │
│    - 返回 task_id: ses_002（可用于后续恢复）                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. 主 Agent 整合结果                                          │
│    - 接收子 Agent 的分析报告                                   │
│    - 继续后续处理                                              │
└─────────────────────────────────────────────────────────────┘
```

### 关键步骤说明

| 步骤 | 机制          | 说明                            |
| ---- | ------------- | ------------------------------- |
| 2    | Task 工具调用 | 主 Agent 通过 task 工具委派任务 |
| 3    | Session 创建  | 创建子 Session，设置 parentID   |
| 4    | 权限控制      | 根据子 Agent 权限规则禁用工具   |
| 5    | 子 Agent 执行 | 在受限环境中执行任务            |
| 6    | 结果返回      | 返回结果和 task_id              |

---

## 最佳实践

### 1. 选择合适的 Agent 类型

```typescript
// ✅ 正确：使用专门的 Agent
task({ subagent_type: "explore", prompt: "搜索认证相关代码" })
task({ subagent_type: "librarian", prompt: "查找 React 最佳实践文档" })

// ❌ 错误：滥用通用 Agent
task({ subagent_type: "general", prompt: "搜索代码" }) // 应使用 explore
```

### 2. 利用会话恢复减少重复

```typescript
// ✅ 正确：恢复之前的会话
const result1 = await task({
  subagent_type: "explore",
  description: "探索模块",
  prompt: "分析 src/auth",
})
// 返回 task_id: "task_abc"

// 后续需要继续探索时
const result2 = await task({
  session_id: "task_abc", // 恢复之前的会话
  prompt: "继续分析登录流程",
})
```

### 3. 后台任务并行执行

```typescript
// ✅ 正确：并行启动多个后台任务
const task1 = task({
  subagent_type: "explore",
  description: "探索模块A",
  prompt: "分析 src/moduleA",
  run_in_background: true,
})

const task2 = task({
  subagent_type: "explore",
  description: "探索模块B",
  prompt: "分析 src/moduleB",
  run_in_background: true,
})

// 稍后收集结果
const result1 = await background_output({ task_id: task1 })
const result2 = await background_output({ task_id: task2 })
```

### 4. 理解权限限制

```typescript
// explore Agent 权限受限，以下调用会失败：
task({
  subagent_type: "explore",
  prompt: "修改 config.json 文件", // ❌ explore 禁止写入
})

// general Agent 权限受限，以下调用会失败：
task({
  subagent_type: "general",
  prompt: "创建 todo 任务", // ❌ general 禁止 todowrite
})
```

---

## 总结

OpenCode 的多 Agent 管理机制通过以下核心设计实现了灵活的 Agent 协作：

1. **清晰的 Agent 类型系统**：`primary` 与 `subagent` 的区分
2. **Session 层级关联**：通过 `parentID` 建立父子关系
3. **细粒度权限控制**：通过 Permission.Ruleset 限制子 Agent 能力
4. **会话恢复机制**：通过 `task_id` 保持上下文连续性
5. **扩展性**：oh-my-opencode 提供了增强的 delegate_task 和 BackgroundManager

这种设计使得主 Agent 可以高效地委派任务给专门的子 Agent，同时保持安全性和可控性。

---

## 参考文件

- `packages/opencode/src/agent/agent.ts` - Agent 类型定义
- `packages/opencode/src/tool/task.ts` - Task 工具实现
- `packages/opencode/src/session/index.ts` - Session 服务
- `packages/opencode/src/session/prompt.ts` - Prompt 处理
- `packages/opencode/src/permission/index.ts` - 权限系统

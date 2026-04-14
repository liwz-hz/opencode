# OpenCode Provider 模块架构详解

本文档深入分析 OpenCode 的 Provider 模块软件架构，包括暴露接口、核心数据结构、主要流程，结合 Vercel AI SDK 背景知识，彻底讲清楚 Provider 系统的设计与实现。

## 目录

1. [背景知识](#背景知识)
2. [架构概览](#架构概览)
3. [核心数据结构](#核心数据结构)
4. [Provider.Interface 接口详解](#providerinterface-接口详解)
5. [Provider 服务实现](#provider-服务实现)
6. [ProviderTransform 转换层](#providertransform-转换层)
7. [AI SDK 集成模式](#ai-sdk-集成模式)
8. [Provider 初始化流程](#provider-初始化流程)
9. [模型获取与 LanguageModel 创建](#模型获取与-languageModel-创建)
10. [自定义 Provider 扩展](#自定义-provider-扩展)
11. [参考文件](#参考文件)

---

## 背景知识

### Vercel AI SDK

OpenCode 使用 **Vercel AI SDK** 作为 LLM 交互的基础框架。核心概念：

| 概念              | 说明                                                        |
| ----------------- | ----------------------------------------------------------- |
| `LanguageModelV3` | AI SDK 的语言模型接口，定义 `doGenerate` 和 `doStream` 方法 |
| `streamText`      | 流式文本生成函数，接收 `LanguageModelV3` 作为 model 参数    |
| `providerOptions` | Provider 特定选项，以 `{ [providerKey]: options }` 格式传递 |
| `createProvider`  | 创建 Provider SDK，返回 `languageModel` 函数                |

### Provider SDK 结构

每个 AI SDK Provider 包（如 `@ai-sdk/anthropic`）提供：

```typescript
// @ai-sdk/anthropic 示例
import { createAnthropic } from "@ai-sdk/anthropic"

const anthropic = createAnthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

// 创建语言模型
const model = anthropic.languageModel("claude-3-5-sonnet-20241022")

// 使用模型
const result = streamText({
  model,
  prompt: "Hello",
})
```

### Provider ID 与 SDK Key

OpenCode 内部使用 `ProviderID`（品牌标识，如 `anthropic`），而 AI SDK 使用 SDK Key（如 npm 包名映射的 key）：

| ProviderID       | SDK Key      | npm 包                        |
| ---------------- | ------------ | ----------------------------- |
| `anthropic`      | `anthropic`  | `@ai-sdk/anthropic`           |
| `openai`         | `openai`     | `@ai-sdk/openai`              |
| `amazon-bedrock` | `bedrock`    | `@ai-sdk/amazon-bedrock`      |
| `google`         | `google`     | `@ai-sdk/google`              |
| `openrouter`     | `openrouter` | `@openrouter/ai-sdk-provider` |

---

## 架构概览

### Provider 模块三层架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Provider 服务层 (Provider.Service)                         │
│                                                                             │
│  Provider.Service                                                           │
│      │                                                                      │
│      ├─► state: InstanceState<State>                                        │
│      │     ├── providers: Map<ProviderID, Info>                             │
│      │     ├── models: Map<ModelID, Model>                                  │
│      │     └── sdk: Map<ProviderID, SDKModule>                              │
│      │                                                                      │
│      ├─► list(): Effect<Info[]>                                             │
│      ├─► getProvider(id): Effect<Info>                                      │
│      ├─► getModel(providerID, modelID): Effect<Model>                       │
│      ├─► getLanguage(model): Effect<LanguageModelV3>                        │
│      ├─► defaultModel(): Effect<{providerID, modelID}>                      │
│      └─► closest(providerID, query): Effect<Model[]>                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                │ 数据源
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ModelsDev 数据层                                           │
│                                                                             │
│  ModelsDev.get() → 从 models.dev API 或数据库获取 Provider/Model 信息        │
│      │                                                                      │
│      ├─► providers.json → Provider.Info 列表                                │
│      └─► models.json   → Provider.Model 列表                                │
│                                                                             │
│  数据来源：                                                                   │
│      ├── models.dev API (https://models.dev/api.json)                       │
│      ├── 本地缓存数据库                                                       │
│      └── 用户自定义配置                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                │ 转换层
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ProviderTransform 转换层                                   │
│                                                                             │
│  ProviderTransform 命名空间                                                  │
│      │                                                                      │
│      ├─► message(messages): 转换消息格式                                     │
│      ├─► options(model): 生成 Provider 特定选项                              │
│      ├─► temperature(model): 模型默认温度                                    │
│      ├─► topP(model): 模型默认 topP                                         │
│      ├─► topK(model): 模型默认 topK                                         │
│      ├─► maxOutputTokens(model): 输出 token 限制                             │
│      ├─► variants(model, effort): 推理模型变体配置                           │
│      ├─► providerOptions(model, options): 转换为 AI SDK 格式                 │
│      └─► smallOptions(model): 小模型选项                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                │ SDK 加载
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SDK 加载层                                                 │
│                                                                             │
│  BUNDLED_PROVIDERS: 内置 SDK 包列表                                          │
│      │                                                                      │
│      ├── @ai-sdk/anthropic                                                  │
│      ├── @ai-sdk/openai                                                     │
│      ├── @ai-sdk/azure                                                      │
│      ├── @ai-sdk/amazon-bedrock                                             │
│      ├── @ai-sdk/google                                                     │
│      ├── @ai-sdk/google-vertex                                              │
│      ├── @ai-sdk/mistral                                                    │
│      ├── @ai-sdk/groq                                                       │
│      ├── @ai-sdk/xai                                                        │
│      ├── @openrouter/ai-sdk-provider                                        │
│      ├── gitlab-ai-provider                                                 │
│      └── ... (20+ SDK)                                                      │
│                                                                             │
│  自定义 Provider 加载器：                                                     │
│      ├── amazon-bedrock: AWS 凭证配置                                        │
│      ├── gitlab: GitLab OAuth 配置                                           │
│      └── copilot: GitHub Copilot SDK 扩展                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心数据结构

### Provider.Info

Provider 信息结构，描述一个 AI 服务提供商：

```typescript
interface Provider.Info {
  id: ProviderID           // Provider 标识（如 "anthropic", "openai"）
  name: string             // 显示名称（如 "Anthropic", "OpenAI"）
  source: ProviderSource   // 数据来源："env" | "config" | "custom" | "api"
  env: string[]            // 环境变量列表（用于 API Key，如 ["ANTHROPIC_API_KEY"]）
  options: Record<string, unknown>  // Provider 特定选项
  models: Record<ModelID, Provider.Model>  // 可用模型列表
}
```

| 字段      | 说明                                            |
| --------- | ----------------------------------------------- |
| `id`      | Provider 唯一标识，用于路由和 SDK 选择          |
| `name`    | 显示名称，用于 UI 展示                          |
| `source`  | 数据来源，区分内置、配置文件、环境变量、自定义  |
| `env`     | API Key 环境变量名列表，多个 Key 时按优先级排序 |
| `options` | Provider 级配置，如 baseURL、headers 等         |
| `models`  | 该 Provider 下所有可用模型的映射表              |

### Provider.Model

模型信息结构，描述一个具体的 AI 模型：

```typescript
interface Provider.Model {
  id: ModelID              // 模型标识（如 "claude-3-5-sonnet-20241022"）
  providerID: ProviderID   // 所属 Provider
  api: {
    id: string             // API 调用时的模型 ID
    url?: string           // 自定义 API URL
    npm?: string           // SDK npm 包名
  }
  capabilities: {
    temperature: boolean   // 是否支持温度调节
    reasoning: boolean     // 是否支持推理/思考
    attachment: {
      file: boolean        // 是否支持文件附件
      image: boolean       // 是否支持图片附件
      video: boolean       // 是否支持视频附件
    }
    toolcall: boolean      // 是否支持工具调用
    input: number          // 输入 token 限制
    output: number         // 输出 token 限制
  }
  cost: {
    input: number          // 输入 token 价格（每百万）
    output: number         // 输出 token 价格（每百万）
    cache?: {
      read: number         // 缓存读取价格
      write: number        // 缓存写入价格
    }
  }
  limit: {
    context: number        // 上下文长度限制
    output: number         // 输出长度限制
  }
  variants?: Record<string, unknown>  // 推理模型变体配置
}
```

| 字段分组       | 说明                                              |
| -------------- | ------------------------------------------------- |
| `api`          | API 调用配置，包含 SDK 包名和可选的自定义 URL     |
| `capabilities` | 模型能力标志，决定是否启用特定功能                |
| `cost`         | 价格信息，用于成本计算和显示                      |
| `limit`        | Token 限制，用于输入输出截断                      |
| `variants`     | 推理模型变体（如 thinking、reasoning effort）配置 |

### ProviderID 与 ModelID Schema

```typescript
// src/provider/schema.ts
const ProviderID = Schema.String.pipe(Schema.brand("ProviderID"))
const ModelID = Schema.String.pipe(Schema.brand("ModelID"))
```

使用 Schema.brand 创建品牌类型，确保类型安全：

```typescript
// 正确用法
const providerID: ProviderID = "anthropic" as ProviderID
const modelID: ModelID = "claude-3-5-sonnet-20241022" as ModelID

// 错误用法（类型检查会报错）
const wrongID: ProviderID = "unknown-provider" // 未经过 brand 转换
```

---

## Provider.Interface 接口详解

Provider 服务暴露的核心接口：

```typescript
interface Provider.Interface {
  // 获取所有 Provider 列表
  list(): Effect<Record<ProviderID, Provider.Info>>

  // 获取指定 Provider 信息
  getProvider(providerID: ProviderID): Effect<Provider.Info>

  // 获取指定模型
  getModel(providerID: ProviderID, modelID: ModelID): Effect<Provider.Model>

  // 获取 LanguageModelV3（AI SDK 模型）
  getLanguage(model: Provider.Model): Effect<LanguageModelV3>

  // 模糊匹配模型
  closest(providerID: ProviderID, query: string): Effect<Provider.Model[]>

  // 获取小模型（用于 compaction）
  getSmallModel(providerID: ProviderID): Effect<Provider.Model>

  // 获取默认模型
  defaultModel(): Effect<{ providerID: ProviderID; modelID: ModelID }>
}
```

### list()

获取所有可用 Provider：

```typescript
// 使用示例
const providers = await Provider.list()
// 返回: { anthropic: Info, openai: Info, ... }
```

**实现逻辑**：

1. 从 `state.providers` Map 中获取所有条目
2. 按 Provider ID 排序
3. 返回 Record 格式

### getProvider()

获取单个 Provider 信息：

```typescript
// 使用示例
const info = await Provider.getProvider("anthropic")
// 返回: { id: "anthropic", name: "Anthropic", models: {...} }
```

**错误处理**：

- Provider 不存在时抛出 `ProviderError.NotFound`

### getModel()

获取指定 Provider 下的模型：

```typescript
// 使用示例
const model = await Provider.getModel("anthropic", "claude-3-5-sonnet-20241022")
// 返回: { id: "...", capabilities: {...}, cost: {...} }
```

**查找顺序**：

1. 先从 `state.models` Map 查找
2. 未找到则调用 `closest()` 进行模糊匹配
3. 仍未找到则抛出 `ProviderError.ModelNotFound`

### getLanguage()

**核心方法**：将 Provider.Model 转换为 AI SDK 的 LanguageModelV3：

```typescript
// 使用示例
const model = await Provider.getModel("anthropic", "claude-3-5-sonnet")
const languageModel = await Provider.getLanguage(model)

// 在 streamText 中使用
const result = streamText({
  model: languageModel,
  prompt: "Hello",
})
```

**实现流程**：

```
getLanguage(model)
    │
    ├─► resolveSDK(model)      ← 加载 SDK 模块
    │       │
    │       ├── BUNDLED_PROVIDERS[model.api.npm]  ← 内置 SDK
    │       └── import(model.api.npm)             ← 动态加载 npm 包
    │
    ├─► createProviderSDK(providerID, apiKey, options)
    │       │
    │       └── provider.languageModel(model.api.id)
    │
    └─► wrapLanguageModel(model, languageModel)  ← 添加 middleware
            │
            └── 添加 cache middleware、usage tracking 等
```

### closest()

模糊匹配模型名称：

```typescript
// 使用示例
const models = await Provider.closest("anthropic", "claude-sonnet")
// 返回: [Model(claude-3-5-sonnet), Model(claude-sonnet-4), ...]
```

**匹配逻辑**：

- 使用字符串相似度算法
- 支持模型别名和简称
- 返回按匹配度排序的模型列表

### defaultModel()

获取默认模型配置：

```typescript
// 使用示例
const { providerID, modelID } = await Provider.defaultModel()
// 返回: { providerID: "anthropic", modelID: "claude-3-5-sonnet" }
```

**默认规则**：

- 用户配置的默认 Provider/Model 优先
- 未配置则使用系统默认（通常是 anthropic/claude-3-5-sonnet）

---

## Provider 服务实现

### InstanceState 状态管理

Provider 服务使用 Effect 的 `InstanceState` 进行状态管理：

```typescript
// src/provider/provider.ts
class State {
  providers: Map<ProviderID, Provider.Info> = new Map()
  models: Map<ModelID, Provider.Model> = new Map()
  sdk: Map<ProviderID, SDKModule> = new Map()
}

const state = InstanceState.make<State>("provider")
```

**InstanceState 特点**：

- 按项目目录隔离状态
- 自动清理（当项目目录删除时）
- 支持 Effect 服务模式

### Provider.Service 实现

```typescript
const Provider = Effect.gen(function* () {
  const s = yield* state

  // 初始化：从 ModelsDev 加载数据
  const providers = yield* ModelsDev.get()
  for (const provider of providers) {
    s.providers.set(provider.id, provider)
    for (const [modelID, model] of Object.entries(provider.models)) {
      s.models.set(modelID, model)
    }
  }

  // 暴露接口
  return {
    list: () => Effect.succeed(Object.fromEntries(s.providers)),
    getProvider: (id) => Effect.succeed(s.providers.get(id)),
    getModel: (providerID, modelID) => ...,
    getLanguage: (model) => ...,
    ...
  }
})
```

---

## ProviderTransform 转换层

ProviderTransform 是 Provider 配置与 AI SDK 调用之间的转换桥梁。

### OUTPUT_TOKEN_MAX

输出 token 最大限制：

```typescript
// src/provider/transform.ts (line 21)
export const OUTPUT_TOKEN_MAX = Flag.OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX || 32_000
```

- 默认限制：32,000 tokens
- 可通过环境变量 `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX` 覆盖
- 用于防止模型输出过长导致成本过高

### maxOutputTokens()

计算实际输出 token 限制：

```typescript
export function maxOutputTokens(model: Provider.Model): number {
  return Math.min(model.limit.output, OUTPUT_TOKEN_MAX) || OUTPUT_TOKEN_MAX
}
```

取模型限制和全局限制的最小值。

### temperature() - 模型默认温度

不同模型有不同的推荐温度值：

```typescript
export function temperature(model: Provider.Model) {
  const id = model.id.toLowerCase()
  if (id.includes("qwen")) return 0.55
  if (id.includes("claude")) return undefined // Claude 使用默认
  if (id.includes("gemini")) return 1.0
  if (id.includes("glm-4.6")) return 1.0
  if (id.includes("kimi-k2")) {
    if (["thinking", "k2.", "k2p", "k2-5"].some((s) => id.includes(s))) return 1.0
    return 0.6
  }
  return undefined
}
```

| 模型家族    | 默认温度 | 原因                            |
| ----------- | -------- | ------------------------------- |
| Qwen        | 0.55     | 推荐值，平衡创造性和准确性      |
| Claude      | 默认     | Claude 自带优化                 |
| Gemini      | 1.0      | 推荐值                          |
| GLM-4.6/4.7 | 1.0      | 推荐值                          |
| Kimi K2     | 1.0/0.6  | Thinking 版本 1.0，普通版本 0.6 |

### topP() / topK() - 其他采样参数

```typescript
export function topP(model: Provider.Model) {
  const id = model.id.toLowerCase()
  if (id.includes("qwen")) return 1
  if (["minimax-m2", "gemini", "kimi-k2.5"].some((s) => id.includes(s))) return 0.95
  return undefined
}

export function topK(model: Provider.Model) {
  const id = model.id.toLowerCase()
  if (id.includes("minimax-m2")) {
    if (["m2.", "m25", "m21"].some((s) => id.includes(s))) return 40
    return 20
  }
  if (id.includes("gemini")) return 64
  return undefined
}
```

### message() - 消息转换

转换消息以适应不同 Provider 的要求：

```typescript
export function message(model: Provider.Model, messages: Message[]) {
  return messages.map((msg) => {
    // 1. 过滤空内容（Anthropic/Bedrock 不接受空字符串）
    if (isEmptyContent(msg.content) && needsFilter(model)) {
      return { ...msg, content: " " }
    }

    // 2. 清理 tool call ID（Claude/Mistral 特殊字符处理）
    if (needsScrub(model)) {
      msg.content = scrubToolCallIds(msg.content)
    }

    // 3. 应用缓存控制（Anthropic style caching）
    if (supportsCaching(model)) {
      msg.content = applyCacheControl(msg.content)
    }

    // 4. 重映射 providerOptions key
    msg.providerOptions = remapKeys(msg.providerOptions, model.providerID)

    return msg
  })
}
```

**Provider 特定处理**：

| Provider       | 处理内容                                |
| -------------- | --------------------------------------- |
| Anthropic      | 过滤空内容、应用缓存、清理 tool call ID |
| Amazon Bedrock | 过滤空内容、应用缓存                    |
| Mistral        | 清理 tool call ID                       |
| OpenAI         | 无特殊处理                              |

### options() - Provider 特定选项

生成 Provider 级别的特定选项：

```typescript
export function options(input: { model; sessionID; providerOptions }) {
  const model = input.model
  const id = model.providerID

  // OpenAI / GitHub Copilot: 禁用存储
  if (id === "openai" || id === "github-copilot") {
    return { store: false }
  }

  // OpenRouter: 启用 usage 统计
  if (id === "openrouter") {
    return {
      usage: { include: true },
      reasoning: { effort: "medium" }, // for Gemini-3
    }
  }

  // Google: thinking 配置
  if (id === "google" && isReasoningModel(model)) {
    return {
      thinkingConfig: { includeThoughts: true },
    }
  }

  // Anthropic Kimi K2.5: thinking budget
  if (id === "anthropic" && isKimiThinking(model)) {
    return {
      thinking: { type: "enabled", budgetTokens: 8192 },
    }
  }

  // GPT-5: reasoning effort
  if (isGPT5(model)) {
    return {
      reasoningEffort: "medium",
      textVerbosity: "low", // 非 chat 模型
    }
  }

  // ... 其他 Provider 特定配置
}
```

**主要 Provider 特定选项**：

| Provider       | 选项内容                                     |
| -------------- | -------------------------------------------- |
| OpenAI         | `store: false`                               |
| GitHub Copilot | `store: false`                               |
| OpenRouter     | `usage: { include: true }`, reasoning effort |
| Google         | `thinkingConfig` for reasoning models        |
| Anthropic      | `thinking` for Kimi K2.5                     |
| Alibaba-cn     | `enable_thinking: true`                      |
| Venice         | `veniceParameters`                           |

### providerOptions() - AI SDK 格式转换

将选项转换为 AI SDK 接受的格式：

```typescript
export function providerOptions(model: Provider.Model, options: Record<string, any>) {
  const sdkKey = getSDKKey(model.providerID)

  // Gateway: 分离路由选项和上游选项
  if (model.providerID === "gateway") {
    const { gateway, ...upstream } = options
    return {
      gateway: gateway,
      [upstreamProviderKey]: upstream,
    }
  }

  // Azure: 双层选项
  if (model.providerID === "azure") {
    return {
      openai: options,
      azure: options,
    }
  }

  // 其他: 单层映射
  return { [sdkKey]: options }
}
```

**SDK Key 映射表**：

| ProviderID       | SDK Key        |
| ---------------- | -------------- |
| `anthropic`      | `anthropic`    |
| `openai`         | `openai`       |
| `amazon-bedrock` | `bedrock`      |
| `google-vertex`  | `googlevertex` |
| `openrouter`     | `openrouter`   |
| `gitlab`         | `gitlab`       |

### variants() - 推理模型变体

为推理模型生成变体配置：

```typescript
export function variants(model: Provider.Model, effort: "low" | "medium" | "high") {
  if (!model.capabilities.reasoning) return {}

  const id = model.providerID

  // OpenAI o1/o3 系列
  if (id === "openai" && isO1O3(model)) {
    return { reasoningEffort: effort }
  }

  // Anthropic Claude extended thinking
  if (id === "anthropic") {
    return {
      thinking: {
        type: "enabled",
        budgetTokens: getBudgetForEffort(effort), // 1024/4096/16384
      },
    }
  }

  // Google Gemini thinking
  if (id === "google") {
    return {
      thinkingConfig: {
        thinkingLevel: effort, // "minimal" | "low" | "medium" | "high"
      },
    }
  }

  // ... 其他 Provider
}
```

### smallOptions() - 小模型选项

用于 compaction 等轻量操作的选项：

```typescript
export function smallOptions(model: Provider.Model) {
  // OpenAI: 最小推理
  if (model.providerID === "openai") {
    return {
      store: false,
      reasoningEffort: isReasoning(model) ? "minimal" : undefined,
    }
  }

  // Google: 禁用 thinking
  if (model.providerID === "google") {
    return isReasoning(model) ? { thinkingBudget: 0 } : { thinkingConfig: { thinkingLevel: "minimal" } }
  }

  // Venice: 禁用 thinking
  if (model.providerID === "venice") {
    return { veniceParameters: { disableThinking: true } }
  }

  return {}
}
```

---

## AI SDK 集成模式

### streamText 调用集成

Provider 与 AI SDK `streamText` 的集成：

```typescript
// src/session/llm.ts
const params = {
  model: await Provider.getLanguage(input.model),
  messages: ProviderTransform.message(input.model, input.messages),
  temperature: ProviderTransform.temperature(input.model),
  topP: ProviderTransform.topP(input.model),
  maxOutputTokens: ProviderTransform.maxOutputTokens(input.model),
  providerOptions: ProviderTransform.providerOptions(input.model, options),
  tools: buildTools(input),
}

const result = streamText(params)
```

### wrapLanguageModel 中间件

为 LanguageModelV3 添加中间件：

```typescript
// src/provider/provider.ts
function wrapLanguageModel(model: Provider.Model, languageModel: LanguageModelV3) {
  return wrapLanguageModel({
    model: languageModel,
    middleware: {
      // 缓存中间件
      transformParams: async ({ params }) => {
        if (supportsCaching(model)) {
          params.messages = applyCacheControl(params.messages)
        }
        return params
      },

      // Usage tracking 中间件
      onChunk: async ({ chunk }) => {
        if (chunk.type === "finish") {
          trackUsage(model, chunk.usage)
        }
      },
    },
  })
}
```

---

## Provider 初始化流程

### 启动时初始化

```
项目启动
    │
    ├─► Provider.Service 初始化
    │       │
    │       ├─► InstanceState.make<State>()
    │       │
    │       ├─► ModelsDev.get()  ← 获取 Provider/Model 数据
    │       │       │
    │       │       ├── 检查本地缓存数据库
    │       │       ├── 缓存过期 → 调用 models.dev API
    │       │       └── 解析 JSON 数据
    │       │
    │       ├─► populateState(providers)
    │       │       │
    │       │       ├── for (provider of providers)
    │       │       │       ├── state.providers.set(id, info)
    │       │       │       └── for (model of provider.models)
    │       │       │               state.models.set(modelID, model)
    │       │
    │       └─► 加载环境变量配置
    │               │
    │               ├── 检查 API Key 环境变量
    │               ├── 配置 Provider 选项
    │               └── 合并用户自定义 Provider
    │
    └─► Provider 服务就绪
```

### ModelsDev 数据源

```typescript
// src/provider/models.ts
const ModelsDev = {
  get: async () => {
    // 1. 检查缓存
    const cached = await db.query("SELECT * FROM providers")
    if (cached && !isExpired(cached)) {
      return cached
    }

    // 2. 获取新数据
    const response = await fetch("https://models.dev/api.json")
    const data = await response.json()

    // 3. 存储缓存
    await db.insert("providers", data)

    return data.providers
  },
}
```

---

## 模型获取与 LanguageModel 创建

### 完整流程

```
用户请求模型 "claude-3-5-sonnet"
    │
    ├─► Provider.getModel("anthropic", "claude-3-5-sonnet")
    │       │
    │       ├── state.models.get(modelID)
    │       │       │
    │       │       └── 找到 → 返回 Model 对象
    │       │       └
    │       │       └── 未找到 → closest(providerID, modelID)
    │       │               │
    │       │               └── 模糊匹配 → 返回最相似 Model
    │       │               └
    │       │               └── 抛出 ProviderError.ModelNotFound
    │       │
    │       └─► 返回 Provider.Model
    │
    ├─► Provider.getLanguage(model)
    │       │
    │       ├─► resolveSDK(model)
    │       │       │
    │       │       ├── 检查 state.sdk 缓存
    │       │       │       │
    │       │       │       └── 已缓存 → 返回 SDK 模块
    │       │       │       │
    │       │       │       └── 未缓存 → 加载 SDK
    │       │       │               │
    │       │       │               ├── BUNDLED_PROVIDERS[model.api.npm]
    │       │       │               │       │
    │       │       │               │       └── 内置 SDK → 返回模块
    │       │       │               │       │
    │       │       │               │       └── 未内置 → 动态 import
    │       │       │               │               │
    │       │       │               │               └── import(model.api.npm)
    │       │       │               │               │
    │       │       │               │               └── 返回 SDK 模块
    │       │       │               │
    │       │       │               └── 存入 state.sdk 缓存
    │       │       │
    │       │       └─► 返回 SDK 模块
    │       │
    │       ├─► createProvider(sdk, apiKey, options)
    │       │       │
    │       │       ├── 获取 API Key（从环境变量）
    │       │       │       │
    │       │       │       └── process.env[model.env[0]]
    │       │       │
    │       │       ├── 获取 Provider 选项
    │       │       │       │
    │       │       │       └── model.options (baseURL, headers 等)
    │       │       │
    │       │       └─► sdk.createProvider({ apiKey, ...options })
    │       │
    │       ├─► provider.languageModel(model.api.id)
    │       │       │
    │       │       └─► 返回 LanguageModelV3
    │       │
    │       ├─► wrapLanguageModel(model, languageModel)
    │       │       │
    │       │       └─► 添加 cache/usage 中间件
    │       │
    │       └─► 返回 LanguageModelV3
    │
    └─► streamText({ model: LanguageModelV3, ... })
```

### SDK 加载优先级

```
resolveSDK(model)
    │
    ├── 优先级 1: state.sdk 缓存
    │       └── 已缓存 → 直接返回
    │
    ├── 优先级 2: BUNDLED_PROVIDERS 内置
    │       └── 检查 model.api.npm 是否在内置列表
    │       └── 是 → 返回内置模块
    │
    ├── 优先级 3: 动态 import
    │       └── import(model.api.npm)
    │       └── 返回动态加载模块
    │
    └─► 存入缓存 → 返回 SDK 模块
```

---

## 自定义 Provider 扩展

### 内置 BUNDLED_PROVIDERS

```typescript
// src/provider/provider.ts
const BUNDLED_PROVIDERS = {
  "@ai-sdk/anthropic": () => import("@ai-sdk/anthropic"),
  "@ai-sdk/openai": () => import("@ai-sdk/openai"),
  "@ai-sdk/azure": () => import("@ai-sdk/azure"),
  "@ai-sdk/amazon-bedrock": () => import("@ai-sdk/amazon-bedrock"),
  "@ai-sdk/google": () => import("@ai-sdk/google"),
  "@ai-sdk/google-vertex": () => import("@ai-sdk/google-vertex"),
  "@ai-sdk/mistral": () => import("@ai-sdk/mistral"),
  "@ai-sdk/groq": () => import("@ai-sdk/groq"),
  "@ai-sdk/xai": () => import("@ai-sdk/xai"),
  "@openrouter/ai-sdk-provider": () => import("@openrouter/ai-sdk-provider"),
  "gitlab-ai-provider": () => import("gitlab-ai-provider"),
  // ... 20+ 内置 SDK
}
```

### Custom Loader 扩展

部分 Provider 需要特殊的初始化逻辑：

```typescript
// amazon-bedrock custom loader
const amazonBedrockLoader = {
  getModel: (model) => {
    // AWS 凭证配置
    const credentials = {
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
      region: process.env.AWS_REGION || "us-east-1",
    }
    return createBedrockProvider({ credentials }).languageModel(model.api.id)
  },

  vars: () => ["AWS_ACCESS_KEY_ID", "AWS_SECRET_ACCESS_KEY", "AWS_REGION"],
  options: () => {},
}

// gitlab custom loader
const gitlabLoader = {
  getModel: (model) => {
    // GitLab OAuth 配置
    const token = process.env.GITLAB_TOKEN
    return createGitLabProvider({ token }).languageModel(model.api.id)
  },

  vars: () => ["GITLAB_TOKEN"],
  options: () => {},
}
```

### GitHub Copilot SDK 扩展

```typescript
// src/provider/sdk/copilot/
const copilotLoader = {
  getModel: async (model) => {
    // 获取 GitHub Copilot token
    const token = await getCopilotToken()

    // 创建 Copilot Provider
    return createCopilotProvider({
      token,
      baseURL: "https://api.githubcopilot.com"
    }).languageModel(model.api.id)
  },

  discoverModels: async () => {
    // 动态发现 Copilot 可用模型
    const models = await fetchCopilotModels()
    return models.map(m => ({
      id: m.id,
      providerID: "github-copilot",
      capabilities: { ... },
      ...
    }))
  }
}
```

---

## 参考文件

| 文件路径                                            | 说明                                |
| --------------------------------------------------- | ----------------------------------- |
| `packages/opencode/src/provider/provider.ts`        | Provider 服务核心实现（1733 行）    |
| `packages/opencode/src/provider/schema.ts`          | ProviderID/ModelID Schema 定义      |
| `packages/opencode/src/provider/transform.ts`       | ProviderTransform 转换层（1051 行） |
| `packages/opencode/src/provider/models.ts`          | ModelsDev 数据源                    |
| `packages/opencode/src/provider/error.ts`           | Provider 错误定义                   |
| `packages/opencode/src/provider/sdk/copilot/`       | GitHub Copilot SDK 扩展             |
| `packages/opencode/src/session/llm.ts`              | LLM 流式调用实现                    |
| `packages/opencode/test/provider/transform.test.ts` | ProviderTransform 测试套件          |

---

## 总结

Provider 模块是 OpenCode 与各种 AI 服务提供商交互的核心：

1. **数据层**：ModelsDev 提供统一的 Provider/Model 数据来源
2. **服务层**：Provider.Service 提供 `getLanguage()` 等核心接口
3. **转换层**：ProviderTransform 处理 Provider 特定的参数和消息转换
4. **SDK 层**：BUNDLED_PROVIDERS + Custom Loader 支持多种 AI SDK
5. **AI SDK 集成**：通过 `LanguageModelV3` 与 `streamText` 无缝对接

这种分层设计使得 OpenCode 能够：

- 支持数十种 AI Provider（Anthropic、OpenAI、Google、Bedrock 等）
- 统一处理不同 Provider 的参数差异
- 轻松扩展新的 Provider 支持
- 在 AI SDK 基础上添加缓存、usage tracking 等中间件

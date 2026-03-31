# OpenCode 工程模块架构

本文档从整体到部分，系统性介绍 OpenCode 工程各模块的功能定位、技术架构和相互关系。

---

## 1. Monorepo 整体架构

OpenCode 是一个基于 **Bun** 的 TypeScript monorepo 项目，使用 **Turborepo** 进行构建编排，**SST** 管理云基础设施。

### 1.1 包管理结构

```
opencode/                      # 工程根目录
├── package.json               # Monorepo 根配置
├── turbo.json                 # Turborepo 构建编排
├── bunfig.toml                # Bun 运行时配置
├── sst.config.ts              # SST 云基础设施配置
├── tsconfig.json              # TypeScript 基础配置
├── patches/                   # 依赖补丁
├── infra/                     # 基础设施代码
├── script/                    # 构建脚本
├── specs/                     # 规格文档
├── sdks/                      # SDK (VSCode 扩展)
└── packages/                  # 所有子包
    ├── opencode/              # 核心 CLI/TUI 应用
    ├── app/                   # Web 前端应用
    ├── ui/                    # 共享 UI 组件库
    ├── desktop/               # Tauri 桌面应用
    ├── desktop-electron/      # Electron 桌面应用
    ├── console/               # Console 平台
    ├── sdk/                   # JavaScript SDK
    ├── web/                   # 官方文档站点
    ├── plugin/                # 插件系统定义
    ├── util/                  # 共享工具库
    ├── script/                # 构建脚本工具
    ├── function/              # Cloudflare 云函数
    ├── enterprise/            # 企业版 Web 应用
    ├── identity/              # 身份认证
    ├── slack/                 # Slack 集成
    ├── storybook/             # UI 组件展示
    ├── extensions/            # 扩展
    ├── containers/            # Docker 容器配置
    └── docs/                  # 内部文档
```

### 1.2 Workspaces 定义

根 `package.json` 采用 Catalog 模式统一管理依赖版本：

```json
{
  "workspaces": ["packages/*", "packages/console/*", "packages/sdk/js", "packages/slack"]
}
```

**Catalog 依赖**: 42+ 共享依赖统一版本管理，包括 effect、typescript、solid-js、vite、hono、drizzle-orm 等。

### 1.3 构建系统

**根命令**:

| 命令              | 用途                            |
| ----------------- | ------------------------------- |
| `bun install`     | 安装依赖                        |
| `bun typecheck`   | Turborepo 并行类型检查          |
| `bun dev`         | 启动 opencode 核心开发服务器    |
| `bun dev:web`     | 启动 Web 前端开发服务器         |
| `bun dev:desktop` | 启动 Tauri 桌面应用             |
| `bun dev:console` | 启动 Console 应用               |
| `bun test`        | **已禁用** — 需在具体包目录运行 |

**Turbo Tasks** (`turbo.json`):

| Task                    | 依赖     | 输出      |
| ----------------------- | -------- | --------- |
| `typecheck`             | 无       | 无        |
| `build`                 | 无       | `dist/**` |
| `opencode#test`         | `^build` | —         |
| `@opencode-ai/app#test` | `^build` | —         |

### 1.4 云基础设施 (SST)

- **云环境**: Cloudflare
- **数据库**: PlanetScale
- **支付**: Stripe
- **基础设施模块**: `infra/app.ts`, `infra/console.ts`, `infra/enterprise.ts`

---

## 2. 核心包: packages/opencode

详见第一篇文档 `01-system-introduction.md`。此包是 OpenCode 的核心 CLI/TUI 应用，提供：

- Agent 系统与会话管理
- 工具系统与权限控制
- AI 提供商抽象层
- HTTP 服务端与 SSE 事件流
- SQLite 数据持久化

---

## 3. 前端与桌面应用层

### 3.1 packages/app — Web 前端应用

**是什么**: OpenCode 的 Web 界面客户端，提供浏览器访问能力。

**技术栈**:

| 技术                  | 版本   | 用途         |
| --------------------- | ------ | ------------ |
| SolidJS               | 1.9.10 | UI 框架      |
| Vite                  | 7.1.4  | 构建工具     |
| Tailwind CSS          | 4.0    | 样式系统     |
| @kobalte/core         | 0.9.0  | UI 组件基础  |
| @tanstack/solid-query | 5.0    | 数据请求管理 |
| Shiki                 | 1.26   | 代码高亮     |

**核心架构**:

```
packages/app/src/
├── app.tsx                # 根组件，Provider 嵌套
├── context/
│   ├── sdk.tsx            # SDK 客户端初始化
│   ├── session.tsx        # 会话状态管理
│   ├── file-tree.tsx      # 文件树状态
│   ├── review.tsx         # Review 面板状态
│   └── settings.tsx       # 设置状态
├── component/
│   ├── session/           # 会话 UI (消息、工具调用、diff)
│   ├── file-tree/         # 文件树 UI
│   ├── review/            # Review 面板 UI
│   ├── terminal/          # 终端 UI (xterm.js)
│   ├── settings/          # 设置面板 UI
│   └── layout/            # 布局组件
└── route/                 # 路由定义
```

**与核心交互**: 通过 `@opencode-ai/sdk` 的 HTTP API 和 SSE 事件流：

```typescript
// src/context/sdk.tsx
const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  directory: projectDir,
})
```

**主要功能**:

- Session 管理 (创建、消息、fork、share)
- 文件树导航与编辑
- Review 面板 (代码变更审查)
- 终端集成 (PTY 会话)
- 设置面板 (配置、提供商、模型)
- 多语言支持 (17 种语言)

---

### 3.2 packages/ui — 共享 UI 组件库

**是什么**: 跨前端应用共享的 UI 组件库，支持 OpenCode 特有的功能需求。

**技术栈**: SolidJS + @kobalte/core + Tailwind CSS + Shiki + KaTeX + Marked

**导出内容**:

| 类型       | 数量  | 举例                                             |
| ---------- | ----- | ------------------------------------------------ |
| Components | 75+   | Button, Dialog, Toast, Tabs, Dropdown, CodeBlock |
| Themes     | 35+   | light, dark, catppuccin, tokyo-night, gruvbox    |
| Languages  | 17 种 | en, zh, ja, ko, de, fr, es, ar 等                |
| Hooks      | 多个  | useIntersectionObserver, useDebounce             |
| Context    | 多个  | ThemeContext, LocaleContext                      |
| Pierre     | —     | Diff 渲染器 (代码变更可视化)                     |

**被依赖关系**:

| 包                     | 用途             |
| ---------------------- | ---------------- |
| `packages/app`         | Web 前端核心组件 |
| `packages/console/app` | Console 管理平台 |
| `packages/desktop`     | 桌面应用前端     |
| `packages/storybook`   | 组件文档展示     |
| `packages/enterprise`  | 企业版 Web 应用  |

**目录结构**:

```
packages/ui/src/
├── component/             # UI 组件
├── context/               # Context providers
├── hook/                  # 自定义 hooks
├── theme/                 # 主题定义
├── locale/                # 国际化翻译
├── pierre/                # Diff 渲染引擎
└── index.ts               # 导出入口
```

---

### 3.3 packages/desktop — Tauri 桌面应用

**是什么**: 基于 Tauri v2 的跨平台桌面应用，提供原生桌面体验。

**技术栈**:

| 层   | 技术                                        |
| ---- | ------------------------------------------- |
| 前端 | SolidJS + Vite + Tailwind                   |
| 后端 | Tauri v2 + Rust                             |
| 通信 | tauri-specta (自动生成 TypeScript bindings) |

**架构模式: Sidecar**

Desktop 应用不直接包含 OpenCode 逻辑，而是启动 `opencode-cli serve` 作为 HTTP 服务端：

```
┌─────────────────┐     HTTP/SSE      ┌─────────────────┐
│  Tauri Desktop  │ ────────────────→ │  opencode-cli   │
│  (SolidJS UI)   │                   │  (serve mode)   │
└─────────────────┘                   └─────────────────┘
         │                                    │
         │ tauri-specta                       │ SQLite
         │ bindings                           │
         ↓                                    ↓
    Rust Commands                        Core Logic
```

**Rust 核心模块** (`src-tauri/src/`):

| 文件          | 功能                       |
| ------------- | -------------------------- |
| `cli.rs`      | Sidecar 启动/停止/状态管理 |
| `deeplink.rs` | 深链接处理 (`opencode://`) |
| `update.rs`   | 自动更新逻辑               |
| `wsl.rs`      | Windows WSL 支持           |
| `tray.rs`     | 系统托盘                   |

**关键代码: Sidecar 启动**:

```rust
// src-tauri/src/cli.rs
pub async fn start_sidecar(app: AppHandle, state: State<'_, CliState>) -> Result<SidecarProcess> {
    let sidecar = app.shell().sidecar("opencode-cli")?;
    let (mut rx, _child) = sidecar
        .args(&["serve", "--port", &port.to_string()])
        .spawn()?;
    // ...
}
```

**TypeScript Bindings**: 通过 tauri-specta 自动生成到 `src/bindings.ts`:

```typescript
// 自动生成的命令调用
await commands.startSidecar()
await commands.stopSidecar()
await commands.checkUpdate()
```

---

### 3.4 packages/desktop-electron — Electron 桌面应用

**是什么**: 基于 Electron 的桌面应用实现（备选方案）。

**技术栈**: Electron + SolidJS + Vite

**状态**: 作为 Tauri 版本的替代方案存在，功能类似。

---

## 4. Console 管理平台

### 4.1 packages/console/app — Console Web 应用

**是什么**: OpenCode 的官方云端管理平台，提供 Workspace 管理、API Key 管理、计费等功能。

**技术栈**:

| 技术               | 用途         |
| ------------------ | ------------ |
| SolidStart         | SSR 框架     |
| Nitro              | 服务端运行时 |
| Cloudflare Workers | 部署平台     |
| Drizzle ORM        | 数据库操作   |
| Stripe             | 计费系统     |
| Resend             | 邮件发送     |

**核心路由**:

```
packages/console/app/src/routes/
├── index.tsx              # Landing Page
├── login.tsx              # 登录页
├── workspace/[id]/        # Workspace 管理区域
│   ├── index.tsx          # Workspace 首页
│   ├── zen.tsx            # Zen API 配置
│   ├── go.tsx             # Go 模型配置
│   ├── usage.tsx          # 使用量统计
│   ├── keys.tsx           # API Key 管理
│   ├── members.tsx        # 成员管理
│   ├── billing.tsx        # Stripe 计费
│   └── settings.tsx       # Workspace 设置
└── api/                   # API 端点
```

**主要功能**:

- **Landing Page**: 产品介绍、定价展示
- **Workspace 管理**: 创建、邀请成员、权限控制
- **Zen API**: OpenCode Zen API 代理配置
- **Go 模型**: 模型参数调优
- **Usage**: Token 使用量统计与可视化
- **API Keys**: 创建、管理 API Key
- **Billing**: Stripe 订阅与支付
- **认证**: OAuth (GitHub/Google)、邮箱验证

**与核心关系**: Console 是独立的云端产品，生成的 API Key 供 CLI/Desktop 使用连接 Zen API。

---

### 4.2 packages/console/core — Console 核心业务逻辑

**是什么**: Console 应用共享的业务逻辑层。

**导出内容**:

- Workspace 数据模型
- API Key 生成逻辑
- 计费计算
- 权限检查

---

### 4.3 packages/console/mail — 邮件服务

**是什么**: Console 邮件发送服务。

**技术栈**: Resend API

**功能**: 验证邮件、邀请邮件、通知邮件。

---

## 5. SDK 层

### 5.1 packages/sdk/js — JavaScript SDK

**是什么**: OpenCode 的 JavaScript/TypeScript SDK，提供与 OpenCode 服务端交互的完整 API。

**技术栈**: TypeScript + @hey-api/openapi-ts (自动生成)

**导出结构**:

| Export            | 用途                                                  |
| ----------------- | ----------------------------------------------------- |
| `.`               | 组合入口：`createOpencode()` 同时启动 server + client |
| `./client`        | 纯客户端：`createOpencodeClient()`                    |
| `./server`        | 纯服务端：`createOpencodeServer()`                    |
| `./v2`            | v2 版本（支持 workspace）                             |
| `./v2/gen/client` | 自动生成的原始客户端代码                              |

**客户端 API 模块**:

| 模块                | 功能                                     |
| ------------------- | ---------------------------------------- |
| `session`           | 会话管理：创建、消息、fork、abort、share |
| `project`           | 项目信息：当前项目、git 初始化           |
| `pty`               | PTY 终端会话                             |
| `config`            | 配置管理                                 |
| `tool`              | 工具管理                                 |
| `mcp`               | MCP 服务器状态                           |
| `lsp` / `formatter` | LSP/格式化器状态                         |
| `tui`               | TUI 控制                                 |
| `file`              | 文件操作                                 |
| `find`              | 搜索功能                                 |
| `provider`          | AI 提供商管理                            |
| `auth`              | 认证设置                                 |
| `event`             | SSE 事件订阅                             |
| `experimental`      | 实验性 API (workspace)                   |

**构建流程**:

```
script/build.ts:
1. bun dev generate → 从 packages/opencode 生成 openapi.json
2. @hey-api/openapi-ts → 生成 TypeScript 客户端到 src/v2/gen/
3. prettier → 格式化
4. tsc → 编译到 dist/
```

**使用示例**:

```typescript
import { createOpencode } from "@opencode-ai/sdk"

// 一行启动服务端 + 创建客户端
const { client, server } = await createOpencode()

// 使用客户端
await client.session.create({ title: "My Session" })
await client.session.prompt({ sessionID, parts: [{ type: "text", text: "Hello" }] })

server.close()
```

---

## 6. 文档与网站

### 6.1 packages/web — 官方文档站点

**是什么**: OpenCode 的官方网站和文档站点，托管在 `opencode.ai`。

**技术栈**:

| 技术       | 用途           |
| ---------- | -------------- |
| Astro 5.7  | 静态站点生成   |
| Starlight  | Astro 文档主题 |
| SolidJS    | 交互组件       |
| MDX        | 文档内容格式   |
| Cloudflare | 部署平台       |
| Shiki      | 代码高亮       |

**主要功能**:

1. **文档站点** (`/docs`):
   - 40+ MDX 文档文件
   - 17 种语言国际化
   - 侧边栏导航、搜索、面包屑

2. **首页着陆页** (`/`):
   - 安装命令（一键复制）
   - 功能特性列表
   - 产品截图展示
   - 多种安装方式介绍

3. **会话分享** (`/s/[id]`):
   - WebSocket 实时同步
   - SolidJS 组件渲染对话
   - 显示成本、token 统计

**目录结构**:

```
packages/web/src/
├── content/
│   ├── docs/              # 文档 MDX 文件
│   │   ├── config.mdx     # 配置指南
│   │   ├── tools.mdx      # 工具文档
│   │   ├── agents.mdx     # Agent 文档
│   │   └── ...            # 40+ 其他文档
│   └── i18n/              # 国际化 JSON (17 种语言)
├── components/
│   ├── Lander.astro       # 首页着陆页
│   ├── Share.tsx          # 分享功能组件
│   └── share/             # 分享子组件
├── pages/
│   └── s/[id].astro       # 分享页面路由
└── styles/
    └── custom.css         # 自定义样式
```

---

## 7. 辅助工具包

### 7.1 packages/util — 共享工具库

**是什么**: 跨包共享的基础工具函数集合。

**模块列表**:

| 文件            | 功能                                          |
| --------------- | --------------------------------------------- |
| `fn.ts`         | 带 Zod schema 验证的函数包装器                |
| `error.ts`      | `NamedError` 类，创建带类型的命名错误         |
| `path.ts`       | 文件路径处理 (getFilename, truncateMiddle 等) |
| `lazy.ts`       | 惰性初始化                                    |
| `retry.ts`      | 重试逻辑                                      |
| `encode.ts`     | 编码工具                                      |
| `identifier.ts` | 标识符生成                                    |
| `array.ts`      | 数组工具                                      |
| `slug.ts`       | Slug 生成                                     |

**被依赖**: `opencode`, `app`, `ui`, `enterprise`

---

### 7.2 packages/plugin — 插件系统定义

**是什么**: OpenCode 的插件扩展接口定义，支持服务端插件和 TUI 插件。

**核心导出**:

**服务端插件 Hooks** (`Plugin` 类型):

| Hook                          | 用途                     |
| ----------------------------- | ------------------------ |
| `event`                       | 事件监听                 |
| `config`                      | 配置处理                 |
| `tool`                        | 自定义工具注册           |
| `auth`                        | 认证钩子 (OAuth/API key) |
| `chat.message/params/headers` | 对话扩展                 |
| `permission.ask`              | 权限拦截                 |
| `shell.env`                   | 环境变量注入             |
| `tool.execute.before/after`   | 工具执行拦截             |

**TUI 插件** (`TuiPlugin` 类型):

- 命令注册
- 路由扩展
- 对话框扩展
- 主题扩展
- 事件订阅

**工具定义辅助**:

```typescript
import { tool } from "@opencode-ai/plugin"

const myTool = tool({
  name: "my_tool",
  description: "...",
  parameters: z.object({ ... }),
  execute: async (params) => { ... }
})
```

**被依赖**: `opencode` 核心

---

### 7.3 packages/script — 构建/发布脚本工具

**是什么**: 版本管理、渠道检测、发布配置的脚本工具。

**导出内容**:

```typescript
Script = {
  channel, // 'latest' | 分支名(预览版)
  version, // 自动计算版本号
  preview, // 是否预览版
  release, // 是否正式发布
  team, // 团队成员列表
}
```

**使用场景**: CI/CD 流程中确定版本号、渠道、团队成员。

---

### 7.4 packages/function — Cloudflare 云函数

**是什么**: 部署在 Cloudflare Workers 上的边缘函数，提供云端服务。

**技术栈**: Hono + Cloudflare Durable Objects + R2

**API 端点**:

| 端点                           | 功能                             |
| ------------------------------ | -------------------------------- |
| `/share_create`                | 创建会话分享                     |
| `/share_sync`                  | 同步分享数据                     |
| `/share_poll`                  | WebSocket 订阅更新               |
| `/share_data`                  | 获取分享数据                     |
| `/share_delete`                | 删除分享                         |
| `/exchange_github_app_token`   | GitHub OIDC → Installation Token |
| `/get_github_app_installation` | GitHub App 安装状态              |
| `/feishu`                      | 飞书机器人 webhook → Discord     |

**部署**: 独立部署的云服务，不作为内部依赖使用。

---

### 7.5 packages/enterprise — 企业版 Web 应用

**是什么**: 基于 SolidStart 的独立企业版 Web 应用。

**技术栈**: SolidStart + Hono + Tailwind + OpenAPI

**核心模块**:

- `src/routes/api/[...path].ts` — OpenAPI 文档化的 API 端点
- `src/core/share.ts` — 分享会话业务逻辑
- `src/core/storage.ts` — S3/R2 存储抽象层

**部署**: 独立部署，用于企业级自托管实例。

---

## 8. 其他辅助包

### 8.1 packages/identity — 身份认证

**是什么**: 身份认证相关功能模块。

**功能**: OAuth 流程、JWT 处理、会话管理。

---

### 8.2 packages/slack — Slack 集成

**是什么**: Slack Bot 集成模块。

**功能**: Slack 消息处理、命令响应。

---

### 8.3 packages/storybook — UI 组件展示

**是什么**: packages/ui 的组件文档和展示工具。

**技术栈**: Storybook for SolidJS

**功能**: 可视化浏览和测试 UI 组件。

---

### 8.4 packages/extensions — 扩展

**是什么**: 扩展功能模块。

**功能**: VSCode 扩展、浏览器扩展等。

---

### 8.5 packages/containers — Docker 容器配置

**是什么**: Docker 容器构建配置。

**功能**: OpenCode 的 Docker 化部署配置。

---

### 8.6 packages/docs — 内部文档

**是什么**: 内部开发文档。

**功能**: 架构文档、API 文档、开发指南。

---

## 9. 模块关系总览

### 9.1 依赖关系图

```
                    packages/opencode (核心 CLI/TUI)
                            │
                            │ HTTP API + SSE
                            │
                            ↓
    ┌───────────────────────┼───────────────────────┐
    │                       │                       │
    ↓                       ↓                       ↓
packages/sdk/js      packages/app          packages/desktop
(JavaScript SDK)     (Web 前端)            (Tauri 桌面)
    │                       │                       │
    │                       │                       │
    │                       ↓                       │
    │               packages/ui (共享 UI 组件)       │
    │                       │                       │
    │                       │                       │
    ↓                       ↓                       ↓
┌─────────────────────────────────────────────────────────┐
│                     packages/util                        │
│                    (共享工具函数)                         │
└─────────────────────────────────────────────────────────┘
                            │
                            │
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   packages/plugin                        │
│                   (插件系统定义)                          │
└─────────────────────────────────────────────────────────┘
```

### 9.2 独立部署模块

| 模块                   | 部署方式           | 用途         |
| ---------------------- | ------------------ | ------------ |
| `packages/console/app` | Cloudflare Workers | 云端管理平台 |
| `packages/function`    | Cloudflare Workers | 云函数服务   |
| `packages/enterprise`  | SolidStart SSR     | 企业自托管   |
| `packages/web`         | Cloudflare Pages   | 官方文档站点 |

### 9.3 数据流关系

**CLI → SDK → Web/Desktop**:

- CLI 启动 HTTP 服务端 (`opencode serve`)
- SDK 创建客户端连接 CLI
- Web/Desktop 通过 SDK 调用 CLI API

**Console → Zen API → CLI**:

- Console 管理 API Key
- CLI 使用 API Key 连接 Zen API
- Zen API 代理请求到 AI 提供商

**Web → Function → 分享功能**:

- CLI 发送分享请求到 Function
- Function 存储分享数据到 R2
- Web 页面通过 WebSocket 获取分享内容

---

## 10. 开发工作流

### 10.1 本地开发

**核心 + Web 前端**:

```bash
# 终端 1: 后端
bun run --cwd packages/opencode --conditions=browser ./src/index.ts serve --port 4096

# 终端 2: 前端
bun dev --cwd packages/app -- --port 4444

# 访问 http://localhost:4444
```

**桌面应用**:

```bash
bun run --cwd packages/desktop tauri dev
```

### 10.2 构建与发布

```bash
# 类型检查
bun typecheck

# 构建 CLI
bun run --cwd packages/opencode build

# 构建 SDK
bun run --cwd packages/sdk/js build

# 发布流程由 script 包管理
```

---

## 11. 总结

OpenCode 采用清晰的模块化架构：

| 层级         | 模块                                       | 职责                         |
| ------------ | ------------------------------------------ | ---------------------------- |
| **核心层**   | `opencode`                                 | CLI/TUI、服务端逻辑、AI 交互 |
| **SDK 层**   | `sdk/js`                                   | HTTP 客户端封装              |
| **前端层**   | `app`, `desktop`                           | 用户界面                     |
| **UI 组件**  | `ui`                                       | 共享组件库                   |
| **工具层**   | `util`, `plugin`, `script`                 | 基础工具、插件定义、构建脚本 |
| **云服务层** | `console`, `function`, `enterprise`, `web` | 云端部署的独立服务           |

各模块职责明确，依赖关系清晰，支持多种部署和访问方式（CLI、Web、桌面、云端 API）。

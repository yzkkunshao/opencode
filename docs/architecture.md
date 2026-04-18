# OpenCode 架构文档

## 1. 项目概述

OpenCode 是一个开源 AI 编码代理（Coding Agent），提供 CLI/TUI、Web UI 和桌面应用三种交互方式。它通过统一的抽象层支持 20+ LLM 提供商（Claude、OpenAI、Google 等），基于 **Bun** 运行时、**Effect** 框架和 **SolidJS** 构建。

```
┌─────────────────────────────────────────────────────────┐
│                    OpenCode 架构总览                      │
├─────────────┬──────────────┬──────────────┬─────────────┤
│  Desktop    │     TUI      │   Web UI     │ VS Code     │
│  (Tauri v2) │  (opentui)   │  (SolidJS)   │ Extension   │
├─────────────┴──────────────┴──────┬───────┴─────────────┤
│          共享 UI 库 (packages/ui)  │                      │
├───────────────────────────────────┼─────────────────────┤
│                   HTTP Server (Hono)                      │
│              REST API / WebSocket / SSE                   │
├─────────────────────────────────────────────────────────┤
│                   Core Runtime (Effect)                   │
│  ┌─────────┐ ┌────────┐ ┌────────┐ ┌───────┐ ┌───────┐ │
│  │ Agent   │ │Session │ │ Tool   │ │Perm   │ │ MCP   │ │
│  │ Runtime │ │Manager │ │System  │ │System │ │Client │ │
│  └─────────┘ └────────┘ └────────┘ └───────┘ └───────┘ │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌────────────┐  ┌──────────────────┐ │
│  │   Provider   │  │  Storage   │  │  Snapshot / VCS  │ │
│  │  Abstraction │  │  (SQLite)  │  │                  │ │
│  └──────────────┘  └────────────┘  └──────────────────┘ │
├─────────────────────────────────────────────────────────┤
│  AI SDK / Anthropic / OpenAI / Google / Bedrock / ...    │
└─────────────────────────────────────────────────────────┘
```

## 2. Monorepo 结构

使用 **Turbo** 管理的 monorepo，采用 **Bun workspaces** 组织。默认分支为 `dev`。

```
opencode/
├── packages/
│   ├── opencode/       # 核心包：CLI、Server、Agent 运行时、Provider、Tool、Session
│   ├── app/            # Web UI（SolidJS + Vite），通过 SDK 连接核心
│   ├── ui/             # 共享 UI 组件库（SolidJS），供 TUI、Desktop、Web 共用
│   ├── desktop/        # 桌面应用（Tauri v2），包装 Web UI
│   ├── console/        # 控制台 Web 应用（计费、Stripe 集成）
│   ├── plugin/         # 插件系统（@opencode-ai/plugin）
│   ├── sdk/            # 从 OpenAPI 规范自动生成的 TypeScript SDK
│   │   └── js/         # JavaScript SDK
│   ├── util/           # 跨包共享工具库（@opencode-ai/util）
│   ├── slack/          # Slack 集成
├── sdks/               # IDE 扩展
│   └── vscode/         # VS Code 扩展
├── script/             # 代码生成脚本（SDK 重新生成等）
├── turbo.json          # Turbo 构建配置
└── CLAUDE.md           # AI 辅助开发指引
```

### 包间依赖关系

```
desktop ──> app ──> ui ──> sdk/js ──> opencode ──> util
                          console ──> util
```

## 3. 核心包详解（packages/opencode/src/）

### 3.1 目录结构

```
src/
├── agent/          # Agent 定义（build、plan、general subagent）
├── auth/           # 认证系统
├── bus/            # 事件总线（Event Bus）
├── cli/            # CLI 入口（yargs），TUI 在 cli/cmd/tui/
├── command/        # 命令执行层
├── config/         # 配置加载与管理
├── control-plane/  # 控制平面实现
├── effect/         # Effect 框架服务定义
├── installation/   # 安装管理
├── lsp/            # LSP 客户端
├── mcp/            # MCP（Model Context Protocol）集成
├── permission/     # 运行时权限系统
├── plugin/         # 插件系统实现
├── project/        # 项目实例管理
├── provider/       # LLM Provider 抽象层
├── session/        # Session 生命周期管理
├── server/         # HTTP 服务器（Hono）
├── snapshot/       # 快照管理
├── storage/        # 数据库层（SQLite + Drizzle ORM）
├── tool/           # Tool 定义（每个 tool 一个 .ts 文件）
└── util/           # 内部工具函数
```

### 3.2 Provider 系统

Provider 系统提供统一的 LLM 接口抽象，支持 20+ 提供商。

```
provider/
├── provider.ts     # 核心：Provider 注册、模型发现、实例创建
├── schema.ts       # Zod schema（ProviderID、ModelID 类型定义）
├── auth.ts         # 认证处理（API Key、OAuth 等）
├── transform.ts    # 消息格式转换（统一不同 Provider 的消息格式）
├── models.ts       # 模型发现（从 models.dev 获取模型信息）
├── error.ts        # Provider 相关错误定义
└── sdk/            # 特定 Provider 的 SDK 适配器（如 GitHub Copilot）
```

**支持的 Provider**（通过 Vercel AI SDK 集成）：

| Provider | SDK 包 |
|----------|--------|
| Anthropic (Claude) | `@ai-sdk/anthropic` |
| OpenAI | `@ai-sdk/openai` |
| Google | `@ai-sdk/google` |
| Azure | `@ai-sdk/azure` |
| Amazon Bedrock | `@ai-sdk/amazon-bedrock` |
| Google Vertex | `@ai-sdk/google-vertex` |
| xAI (Grok) | `@ai-sdk/xai` |
| Mistral | `@ai-sdk/mistral` |
| Groq | `@ai-sdk/groq` |
| DeepInfra | `@ai-sdk/deepinfra` |
| Cerebras | `@ai-sdk/cerebras` |
| Cohere | `@ai-sdk/cohere` |
| Together AI | `@ai-sdk/togetherai` |
| Perplexity | `@ai-sdk/perplexity` |
| OpenRouter | `@openrouter/ai-sdk-provider` |
| Venice | `venice-ai-sdk-provider` |
| GitLab | `gitlab-ai-provider` |
| Vercel | `@ai-sdk/vercel` |
| Gateway | `@ai-sdk/gateway` |
| 自定义 | `@ai-sdk/openai-compatible` |

**核心流程**：
1. 从配置和插件收集 Provider 定义
2. 通过 `models.dev` 进行模型发现和校验
3. 使用 AI SDK 创建统一 `LanguageModelV3` 实例
4. 通过 `transform.ts` 转换消息格式

### 3.3 Tool 系统

Tool 是 Agent 与外部世界交互的基本单元。每个 Tool 是一个自包含的 `.ts` 文件，附带可选的 `.txt` 描述文件。

```
tool/
├── tool.ts              # Tool 定义基础框架
├── bash.ts              # Shell 命令执行
├── read.ts              # 文件读取
├── write.ts             # 文件写入
├── edit.ts              # 文件编辑
├── multiedit.ts         # 多文件编辑
├── apply_patch.ts       # Patch 应用
├── glob.ts              # 文件模式匹配
├── grep.ts              # 内容搜索
├── codesearch.ts        # 代码语义搜索
├── webfetch.ts          # 网页获取
├── websearch.ts         # 网页搜索
├── lsp.ts               # LSP 代码智能
├── question.ts          # 用户交互提问
├── plan.ts              # 计划管理
├── task.ts              # 任务管理
├── todo.ts              # 待办管理
├── schema.ts            # Schema 操作
├── registry.ts          # 注册表操作
├── skill.ts             # 技能调用
├── truncate.ts          # 输出截断
├── external-directory.ts # 外部目录访问
├── mcp-exa.ts           # MCP Exa 搜索
├── invalid.ts           # 无效 Tool 处理
└── *.txt                # 各 Tool 的描述文本（供 LLM 参考）
```

**Tool 定义模式**：

```typescript
// 每个 Tool 使用 Tool.define() 或 Tool.Info 定义
// 包含：id、描述、参数（Zod schema）、执行函数（返回 Effect）
import { Tool } from "./tool"
import { Effect } from "effect"

// Tool 描述存储在同名 .txt 文件中
import DESCRIPTION from "./bash.txt"

// 参数使用 Zod schema 校验
const Schema = z.object({ command: z.string() })
```

**Tool 执行流程**：

```
Agent 调用 Tool
    │
    ▼
权限系统检查（Permission）
    │
    ├── allow  → 直接执行
    ├── deny   → 拒绝执行
    └── ask    → 请求用户确认
         │
         ▼
    Tool 执行（Effect 运行时）
         │
         ▼
    返回结果（文本/流式）
```

### 3.4 Session 管理

Session 管理对话状态、消息历史和会话生命周期。

```
session/
├── index.ts            # Session 核心逻辑（创建、查询、更新）
├── message-v2.ts       # 消息模型（v2 格式）
├── processor.ts        # 消息处理器（LLM 请求/响应处理）
├── status.ts           # 会话状态枚举
├── session.sql.ts      # Session 表 Schema
├── schema.ts           # 类型定义（SessionID、MessageID、PartID）
└── summary.ts          # 会话摘要
```

**Session 数据模型**：

```
Session
├── id: SessionID
├── title: string
├── status: SessionStatus
├── summary: { additions, deletions, files, diffs }
├── parent_session_id: SessionID | null
├── created_at / updated_at
│
└── Messages[]
    ├── id: MessageID
    ├── role: "user" | "assistant"
    ├── content: string
    │
    └── Parts[]
        ├── id: PartID
        ├── type: "text" | "tool-invocation" | "tool-result" | ...
        └── content: any
```

**消息处理流程**：

```
用户输入
    │
    ▼
Session.createMessage()
    │
    ▼
Prompt 构建（系统提示 + 历史 + 工具定义）
    │
    ▼
Provider.send() → LLM API 调用
    │
    ▼
响应处理（文本 + Tool 调用）
    │
    ├── 文本响应 → 直接展示
    └── Tool 调用 → 权限检查 → 执行 → 结果回传 LLM（循环）
    │
    ▼
Session.saveMessage() → 持久化到 SQLite
    │
    ▼
通过 SSE/WebSocket 推送给客户端
```

### 3.5 Snapshot / VCS 系统

基于独立 Git 仓库的文件状态追踪和恢复系统，用于 Tool 执行后的状态管理。

```
snapshot/
└── index.ts           # Snapshot.Service — 文件追踪、恢复、回滚
```

**核心能力**：

| 方法 | 用途 |
|------|------|
| `track(files)` | 追踪文件变更，记录到独立 Git 仓库 |
| `restore()` | 恢复到上一个快照状态 |
| `revert()` | 回滚所有未保存变更 |
| `diff()` | 查看当前变更差异 |
| `diffFull()` | 查看完整差异（含新增文件） |

**设计要点**：
- 使用独立 Git 仓库（非用户项目仓库）存储文件快照
- 大文件自动排除（2MB 限制）
- 7 天自动清理过期快照
- 支持批量恢复操作

### 3.6 HTTP 服务器

基于 **Hono** 框架，提供 REST API、WebSocket 和 SSE 流式传输。

```
server/
├── server.ts          # 服务器创建、路由注册、OpenAPI 规范生成
├── middleware.ts       # 中间件（认证、日志、压缩、CORS、错误处理）
├── proxy.ts           # 代理配置
├── mdns.ts            # mDNS 服务发现
├── instance/          # 实例级路由控制器
├── control/           # 控制平面路由
├── ui/                # UI 静态资源路由
└── projectors/        # 事件投影器（Event Sourcing）
```

**中间件栈**（按注册顺序）：

```
请求 → ErrorMiddleware → AuthMiddleware → LoggerMiddleware
     → CompressionMiddleware → CorsMiddleware → 路由处理
```

**路由层级**：

```
/  → ControlPlaneRoutes()    # 控制平面 API
/  → InstanceRoutes()         # 实例级 API（Session、Tool、Provider 等）
/  → UIRoutes()               # Web UI 静态资源
```

### 3.7 Agent 系统

Agent 是 OpenCode 的核心执行单元，定义了不同模式的 AI 行为。

```
agent/
└── agent.ts           # Agent 定义、提示词管理、模型配置
```

**Agent 模式**：

| 模式 | 用途 |
|------|------|
| Primary | 主 Agent，处理用户直接请求 |
| Subagent | 子 Agent，处理特定子任务 |
| Build | 构建 Agent，专注于代码实现 |
| Plan | 规划 Agent，专注于方案设计 |
| All | 通用模式 |

### 3.8 Effect 框架

OpenCode 大量使用 **Effect** 进行依赖注入、错误处理和资源管理。

```
effect/
├── app-runtime.ts     # 核心：AppRuntime — 合并所有服务 Layer 的完整托管运行时
├── run-service.ts     # 工具：makeRuntime/attach — 创建单个服务运行时并注入上下文
├── instance-state.ts  # 实例状态服务
├── instance-ref.ts    # 实例引用
├── workspace-context.ts # 工作空间上下文
├── logger.ts          # 日志服务
├── oltp.ts            # OpenTelemetry 集成
└── runner.ts          # 运行器服务
```

**AppRuntime（`app-runtime.ts`）**：

AppRuntime 是系统启动时创建的完整托管运行时，通过 `Layer.merge()` 将所有服务 Layer 合并为一个统一的 `ManagedRuntime`：

```
AppRuntime 合并的服务 Layer：
├── AppFileSystem     # 文件系统抽象
├── Bus               # 事件总线
├── Auth              # 认证服务
├── Git               # Git 操作
├── Storage           # 数据库持久化
├── Plugin            # 插件服务
├── Format            # 代码格式化
├── LSP               # 语言服务
├── Snapshot          # 文件快照
├── FileWatcher       # 文件监听
├── VCS               # 版本控制
└── ...
```

**run-service.ts（工具函数）**：

提供 `makeRuntime()` 和 `attach()` 两个核心工具函数，用于为单个服务创建独立的 ManagedRuntime：

```typescript
// makeRuntime: 为单个服务创建托管运行时
const { runPromise, runSync, runFork } = makeRuntime(MyService, MyServiceLive)

// attach: 将当前实例上下文（InstanceRef + WorkspaceRef）注入 Effect
const result = attach(effect)
```

**核心模式**：

```typescript
// 1. 使用 Context.Tag 定义服务
const MyService = Context.GenericTag<MyService>("MyService")

// 2. 使用 Layer 提供依赖
const MyServiceLive = Layer.effect(MyService, Effect.gen(...))

// 3. AppRuntime 启动时合并所有 Layer
const runtime = ManagedRuntime.make(Layer.merge(...allLayers))

// 4. 或通过 makeRuntime 为单个服务创建独立运行时
const { runPromise } = makeRuntime(MyService, MyServiceLive)
```

### 3.9 权限系统

基于规则的运行时权限控制，保护 Tool 执行安全。

```
permission/
├── index.ts           # 权限检查入口
├── arity.ts           # Bash 命令复杂度分析
└── ...
```

**权限规则**：

```
Rule = { pattern: string, action: "allow" | "deny" | "ask" }

匹配流程：
  Tool 调用请求 → 遍历规则列表 → 匹配 pattern
    ├── allow  → 自动放行
    ├── deny   → 自动拒绝
    └── ask    → 暂停等待用户确认
```

### 3.9 存储层

使用 **SQLite**（WAL 模式）+ **Drizzle ORM** 进行数据持久化。

```
storage/
├── db.ts              # Drizzle 客户端初始化（#db 平台抽象）
├── storage.ts         # 存储服务定义
├── schema.sql.ts      # 数据库 Schema 定义（snake_case 字段）
└── ...                # 各模块的 SQL 表定义
```

**关键设计**：
- 字段命名使用 `snake_case`，避免列名字符串重定义
- 数据库迁移在启动时自动运行
- 使用 `#db` import path 实现平台特定的数据库实现

**JSON → SQLite 迁移**：

首次运行时，系统自动将旧版 JSON 存储的数据迁移至 SQLite：
1. 读取旧格式的项目和会话 JSON 数据
2. 转换为 SQLite 表结构
3. 提取会话 diff 到独立文件
4. 显示迁移进度，迁移完成后自动清理

### 3.10 MCP 集成

Model Context Protocol 集成，支持外部 Tool 和资源发现。

```
mcp/
├── index.ts           # MCP 客户端核心
├── auth.ts            # MCP 认证
├── oauth-provider.ts  # OAuth 提供商
└── oauth-callback.ts  # OAuth 回调处理
```

**支持的传输方式**：HTTP、SSE、stdio

### 3.11 LSP 客户端

Language Server Protocol 客户端，提供代码智能功能。

```
lsp/
├── server.ts          # LSP 服务器管理
├── client.ts          # LSP 客户端
├── language.ts        # 语言定义
└── launch.ts          # LSP 进程启动
```

### 3.12 事件总线

用于模块间解耦通信的事件系统，基于 Effect PubSub 实现类型化事件。事件通过 SSE 推送给客户端。

```
bus/
├── bus.ts             # 事件总线核心（Effect PubSub）
└── bus-event.ts       # 事件类型定义（Zod schema 校验）
```

**核心事件类型**：

| 事件名 | 类型 | 用途 |
|--------|------|------|
| `message.part.delta` | `EventMessagePartDelta` | 流式推送 AI 响应片段 |
| `session.status` | `EventSessionStatus` | 通知客户端 Agent 忙闲状态变化 |
| `permission.asked` | `EventPermissionAsked` | 请求用户批准 Tool 执行 |
| `file.edited` | `EventFileEdited` | 通知 UI 文件系统变更 |
| `lsp.updated` | `EventLspUpdated` | 通知客户端 LSP 服务器状态变化 |
| `pty.created` | — | 伪终端创建 |
| `pty.updated` | — | 伪终端输出更新 |
| `pty.exited` | — | 伪终端退出 |
| `pty.deleted` | — | 伪终端删除 |
| `server.instance.disposed` | — | 服务实例销毁 |

**事件流**：

```
Core Runtime（Tool 执行、Session 状态变更等）
    │
    ▼
Bus.publish(event)    # 内部模块间通信
    │
    ▼
Server SSE 端点推送    # /global/event（Server-Sent Events）
    │
    ▼
Client SDK 监听       # SDK 提供类型化事件回调
```

### 3.13 配置系统

管理项目级和全局配置。

```
config/
└── config.ts          # 配置加载、插件配置、模型选择、工作空间配置
```

**配置层级**：
1. 全局配置（`~/.config/opencode/`）
2. 项目配置（`.opencode/`）
3. 环境变量覆盖
4. CLI 参数覆盖

## 4. Bootstrap 启动序列

项目启动时，`src/project/bootstrap.ts` 按以下顺序初始化服务：

```
1. Log 初始化         # 日志系统
2. Plugin 初始化      # 加载外部插件
3. ShareNext 初始化    # 分享服务
4. Format 初始化      # 代码格式化器
5. LSP 初始化         # 启动语言服务器
6. Filesystem 初始化   # 文件系统抽象
7. FileWatcher 初始化  # 文件变更监听
8. VCS 初始化         # 版本控制集成
9. Snapshot 初始化     # 文件快照系统
10. Command.Executed  # 标记项目初始化完成
```

## 5. CLI 入口点

CLI 使用 **yargs** 解析命令行参数，入口在 `src/index.ts`。

**启动流程**：

```
yargs 解析 CLI 参数
    │
    ▼
中间件：日志初始化 + 数据库迁移
    │
    ▼
JSON → SQLite 迁移（首次运行，显示进度）
    │
    ▼
分发到对应命令处理器
```

**注册的命令**：

| 命令 | 用途 |
|------|------|
| `tui` / 默认 | 启动 TUI 交互界面 |
| `serve` | 启动无头 API 服务器（端口 4096） |
| `web` | 启动 Server + Web 界面 |
| `run` | 执行单次 Agent 任务 |
| `attach` | 连接到运行中的实例 |
| `generate` | 代码生成 |
| `debug` | 调试模式 |
| `account` | 账户管理 |
| `providers` | Provider 管理 |
| `agent` | Agent 管理 |
| `mcp` | MCP 服务器管理 |
| `session` | 会话管理 |
| `models` | 模型列表 |
| `stats` | 统计信息 |
| `export` / `import` | 数据导入导出 |
| `github` / `pr` | GitHub 集成 |
| `plug` | 插件管理 |
| `db` | Drizzle Kit 数据库操作 |
| `upgrade` | 自动升级 |
| `uninstall` | 卸载 |

## 6. Web UI（packages/app/）

基于 **SolidJS** + **Vite** 构建的 Web 前端。

```
src/
├── app.tsx            # 应用入口
├── entry.tsx          # 渲染入口
├── pages/             # 页面路由
│   ├── home.tsx       # 首页
│   ├── session.tsx    # 会话页
│   ├── directory-layout.tsx # 目录布局
│   └── error.tsx      # 错误页
├── components/        # UI 组件
│   ├── session/       # 会话相关组件
│   ├── prompt-input/  # 输入框组件
│   ├── server/        # 服务端组件
│   ├── dialog-*.tsx   # 各种对话框
│   ├── file-tree.tsx  # 文件树
│   ├── terminal.tsx   # 终端模拟
│   └── settings-*.tsx # 设置面板
├── context/           # SolidJS Context（状态管理）
├── hooks/             # 自定义 Hooks
├── i18n/              # 国际化
├── addons/            # 插件/扩展
└── utils/             # 工具函数
```

**与核心通信**：通过自动生成的 **SDK** 调用 HTTP API，使用 **WebSocket** 接收实时更新。

## 7. 桌面应用（packages/desktop/）

基于 **Tauri v2** 的原生桌面应用，包装 Web UI。

- 共享 Web UI 组件
- 系统集成（剪贴板、通知、窗口状态管理）
- 自动更新支持
- 跨平台支持（macOS、Windows、Linux）

## 8. 共享 UI 库（packages/ui/）

供 TUI、Desktop 和 Web 客户端共用的 SolidJS 组件库。

- 共享组件（卡片、文件树、对话框等）
- Context Provider（状态管理）
- 国际化支持
- Worker Pool 集成
- Storybook 组件文档

## 9. SDK（packages/sdk/）

从 OpenAPI 规范（`packages/sdk/openapi.json`）自动生成的 **TypeScript SDK**，提供类型安全的 API 调用。

**双传输模式**：
- **Same-process**：内部进程内调用，零网络开销
- **HTTP**：通过 REST API 远程调用

- API 变更后运行 `./packages/sdk/js/script/build.ts` 重新生成
- 供 Web UI 和外部集成使用

## 10. 插件系统（packages/plugin/）

提供 `@opencode-ai/plugin` 接口，支持扩展：

- 自定义 Provider
- 自定义 Tool
- TUI 集成支持

## 11. IDE 扩展

### VS Code 扩展（sdks/vscode/）

提供 IDE 内集成，支持快捷键打开 OpenCode：

| 快捷键 | 功能 |
|--------|------|
| `Cmd/Ctrl + Esc` | 打开 OpenCode |
| `Cmd/Ctrl + Shift + Esc` | 新建会话 |
| `Cmd/Ctrl + Option + K` | 插入文件引用 |

## 12. 数据流总览

```
┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────┐
│  用户     │────>│  Client  │────>│ HTTP Server  │────>│ Session  │
│ (CLI/Web)│     │ (SDK)    │     │ (Hono)       │     │ Manager  │
└──────────┘     └──────────┘     └──────────────┘     └────┬─────┘
                                                       │
                  ┌────────────────────────────────────┘
                  │
                  ▼
             ┌─────────┐     ┌──────────┐     ┌──────────────┐
             │  Agent  │────>│ Provider │────>│  LLM API     │
             │ Runtime │     │ Layer    │     │ (外部服务)    │
             └────┬────┘     └──────────┘     └──────────────┘
                  │
                  ▼
             ┌─────────┐     ┌──────────┐
             │  Tool   │────>│ Storage  │
             │ System  │     │ (SQLite) │
             └─────────┘     └──────────┘
                  │
                  ▼
             ┌─────────┐
             │  LSP /  │
             │  MCP    │
             └─────────┘
```

**关键数据流**：
1. **请求流**：用户 → Client → Server → Session → Agent → Provider → LLM
2. **响应流**：LLM → Provider → Agent → Session → Server → SSE/WebSocket → Client
3. **Tool 流**：Agent → Permission → Tool 执行 → 结果回传 Agent → LLM（循环直到完成）
4. **持久化**：Session/Message → Storage（SQLite），异步写入

## 13. 平台抽象

| Import Path | 用途 | 实现 |
|-------------|------|------|
| `#db` | 数据库客户端 | 平台特定的 SQLite 实现 |
| `#pty` | 伪终端 | Bun 原生 / Node node-pty |
| `#hono` | HTTP 适配器 | Hono Bun/Node 适配器 |

## 14. 关键技术栈

| 类别 | 技术 |
|------|------|
| 运行时 | Bun 1.3+ |
| 构建工具 | Turbo, esbuild |
| 类型系统 | TypeScript, tsgo |
| 函数式框架 | Effect |
| Schema 校验 | Zod |
| Web 框架 | Hono |
| ORM | Drizzle ORM |
| 数据库 | SQLite (WAL mode) |
| 前端框架 | SolidJS |
| 桌面框架 | Tauri v2 |
| AI SDK | Vercel AI SDK |
| 协议 | LSP, MCP, SSE, WebSocket |
| 测试 | Bun test, Playwright |
| 代码规范 | Prettier (semi: false, printWidth: 120) |

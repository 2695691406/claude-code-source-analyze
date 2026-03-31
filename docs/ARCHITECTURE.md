# Claude Code 整体框架设计文档

> 基于 `@anthropic-ai/claude-code` v2.1.88 源码映射还原分析

---

## 1. 项目概述

**Claude Code** 是 Anthropic 开发的 AI 驱动终端编码助手，将 Claude AI 深度集成到开发工作流中。它是一个功能完备的命令行工具，支持代码理解、智能体自动化、文件操作、终端集成、Git 工作流等核心能力。

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 包名 | `@anthropic-ai/claude-code` |
| 版本 | 2.1.88 |
| 入口命令 | `claude` |
| Node.js 要求 | ≥18.0.0 |
| 模块类型 | ES Module (`"type": "module"`) |
| 构建运行时 | Bun |
| 语言 | TypeScript / TSX |
| UI 框架 | React + Ink（终端渲染） |

### 1.2 规模统计

| 指标 | 数值 |
|------|------|
| 源码文件总数 | 4,756 |
| TypeScript/TSX 文件 | 1,884 |
| 总代码行数 | 205,487 |
| CLI 命令数 | 86+ |
| 智能体工具数 | 43+ |
| 服务模块数 | 21+ |
| React Hooks | 33+ |
| React 组件 | 33+ |

---

## 2. 整体架构

### 2.1 架构分层

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户层 (User Layer)                         │
│   CLI 输入 / IDE 集成 / SDK 调用 / Bridge 远程控制                    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                       入口层 (Entrypoint Layer)                      │
│   main.tsx → cli.tsx → init.ts → 命令路由 / 模式选择                  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                       命令层 (Command Layer)                         │
│   commands.ts 命令注册表 → 86+ 命令实现 → 斜杠命令处理                 │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                      查询引擎层 (Query Engine Layer)                  │
│   QueryEngine.ts → query.ts → 消息流 → 工具调用循环                    │
└────────────┬───────────────────┼───────────────────┬───────────────┘
             │                   │                   │
┌────────────▼──────┐ ┌─────────▼────────┐ ┌───────▼────────────────┐
│   工具层 (Tools)   │ │ 服务层 (Services) │ │  智能体层 (Agent Layer) │
│  43+ 工具实现      │ │  21+ 服务模块     │ │ AgentTool / 任务框架   │
│  Tool 抽象基类     │ │  API / MCP / LSP  │ │ Coordinator / Bridge   │
│  权限检查          │ │  OAuth / Analytics │ │ 多智能体协调           │
└────────────┬──────┘ └─────────┬────────┘ └───────┬────────────────┘
             │                   │                   │
┌────────────▼───────────────────▼───────────────────▼───────────────┐
│                       状态层 (State Layer)                           │
│   AppStateStore → React Context → 消息历史 / 工具权限 / 任务状态       │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                      基础设施层 (Infrastructure Layer)                │
│   utils/ → 权限系统 / 设置管理 / 模型选择 / Git / Shell / 文件操作     │
│   constants/ / schemas/ / types/ / migrations/                       │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        渲染层 (Rendering Layer)                      │
│   Ink 终端 UI → React 组件 → 屏幕 → Hooks → 键盘绑定                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心数据流

```
用户输入 (CLI/IDE/SDK)
    │
    ▼
CommandRegistry ─── 匹配斜杠命令 ──→ 直接执行命令逻辑
    │ (非斜杠命令)
    ▼
processUserInput() ─── 解析输入类型
    │
    ▼
QueryEngine.submitMessage()
    │
    ├── 构建系统提示词 (SystemPrompt)
    ├── 注入用户上下文 (Git状态 / 文件树 / 记忆文件)
    │
    ▼
query() ─── 工具调用循环 (核心)
    │
    ├── 调用 Claude API (流式响应)
    │       │
    │       ▼
    │   AssistantMessage ──→ 包含 ToolUseBlock?
    │       │ 是                    │ 否
    │       ▼                      ▼
    │   CanUseTool() 权限检查     返回最终响应
    │       │
    │       ▼
    │   Tool.call() 执行工具
    │       │
    │       ▼
    │   ToolResultBlock ──→ 回传给 Claude API
    │       │
    │       └── 继续循环 ───────→ query()
    │
    ▼
流式输出 → Ink UI 渲染 → 终端显示
```

---

## 3. 核心设计模式

### 3.1 异步生成器管道 (Async Generator Pipeline)

整个系统的核心通信模式基于 **AsyncGenerator**，实现了流式处理和实时更新：

```typescript
// QueryEngine 入口
async *submitMessage(prompt, options): AsyncGenerator<SDKMessage>

// 查询循环
async function* query(params): AsyncGenerator<Message>

// 智能体执行
async function* runAgent(config): AsyncGenerator<Message>
```

**设计优势**：
- 生产者-消费者解耦
- 支持实时流式响应
- 允许中间状态更新
- 支持取消操作

### 3.2 工具抽象系统 (Tool Abstraction)

所有工具遵循统一的抽象接口：

```typescript
type Tool = {
  name: string                                    // 工具标识
  searchHint: string                              // 搜索提示
  inputSchema: ZodSchema                          // 输入校验
  outputSchema: ZodSchema                         // 输出校验
  description(input, context): Promise<string>    // 动态描述
  prompt(): Promise<string>                       // 使用说明
  call(input, context): Promise<ToolResult>       // 核心执行
  checkPermissions(): PermissionResult            // 权限检查
  isConcurrencySafe(): boolean                    // 并发安全
  isReadOnly(): boolean                           // 只读标记
  shouldDefer: boolean                            // 延迟加载
}
```

**扩展机制**：
- 内置工具 (43+)
- MCP 协议工具 (动态发现)
- Skill 技能工具 (项目自定义)
- Plugin 插件工具 (第三方扩展)

### 3.3 分层权限系统 (Layered Permission System)

```
┌──────────────────────────────────────┐
│  层级 1: 工具池组装 (Tool Pool)        │
│  决定哪些工具对当前智能体可用           │
├──────────────────────────────────────┤
│  层级 2: 权限模式 (Permission Mode)    │
│  manual / ask / auto / bypass         │
├──────────────────────────────────────┤
│  层级 3: 权限规则 (Permission Rules)   │
│  per-tool 规则 / 文件系统规则 /         │
│  Shell 命令规则 / 危险模式检测          │
├──────────────────────────────────────┤
│  层级 4: 用户确认 (User Approval)      │
│  交互式对话框 / 自动审批               │
└──────────────────────────────────────┘
```

**关键组件**：
- `CanUseToolFn`: 权限决策函数
- `bashClassifier`: 危险 Bash 命令检测
- `dangerousPatterns`: 危险操作模式数据库
- `PermissionRule`: 权限规则定义 DSL

### 3.4 状态管理架构

采用类 Zustand 的集中式状态管理：

```typescript
type AppState = {
  messages: Message[]              // 消息历史
  tools: Tools                     // 可用工具
  fileCache: FileStateCache        // 文件状态缓存
  permissions: ToolPermissionContext // 工具权限
  tasks: TaskState[]               // 后台任务
  settings: SettingsJson           // 用户设置
  modelSelection: ModelState       // 模型选择
  pluginStatus: PluginState        // 插件状态
  speculation: SpeculationState    // 预测执行
  uiState: UIState                 // UI 状态
}
```

**设计要点**：
- React Context Provider 提供全局状态
- 不可变状态更新 (Immutable Updates)
- 设置变更检测与自动应用
- 任务状态追踪与通知

### 3.5 Feature Gate 编译系统

使用 Bun 的 `bun:bundle` 特性门控进行死代码消除：

```typescript
// 条件编译示例
import { feature } from 'bun:bundle'

if (feature('KAIROS')) {
  // KAIROS 相关代码 - 不满足条件时被完全移除
}
```

**已知特性门控 (20+)**：

| 特性标识 | 描述 |
|---------|------|
| `KAIROS` | 助手模式 (主动建议/后台任务) |
| `COORDINATOR_MODE` | 多智能体协调器 |
| `BRIDGE_MODE` | 远程桥接模式 |
| `VOICE_MODE` | 语音交互 |
| `DAEMON` | 后台守护进程 |
| `PROACTIVE` | 主动建议 |
| `WORKFLOW_SCRIPTS` | 工作流脚本 |
| `BUDDY` | AI 伴侣 |
| `FORK_SUBAGENT` | 分叉子智能体 |
| `UDS_INBOX` | Unix Domain Socket 收件箱 |
| `TORCH` | Torch 模式 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `TERMINAL_PANEL` | 终端面板 |
| `ULTRAPLAN` | 超级规划 |
| `CCR_REMOTE_SETUP` | 远程配置 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub Webhook 集成 |
| `KAIROS_BRIEF` | 简报生成 |
| `ABLATION_BASELINE` | 消融基线测试 |

---

## 4. 多模式架构

Claude Code 支持多种运行模式，通过特性门控和入口参数选择：

### 4.1 CLI 交互模式 (默认)

```
用户 → 终端 → Ink REPL → QueryEngine → Claude API
```

- 标准终端交互界面
- 支持斜杠命令、Vim 模式、键盘快捷键
- 实时流式输出

### 4.2 非交互模式

```
管道输入/参数 → QueryEngine → 输出结果 → 退出
```

- 支持 `--prompt` 参数直接查询
- 支持管道输入 (`echo "..." | claude`)
- 适用于 CI/CD 和脚本化场景

### 4.3 SDK 模式

```
SDK 调用 → query() / createSession() → AsyncGenerator → SDK 消息
```

- 编程式 API 接入
- 支持 `query()`、`createSession()` 等入口
- 结构化输入/输出
- 控制消息 (中断/模型切换)

### 4.4 Coordinator 多智能体模式

```
Coordinator (主智能体)
    ├── Worker 1 (AgentTool → 独立 QueryEngine)
    ├── Worker 2 (AgentTool → 独立 QueryEngine)
    └── Worker N (AgentTool → 独立 QueryEngine)
```

- 一个协调器指挥多个工作者
- 工作者通过 `<task-notification>` 汇报结果
- 支持并行研究和顺序实现
- 详见 [Agent 设计文档](./AGENT_DESIGN.md)

### 4.5 Bridge 远程控制模式

```
claude.ai Web → Bridge API → BridgeApiClient → 本地 Agent 子进程
```

- 将本地 CLI 智能体连接到 claude.ai
- WebSocket 通信 + JWT 认证
- 会话轮询 + 控制消息传递
- 支持中断、模型切换等远程控制

### 4.6 Daemon 后台守护模式

```
Daemon 主进程 → Worker 子进程 → 持久化会话
```

- 后台服务持续运行
- 管理工作者生命周期
- 支持持久化会话

### 4.7 KAIROS 助手模式

```
调度器 → CronTask → 自动触发 → 主动建议/后台执行
```

- 计划任务 (Cron 表达式)
- 主动建议和推送通知
- 后台自主执行

### 4.8 Voice 语音模式

```
麦克风 → STT (语音转文字) → QueryEngine → TTS (文字转语音) → 扬声器
```

- 流式语音识别
- 关键词检测
- 语音输出

---

## 5. 扩展性设计

### 5.1 Plugin 插件系统

```
插件市场 → 安装管理器 → 后台安装 → 命令/工具注入
```

- 非阻塞后台安装
- 市场对账 (Marketplace Reconciliation)
- 插件可提供新命令和工具
- 内置插件 (`src/plugins/bundled/`)

### 5.2 Skill 技能系统

```
.claude/skills/ 目录 → 技能发现 → SkillTool 注册 → 可调用
```

- 项目级自定义技能
- YAML 前置元数据 + Markdown 正文
- 内置技能 (`src/skills/bundled/`)
- 动态发现和加载

### 5.3 MCP 协议集成

```
MCP 服务器注册 → 连接管理器 → 工具发现 → MCPTool 代理
```

- 支持多种传输协议 (stdio, SSE, HTTP, WebSocket)
- 官方注册表 + 自定义服务器
- OAuth 认证支持
- 资源读取 (ListMcpResources / ReadMcpResource)
- 通道权限控制

### 5.4 Agent 定义系统

```
.claude/agents/ 目录 → YAML 定义 → AgentTool 加载
```

- 自定义智能体类型
- 模型覆盖、工具白名单
- 专属 MCP 服务器配置
- 详见 [Agent 设计文档](./AGENT_DESIGN.md)

---

## 6. 技术栈

### 6.1 核心运行时

| 技术 | 用途 |
|------|------|
| **Bun** | 构建工具 + 原生 TypeScript 支持 |
| **Node.js ≥18** | 运行时引擎 |
| **TypeScript** | 主要编程语言 |

### 6.2 AI 与 API

| 技术 | 用途 |
|------|------|
| **@anthropic-ai/sdk** | Anthropic Claude API 客户端 |
| **@modelcontextprotocol/sdk** | MCP 工具集成协议 |

### 6.3 UI 与渲染

| 技术 | 用途 |
|------|------|
| **React** | 组件化 UI 框架 (终端，非 Web) |
| **Ink** | React 终端渲染器 |
| **chalk** | 终端颜色/样式 |
| **log-update** | 终端内容更新 |

### 6.4 CLI 基础设施

| 技术 | 用途 |
|------|------|
| **commander.js** | CLI 框架 |
| **yargs** | 参数解析 |

### 6.5 数据验证与工具

| 技术 | 用途 |
|------|------|
| **zod** | TypeScript Schema 验证 |
| **lodash-es** | 函数式工具库 |
| **axios** | HTTP 客户端 |
| **p-map** | 并行 Promise 映射 |
| **strip-ansi** | ANSI 码清理 |

### 6.6 认证与安全

| 技术 | 用途 |
|------|------|
| **OAuth 2.0** | 用户认证 (PKCE 流程) |
| **Keychain** | macOS 安全凭证存储 |
| **MTLS** | 双向 TLS 证书 |

### 6.7 遥测与实验

| 技术 | 用途 |
|------|------|
| **OpenTelemetry** | 可观测性 (延迟加载) |
| **GrowthBook** | 特性标志与 A/B 测试 |
| **Datadog** | 事件投递后端 |

### 6.8 构建与打包

| 技术 | 用途 |
|------|------|
| **Bun:bundle** | 打包 + 特性门控死代码消除 |
| **Biome** | Linter / Formatter |

---

## 7. 目录结构总览

```
restored-src/src/
├── main.tsx                    # CLI 主入口 (4,683 LOC)
├── QueryEngine.ts              # 查询引擎 (1,295 LOC)
├── query.ts                    # 查询执行 (1,729 LOC)
├── Tool.ts                     # 工具抽象基类 (792 LOC)
├── commands.ts                 # 命令注册表 (754 LOC)
├── tools.ts                    # 工具集合管理器 (389 LOC)
├── context.ts                  # 上下文构建
├── history.ts                  # 会话历史
├── cost-tracker.ts             # Token 追踪
├── Task.ts                     # 任务定义
│
├── entrypoints/                # 启动入口
├── commands/                   # 86+ CLI 命令
├── tools/                      # 43+ 智能体工具
├── services/                   # 21+ 服务模块
├── utils/                      # 100+ 工具函数
├── state/                      # 状态管理
├── components/                 # 33+ React 组件
├── screens/                    # UI 屏幕
├── hooks/                      # 33+ React Hooks
├── types/                      # TypeScript 类型
├── context/                    # React Context
├── ink/                        # 终端 UI 渲染层
├── bridge/                     # 远程桥接
├── cli/                        # CLI 基础设施
├── tasks/                      # 任务执行框架
├── coordinator/                # 多智能体协调
├── skills/                     # 技能系统
├── plugins/                    # 插件系统
├── constants/                  # 全局常量
├── bootstrap/                  # 启动引导
├── schemas/                    # 数据校验
├── migrations/                 # 数据迁移
├── keybindings/                # 键盘绑定
├── native-ts/                  # 原生模块
├── buddy/                      # AI 伴侣
├── assistant/                  # 助手模式
├── voice/                      # 语音交互
├── remote/                     # 远程会话
├── vim/                        # Vim 模式
└── memdir/                     # 记忆目录
```

---

## 8. 关联文档

- [模块详细设计](./MODULES.md) — 按文件夹逐一分析每个模块
- [命令详述](./COMMANDS.md) — 86+ 命令逐一详述 + 分类汇总
- [工具详述](./TOOLS.md) — 43+ 工具逐一详述 + 分类汇总
- [Agent 设计](./AGENT_DESIGN.md) — 智能体架构、多智能体协调、生命周期

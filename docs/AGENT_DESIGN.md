# Claude Code Agent 设计文档

> 智能体架构、多智能体协调、生命周期与通信机制的完整设计分析

---

## 目录

- [概述](#1-概述)
- [核心架构](#2-核心架构)
- [智能体类型与定义](#3-智能体类型与定义)
- [智能体生命周期](#4-智能体生命周期)
- [查询引擎 (QueryEngine)](#5-查询引擎-queryengine)
- [工具调用循环 (query)](#6-工具调用循环-query)
- [智能体执行引擎 (runAgent)](#7-智能体执行引擎-runagent)
- [任务框架](#8-任务框架)
- [多智能体协调 (Coordinator)](#9-多智能体协调-coordinator)
- [智能体间通信](#10-智能体间通信)
- [权限模型](#11-权限模型)
- [上下文管理](#12-上下文管理)
- [Bridge 远程控制](#13-bridge-远程控制)
- [KAIROS 助手模式](#14-kairos-助手模式)
- [Buddy AI 伴侣](#15-buddy-ai-伴侣)
- [SDK 集成](#16-sdk-集成)
- [设计模式总结](#17-设计模式总结)

---

## 1. 概述

Claude Code 的 Agent 系统是整个应用的核心，它实现了从简单的单轮对话到复杂的多智能体协同工作的完整能力链。

### 核心设计理念

- **智能体即一等公民**：每个智能体拥有独立的 QueryEngine、工具池、上下文和生命周期管理
- **递归嵌套**：父会话 → 子智能体 → 嵌套子智能体（无限递归支持）
- **隔离性**：每个智能体拥有独立的文件缓存、消息历史和工具权限
- **可恢复性**：智能体可被终止/恢复，完整上下文保留
- **多模态执行**：支持同步、异步、远程、进程内等多种执行模式

### 关键组件关系

```
┌─────────────────────────────────────────────────────────┐
│                   用户会话 (Main Session)                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │              QueryEngine (主引擎)                 │    │
│  │  ┌─────────────────────────────────┐            │    │
│  │  │  query() 工具调用循环             │            │    │
│  │  │  ├── Claude API 调用              │            │    │
│  │  │  ├── 工具执行                     │            │    │
│  │  │  │   ├── AgentTool ──→ 子智能体   │            │    │
│  │  │  │   ├── BashTool ──→ Shell      │            │    │
│  │  │  │   ├── FileEditTool ──→ 文件    │            │    │
│  │  │  │   └── ...                      │            │    │
│  │  │  └── 上下文压缩                   │            │    │
│  │  └─────────────────────────────────┘            │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──────────────────────┐  ┌──────────────────────┐     │
│  │ LocalAgentTask #1    │  │ LocalAgentTask #2    │     │
│  │ (后台智能体)          │  │ (后台智能体)          │     │
│  │ QueryEngine (独立)    │  │ QueryEngine (独立)    │     │
│  └──────────────────────┘  └──────────────────────┘     │
│                                                         │
│  ┌──────────────────────┐                               │
│  │ RemoteAgentTask      │                               │
│  │ (云端智能体)          │                               │
│  │ CCR 环境轮询         │                               │
│  └──────────────────────┘                               │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 核心架构

### 2.1 文件结构

```
src/
├── QueryEngine.ts                    # 查询引擎 (1,295 LOC) — 每个智能体一个实例
├── query.ts                          # 工具调用循环 (1,729 LOC)
├── Tool.ts                           # 工具抽象基类
├── Task.ts                           # 任务定义与状态
├── tools/
│   ├── AgentTool/                    # 智能体生成工具
│   │   ├── AgentTool.tsx             # 主入口 (~1000 LOC)
│   │   ├── runAgent.ts              # 执行引擎 (~866 LOC)
│   │   ├── loadAgentsDir.ts         # Agent 定义加载
│   │   ├── agentToolUtils.ts        # 工具解析
│   │   ├── agentMemory.ts           # Agent 记忆
│   │   └── agentMemorySnapshot.ts   # 记忆快照
│   ├── SendMessageTool/             # 智能体间消息 (~917 LOC)
│   ├── TeamCreateTool/              # 团队创建
│   ├── TeamDeleteTool/              # 团队解散
│   ├── SkillTool/                   # 技能执行
│   ├── TaskOutputTool/              # 任务输出读取
│   ├── TaskStopTool/                # 任务终止
│   ├── MonitorTool/                 # 监控工具
│   ├── SleepTool/                   # 延迟工具
│   ├── EnterPlanModeTool/           # 进入计划模式
│   └── ExitPlanModeTool/            # 退出计划模式
├── tasks/
│   ├── LocalAgentTask/              # 本地后台智能体 (~682 LOC)
│   ├── RemoteAgentTask/             # 远程智能体 (~855 LOC)
│   ├── InProcessTeammateTask/       # 进程内队友 (~125 LOC)
│   ├── LocalShellTask/              # 本地 Shell 任务
│   ├── LocalWorkflowTask/           # 工作流任务
│   └── MonitorMcpTask/              # MCP 监控任务
├── coordinator/
│   └── coordinatorMode.ts           # 协调器模式 (~370 LOC)
├── bridge/
│   ├── bridgeMain.ts                # 远程桥接主模块 (~2999 LOC)
│   ├── replBridge.ts                # REPL 桥接 (~2406 LOC)
│   └── bridgePermissionCallbacks.ts # 桥接权限回调
├── assistant/
│   └── index.ts                     # KAIROS 助手模式
├── buddy/
│   ├── companion.ts                 # AI 伴侣逻辑
│   ├── prompt.ts                    # 伴侣提示词
│   └── CompanionSprite.tsx          # 伴侣 UI 组件
├── hooks/
│   ├── useCanUseTool.tsx            # 工具权限检查
│   └── useMergedTools.tsx           # 工具合并
├── services/
│   └── compact/                     # 上下文压缩服务
└── utils/
    ├── systemPrompt/                # 系统提示词构建
    └── teammateMailbox.ts           # 队友邮箱系统
```

### 2.2 核心抽象

| 抽象 | 文件 | 职责 |
|------|------|------|
| **QueryEngine** | `QueryEngine.ts` | 每个对话/智能体的查询引擎实例 |
| **query()** | `query.ts` | 工具调用循环编排 |
| **AgentTool** | `tools/AgentTool/` | 智能体生成入口 |
| **runAgent()** | `AgentTool/runAgent.ts` | 智能体执行引擎 |
| **Task** | `Task.ts` | 任务基础状态定义 |
| **LocalAgentTask** | `tasks/LocalAgentTask/` | 本地后台智能体生命周期 |
| **RemoteAgentTask** | `tasks/RemoteAgentTask/` | 远程智能体生命周期 |
| **InProcessTeammateTask** | `tasks/InProcessTeammateTask/` | 进程内队友管理 |
| **SendMessageTool** | `tools/SendMessageTool/` | 智能体间通信 |
| **ProgressTracker** | 任务相关 | 进度追踪 (工具计数/Token/活动) |

---

## 3. 智能体类型与定义

### 3.1 任务类型 (Task Types)

定义于 `src/Task.ts`：

| 类型 | 标识 | 说明 |
|------|------|------|
| `local_bash` | `b` | 本地 Bash 命令执行 |
| `local_agent` | `a` | 本地后台智能体 |
| `remote_agent` | `r` | 远程云端智能体 |
| `in_process_teammate` | `t` | 进程内队友 (Swarm) |
| `local_workflow` | `w` | 本地工作流 |
| `monitor_mcp` | `m` | MCP 服务器监控 |
| `dream` | `d` | 探索性/推测性任务 |

**任务状态机**：

```
pending → running → completed
                 → failed
                 → killed
```

### 3.2 Agent 定义系统

**文件**：`src/tools/AgentTool/loadAgentsDir.ts`

智能体可通过 `.claude/agents/` 目录的 YAML 文件定义：

```yaml
---
type: explorer
model: haiku
tools:
  - Read
  - Grep
  - Glob
mcpServers:
  - github-server
omitClaudeMd: true
---

你是一个代码探索专家。请分析指定的代码文件并提供详细报告。
```

**定义属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `type` | string | 智能体类型标识 |
| `model` | string | 模型覆盖 (继承自父级如省略) |
| `tools` | string[] | 工具白名单 (限制可用工具) |
| `mcpServers` | string[] | 专属 MCP 服务器 |
| `omitClaudeMd` | boolean | 跳过 CLAUDE.md (只读智能体优化) |

**内置智能体**：
- **GeneralPurpose** — 通用智能体 (完整工具池)
- **Explore** — 探索智能体 (只读工具)
- **Plan** — 规划智能体 (计划模式)

### 3.3 AgentId 生成

```typescript
function createAgentId(taskType: TaskType): string {
  // 前缀 = 任务类型首字母 (a/b/r/t/w/m/d)
  // 后缀 = 唯一标识符
  return `${taskType[0]}_${generateUniqueId()}`
}
```

---

## 4. 智能体生命周期

### 4.1 同步智能体

```
AgentTool.call()
    │
    ▼
runAgent() ── 创建子 QueryEngine
    │
    ├── 构建初始消息
    ├── 克隆文件状态缓存
    ├── 获取用户/系统上下文
    ├── 解析智能体工具池
    ├── 构建有效系统提示词
    │
    ▼
query() 工具调用循环
    │
    ├── Claude API 调用 (流式)
    ├── 工具执行
    ├── 上下文压缩 (如需)
    │
    ▼
返回最终结果 → 父会话继续
```

### 4.2 异步智能体

```
AgentTool.call(run_in_background: true)
    │
    ▼
registerAsyncAgent() ── 在 AppState 注册任务
    │
    ▼
立即返回 { status: 'async_launched', agentId }
    │
    ▼ (后台)
runAsyncAgentLifecycle()
    │
    ├── 创建独立 QueryEngine
    ├── 执行查询循环
    ├── updateAsyncAgentProgress() ── 实时进度
    │
    ▼
completeAsyncAgent() 或 failAsyncAgent()
    │
    ▼
enqueueAgentNotification() ── 发送 <task-notification>
```

### 4.3 远程智能体

```
AgentTool.call(isolation: 'remote')
    │
    ▼
checkRemoteAgentEligibility() ── 验证用户资格
    │
    ▼
registerRemoteAgentTask() ── 创建远程任务
    │
    ▼
立即返回 { status: 'async_launched' }
    │
    ▼ (后台轮询)
pollRemoteSessionEvents()
    │
    ├── 获取远程状态
    ├── 检查完成条件
    │
    ▼
任务完成通知
```

### 4.4 进程内队友

```
TeamCreateTool → 创建团队
    │
    ▼
AgentTool(team_name, name) → 生成队友
    │
    ▼
InProcessTeammateTask 注册
    │
    ├── AsyncLocalStorage 隔离
    ├── 独立消息历史
    ├── pendingUserMessages 队列
    │
    ▼
injectUserMessageToTeammate() ← SendMessage 注入
    │
    ▼
appendTeammateMessage() → 显示队友对话
```

---

## 5. 查询引擎 (QueryEngine)

### 5.1 概述

**文件**：`src/QueryEngine.ts` (~1,295 LOC)

`QueryEngine` 是每个对话/智能体的核心驱动引擎。每个智能体实例都拥有自己的 QueryEngine。

### 5.2 配置结构

```typescript
type QueryEngineConfig = {
  // 环境
  cwd: string                         // 工作目录
  tools: Tools                        // 工具池
  commands: Command[]                 // 可用命令
  mcpClients: MCPServerConnection[]   // MCP 连接
  agents: AgentDefinition[]           // 可用智能体定义

  // 权限
  canUseTool: CanUseToolFn            // 权限决策函数

  // 状态
  getAppState: () => AppState         // 状态读取器
  setAppState: (fn) => void           // 状态更新器
  initialMessages?: Message[]         // 初始消息

  // 缓存
  readFileCache: FileStateCache       // 文件读取缓存

  // 提示词
  customSystemPrompt?: string         // 自定义系统提示
  appendSystemPrompt?: string         // 追加系统提示

  // 模型
  userSpecifiedModel?: string         // 用户指定模型
  fallbackModel?: string              // 备用模型

  // 限制
  thinkingConfig?: ThinkingConfig     // 思考配置
  maxTurns?: number                   // 最大轮次
  maxBudgetUsd?: number               // 最大预算 (USD)

  // Schema
  jsonSchema?: Record<string, unknown> // JSON Schema (结构化输出)

  // 回放
  snipReplay?: (msg, store) => { messages, executed } | undefined
}
```

### 5.3 入口方法

```typescript
async *submitMessage(
  prompt: string | UserMessage,
  options?: {
    abortController?: AbortController
    attachments?: Attachment[]
    skipUserContext?: boolean
  }
): AsyncGenerator<SDKMessage>
```

**执行流程**：
1. 处理用户输入（斜杠命令、附件）
2. 构建系统提示 + 用户上下文
3. 产出 `SDKSystemInitMessage`
4. 进入 `query()` 工具调用循环
5. 流式产出助手消息、工具调用和结果
6. 需要时进行上下文压缩
7. 产出最终结果

---

## 6. 工具调用循环 (query)

### 6.1 概述

**文件**：`src/query.ts` (~1,729 LOC)

`query()` 函数是 Claude 与工具交互的核心循环，管理整个工具调用→结果→继续的闭环。

### 6.2 循环流程

```
┌────────────────────────────────────────────────────────┐
│                    query() 循环                         │
│                                                        │
│  1. 准备消息 + 系统提示                                  │
│  2. 调用 Claude API (流式)                              │
│         │                                              │
│         ▼                                              │
│  3. 接收 AssistantMessage                              │
│         │                                              │
│         ├── 无 ToolUse → 返回最终响应 ──→ 退出循环       │
│         │                                              │
│         └── 有 ToolUse → 进入工具执行                    │
│                │                                       │
│                ▼                                       │
│  4. 遍历所有 ToolUseBlock                              │
│         │                                              │
│         ├── CanUseTool() 权限检查                       │
│         │   ├── allow → 执行工具                       │
│         │   ├── deny → 返回拒绝消息                     │
│         │   └── ask → 用户交互审批                      │
│         │                                              │
│         ▼                                              │
│  5. StreamingToolExecutor 并发执行                      │
│         │                                              │
│         ▼                                              │
│  6. 收集 ToolResultBlock                               │
│         │                                              │
│         ▼                                              │
│  7. Token 预算检查                                     │
│         │                                              │
│         ├── 超预算 → 微压缩 (microCompact)               │
│         │                                              │
│         ▼                                              │
│  8. 继续循环 → 回到步骤 2                                │
└────────────────────────────────────────────────────────┘
```

### 6.3 关键特性

- **流式工具执行**：支持并发安全的工具并行执行
- **工具结果预算**：限制工具结果大小，超出时截断
- **微压缩**：在循环内进行轻量级上下文优化
- **后采样钩子**：支持 SessionMemory、MagicDocs 等后处理
- **重试逻辑**：API 调用失败时的指数退避重试
- **取消支持**：通过 AbortController 中断

---

## 7. 智能体执行引擎 (runAgent)

### 7.1 概述

**文件**：`src/tools/AgentTool/runAgent.ts` (~866 LOC)

`runAgent()` 是智能体实际执行的核心引擎，负责创建隔离的执行环境并驱动查询循环。

### 7.2 执行参数

```typescript
export async function* runAgent({
  agentDefinition,        // 智能体定义
  promptMessages,         // 初始提示消息
  toolUseContext,         // 工具使用上下文
  canUseTool,             // 权限决策函数
  isAsync,                // 是否异步
  forkContextMessages?,   // 分叉上下文消息
  querySource,            // 查询来源标识
  override?,              // 覆盖配置
  model?,                 // 模型覆盖
  maxTurns?,              // 最大轮次
  availableTools,         // 可用工具列表
  allowedTools?,          // 工具白名单
  onCacheSafeParams?,     // 缓存安全回调
  worktreePath?,          // Worktree 路径
  description?,           // 智能体描述
  transcriptSubdir?,      // 转录子目录
  onQueryProgress?        // 进度回调
}): AsyncGenerator<Message, void>
```

### 7.3 执行流程

```
runAgent() 调用
│
├── 1. 创建 Agent Identity (agentId)
├── 2. 注册 Perfetto 追踪
├── 3. 构建初始消息（从上下文 + 提示）
├── 4. 克隆父级文件状态缓存 (或创建新的)
├── 5. 获取用户上下文 & 系统上下文
├── 6. 是否省略 CLAUDE.md (只读智能体优化)
├── 7. 初始化智能体专属 MCP 服务器
├── 8. 解析智能体工具池 → resolveAgentTools()
├── 9. 构建有效系统提示词
├── 10. 创建子智能体上下文 & QueryEngine
│
▼
query() 循环执行 → 产出消息流
│
▼
注册/注销 Perfetto agent 完成
```

### 7.4 工具解析 (resolveAgentTools)

**文件**：`src/tools/AgentTool/agentToolUtils.ts`

```
resolveAgentTools(agentDefinition, availableTools)
│
├── 检查智能体定义中的 tools 白名单
├── 过滤可用工具列表
├── 应用权限模式（只读智能体跳过写入工具）
├── 检查 MCP 服务器可用性
└── 返回最终工具池
```

---

## 8. 任务框架

### 8.1 LocalAgentTask — 本地后台智能体

**文件**：`src/tasks/LocalAgentTask/LocalAgentTask.tsx` (~682 LOC)

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string               // 智能体 ID
  prompt: string                // 原始提示
  agentType: string             // 智能体类型
  model?: string                // 使用的模型
  error?: string                // 错误信息
  result?: AgentToolResult      // 最终结果
  progress?: AgentProgress      // 实时进度
  isBackgrounded: boolean       // 是否后台化
  pendingMessages: string[]     // 队列中的 SendMessage
  retain: boolean               // UI 是否保留此任务
  diskLoaded: boolean           // 转录是否加载完成
  evictAfter?: number           // GC 截止时间
}
```

**生命周期方法**：

| 方法 | 说明 |
|------|------|
| `registerAsyncAgent()` | 在 AppState 注册任务 |
| `runAsyncAgentLifecycle()` | 执行智能体查询循环 |
| `updateAsyncAgentProgress()` | 报告工具/Token 进度 |
| `completeAsyncAgent()` | 标记完成 |
| `failAsyncAgent()` | 标记失败 |
| `enqueueAgentNotification()` | 队列任务通知 |

**进度追踪**：

```typescript
type ProgressTracker = {
  toolUseCount: number          // 工具调用次数
  inputTokens: number           // 输入 Token 数
  outputTokens: number          // 输出 Token 数
  recentActivities: Activity[]  // 最近活动列表
}
```

通过 `emitTaskProgress()` 将进度发送到 UI 和 SDK。

### 8.2 RemoteAgentTask — 远程智能体

**文件**：`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` (~855 LOC)

```typescript
// 远程任务类型
type RemoteTaskType =
  | 'remote-agent'     // 通用远程执行
  | 'ultraplan'        // 规划阶段
  | 'ultrareview'      // 代码审查
  | 'autofix-pr'       // PR 自动修复
  | 'background-pr'    // 后台 PR 操作
```

**执行流程**：
1. `checkRemoteAgentEligibility()` — 验证用户登录 + 包可用性
2. `registerRemoteAgentTask()` — 创建远程任务
3. `pollRemoteSessionEvents()` — 轮询远程状态与完成
4. 可插拔的完成检查器 (per task type)

### 8.3 InProcessTeammateTask — 进程内队友

**文件**：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` (~125 LOC)

```typescript
type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: {
    agentId: string             // 智能体 ID
    agentName: string           // 智能体名称
    teamName: string            // 所属团队
  }
  messages: Message[]           // 对话历史
  pendingUserMessages: string[] // 待处理消息队列
  shutdownRequested?: boolean   // 是否请求关闭
}
```

**关键操作**：
- `injectUserMessageToTeammate()` — 注入消息到队友
- `appendTeammateMessage()` — 追加队友对话
- `findTeammateTaskByAgentId()` — 按 ID 查找（优先运行中的）
- `getRunningTeammatesSorted()` — 获取排序后的运行中队友

### 8.4 其他任务类型

| 类型 | 说明 |
|------|------|
| **LocalShellTask** | Bash 命令执行任务 |
| **LocalWorkflowTask** | 工作流自动化任务 |
| **MonitorMcpTask** | MCP 服务器监控任务 |
| **DreamTask** | 推测性/探索性任务 |

---

## 9. 多智能体协调 (Coordinator)

### 9.1 概述

**文件**：`src/coordinator/coordinatorMode.ts` (~370 LOC)

Coordinator 模式实现了一个协调器指挥多个工作者的多智能体编排系统。

### 9.2 架构

```
┌──────────────────────────────────────────────────────┐
│                   Coordinator (协调器)                  │
│                                                      │
│  系统提示增强:                                         │
│  - 并行指导                                           │
│  - 综合模式                                           │
│  - Worker 管理规则                                    │
│                                                      │
│  内部工具 (对 Worker 隐藏):                             │
│  - TEAM_CREATE                                       │
│  - TEAM_DELETE                                       │
│  - SEND_MESSAGE                                      │
│  - SYNTHETIC_OUTPUT                                  │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Worker 1 │  │ Worker 2 │  │ Worker N │           │
│  │ 研究任务  │  │ 实现任务  │  │ 测试任务  │           │
│  │ 标准工具  │  │ 标准工具  │  │ 标准工具  │           │
│  │ + MCP    │  │ + MCP    │  │ + MCP    │           │
│  │ + Skills │  │ + Skills │  │ + Skills │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │              │              │                 │
│       └──────────────┼──────────────┘                 │
│                      │                                │
│              <task-notification>                       │
│              汇报结果到协调器                             │
└──────────────────────────────────────────────────────┘
```

### 9.3 协调器系统提示增强

Coordinator 模式会向系统提示注入特殊指令：

- **并行策略**：指导何时并行生成多个 Worker
- **研究→实现模式**：先并行研究，再顺序实现
- **综合策略**：如何将多个 Worker 的结果合并为最终输出
- **Worker 管理规则**：Worker 超时、失败重试、资源限制

### 9.4 Worker 通信

1. Coordinator 通过 `AgentTool` 生成 Worker
2. Worker 通过 `<task-notification>` XML 汇报结果
3. Coordinator 收到通知后进行决策和综合
4. 支持广播消息 (`SendMessage to: "*"`)

---

## 10. 智能体间通信

### 10.1 SendMessageTool

**文件**：`src/tools/SendMessageTool/SendMessageTool.ts` (~917 LOC)

**路由逻辑**：

```
SendMessage(to, message)
│
├── to === "teammate_name"
│   └── 在 tasks 中查找 InProcessTeammateTask
│       └── injectUserMessageToTeammate()
│
├── to === "*"  (广播)
│   └── 遍历所有 InProcessTeammateTask
│       └── 每个队友注入消息
│
├── to === "uds:<path>"  (Unix Domain Socket)
│   └── UDS 协议发送到外部 Peer
│
└── to === "bridge:<id>"  (Bridge)
    └── Bridge 协议发送到远程控制端
```

### 10.2 结构化消息

```typescript
type StructuredMessage =
  | { type: 'shutdown_request' }         // 请求停止
  | { type: 'shutdown_response',         // 停止响应
      approved: boolean }
  | { type: 'plan_approval_response',    // 计划审批
      approved: boolean,
      feedback?: string }
```

### 10.3 邮箱系统

**文件**：`src/utils/teammateMailbox.ts`

提供异步 IPC 机制：

| 函数 | 说明 |
|------|------|
| `writeToMailbox()` | 队友写入共享邮箱 |
| `createShutdownRequestMessage()` | 创建停止请求消息 |
| `createShutdownApprovedMessage()` | 创建停止审批消息 |
| `createShutdownRejectedMessage()` | 创建停止拒绝消息 |

### 10.4 任务通知

后台智能体完成时，通过 `enqueueAgentNotification()` 向父会话发送：

```xml
<task-notification agent-id="a_xxxxx" status="completed">
  任务结果内容...
</task-notification>
```

该通知以 User 角色消息注入父会话，触发 Coordinator 的后续决策。

---

## 11. 权限模型

### 11.1 CanUseToolFn

**文件**：`src/hooks/useCanUseTool.tsx`

```typescript
type CanUseToolFn = (
  tool: Tool,
  input: Record<string, unknown>,
  toolUseContext: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: PermissionDecision
) => Promise<PermissionDecision>
```

**决策类型**：
- `allow` — 允许执行（可附带 `updatedInput` 净化输入）
- `deny` — 拒绝执行
- `ask` — 请求用户审批

### 11.2 三层权限

```
┌────────────────────────────────────────────────┐
│ 层级 1: 工具池组装 (什么工具存在于此智能体)       │
│                                                │
│  - Agent 定义的 tools 白名单                    │
│  - 只读智能体跳过写入工具                       │
│  - Feature Gate 条件工具                        │
├────────────────────────────────────────────────┤
│ 层级 2: 权限模式 (默认行为)                      │
│                                                │
│  - manual: 每次都询问                           │
│  - ask: 默认询问，可记住选择                     │
│  - auto: 自动审批 (YOLO 分类器)                 │
│  - bypass: 绕过所有检查                         │
│  - plan: 仅在计划审批后执行                      │
├────────────────────────────────────────────────┤
│ 层级 3: 权限规则 (per-tool 粒度)                 │
│                                                │
│  - alwaysAllowRules: 预审批的命令/工具           │
│  - 文件系统路径规则                              │
│  - Shell 命令模式匹配                           │
│  - 危险模式检测 (bashClassifier)                │
└────────────────────────────────────────────────┘
```

### 11.3 权限处理器

| 处理器 | 场景 | 说明 |
|--------|------|------|
| `interactiveHandler` | REPL 模式 | 显示 UI 提示让用户决定 |
| `coordinatorHandler` | Coordinator 模式 | 等待自动化检查后再对话 |
| `swarmWorkerHandler` | Swarm Worker | Worker 专用权限规则 |

---

## 12. 上下文管理

### 12.1 文件状态缓存

**文件**：`src/utils/fileStateCache.ts`

- 每个智能体继承或克隆父级的文件缓存
- 缓存大小限制 (`READ_FILE_STATE_CACHE_SIZE`)
- 所有文件读取工具共享此缓存
- 避免重复读取相同文件

### 12.2 上下文压缩

**文件**：`src/services/compact/`

| 类型 | 文件 | 说明 |
|------|------|------|
| `autoCompact` | `autoCompact.ts` | Token 阈值自动触发 |
| `compact` | `compact.ts` | 主压缩算法 (分叉智能体执行) |
| `microCompact` | `microCompact.ts` | 循环内轻量压缩 |
| `sessionMemoryCompact` | `sessionMemoryCompact.ts` | 记忆感知压缩 |
| `snipCompact` | — | 片段级历史压缩 |

**压缩标记**：

```xml
<system subtype="compact_boundary">
  此处标记压缩边界
</system>
```

QueryEngine 的 `snipReplay` 回调用于 headless 场景的裁剪。

### 12.3 系统提示构建

**文件**：`src/utils/queryContext.ts`, `src/utils/systemPrompt.ts`

`fetchSystemPromptParts()` 收集：
- 工具描述和使用说明
- 可用命令列表
- 权限模式说明
- 智能体专属 MCP 服务器列表
- Skill 工具枚举
- CLAUDE.md 记忆文件内容

### 12.4 智能体记忆

**文件**：`src/tools/AgentTool/agentMemory.ts`, `agentMemorySnapshot.ts`

- 每个智能体的记忆快照存储在转录旁边
- 用于后续智能体生成时的上下文注入
- 支持跨会话的智能体记忆持久化

---

## 13. Bridge 远程控制

### 13.1 概述

**文件**：`src/bridge/bridgeMain.ts` (~2,999 LOC), `src/bridge/replBridge.ts` (~2,406 LOC)

Bridge 模式将本地 CLI 智能体连接到 claude.ai Web 界面，实现远程控制。

### 13.2 架构

```
┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
│  claude.ai   │ ◄─────► │  Bridge Server   │ ◄─────► │  Local CLI   │
│  Web 界面    │  HTTP/   │  (Orchestrator)  │  API/   │  Agent 进程  │
│              │  WS      │                  │  Poll   │              │
└──────────────┘         └──────────────────┘         └──────────────┘
```

### 13.3 组件

| 组件 | 说明 |
|------|------|
| `BridgeApiClient` | HTTP 客户端连接到 Orchestrator |
| `SessionSpawner` | 生成隔离的 Agent 子进程 |
| `BridgeLogger` | 状态更新和显示 |
| `JWT Token Refresh` | 定期刷新 JWT Token |
| `Session Poll Loop` | 获取入站提示和控制消息 |
| `Error Budget` | 指数退避错误预算 |

### 13.4 远程控制 Handle

```typescript
type RemoteControlHandle = {
  write(msg: SDKMessage): void               // 发送消息到 Orchestrator
  sendResult(): void                         // 标记工作完成
  sendControlRequest(req): void              // 发送控制消息
  inboundPrompts(): AsyncGenerator<UserMsg>  // 接收用户消息流
  controlRequests(): AsyncGenerator<CtrlReq> // 接收控制请求流
  permissionResponses(): AsyncGenerator<Dec> // 接收权限决策流
  onStateChange(cb): void                    // 监听连接状态变化
}
```

### 13.5 通信流

1. Agent 子进程通过 `SessionSpawner` 生成
2. JWT Token 通过 `/work-secret` 传递
3. WebSocket 连接到 Orchestrator
4. 轮询入站提示 → 注入 Agent
5. Agent 产出消息 → 发送到 Orchestrator
6. 接收控制消息 (中断/模型切换等)

---

## 14. KAIROS 助手模式

### 14.1 计划任务

**文件**：`src/entrypoints/agentSdkTypes.ts`

```typescript
type CronTask = {
  id: string           // 任务 ID
  cron: string         // Cron 表达式
  prompt: string       // 执行提示
  createdAt: number    // 创建时间
  recurring?: boolean  // 是否循环
}

type ScheduledTaskEvent =
  | { type: 'fire'; task: CronTask }    // 触发
  | { type: 'missed'; tasks: CronTask[] } // 错过
```

### 14.2 调度配置

```typescript
type CronJitterConfig = {
  recurringFrac: number        // 循环任务抖动比例
  recurringCapMs: number       // 循环任务抖动上限
  oneShotMaxMs: number         // 一次性任务最大抖动
  oneShotFloorMs: number       // 一次性任务最小抖动
  oneShotMinuteMod: number     // 分钟级取模
  recurringMaxAgeMs: number    // 循环任务老化阈值
}
```

### 14.3 主动功能

- 基于 Cron 表达式的定时执行
- 推送通知 (`PushNotificationTool`)
- 主动建议 (`PromptSuggestion` 服务)
- 后台自主操作
- 守护进程模式 (`DAEMON`)

---

## 15. Buddy AI 伴侣

### 15.1 结构

**目录**：`src/buddy/`

| 文件 | 说明 |
|------|------|
| `companion.ts` | 核心伴侣逻辑 |
| `prompt.ts` | 系统提示词 |
| `CompanionSprite.tsx` | UI 精灵组件 |
| `types.ts` | 类型定义 |
| `useBuddyNotification.tsx` | 通知 Hook |

### 15.2 设计

- 轻量级会话内助手
- 对用户操作的响应式交互
- 独立的对话轨道（不影响主对话）
- 特性门控：`BUDDY`

---

## 16. SDK 集成

### 16.1 SDK 类型

**文件**：`src/entrypoints/agentSdkTypes.ts`

核心导出：
- `SDKMessage` — 消息类型联合
- `SDKUserMessage` — 用户消息
- `SDKAssistantMessage` — 助手消息
- `SDKControlRequest` — 控制请求
- `SDKControlResponse` — 控制响应

### 16.2 SDK 入口

| 函数 | 说明 |
|------|------|
| `query()` | 单次查询 |
| `unstable_v2_createSession()` | 创建持久会话 |
| `getSessionMessages()` | 获取会话消息 |

### 16.3 SDK 构建器

**目录**：`src/entrypoints/sdk/`

包含核心类型、运行时类型、工具类型、控制类型和设置的 SDK 构建模块。

---

## 17. 设计模式总结

### 17.1 异步生成器模式

```typescript
async function* pattern(): AsyncGenerator<Message> {
  yield startMessage
  for await (const response of apiCall()) {
    yield processedMessage
  }
  yield finalMessage
}
```

**应用**：`submitMessage()`、`query()`、`runAgent()` — 整个消息管道都基于此模式。

**优势**：
- 流式处理和实时更新
- 生产者-消费者解耦
- 惰性求值（按需产出）
- 支持取消 (AbortController)

### 17.2 上下文隔离模式

每个智能体拥有独立的：
- QueryEngine 实例
- 文件状态缓存（克隆）
- 工具池
- 消息历史
- 权限上下文

通过 `AsyncLocalStorage` 实现进程内队友的额外隔离。

### 17.3 进度追踪模式

```
ProgressTracker → emitTaskProgress() → UI/SDK
```

即使后台智能体也能实时报告：
- 工具使用计数
- Token 消耗
- 最近活动列表

### 17.4 层次化权限模式

```
工具池过滤 → 权限模式 → 权限规则 → 用户确认
```

每层逐步收窄可用能力。

### 17.5 递归嵌套模式

```
父会话 → Agent → 子 Agent → 子子 Agent → ...
```

无限递归支持，每层独立的 QueryEngine 和上下文。

### 17.6 消息传递模式

```
SendMessage → 路由查找 → 邮箱注入 → 异步处理
```

支持点对点、广播、UDS 和 Bridge 等多种路由方式。

### 17.7 关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 每个 Agent 独立 QueryEngine | 完全隔离 | 避免状态污染，支持并行 |
| AsyncGenerator 管道 | 流式处理 | 实时响应，内存高效 |
| 文件缓存克隆 | 继承+独立 | 减少重复 IO，保证隔离 |
| XML 通知格式 | `<task-notification>` | 与 LLM 上下文兼容 |
| Zod Schema 验证 | 类型安全 | 运行时输入验证 + TypeScript 推导 |
| Feature Gate 编译 | 死代码消除 | 减小最终包体积 |

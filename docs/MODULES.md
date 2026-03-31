# Claude Code 模块详细设计

> 按文件夹逐一分析每个模块的职责、关键文件、核心抽象与模块关系

---

## 目录

- [核心顶层文件](#核心顶层文件)
- [entrypoints/ — 启动入口](#1-entrypoints--启动入口)
- [commands/ — CLI 命令](#2-commands--cli-命令)
- [tools/ — 智能体工具](#3-tools--智能体工具)
- [services/ — 服务层](#4-services--服务层)
- [utils/ — 工具函数库](#5-utils--工具函数库)
- [state/ — 状态管理](#6-state--状态管理)
- [components/ — React UI 组件](#7-components--react-ui-组件)
- [screens/ — UI 屏幕](#8-screens--ui-屏幕)
- [hooks/ — React Hooks](#9-hooks--react-hooks)
- [types/ — 类型定义](#10-types--类型定义)
- [context/ — React Context](#11-context--react-context)
- [ink/ — 终端 UI 渲染层](#12-ink--终端-ui-渲染层)
- [bridge/ — 远程桥接](#13-bridge--远程桥接)
- [cli/ — CLI 基础设施](#14-cli--cli-基础设施)
- [tasks/ — 任务执行框架](#15-tasks--任务执行框架)
- [coordinator/ — 多智能体协调](#16-coordinator--多智能体协调)
- [skills/ — 技能系统](#17-skills--技能系统)
- [plugins/ — 插件系统](#18-plugins--插件系统)
- [constants/ — 全局常量](#19-constants--全局常量)
- [bootstrap/ — 启动引导](#20-bootstrap--启动引导)
- [schemas/ — 数据校验](#21-schemas--数据校验)
- [migrations/ — 数据迁移](#22-migrations--数据迁移)
- [keybindings/ — 键盘绑定](#23-keybindings--键盘绑定)
- [native-ts/ — 原生模块绑定](#24-native-ts--原生模块绑定)
- [其他模块](#25-其他模块)

---

## 核心顶层文件

位于 `src/` 根目录的核心编排文件：

### main.tsx (4,683 LOC) — CLI 主入口

| 属性 | 说明 |
|------|------|
| **职责** | CLI 主入口，模块编排与启动引导 |
| **关键逻辑** | 性能检查点、快速路径（`--version`/`--dump-system-prompt`/MCP）、动态导入 Feature Gate 模块、Bootstrap 命令加载、子命令路由 |
| **依赖** | entrypoints/cli.tsx, entrypoints/init.ts, coordinator, KAIROS |
| **被依赖** | package.json bin 入口 |

### QueryEngine.ts (1,295 LOC) — 查询引擎

| 属性 | 说明 |
|------|------|
| **职责** | 驱动每个对话/智能体的查询处理 |
| **核心方法** | `submitMessage()` — AsyncGenerator 入口 |
| **配置** | cwd, tools, commands, mcpClients, agents, canUseTool, state |
| **依赖** | query.ts, services/api, services/compact, utils/systemPrompt |

### query.ts (1,729 LOC) — 工具调用循环

| 属性 | 说明 |
|------|------|
| **职责** | 编排工具调用循环：API 调用 → 工具执行 → 结果回传 |
| **核心函数** | `query()` — AsyncGenerator |
| **管理** | 工具可用性检查、流式执行、结果预算、微压缩、后采样钩子、重试 |

### Tool.ts (792 LOC) — 工具抽象基类

| 属性 | 说明 |
|------|------|
| **职责** | 定义所有工具的统一抽象接口 |
| **核心类型** | `Tool`, `ToolUseContext`, `ToolResult`, `ToolInputJSONSchema` |
| **扩展** | 所有 43+ 工具实现此接口 |

### commands.ts (754 LOC) — 命令注册表

| 属性 | 说明 |
|------|------|
| **职责** | 汇总所有命令来源，提供记忆化命令列表 |
| **来源** | 内置命令、根级文件、Skill、Plugin、工作流、MCP |
| **核心函数** | `COMMANDS()` — 记忆化命令数组 |

### tools.ts (389 LOC) — 工具集合管理器

| 属性 | 说明 |
|------|------|
| **职责** | 汇总内置工具、MCP 工具、Skill 工具，应用 Feature Gate |
| **核心函数** | `getTools()` |

### context.ts — 上下文构建

| 属性 | 说明 |
|------|------|
| **职责** | Git 状态提取、文件树生成、记忆文件注入、系统提示构建 |
| **缓存** | 记忆化 Git 状态和文件树 |

### history.ts (464 LOC) — 会话历史

| 属性 | 说明 |
|------|------|
| **职责** | 粘贴内容管理、历史格式化/解析、`[Pasted text #N]` 引用 |
| **持久化** | 历史写入磁盘 |

### Task.ts (125 LOC) — 任务定义

| 属性 | 说明 |
|------|------|
| **职责** | 任务类型定义、状态机、ID 生成 |
| **类型** | `local_bash`, `local_agent`, `remote_agent`, `in_process_teammate` 等 |

### cost-tracker.ts — Token 追踪

| 属性 | 说明 |
|------|------|
| **职责** | Token 使用量和 API 费用追踪 |

---

## 1. entrypoints/ — 启动入口

**大小**：~53K LOC | **文件数**：7+ | **职责**：应用引导、CLI 初始化、SDK 入口

### 关键文件

| 文件 | LOC | 职责 |
|------|-----|------|
| `cli.tsx` | ~39,275 | CLI 主逻辑（因源码映射还原为单文件，实际为多模块合并）：进程环境设置、版本快速路径、Daemon Worker 路径、配置初始化、优雅关闭 |
| `init.ts` | 13,780 | 核心初始化：配置加载、信任对话、MDM 预取、OAuth/API Key 认证、策略限制、遥测、GrowthBook |
| `agentSdkTypes.ts` | — | SDK 类型定义：SDKMessage、Query 接口、控制类型、计划任务 |
| `mcp.ts` | — | MCP 协议入口集成 |
| `sdk/` | — | SDK 构建层：核心类型、运行时类型、工具类型、控制类型、设置 |

### 模块关系

```
main.tsx ──→ cli.tsx ──→ init.ts ──→ [认证/遥测/配置]
                │
                ├──→ 正常模式 → REPL 渲染
                ├──→ SDK 模式 → agentSdkTypes.ts
                ├──→ Daemon 模式 → Worker 路径
                └──→ MCP 模式 → mcp.ts
```

---

## 2. commands/ — CLI 命令

**大小**：3.3 MB | **子目录**：86+ | **职责**：所有 CLI 命令实现

### 目录结构

每个命令一个子目录，包含 `index.ts` 或 `index.tsx` 入口文件。

### 分类总览

| 分类 | 命令数 | 示例 |
|------|--------|------|
| 会话管理 | 10 | add-dir, clear, compact, context, diff, export, plan, rename, resume, tag |
| 模型配置 | 7 | effort, fast, model, advisor, color, theme, output-style |
| 状态信息 | 4 | cost, status, usage, version |
| 设置 | 6 | config, help, keybindings, memory, permissions, vim |
| 认证 | 3 | login, logout, oauth-refresh |
| 订阅 | 5 | upgrade, passes, extra-usage, rate-limit-options, privacy-settings |
| 集成 | 6 | install-github-app, install-slack-app, ide, desktop, mobile, chrome |
| 开发 | 7 | doctor, plugin, reload-plugins, skills, hooks, mcp, files |
| 审查 | 3 | review, ultrareview, security-review |
| 通信 | 4 | feedback, btw, copy, tasks |
| 协作 | 4 | branch, remote-control, session, remote-env |
| 项目 | 2 | init, init-verifiers |
| 内部 | 16+ | commit, autofix-pr, bughunter, teleport 等 |

> 详细命令列表见 [COMMANDS.md](./COMMANDS.md)

### 模块关系

```
commands.ts (注册表) ←── commands/*/index.ts (实现)
                     ←── skills/bundled/ (技能命令)
                     ←── plugins/ (插件命令)
```

---

## 3. tools/ — 智能体工具

**大小**：3.2 MB | **子目录**：43+ | **职责**：所有智能体工具实现

### 目录结构

每个工具一个子目录，包含主实现文件和辅助文件。

### 分类总览

| 分类 | 数量 | 工具 |
|------|------|------|
| 文件操作 | 5 | FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool |
| 代码导航 | 1 | LSPTool |
| Shell 执行 | 3 | BashTool, PowerShell, REPLTool |
| 规划 | 2 | EnterPlanModeTool, ExitPlanModeTool |
| 工作树 | 2 | EnterWorktreeTool, ExitWorktreeTool |
| 多智能体 | 5 | AgentTool, SendMessageTool, TeamCreateTool, TeamDeleteTool, SkillTool |
| 任务管理 | 7 | TaskCreate~TaskStop, TodoWriteTool |
| 用户交互 | 2 | AskUserQuestionTool, SendUserMessage |
| Web | 2 | WebFetchTool, WebSearchTool |
| Notebook | 1 | NotebookEditTool |
| MCP | 4 | ListMcpResourcesTool, ReadMcpResourceTool, MCPTool, McpAuthTool |
| 调度 | 4 | CronCreate~CronList, RemoteTrigger |
| 实用 | 3 | ToolSearch, SleepTool, StructuredOutput |

> 详细工具列表见 [TOOLS.md](./TOOLS.md)

### 模块关系

```
Tool.ts (抽象基类) ←── tools/*/index.ts (实现)
tools.ts (集合) ──→ 汇总 → QueryEngine
```

---

## 4. services/ — 服务层

**大小**：2.2 MB | **子目录**：21+ | **职责**：核心业务服务

### 4.1 API 服务 (`services/api/`)

| 文件 | 职责 |
|------|------|
| `claude.ts` | Claude API 客户端：消息流、工具处理、响应标准化 |
| `client.ts` | 多 Provider 工厂 (Direct/Bedrock/Foundry/Vertex) + 认证路由 |
| `bootstrap.ts` | 模型选项和配置 Bootstrap |
| `filesApi.ts` | Public Files API 文件上传/下载 |
| `logging.ts` | API 调用日志、计时、Token 计数、错误追踪 |
| `errors.ts` | API 错误分类和用户提示 |
| `usage.ts` | Token 使用量和费用计算 |
| `withRetry.ts` | 指数退避重试 |
| `promptCacheBreakDetection.ts` | 提示缓存失效检测 |
| `referral.ts` | 推荐计划集成 |
| `sessionIngress.ts` | 会话入口认证 |

**LOC**：~10,477 | **集成**：认证系统、模型选择、错误追踪

### 4.2 MCP 服务 (`services/mcp/`)

| 文件 | 职责 |
|------|------|
| `client.ts` | MCP 客户端编排器：传输管理、工具/资源枚举 |
| `types.ts` | Zod Schema：stdio/SSE/HTTP/WebSocket/SDK 传输 |
| `config.ts` | MCP 服务器配置加载与验证 |
| `MCPConnectionManager.tsx` | React 组件：MCP 连接生命周期管理 |
| `officialRegistry.ts` | 官方 MCP 注册表获取与缓存 |
| `auth.ts` | MCP 服务器 OAuth 认证 |
| `channelPermissions.ts` | 通道级权限处理 |
| `channelAllowlist.ts` | 通道访问控制列表 |
| `elicitationHandler.ts` | 用户引导请求处理 |
| `InProcessTransport.ts` | 进程内 MCP 传输 |
| `SdkControlTransport.ts` | SDK 控制传输 |

**文件数**：23 | **集成**：认证、权限、工具注册

### 4.3 Analytics 分析 (`services/analytics/`)

| 文件 | 职责 |
|------|------|
| `index.ts` | 事件日志 API（零依赖设计避免循环导入） |
| `datadog.ts` | Datadog 事件投递 |
| `firstPartyEventLogger.ts` | 第一方事件日志 |
| `growthbook.ts` | GrowthBook Feature Flag + A/B 测试 |
| `sink.ts` | 事件 Sink 路由（批量、多后端投递） |
| `sinkKillswitch.ts` | 隐私模式 Kill Switch |
| `metadata.ts` | 事件元数据丰富 + PII 过滤 |

**关键设计**：启动前事件排队、Sink 延迟附着、PII 验证类型

### 4.4 LSP 服务 (`services/lsp/`)

| 文件 | 职责 |
|------|------|
| `LSPClient.ts` | LSP 客户端 (vscode-jsonrpc stdio) |
| `LSPServerManager.ts` | 多 LSP 服务器实例管理 |
| `LSPServerInstance.ts` | 单个 LSP 服务器生命周期 |
| `LSPDiagnosticRegistry.ts` | 诊断信息聚合 |
| `config.ts` | LSP 服务器配置 |
| `passiveFeedback.ts` | 被动反馈收集 |

### 4.5 OAuth 服务 (`services/oauth/`)

| 文件 | 职责 |
|------|------|
| `client.ts` | OAuth 流程编排 (PKCE) |
| `auth-code-listener.ts` | 本地回调服务器 |
| `getOauthProfile.ts` | OAuth Token → 用户 Profile |
| `crypto.ts` | PKCE + 加密工具 |

### 4.6 Compact 压缩 (`services/compact/`)

| 文件 | 职责 |
|------|------|
| `compact.ts` | 主压缩编排 + pre/post hooks |
| `autoCompact.ts` | Token 阈值自动触发 |
| `microCompact.ts` | 循环内轻量压缩 |
| `grouping.ts` | 消息分组策略 |
| `prompt.ts` | 压缩提示词生成 |
| `sessionMemoryCompact.ts` | 会话记忆压缩 |
| `postCompactCleanup.ts` | 压缩后清理 |

### 4.7 其他服务

| 服务 | 目录 | 职责 |
|------|------|------|
| **SessionMemory** | `services/SessionMemory/` | 后台 Markdown 笔记自动维护 |
| **MagicDocs** | `services/MagicDocs/` | 自动更新带 `# MAGIC DOC:` 标记的文档 |
| **PromptSuggestion** | `services/PromptSuggestion/` | 上下文提示建议 + 推测执行 |
| **RemoteManagedSettings** | `services/remoteManagedSettings/` | 企业远程设置 (ETag 缓存) |
| **PolicyLimits** | `services/policyLimits/` | 组织级策略限制 |
| **Plugins** | `services/plugins/` | 插件安装管理器 |
| **AgentSummary** | `services/AgentSummary/` | 智能体执行摘要 |
| **extractMemories** | `services/extractMemories/` | 对话记忆提取 |
| **settingsSync** | `services/settingsSync/` | 跨实例设置同步 |
| **teamMemorySync** | `services/teamMemorySync/` | 团队记忆同步 |

### 根级服务文件

| 文件 | 职责 |
|------|------|
| `claudeAiLimits.ts` (16.8K) | Claude.ai 速率限制管理 |
| `mockRateLimits.ts` (29.7K) | 限速模拟测试 |
| `diagnosticTracking.ts` | 诊断事件追踪 |
| `rateLimitMessages.ts` | 限速消息 |
| `awaySummary.ts` | 离开摘要 |
| `tokenEstimation.ts` | Token 估算 |
| `notifier.ts` | 系统通知 |
| `preventSleep.ts` | 防止系统休眠 |
| `voice.ts` / `voiceStreamSTT.ts` | 语音处理 |

---

## 5. utils/ — 工具函数库

**大小**：7.8 MB | **目录**：33+ | **职责**：基础设施工具函数

### 5.1 permissions/ — 权限系统 (24 文件)

| 文件 | 职责 |
|------|------|
| `PermissionMode.ts` | 权限模式：manual/ask/auto/bypass |
| `PermissionResult.ts` | 权限决策结构 |
| `PermissionRule.ts` | 权限规则定义与来源 |
| `bashClassifier.ts` | 危险 Bash 命令检测 |
| `yoloClassifier.ts` | 激进自动分类 (YOLO 模式) |
| `dangerousPatterns.ts` | 危险操作模式数据库 |
| `filesystem.ts` | 文件路径验证与权限 |
| `pathValidation.ts` | 路径标准化与验证 |
| `permissionExplainer.ts` | 用户可读权限说明 |
| `permissionRuleParser.ts` | 权限规则 DSL 解析 |
| `permissions.ts` | 核心权限引擎 |
| `shellRuleMatching.ts` | Shell 命令模式匹配 |
| `classifierDecision.ts` | 基于转录的分类器决策 |
| `autoModeState.ts` | 自动模式分类状态 |
| `denialTracking.ts` | 拒绝权限追踪 |

### 5.2 settings/ — 设置管理 (19 文件)

| 文件 | 职责 |
|------|------|
| `settings.ts` | 主设置加载器 (优先级排序) |
| `types.ts` | 设置 JSON Schema (Zod) |
| `validation.ts` | Schema 验证与错误报告 |
| `changeDetector.ts` | 文件监听器 |
| `applySettingsChange.ts` | 设置变更应用 |
| `permissionValidation.ts` | 权限规则验证 |
| `mdm/` | MDM 集成 (macOS/Windows 注册表) |

### 5.3 model/ — 模型选择 (16 文件)

| 文件 | 职责 |
|------|------|
| `model.ts` | 模型选择与能力检测 |
| `providers.ts` | Provider 路由 (Direct/Bedrock/Foundry/Vertex) |
| `modelCapabilities.ts` | 模型特性矩阵 (思考/视觉等) |
| `bedrock.ts` | AWS Bedrock 特定工具 |
| `aliases.ts` | 模型名称别名 |
| `validateModel.ts` | 模型白名单验证 |
| `deprecation.ts` | 弃用模型处理 |

### 5.4 systemPrompt/ — 系统提示词

| 职责 |
|------|
| 工具描述和使用说明汇总 |
| 命令列表注入 |
| 权限模式说明 |
| MCP 服务器列表 |
| Skill 枚举 |
| CLAUDE.md 内容注入 |

### 5.5 processUserInput/ — 输入处理

| 文件 | 职责 |
|------|------|
| `processUserInput.ts` | 输入类型分发 |
| `processTextPrompt.ts` | 文本提示解析 |
| `processBashCommand.tsx` | Bash 命令执行 |
| `processSlashCommand.tsx` | 斜杠命令处理 |

### 5.6 核心工具文件

| 文件 | 职责 |
|------|------|
| `auth.ts` | OAuth Token 管理、API Key 管理、AWS/GCP 凭证、Keychain |
| `git.ts` | 仓库检测、分支追踪、状态获取、Worktree 检测 |
| `file.ts` | 安全读写、修改时间追踪、编码检测、二进制检测 |
| `Shell.ts` | Shell Provider 抽象、命令执行、沙箱、CWD 管理 |
| `config.ts` | 配置文件加载、优先级排序、锁文件并发控制 |
| `api.ts` | API 工具函数 |
| `cwd.ts` | 工作目录追踪 |
| `messages.ts` | 消息操作工具 |
| `teammateMailbox.ts` | 队友邮箱 IPC 系统 |
| `fileStateCache.ts` | 文件状态缓存 |

---

## 6. state/ — 状态管理

**大小**：76 KB | **文件数**：3 | **职责**：集中式应用状态

### 关键文件

| 文件 | 职责 |
|------|------|
| `AppStateStore.ts` | 完整状态类型定义：设置/模型/UI/消息/权限/插件/任务 |
| `AppState.tsx` | React Context Provider：不可变更新、设置变更检测、子 Provider |
| `store.ts` | Store 工厂 |

### 核心状态结构

```typescript
type AppState = {
  settings: SettingsJson
  modelSelection: ModelState
  uiState: UIState
  messages: Message[]
  speculation: SpeculationState
  permissions: ToolPermissionContext
  pluginStatus: PluginState
  tasks: TaskState[]
  footerItems: FooterItem[]
}
```

---

## 7. components/ — React UI 组件

**大小**：11 MB | **组件数**：33+ | **职责**：终端 UI 组件

### 关键组件

| 组件 | 职责 |
|------|------|
| `App.tsx` | 根组件 |
| `AgentProgressLine.tsx` | 智能体进度条 |
| `BashModeProgress.tsx` | Bash 模式进度 |
| `AutoUpdater.tsx` | 自动更新器 |
| `BypassPermissionsModeDialog.tsx` | 绕过权限确认 |
| `MessageContent.tsx` | 消息内容渲染 |
| `ToolResult.tsx` | 工具结果展示 |
| `InputBar.tsx` | 输入栏 |
| `StatusBar.tsx` | 状态栏 |
| `Footer.tsx` | 底部信息栏 |

---

## 8. screens/ — UI 屏幕

**大小**：1.0 MB | **文件数**：30+ | **职责**：应用页面

### 关键屏幕

| 屏幕 | 职责 |
|------|------|
| REPL 屏幕 | 主交互界面 |
| Chat 屏幕 | 对话界面 |
| Onboarding 屏幕 | 新手引导 |
| Config 屏幕 | 配置面板 |
| Help 屏幕 | 帮助页面 |
| Resume 屏幕 | 会话恢复 |

---

## 9. hooks/ — React Hooks

**大小**：1.5 MB | **Hook 数**：33+ | **职责**：React 状态与副作用

### 关键 Hooks

| Hook | 职责 |
|------|------|
| `useCanUseTool.tsx` | 工具权限决策 |
| `useMergedTools.tsx` | 工具池合并 |
| `useSessionBackgrounding.ts` | 会话后台化 |
| `useBackgroundTaskNavigation.ts` | 后台任务导航 |
| `useKeyBindings.ts` | 键盘绑定 |
| `useAutoCompact.ts` | 自动压缩触发 |
| `useTokenWarning.ts` | Token 告警 |
| `useSettings.ts` | 设置管理 |

### 权限处理器

位于 `hooks/toolPermission/handlers/`：

| 处理器 | 场景 |
|--------|------|
| `interactiveHandler` | REPL 模式：显示 UI 提示 |
| `coordinatorHandler` | Coordinator：等待自动化检查 |
| `swarmWorkerHandler` | Swarm Worker：专用规则 |

---

## 10. types/ — 类型定义

**大小**：172 KB | **文件数**：8 | **职责**：全局 TypeScript 类型

### 关键类型文件

| 文件 | 定义 |
|------|------|
| `message.ts` | `UserMessage`, `AssistantMessage`, `SystemMessage`, `ToolUseMessage`, `ProgressMessage` |
| `permissions.ts` | `PermissionMode`, `PermissionDecision`, `PermissionRule` |
| `command.ts` | `Command`, `CommandType` |
| `tools.ts` | `ToolProgressType`, `ToolResult` |
| `logs.ts` | 日志类型 |

---

## 11. context/ — React Context

**大小**：128 KB | **文件数**：12+ | **职责**：React Context Provider

### 关键 Context

| Context | 职责 |
|---------|------|
| `notifications.ts` | 系统通知 |
| `voice.ts` | 语音交互 (KAIROS) |
| `mailbox.ts` | 智能体邮箱 IPC |

---

## 12. ink/ — 终端 UI 渲染层

**大小**：1.3 MB | **文件数**：~40 | **职责**：Ink 框架适配与终端渲染

### 关键文件

| 文件 | 职责 |
|------|------|
| `App.tsx` | Ink 主应用 |
| `clearTerminal.ts` | 终端清屏 |
| `log-update.ts` | 终端内容更新 |
| `terminal-querier.ts` | 终端能力查询 |
| `supports-hyperlinks.ts` | 超链接支持检测 |

---

## 13. bridge/ — 远程桥接

**大小**：536 KB | **文件数**：5+ | **职责**：连接本地 CLI 到 claude.ai

### 关键文件

| 文件 | LOC | 职责 |
|------|-----|------|
| `bridgeMain.ts` | 2,999 | 主模块：BridgeApiClient, SessionSpawner, JWT 刷新, 轮询循环 |
| `replBridge.ts` | 2,406 | REPL 桥接集成 |
| `bridgePermissionCallbacks.ts` | — | 权限回调处理 |

> 详见 [Agent 设计文档 — Bridge 部分](./AGENT_DESIGN.md#13-bridge-远程控制)

---

## 14. cli/ — CLI 基础设施

**大小**：536 KB | **职责**：CLI 参数解析、子命令路由

基于 commander.js 和 yargs 构建的 CLI 基础设施。

---

## 15. tasks/ — 任务执行框架

**大小**：368 KB | **职责**：后台任务生命周期管理

### 任务类型

| 目录 | 类型 | 职责 |
|------|------|------|
| `LocalAgentTask/` | `local_agent` | 本地后台智能体 (~682 LOC) |
| `RemoteAgentTask/` | `remote_agent` | 远程云端智能体 (~855 LOC) |
| `InProcessTeammateTask/` | `in_process_teammate` | 进程内 Swarm 队友 (~125 LOC) |
| `LocalShellTask/` | `local_bash` | Bash 命令任务 |
| `LocalWorkflowTask/` | `local_workflow` | 工作流任务 |
| `MonitorMcpTask/` | `monitor_mcp` | MCP 监控任务 |

> 详见 [Agent 设计文档 — 任务框架](./AGENT_DESIGN.md#8-任务框架)

---

## 16. coordinator/ — 多智能体协调

**大小**：24 KB | **文件数**：1 | **职责**：Coordinator 模式编排

**文件**：`coordinatorMode.ts` (~370 LOC)

- 系统提示增强（并行指导、综合策略）
- 内部工具隐藏 (TEAM_CREATE, TEAM_DELETE, SEND_MESSAGE, SYNTHETIC_OUTPUT)
- Worker 生成与结果收集

> 详见 [Agent 设计文档 — Coordinator](./AGENT_DESIGN.md#9-多智能体协调-coordinator)

---

## 17. skills/ — 技能系统

**大小**：208 KB | **职责**：可扩展自定义技能

### 结构

| 目录/文件 | 职责 |
|-----------|------|
| `loadSkillsDir.ts` | 技能目录加载与解析 |
| `bundled/` | 内置技能 (commit, verify 等) |

### 技能定义格式

YAML 前置元数据 + Markdown 正文，放置于 `.claude/skills/` 目录。

---

## 18. plugins/ — 插件系统

**大小**：20 KB | **职责**：第三方插件管理

### 结构

| 文件 | 职责 |
|------|------|
| `bundled/` | 内置插件 |
| `bundledPlugins.ts` | 插件注册 |

### 集成

- `services/plugins/PluginInstallationManager.ts` — 后台安装
- `commands/plugin/` — 插件管理命令

---

## 19. constants/ — 全局常量

**大小**：164 KB | **文件数**：20+ | **职责**：全局配置常量

包含 API 端点、默认值、特性标志名称、模型列表等常量定义。

---

## 20. bootstrap/ — 启动引导

**大小**：60 KB | **职责**：应用启动阶段状态初始化

### 关键文件

| 文件 | 职责 |
|------|------|
| `state.ts` | 全局状态初始化 |

---

## 21. schemas/ — 数据校验

**大小**：12 KB | **职责**：Zod Schema 定义

提供运行时数据验证 Schema，用于配置、API 响应、用户输入等验证。

---

## 22. migrations/ — 数据迁移

**大小**：48 KB | **职责**：数据格式版本迁移

处理配置文件、历史记录等数据格式在版本间的迁移。

---

## 23. keybindings/ — 键盘绑定

**大小**：172 KB | **职责**：键盘快捷键系统

管理 REPL 模式下的键盘绑定配置、Vim 模式切换等。

---

## 24. native-ts/ — 原生模块绑定

**大小**：148 KB | **职责**：原生系统 API 绑定

TypeScript 包装的原生模块调用 (Keychain, 剪贴板, 系统通知等)。

---

## 25. 其他模块

| 模块 | 大小 | 职责 |
|------|------|------|
| `buddy/` | 88 KB | AI 伴侣 (BUDDY 特性) |
| `assistant/` | 8 KB | KAIROS 助手模式 |
| `voice/` | 8 KB | 语音交互 |
| `remote/` | 48 KB | 远程会话处理 |
| `vim/` | 56 KB | Vim 编辑模式 |
| `memdir/` | 100 KB | 记忆目录系统 |
| `query/` | 36 KB | 查询辅助工具 |
| `outputStyles/` | 8 KB | 输出样式 |
| `moreright/` | 8 KB | — |

---

## 模块依赖关系总图

```
                    ┌─────────────┐
                    │  main.tsx   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ entrypoints │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──┐  ┌─────▼─────┐  ┌──▼────────┐
       │commands │  │QueryEngine│  │  screens  │
       └─────────┘  └─────┬─────┘  └───────────┘
                          │
                   ┌──────▼──────┐
                   │   query()   │
                   └──────┬──────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
     ┌──────▼──┐   ┌─────▼─────┐  ┌───▼─────┐
     │  tools  │   │ services  │  │  tasks  │
     └─────────┘   └───────────┘  └─────────┘
            │             │             │
            └─────────────┼─────────────┘
                          │
              ┌───────────▼───────────┐
              │        state          │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │    utils / types /    │
              │  constants / schemas  │
              └───────────────────────┘
                          │
              ┌───────────▼───────────┐
              │    ink / components   │
              │    hooks / screens    │
              └───────────────────────┘
```

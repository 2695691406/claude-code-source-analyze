# Claude Code 服务层关键分析

> 21+ 服务模块的深度架构分析：设计模式、关键实现与系统集成
>
> 基于 `@anthropic-ai/claude-code` v2.1.88 源码映射还原分析

---

## 目录

- [1. API 调用层](#1-api-调用层)
- [2. MCP 协议集成](#2-mcp-协议集成)
- [3. 上下文压缩服务](#3-上下文压缩服务)
- [4. OAuth 认证体系](#4-oauth-认证体系)
- [5. LSP 集成](#5-lsp-集成)
- [6. 记忆与自动整合](#6-记忆与自动整合)
- [7. 插件服务层](#7-插件服务层)
- [8. 遥测与可观测性](#8-遥测与可观测性)
- [9. 工具执行编排](#9-工具执行编排)
- [10. Token 估算与费用跟踪](#10-token-估算与费用跟踪)
- [11. 远程配置与策略](#11-远程配置与策略)
- [12. 团队记忆同步](#12-团队记忆同步)
- [13. 语音服务](#13-语音服务)
- [14. 辅助服务](#14-辅助服务)
- [15. 服务层架构总结](#15-服务层架构总结)

---

## 1. API 调用层

> `services/api/` — Claude API 的完整交互层

### 1.1 架构概览

```
queryModelWithStreaming()    ← 主入口
  ├── getAnthropicClient()   ← 多供应商客户端
  │     ├── Anthropic SDK (Direct)
  │     ├── AWS Bedrock
  │     ├── Azure Foundry
  │     └── Google Vertex
  │
  ├── withRetry()            ← 重试 + Fallback
  │     ├── 指数退避 + 抖动
  │     ├── Fast Mode 冷却
  │     └── Fallback 模型切换
  │
  └── errors.ts              ← 错误分类
        ├── 429 Rate Limit
        ├── 529 Overload
        ├── prompt-too-long
        └── 连接错误
```

### 1.2 claude.ts — 流式请求调度

**核心函数**：`queryModelWithStreaming()`

负责：
- **Beta 特性管理**：thinking block、prompt cache、AFK 模式、fast mode、task budget
- **系统提示词前缀注入**：自动在系统提示词前添加版本标识
- **工具 Schema 序列化**：将 Tool[] 转换为 API 兼容格式
- **Token 用量追踪**：累计到会话级 cost-tracker
- **Prompt Cache 管理**：检测 cache break 条件，跟踪 1 小时缓存有效期
- **结构化输出**：支持 JSON Schema 约束的输出

### 1.3 withRetry.ts — 重试与降级策略

```typescript
// 核心重试策略
{
  maxRetries: 10,
  baseDelayMs: 500,           // 指数退避基础延迟
  maxDelayMs: 60_000,         // 最大延迟 1 分钟
  
  // 特殊处理
  overload529MaxRetries: 3,   // 529 错误限制 3 次
  persistentRetry: true,      // ant 环境：无限重试 + 心跳
  
  // 前台 529 重试来源
  foreground529Sources: [
    'repl_main_thread', 'sdk', 'agent', 'compact', 'security_classifier'
  ]
}
```

**Fallback 模型切换**：
```
API 调用 (主模型: Claude Opus)
  └── 连续 N 次 overload
        └── FallbackTriggeredError
              └── 调用方切换到 Claude Sonnet
```

**Fast Mode 冷却**：
```
Fast mode 请求被拒
  └── 记录拒绝次数
        └── 指数退避冷却期
              └── 超过阈值 → 降级为标准模式
```

### 1.4 client.ts — 多供应商抽象

```typescript
function getAnthropicClient(): AnthropicClient {
  // 1. OAuth Token 刷新
  await checkAndRefreshOAuthTokenIfNeeded()
  
  // 2. 供应商检测
  if (isBedrockConfigured())  → createBedrockClient()
  if (isFoundryConfigured())  → createFoundryClient()
  if (isVertexConfigured())   → createVertexClient()
  // 默认：直连 Anthropic API
  
  // 3. 附加配置
  // · 自定义请求头 (CLI 标识 + 会话追踪)
  // · 代理 / TLS 配置 (企业部署)
  // · 调试日志选项
}
```

### 1.5 错误分类体系

| 错误码 | 分类 | 处理策略 |
|-------|------|---------|
| 429 | rate_limited | 等待 retry-after → 重试 |
| 529 | overloaded | 有限重试 + fallback |
| 400 (prompt-too-long) | context_too_long | 响应式压缩 → 重试 |
| ECONNRESET | connection | 立即重试（陈旧连接） |
| EPIPE | connection | 立即重试（管道断裂） |
| 401/403 | auth | 刷新 token → 重试 |
| 其他 | other | 传播到上层 |

---

## 2. MCP 协议集成

> `services/mcp/` — 30+ 文件实现完整 MCP 协议栈

### 2.1 架构分层

```
MCPConnectionManager.tsx    ← React 上下文管理
  │
  ├── client.ts              ← 核心 MCP 客户端
  │     ├── 工具包装与 Schema 翻译
  │     ├── 资源管理 (list/read)
  │     ├── 内容截断与 Unicode 净化
  │     └── Elicitation 处理
  │
  ├── Transport 层
  │     ├── stdio (子进程)
  │     ├── SSE (Server-Sent Events)
  │     ├── WebSocket
  │     ├── InProcessTransport.ts (进程内)
  │     └── SdkControlTransport.ts (SDK HTTP)
  │
  ├── auth.ts                ← OAuth 认证
  │     ├── ClaudeAuthProvider
  │     ├── PKCE 流程
  │     ├── 跨应用访问 (XAA)
  │     └── 锁文件并发保护
  │
  ├── config.ts              ← 配置管理
  │     └── getAllMcpConfigs() (静态 + 动态合并)
  │
  └── channelPermissions.ts  ← 通道权限
        └── 消息通道审批 (Telegram/Discord/iMessage)
```

### 2.2 核心客户端 (client.ts)

**关键能力**：
- **工具调用**：Schema 翻译 (MCP → Tool API) + 进度追踪 + 分析事件
- **资源管理**：列表/读取 + 自动截断超大内容
- **内容验证**：二进制内容持久化 + Unicode 净化防终端损坏
- **Elicitation**：浏览器 OAuth 流程的队列化处理
- **MCP Skills**：可选技能集成（特性门控）

### 2.3 传输协议抽象

| 传输方式 | 用途 | 特点 |
|---------|------|------|
| **stdio** | 本地子进程 MCP 服务器 | 最常用，进程生命周期管理 |
| **SSE** | HTTP 长连接服务器 | 适合远程服务 |
| **WebSocket** | 双向实时通信 | 适合交互式服务 |
| **InProcess** | 进程内链接 | 零序列化开销，同进程 |
| **SDK HTTP** | Anthropic SDK HTTP | 控制消息传输 |

### 2.4 OAuth 认证流 (auth.ts)

```
用户启动 MCP 服务器认证
  ├── PKCE 准备
  │     ├── generateCodeVerifier()   // 32 字节随机
  │     ├── generateCodeChallenge()  // SHA-256 哈希
  │     └── generateState()          // CSRF 保护
  │
  ├── 浏览器授权
  │     ├── 启动临时 HTTP 服务器 (接收回调)
  │     ├── 打开浏览器 → OAuth 授权页面
  │     └── 接收 authorization code
  │
  ├── Token 交换
  │     ├── code → access_token + refresh_token
  │     └── 安全存储 (Keychain / 文件)
  │
  └── 跨应用访问 (XAA) [特性门控]
        ├── OIDC 发现
        ├── IDP Token 交换
        └── Step-up 检测 (Token 刷新)
```

**关键设计**：
- **锁文件**：`tryAcquireLock()` 防止并发 OAuth 尝试
- **平台存储**：macOS Keychain / Linux 文件系统
- **失败分析**：详细错误原因追踪到 analytics

### 2.5 通道权限 (channelPermissions.ts)

```typescript
// 消息通道审批（非文本解析）
type ChannelPermissionRequest = {
  requestId: string          // 生成时避免黑名单
  toolName: string
  toolInput: unknown
  channel: 'telegram' | 'discord' | 'imessage'
}

// 基于结构化事件的审批（非自然语言解析）
```

---

## 3. 上下文压缩服务

> `services/compact/` — 管理上下文窗口溢出的多层策略

### 3.1 压缩策略全景

```
压缩优先级 (从轻到重)
  │
  ├── L1: Tool Result Budget    ← 裁剪超大工具结果
  ├── L2: Snip Compaction       ← 剪切旧历史 [HISTORY_SNIP]
  ├── L3: Microcompact          ← 基于时间清理过期内容
  ├── L4: Context Collapse      ← 读时投影视图 [CONTEXT_COLLAPSE]
  └── L5: Auto Compaction       ← AI 摘要重写
```

### 3.2 compact.ts — 核心压缩引擎

**`compactConversation()` 流程**：

```
输入消息 ───→ 图片剥离 ───→ 按 API 轮次分组 ───→ Claude 摘要
  │                                                    │
  │                                            (作为分叉子智能体)
  │                                                    │
  └── 压缩后清理 ←─────────────────────────────────────┘
        ├── 文件恢复 (top 5, 共 50KB)
        ├── 技能重注入 (每技能 5KB, 共 25KB)
        ├── 会话元数据重附加 (plan/tips 等)
        └── 分析事件 + token 统计日志
```

**熔断器**：
```
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
// 基于 BQ 分析：1,279 个会话有 50+ 次连续失败
// 3 次是成本/效果的最佳平衡点
```

### 3.3 autoCompact.ts — 自动触发

```typescript
// 阈值计算
function getAutoCompactThreshold(contextWindow, reservedOutput) {
  return contextWindow - reservedOutput - AUTOCOMPACT_BUFFER_TOKENS  // 13,000
}

// 告警状态分级
type TokenWarningState = {
  warning: boolean    // 接近阈值
  error: boolean      // 超过阈值
  autocompact: boolean // 触发自动压缩
}
```

### 3.4 microCompact.ts — 微压缩策略

| 工具类型 | 策略 | 时机 |
|---------|------|------|
| FILE_READ | 清除完整内容 | 时间窗口过期 |
| Bash | 清除输出 | 时间窗口过期 |
| GREP/GLOB | 清除结果列表 | 时间窗口过期 |
| WEB_* | 清除网页内容 | 时间窗口过期 |
| FILE_EDIT/WRITE | 清除 diff 预览 | 时间窗口过期 |
| 图片 | 截断至 2000 token | 立即 |

**缓存编辑追踪**：微压缩可能改变消息内容，需要通知 prompt cache 系统失效已缓存的前缀。

### 3.5 postCompactCleanup.ts — 压缩后恢复

```
压缩后
  ├── 恢复 top 5 文件 (POST_COMPACT_MAX_FILES_TO_RESTORE)
  │     └── 从压缩前上下文中提取最相关文件
  │
  ├── 重注入技能
  │     ├── 每技能预算: 5KB
  │     └── 总预算: 25KB
  │
  └── 重附加会话元数据
        ├── 当前计划
        ├── Tips 状态
        └── 其他会话级标注
```

---

## 4. OAuth 认证体系

> `services/oauth/` — CLI 场景的安全认证实现

### 4.1 PKCE 流程实现

```typescript
// crypto.ts
function generateCodeVerifier(): string {
  return base64URLEncode(crypto.randomBytes(32))  // 256 位熵
}

function generateCodeChallenge(verifier: string): string {
  return base64URLEncode(
    crypto.createHash('sha256').update(verifier).digest()
  )
}

function generateState(): string {
  return base64URLEncode(crypto.randomBytes(32))  // CSRF 保护
}

// Base64-URL 编码：+ → -, / → _, 移除 = 填充
```

### 4.2 本地回调服务器

```
CLI 发起 OAuth
  ├── 启动临时 HTTP 服务器 (随机端口)
  ├── 打开浏览器 → 授权页面
  │     redirect_uri = http://localhost:{port}/callback
  │
  ├── 用户授权
  │     浏览器 → callback?code=xxx&state=yyy
  │
  ├── 服务器接收
  │     验证 state → 提取 code
  │
  └── Token 交换
        code + code_verifier → access_token + refresh_token
        └── 存储到 Keychain / 文件系统
```

---

## 5. LSP 集成

> `services/lsp/` — Language Server Protocol 接入

### 5.1 管理器架构

```
LSPServerManager (工厂函数 + 闭包状态)
  │
  ├── 扩展名 → 服务器映射
  │     .ts/.tsx → TypeScript Language Server
  │     .py → Python Language Server
  │     .go → Go Language Server
  │
  ├── LSPServerInstance (独立进程)
  │     ├── 进程生命周期管理
  │     ├── JSONRPC 通信 (LSPClient)
  │     ├── 传输选择 (stdio/HTTP/WebSocket)
  │     └── 错误处理与重连
  │
  └── LSPDiagnosticRegistry
        ├── 按文件追踪诊断信息
        └── 变更增量计算
```

### 5.2 关键设计模式

- **工厂模式**：`createLSPServerManager()` 返回闭包接口
- **扩展路由**：文件扩展名自动选择对应服务器
- **懒启动**：服务器仅在首次访问文件时启动
- **生命周期追踪**：打开文件计数，空闲时可关闭

---

## 6. 记忆与自动整合

### 6.1 记忆提取 (extractMemories)

> `services/extractMemories/` — 从对话中提取持久记忆

```
查询循环结束 (模型产出最终响应，无工具调用)
  │
  ├── 初始化提取器 (闭包状态)
  │
  ├── 扫描现有记忆文件 → 构建 manifest
  │
  ├── 分叉子智能体执行
  │     ├── 共享 prompt cache（零额外 token 成本）
  │     ├── 使用提取提示词分析对话
  │     └── 通过 FILE_WRITE_TOOL 持久化
  │
  └── 记忆路径
        ├── .claude/memory/*.md  ← 自动记忆
        └── 团队记忆路径 (feature-gated)
```

### 6.2 自动整合 (autoDream)

> `services/autoDream/` — 后台记忆整合

```
触发条件 (多门控过滤，从便宜到昂贵)
  │
  ├── Gate 1: 时间门控 (1 stat)
  │     └── 距上次整合 >= 24 小时？
  │
  ├── Gate 2: 会话门控 (文件扫描)
  │     └── 新会话数 >= 5？
  │
  ├── Gate 3: 锁门控 (原子操作)
  │     └── tryAcquireConsolidationLock()
  │
  └── 执行整合
        ├── 分叉子智能体 + 整合提示词
        ├── 扫描会话文件获取上下文
        ├── 生成高级摘要
        └── 写入记忆目录
```

**关键常量**：
- `SESSION_SCAN_INTERVAL_MS = 600,000` (10 分钟) — 通过时间门但未通过会话门时的扫描节流
- 闭包隔离状态：每个测试获得独立实例，防止交叉污染

### 6.3 会话记忆 (SessionMemory)

> `services/SessionMemory/` — 当前对话的 Markdown 笔记

```typescript
// 阈值配置
initializationThreshold: 10  // 10 次工具调用后初始化
updateThreshold: 5           // 每 5 次工具调用更新一次

// 运行时机
postSamplingHook → 检查工具调用计数 → 触发更新
  ├── 分叉子智能体
  ├── 读取当前会话记忆
  ├── 使用模板 + 提示词更新
  └── FileEditTool 持久化
```

---

## 7. 插件服务层

> `services/plugins/` — 后台插件安装与市场管理

### 7.1 安装管理器

```
应用初始化
  │
  ├── performBackgroundPluginInstallations()  [非阻塞]
  │     ├── 计算市场差异 (missing / sourceChanged)
  │     ├── 设置 AppState pending 状态 → UI spinner
  │     ├── 对账市场 → 安装缺失插件
  │     ├── 安装后自动刷新插件
  │     └── 仅更新 → 提示用户运行 /reload-plugins
  │
  └── 插件操作
        ├── install(pluginName)
        ├── uninstall(pluginName)
        ├── list()
        └── update(pluginName)
```

**关键设计**：
- **非阻塞启动**：插件安装在后台进行
- **差异对账**：仅安装缺失/变更的市场
- **状态集成**：通过 AppState 通知 UI 进度

---

## 8. 遥测与可观测性

### 8.1 Analytics 架构

> `services/analytics/` — 零依赖核心 + 双后端投递

```
logEvent(eventName, metadata)
  │
  ├── 事件队列 (sink 附加前缓冲)
  │
  ├── 附加 sink 后路由
  │     ├── Datadog
  │     │     └── stripProtoFields() → 移除 _PROTO_* 键
  │     │
  │     └── 1P (First-Party) 日志
  │           └── hoistProtoFields() → PII 标记值提升
  │
  └── 元数据安全标记
        ├── AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
        └── AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED
```

**核心设计**：
- **零导入依赖**：`index.ts` 无任何 import，防止循环依赖
- **编译期安全**：标记类型强制元数据分类
- **双轨投递**：Datadog (去 PII) + 1P (含 PII proto 字段)

### 8.2 GrowthBook 特性标志

```typescript
// growthbook.ts
// GrowthBook SDK 初始化
// 特性标志获取 + 缓存
// 动态配置拉取
// A/B 测试分组
```

### 8.3 诊断追踪 (diagnosticTracking.ts)

```typescript
class DiagnosticTrackingService {
  // 单例，管理 IDE 诊断状态
  
  baseline: Map<string, Diagnostic[]>   // 文件 → 诊断基线
  lastProcessedTimestamps: Map<string, number>  // 处理时间戳
  
  // 常量
  MAX_DIAGNOSTICS_SUMMARY_CHARS = 4_000
  
  // 集成：MCP 客户端通信 IDE
}
```

---

## 9. 工具执行编排

> `services/tools/` — 工具执行的核心编排层

### 9.1 runToolUse — 主执行调度

```
runToolUse(tool, input, context)
  │
  ├── 1. 输入验证 (validateInput)
  ├── 2. 权限检查 (canUseTool)
  │     ├── PreToolUse 钩子执行
  │     └── 权限对话框 (如需要)
  │
  ├── 3. 工具执行 (tool.call)
  │     ├── 进度回调 (onProgress)
  │     └── 超时/中断处理
  │
  ├── 4. PostToolUse 钩子执行
  │     └── 钩子可修改输出 / 阻止结果
  │
  ├── 5. 结果存储
  │
  └── 6. 分析事件发射
```

### 9.2 StreamingToolExecutor — 流式并行执行

```
API 流输出
  │
  ├── tool_use 块到达 → enqueue()
  │     └── 立即开始权限检查
  │
  ├── 流继续...
  │     └── 更多 tool_use 块可能到达
  │
  └── 流结束 → 执行所有排队工具
        │
        ├── isConcurrencySafe = true
        │     └── 并行执行
        │
        └── isConcurrencySafe = false
              └── 排他执行（等待其他完成）
```

**结果排序**：按入队顺序输出，非完成顺序。

**兄弟中断控制器**：Bash 工具出错时，通过子 AbortController 中断其他兄弟工具。

### 9.3 Hook 集成 (toolHooks.ts)

```typescript
// PreToolUse 钩子
async function* runPreToolUseHooks(context, tool, input) {
  for (const hook of getPreToolHooks(tool.name)) {
    const result = await executePreToolHook(hook, { toolName, toolUseId, toolInput })
    // 钩子可返回：allow / deny / ask + updatedInput
  }
}

// PostToolUse 钩子
async function* runPostToolUseHooks(context, tool, input, output) {
  for (const hook of getPostToolHooks(tool.name)) {
    const result = await executePostToolHook(hook, { toolName, toolOutput })
    // 钩子可修改输出或阻止结果
  }
}
```

---

## 10. Token 估算与费用跟踪

### 10.1 Token 估算 (tokenEstimation.ts)

```
Token 计数策略
  │
  ├── 精确计数 (API 返回)
  │     └── usage.input_tokens / usage.output_tokens
  │
  ├── 本地估算 (无 API 调用)
  │     ├── JSON 序列化 + 系数
  │     ├── Thinking block 检测
  │     └── Tool search 字段剥离
  │
  └── 多供应商支持
        ├── Anthropic SDK 计数
        ├── AWS Bedrock countTokens()
        └── Google Vertex
```

**动态加载**：Bedrock SDK 仅在首次调用时加载（节省 279KB 启动开销）。

### 10.2 费用跟踪 (cost-tracker.ts)

```typescript
// 会话级 Token 用量累计
type CostTrackerState = {
  inputTokens: number
  outputTokens: number
  cacheCreationInputTokens: number
  cacheReadInputTokens: number
  totalCostUsd: number
}
```

### 10.3 速率限制感知 (claudeAiLimits.ts)

```typescript
// 早期预警阈值
{
  five_hour: {
    warnAt: 0.90,           // 90% 用量
    timeElapsedMin: 0.72    // 72% 时间
  },
  seven_day: [
    { warnAt: 0.75, timeElapsed: 0.60 },
    { warnAt: 0.50, timeElapsed: 0.35 },
    { warnAt: 0.25, timeElapsed: 0.15 }
  ]
}

// 配额状态
type QuotaStatus = 'allowed' | 'allowed_warning' | 'rejected'
```

---

## 11. 远程配置与策略

### 11.1 远程管理设置 (remoteManagedSettings)

```
应用启动 → loadRemoteManagedSettings() [异步，不阻塞]
  │
  ├── fetchRemoteManagedSettings()
  │     ├── ETag 缓存 → 304 Not Modified 优化
  │     ├── Fail-open → 获取失败不影响运行
  │     └── 文件缓存持久化
  │
  └── 后台轮询
        └── 定期刷新组织级设置
```

### 11.2 策略限制 (policyLimits)

```
loadPolicyLimits() [异步，不阻塞]
  │
  ├── 资格检查
  │     └── Team/Enterprise 订阅者 (OAuth) 才生效
  │
  ├── getPolicyLimits()
  │     ├── ETag 缓存
  │     ├── Fail-open
  │     └── 1 小时轮询间隔
  │
  └── 策略执行
        └── 功能禁用/限制
```

---

## 12. 团队记忆同步

> `services/teamMemorySync/` — 跨组织成员的记忆共享

### 12.1 同步机制

```
双向同步
  │
  ├── pullTeamMemory()
  │     └── 服务端内容 → 覆写本地 (服务端赢)
  │
  ├── pushTeamMemory()
  │     ├── 内容哈希比较 → 仅上传差异
  │     └── 秘密扫描 → 阻止凭证泄露
  │
  └── 文件监控 (watcher.ts)
        └── 本地修改 → 触发 push
```

### 12.2 安全防护

```
上传前扫描管道
  │
  ├── secretScanner.ts
  │     └── 扫描文件中的敏感信息
  │
  ├── teamMemSecretGuard.ts
  │     └── 验证扫描结果 → 过滤文件
  │
  └── 约束
        └── MAX_FILE_SIZE_BYTES = 250,000 (每条目限制)
```

**关键特性**：本地删除不会传播到服务端（下次 pull 时恢复）。

---

## 13. 语音服务

> `services/voice.ts` — 推送讲话语音输入

### 13.1 音频捕获架构

```
语音输入
  │
  ├── 原生模块 (首选)
  │     ├── audio-capture-napi
  │     ├── macOS: CoreAudio
  │     └── Linux: ALSA
  │
  └── 降级方案
        ├── SoX (rec 命令)
        └── arecord (ALSA 工具)
```

**关键参数**：
```
RECORDING_SAMPLE_RATE = 16,000 Hz
SILENCE_DURATION_SECS = 2.0      // 静音检测时长
SILENCE_THRESHOLD = 3%            // 静音检测阈值
```

**懒加载设计**：原生模块仅在首次语音按键时加载，避免启动时 `dlopen` 阻塞。

### 13.2 流式 STT (voiceStreamSTT.ts)

```
音频流 → 分块 → STT API → 增量文本 → 关键词检测
```

---

## 14. 辅助服务

### 14.1 VCR 录制/回放 (vcr.ts)

```typescript
// 测试夹具管理
withVCR(messages, async () => {
  // 1. 对输入取哈希 → 文件名
  // 2. 尝试加载缓存夹具
  // 3. 缺失 + CI + 无 VCR_RECORD → 报错
  // 4. 否则执行 → 缓存结果
})

// 夹具路径
${CLAUDE_CODE_TEST_FIXTURES_ROOT}/fixtures/{name}-{hash}.json
```

### 14.2 通知 (notifier.ts)

| 通道 | 方式 |
|------|------|
| iTerm2 | 转义序列 (可选响铃) |
| Kitty | 转义序列 |
| Ghostty | 转义序列 |
| Terminal Bell | BEL 字符 |
| 系统通知 | 外部命令 (macOS/Linux) |

### 14.3 防睡眠 (preventSleep.ts)

```typescript
// macOS: caffeinate 引用计数
startPreventSleep()  → refCount++ → 首次时 spawn caffeinate
stopPreventSleep()   → refCount-- → 归零时 kill
forceStopPreventSleep() → 强制停止

// 自愈设计
// · 5 分钟超时，4 分钟重启间隔
// · Node 进程被 kill 后，caffeinate 超时自动退出
```

### 14.4 Tips 系统 (tips/)

```
加载中显示 Tip
  ├── 上下文过滤 → 选择相关 Tips
  ├── 轮换策略 → 选择最久未显示的
  └── 会话计数冷却 → 避免重复
```

### 14.5 内部日志 (internalLogging.ts)

```typescript
// Ant 环境专用
getKubernetesNamespace()  → /var/run/secrets/kubernetes.io
getContainerId()          → /proc/self/mountinfo 解析
logPermissionContextForAnts()  → analytics 事件

// 结果缓存：基础设施信息在会话中不变
```

---

## 15. 服务层架构总结

### 15.1 设计模式一览

| 模式 | 应用场景 | 示例 |
|------|---------|------|
| **分叉子智能体** | 隔离执行 + 共享缓存 | compact, extractMemories, autoDream, sessionMemory |
| **多门控过滤** | 按成本排序的条件检查 | autoDream (时间→会话→锁) |
| **闭包状态** | 避免模块级可变状态 | initExtractMemories(), initAutoDream(), initSessionMemory() |
| **熔断器** | 防止级联失败 | autoCompact (3次), tokenRefresh (3次) |
| **零依赖核心** | 防止循环导入 | analytics/index.ts |
| **Fail-open** | 非阻塞配置加载 | remoteManagedSettings, policyLimits |
| **ETag 缓存** | 高效轮询 | remoteManagedSettings, policyLimits |
| **标记类型** | 编译期安全分类 | analytics 元数据 |
| **引用计数** | 共享资源管理 | preventSleep (caffeinate) |
| **懒加载** | 减少启动开销 | voice (原生模块), tokenEstimation (Bedrock SDK) |

### 15.2 服务间依赖关系

```
API 调用层 ←── query.ts (核心循环)
    │
    ├── 被 compact 服务使用 (摘要生成)
    ├── 被 MCP 客户端使用 (远程工具)
    └── 被 extractMemories 使用 (分叉子智能体)

Analytics ←── 全局使用 (无导入依赖)
    │
    ├── Datadog (去 PII)
    └── 1P 日志 (含 PII proto)

OAuth ←── MCP auth
    └── 被 API client 使用 (token 刷新)

State (AppState) ←── React Context
    │
    ├── 被 Plugin 安装管理器写入
    ├── 被 onChangeAppState 监听
    └── 被 Selectors 读取
```

---

## 关联文档

- [核心流程源码级分析](./CORE_FLOW_ANALYSIS.md) — 启动到响应的完整链路
- [整体框架设计](./ARCHITECTURE.md) — 架构分层与技术栈
- [模块详细设计](./MODULES.md) — 按目录逐一分析
- [Agent 设计](./AGENT_DESIGN.md) — 智能体架构与多智能体协调

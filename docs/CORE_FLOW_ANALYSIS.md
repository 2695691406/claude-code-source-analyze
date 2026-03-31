# Claude Code 核心流程源码级分析

> 从启动到响应的完整链路追踪，源码级剖析关键决策点与控制流
>
> 基于 `@anthropic-ai/claude-code` v2.1.88 源码映射还原分析

---

## 目录

- [1. 启动流程深度分析](#1-启动流程深度分析)
- [2. QueryEngine 核心状态机](#2-queryengine-核心状态机)
- [3. query 工具调用循环](#3-query-工具调用循环)
- [4. 权限决策完整链路](#4-权限决策完整链路)
- [5. 上下文管理完整链路](#5-上下文管理完整链路)
- [6. 关键数据结构图解](#6-关键数据结构图解)
- [7. 错误处理与容错架构](#7-错误处理与容错架构)
- [8. 性能优化关键路径](#8-性能优化关键路径)

---

## 1. 启动流程深度分析

### 1.1 入口分层：三级启动链

```
cli.tsx (快速路径) → init.ts (配置加载) → main.tsx (完整初始化)
```

**第一级：cli.tsx — 零开销快速路径**

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2)

  // 快速路径 1: --version（零模块加载）
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`)
    return
  }

  // 快速路径 2: --dump-system-prompt（仅加载 prompts 模块）
  if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
    const { getSystemPrompt } = await import('../constants/prompts.js')
    // ...
    return
  }

  // 快速路径 3: Chrome MCP 服务器模式
  if (process.argv[2] === '--claude-in-chrome-mcp') {
    const { runClaudeInChromeMcpServer } = await import(...)
    await runClaudeInChromeMcpServer()
    return
  }

  // 默认路径：加载完整 CLI
  const { runCli } = await import('./init.ts')
  await runCli()
}
```

**设计要点**：快速路径避免加载完整 205K 行源码，`--version` 在毫秒级返回。

**第二级：init.ts — 配置预加载**

```typescript
export async function runCli(): Promise<void> {
  const { enableConfigs } = await import('../utils/config.js')
  enableConfigs()   // 加载 settings.json / 环境变量 / 迁移

  const { runApp } = await import('../main.tsx')
  await runApp()
}
```

**第三级：main.tsx — 完整初始化 (4,683 LOC)**

### 1.2 main() 函数初始化序列

```
main()
  ├── 安全防护 (Windows PATH 劫持 / 信号处理)
  ├── 深链接/URL 解析 (cc:// / cc+unix:// / assistant / ssh)
  ├── run() → Commander 程序构建
  │     ├── preAction Hook (全局初始化)
  │     │     ├── ensureMdmSettingsLoaded()    // MDM 管理设置
  │     │     ├── ensureKeychainPrefetchCompleted()  // 凭证预取
  │     │     ├── init()                        // 核心初始化
  │     │     ├── loadRemoteManagedSettings()   // 远程设置(异步)
  │     │     └── loadPolicyLimits()            // 策略限制(异步)
  │     │
  │     ├── CLI 选项注册 (30+ 参数)
  │     └── .action() 主处理函数
  │           ├── 模式检测与选择
  │           ├── 工具池组装
  │           ├── 权限上下文构建
  │           └── REPL/Print/SDK 启动
  └── 子命令注册 (config/api-key/doctor/mcp 等)
```

### 1.3 preAction Hook：全局一次性初始化

```typescript
program.hook('preAction', async thisCommand => {
  // 并行加载：MDM 设置 + 凭证预取
  await Promise.all([
    ensureMdmSettingsLoaded(),
    ensureKeychainPrefetchCompleted()
  ])
  
  // 核心初始化
  await init()
  
  // 异步后台加载（不阻塞启动）
  void loadRemoteManagedSettings()  // 远程设置同步
  void loadPolicyLimits()           // 组织策略限制
})
```

**关键设计**：MDM 设置和凭证预取**并行**执行，远程配置加载**不阻塞**主流程。

### 1.4 模式选择决策树

```
.action(async (prompt, options) => {
  │
  ├── --bare → CLAUDE_CODE_SIMPLE 模式 (最小工具集)
  │
  ├── KAIROS 特性门控 + settings.assistant
  │   └── 助手模式初始化
  │
  ├── --print/-p 标志
  │   └── 非交互打印模式
  │
  ├── SDK 调用入口
  │   └── AsyncGenerator 模式
  │
  ├── Bridge 远程模式
  │   └── BridgeApiClient 连接
  │
  └── 默认
      └── 交互式 REPL (Ink TUI)
```

| 模式 | 触发条件 | 行为 |
|------|---------|------|
| **REPL 交互** | 默认 | 完整 TUI，Ink 渲染 |
| **打印模式** | `--print` / `-p` | 单次查询，输出后退出 |
| **SDK** | `ask()` 生成器调用 | AsyncGenerator 产出 SDKMessage |
| **SIMPLE** | `--bare` | 最小工具集 (Bash/Read/Edit 或 REPLTool) |
| **Bridge** | SDK + ElicitRequest 回调 | 远程权限审批 |
| **KAIROS** | 特性门控 + settings.assistant | 计划任务 + 主动建议 |

### 1.5 工具池组装时序

```typescript
// 1. 基础工具收集
const baseTools = getAllBaseTools()  // 43+ 内置工具

// 2. 条件工具注入
if (feature('AGENT_TRIGGERS')) → CronTool 等
if (feature('PROACTIVE'))      → SleepTool
if (hasEmbeddedSearchTools())  → 替换 Glob/Grep

// 3. 权限过滤
const filtered = filterToolsByDenyRules(baseTools, permissionContext)

// 4. MCP 工具合并
const pool = assembleToolPool(permissionContext, mcpTools)
// 内置工具排序后作为前缀 → MCP 工具追加 → 按 name 去重
// 设计目的：保持 prompt cache 稳定性
```

---

## 2. QueryEngine 核心状态机

### 2.1 类结构与关键状态

```typescript
class QueryEngine {
  // 配置（不可变）
  private config: QueryEngineConfig
  
  // 会话状态（可变）
  private mutableMessages: Message[]           // 消息历史
  private abortController: AbortController     // 取消控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage         // Token 累计用量
  private readFileState: FileStateCache        // 文件状态缓存(COW)
  
  // 轮次状态（每轮重置）
  private discoveredSkillNames = new Set<string>()     // 发现的技能
  private loadedNestedMemoryPaths = new Set<string>()  // 加载的记忆路径
  private hasHandledOrphanedPermission = false         // 孤立权限处理
}
```

### 2.2 submitMessage() 七阶段生命周期

```
submitMessage(prompt, options)
  │
  ├── Phase 1: 上下文组装
  │     ├── 包装 canUseTool (追踪权限拒绝)
  │     ├── fetchSystemPromptParts() (工具列表/上下文)
  │     └── 组装系统提示词 (custom + memory + append)
  │
  ├── Phase 2: 用户输入处理
  │     ├── 构建 ProcessUserInputContext (第一次)
  │     ├── processUserInput() (斜杠命令 / 文件资源)
  │     └── 重建 ProcessUserInputContext (第二次，含命令结果)
  │
  ├── Phase 3: 会话记录
  │     ├── recordTranscript() (持久化用户消息)
  │     └── yield 系统初始化消息
  │
  ├── Phase 4: 查询循环执行
  │     ├── yield* query() (核心循环)
  │     ├── 流式消息归一化
  │     └── usage 追踪
  │
  ├── Phase 5: 结构化输出处理
  │     └── JSON Schema 验证 (如果启用)
  │
  ├── Phase 6: 停止钩子执行
  │     ├── 记忆提取 (extractMemories)
  │     ├── 自动整合 (autoDream)
  │     └── 会话记忆更新 (sessionMemory)
  │
  └── Phase 7: 清理与总结
        ├── yield 最终 usage 消息
        └── yield 权限拒绝摘要
```

### 2.3 系统提示词组装机制

**组装优先级**：

```typescript
const systemPrompt = asSystemPrompt([
  // 1. 自定义提示词 OR 默认提示词（二选一）
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  
  // 2. 记忆机制说明（SDK + CLAUDE_COWORK_MEMORY_PATH_OVERRIDE 时注入）
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  
  // 3. 追加提示词（--append-system-prompt）
  ...(appendSystemPrompt ? [appendSystemPrompt] : [])
])
```

**defaultSystemPrompt 来源**（由 `fetchSystemPromptParts()` 生成）：
- 基础系统提示词（角色、能力、限制）
- 工具描述列表（根据当前工具池动态生成）
- Git 状态上下文
- 工作目录文件树
- CLAUDE.md 记忆文件内容

### 2.4 ProcessUserInputContext 双构建模式

```
第一次构建 (斜杠命令处理前)
  │  包含：工具/命令/MCP 客户端/模型配置
  │  用于：processUserInput() 处理斜杠命令
  │
  ▼ processUserInput() 执行
  │  可能修改：模型/工具/消息
  │  返回：shouldQuery / allowedTools / modelFromUserInput
  │
第二次构建 (斜杠命令处理后)
  │  反映命令执行后的状态变更
  │  用于：传递给 query() 作为 toolUseContext
```

**设计原因**：斜杠命令（如 `/model`、`/tools`）可能改变模型选择或工具池，因此需要在命令执行后重建上下文。

---

## 3. query 工具调用循环

### 3.1 循环架构总览

```
query(params)              // 入口：消费命令 UUID
  └── queryLoop(params)    // 外层：管理迭代状态
        └── queryLoopBody()  // 内层：单次 API 调用 + 工具执行
              ├── 上下文压缩管道 (5层)
              ├── Claude API 流式调用
              ├── 工具检测与执行
              └── 结果反馈 → 继续/终止
```

### 3.2 循环可变状态

```typescript
type State = {
  messages: Message[]                   // 消息历史
  toolUseContext: ToolUseContext         // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 压缩追踪
  maxOutputTokensRecoveryCount: number  // 输出超限恢复计数
  hasAttemptedReactiveCompact: boolean  // 是否尝试过响应式压缩
  maxOutputTokensOverride: number | undefined  // 输出 token 覆盖
  pendingToolUseSummary: Promise<...>   // 待处理的工具摘要
  stopHookActive: boolean | undefined   // 停止钩子是否激活
  turnCount: number                     // 轮次计数
  transition: Continue | undefined      // 上一轮为何继续
}
```

### 3.3 上下文压缩管道（5 层优先级）

```
原始消息 ─────────────────────────────────────────────────────────→
  │
  ├── Layer 1: Tool Result Budget (工具结果预算)
  │   applyToolResultBudget()
  │   · 按聚合大小裁剪超大工具结果
  │   · 豁免 maxResultSizeChars=Infinity 的工具
  │   · 运行在所有其他压缩之前
  │
  ├── Layer 2: Snip Compaction (历史剪切) [HISTORY_SNIP 门控]
  │   snipCompactIfNeeded()
  │   · 超过剪切阈值时裁剪历史
  │   · 产出边界消息通知用户
  │   · tokensFreed 传递给 autoCompact 避免误判
  │
  ├── Layer 3: Microcompact (微压缩)
  │   deps.microcompact()
  │   · 基于时间窗口清理过期工具结果
  │   · 按工具类型分策略 (FILE_READ/Bash/Grep/Glob/Web/FileEdit)
  │   · 图片截断至 IMAGE_MAX_TOKEN_SIZE=2000
  │   · 跟踪缓存编辑以处理 prompt cache 失效
  │
  ├── Layer 4: Context Collapse (上下文折叠) [CONTEXT_COLLAPSE 门控]
  │   contextCollapse.applyCollapsesIfNeeded()
  │   · 读时投影：摘要消息存在别处
  │   · 不修改原始消息，仅在 API 调用时替换视图
  │
  └── Layer 5: Auto Compaction (自动压缩)
      deps.autocompact()
      · Token 数超过阈值时触发
      · 熔断器：连续失败 ≥3 次停止重试
      · 成功后：重建追踪状态，产出 postCompact 消息
      · 接收 snipTokensFreed 以修正 token 估算
```

### 3.4 Claude API 流式调用

```typescript
// 预调用设置
const toolUseBlocks: ToolUseBlock[] = []
let needsFollowUp = false
const streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(tools, canUseTool, toolUseContext)
  : null

// 模型选择
const currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel,
  exceeds200kTokens: permissionMode === 'plan' 
    && doesMostRecentAssistantMessageExceed200k(messages)
})

// 流式处理
for await (const event of apiStream) {
  if (event.type === 'tool_use') {
    toolUseBlocks.push(event)
    needsFollowUp = true
    
    // 流式工具执行：权限检查可以在流完成前开始
    if (streamingToolExecutor) {
      streamingToolExecutor.enqueue(event)
    }
  }
  yield event  // 实时产出到上层
}

// 工具执行阶段（流完成后）
if (needsFollowUp) {
  for (const toolUse of toolUseBlocks) {
    // 1. 权限检查 (canUseTool)
    // 2. 工具执行 (tool.call())
    // 3. 结果打包 (ToolResultBlock)
    // 4. 追加到消息历史
  }
  // → 继续循环
}
```

### 3.5 循环终止条件

| 条件 | 行为 |
|------|------|
| 无 ToolUseBlock | 返回最终响应，循环结束 |
| `maxTurns` 达到上限 | 强制终止 |
| `taskBudget` 耗尽 | 停止工具执行 |
| 用户中断 (AbortController) | 取消当前请求 |
| API 错误（不可重试） | 抛出异常 |
| `max_output_tokens` 错误 | 恢复循环（最多 3 次） |

### 3.6 StreamingToolExecutor 并行执行

```
API 流输出中
  │ tool_use_block_1 到达 ──→ enqueue() ──→ 权限检查开始
  │ tool_use_block_2 到达 ──→ enqueue() ──→ 权限检查开始
  │ ...流继续...
  │ tool_use_block_3 到达 ──→ enqueue() ──→ 权限检查开始
  │
流结束
  ├── block_1 (isConcurrencySafe=true)  ──→ 并行执行
  ├── block_2 (isConcurrencySafe=true)  ──→ 并行执行
  └── block_3 (isConcurrencySafe=false) ──→ 等待前序完成 → 串行执行
```

**并发安全判定**：
- `BashTool`: 只读命令 (ls/grep/cat) → 并发安全
- `FileReadTool`: 始终并发安全
- `FileEditTool`: 不安全（影响文件系统状态）
- `AgentTool`: 不安全（子智能体有副作用）

---

## 4. 权限决策完整链路

### 4.1 分层权限决策流水线

```
工具调用请求 (tool_name, input)
  │
  ├── Layer 0: 工具池过滤 (assembleToolPool)
  │   └── filterToolsByDenyRules() → 工具是否在可用池中？
  │       否 → 工具不可见，无法被模型调用
  │
  ├── Layer 1: 输入验证 (validateInput)
  │   └── Zod Schema 校验 → 失败则 deny
  │
  ├── Layer 2: 权限模式快速路径
  │   ├── bypassPermissions → allow (但安全检查例外)
  │   ├── dontAsk → deny
  │   └── plan → block execution（仅规划）
  │
  ├── Layer 3: 工具级权限检查 (checkPermissions)
  │   └── 每个工具的自定义权限逻辑
  │       ├── BashTool → bashToolHasPermission() (见 4.2)
  │       ├── FileEditTool → 文件路径权限
  │       └── AgentTool → 子智能体权限传递
  │
  ├── Layer 4: 规则匹配 (PermissionRule)
  │   ├── 来源优先级：policy > user > project > local > flag > command > session
  │   ├── 匹配工具名 + 可选内容模式
  │   └── allow/deny/ask 三值决策
  │
  ├── Layer 5: PreToolUse 钩子
  │   └── 外部脚本可返回 allow/deny/ask + updatedInput
  │
  ├── Layer 6: 自动分类器 (auto 模式)
  │   └── AI 分类器评估安全性
  │       返回 allow 或 ask（降级为人工审批）
  │
  └── Layer 7: 用户确认对话框
      ├── 交互模式 → Ink UI 对话框
      ├── SDK 模式 → ElicitRequest 回调
      └── Bridge 模式 → 远程审批
```

### 4.2 Bash 命令安全检查（两阶段分类）

**阶段 1：静态模式检测（23+ 检查器）**

```typescript
async function bashCommandIsSafeAsync(command, cwd) {
  // 23 类危险模式检测
  const checks = [
    INCOMPLETE_COMMANDS,           // 未闭合引号
    JQ_SYSTEM_FUNCTION,            // jq 的 system() 函数
    OBFUSCATED_FLAGS,              // 标志混淆尝试
    SHELL_METACHARACTERS,          // 管道/重定向（引号外）
    DANGEROUS_VARIABLES,           // $IFS/$SHELL 修改
    DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION,  // $() / `` / <()
    DANGEROUS_PATTERNS_INPUT_REDIRECTION,     // < /etc/passwd
    DANGEROUS_PATTERNS_OUTPUT_REDIRECTION,    // > /etc/hosts
    PROC_ENVIRON_ACCESS,           // /proc/*/environ
    MALFORMED_TOKEN_INJECTION,     // UTF-8 绕过
    BRACE_EXPANSION,               // {a,b,c} 展开
    CONTROL_CHARACTERS,            // NUL/STX 等
    UNICODE_WHITESPACE,            // 非 ASCII 空白
    ZSH_DANGEROUS_COMMANDS,        // zmodload/emulate 等
    // ... 更多
  ]
  
  for (const check of checks) {
    const result = check(command)
    if (result.isDangerous) {
      return { behavior: 'deny', reason: result.reason, checkId: result.id }
    }
  }
}
```

**阶段 2：规则 + 路径 + 只读约束**

```typescript
// 规则匹配
const ruleResult = checkRuleBasedPermissions(toolName, context)
// 路径约束（工作目录验证）
const pathResult = checkPathConstraints(cwd, command, context)
// 只读约束（--read-only 标志）
const readOnlyResult = checkReadOnlyConstraints(command, mode)
// 可选：AI 分类器（auto 模式）
if (mode === 'auto') {
  const classifierResult = await classifyYoloAction(...)
}
```

**Zsh 特殊处理**：
```typescript
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload', 'emulate', 'sysopen', 'sysread', 'syswrite',
  'zpty', 'ztcp', 'zsocket', 'mapfile',
  'zf_rm', 'zf_mv', 'zf_ln', 'zf_chmod', 'zf_chown', 'zf_mkdir'
])
```

### 4.3 权限决策元数据（可追溯性）

```typescript
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }         // 规则匹配
  | { type: 'mode'; mode: PermissionMode }         // 模式决策
  | { type: 'hook'; hookName: string }             // 钩子决策
  | { type: 'classifier'; classifier: string }     // 分类器决策
  | { type: 'workingDir'; reason: string }         // 工作目录限制
  | { type: 'safetyCheck'; classifierApprovable }  // 安全检查
  | { type: 'sandboxOverride'; reason: string }    // 沙箱覆盖
  | { type: 'asyncAgent'; reason: string }         // 异步智能体
  | { type: 'subcommandResults'; reasons: Map }    // 复合子命令
  | { type: 'other'; reason: string }              // 其他
```

**设计意义**：每个权限决策都携带**原因元数据**，支持调试、审计和 UI 展示。

### 4.4 权限模式配置详解

| 模式 | 标识 | 行为 | 可见性 |
|------|------|------|--------|
| `default` | (无) | 逐一确认工具使用 | 公开 |
| `plan` | ⏸ | 暂停-审查模式，阻止执行 | 公开 |
| `acceptEdits` | — | 自动批准文件编辑 | 公开 |
| `bypassPermissions` | — | 跳过所有权限检查（危险） | 公开 |
| `dontAsk` | — | 静默拒绝所有权限 | 公开 |
| `auto` | ⏵⏵ | AI 分类器自动审批 | 内部 |
| `bubble` | — | 临时气泡模式 | 内部 |

---

## 5. 上下文管理完整链路

### 5.1 系统提示词组装流水线

```
fetchSystemPromptParts()
  │
  ├── 基础提示词 (角色/能力/限制)
  │     └── 从 constants/prompts.ts 加载
  │
  ├── 工具描述注入
  │     ├── 遍历当前工具池
  │     ├── tool.description(input, context) → 动态描述
  │     ├── tool.prompt() → 使用说明
  │     └── shouldDefer=true 的工具 → 仅注入简要描述
  │
  ├── 用户上下文 (userContext)
  │     ├── Git 状态 (branch / modified files / recent commits)
  │     ├── 工作目录文件树 (深度限制)
  │     └── 项目元数据 (package.json / README 片段)
  │
  ├── 系统上下文 (systemContext)
  │     ├── 操作系统信息
  │     ├── 运行时环境
  │     └── 可用 MCP 服务器
  │
  └── 记忆文件注入
        ├── CLAUDE.md (项目根目录)
        ├── ~/.claude/CLAUDE.md (用户全局)
        ├── 嵌套记忆 (.claude/memory/ 目录)
        └── 团队记忆 (如果启用)
```

### 5.2 记忆文件层级

```
~/.claude/CLAUDE.md              ← 用户全局记忆
  │
  ├── /project/CLAUDE.md          ← 项目根记忆
  │     ├── .claude/memory/*.md   ← 自动记忆（extractMemories 写入）
  │     └── .claude/settings.json ← 项目级设置
  │
  └── 团队记忆 (teamMemorySync)
        └── 服务端同步，per-repo 范围
```

### 5.3 Token 预算管理

```typescript
// 上下文窗口计算
有效窗口 = 模型上下文窗口 - 预留输出 token - AUTOCOMPACT_BUFFER_TOKENS(13,000)

// 预算追踪
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null

// 任务预算（跨压缩边界累计）
let taskBudgetRemaining = params.taskBudget?.total

// 硬阻塞限制（仅在 autoCompact 关闭时生效）
const finalContextTokens = tokenCountWithEstimation(messages) - snipTokensFreed
if (finalContextTokens > HARD_BLOCKING_TOKEN_LIMIT) {
  throw new Error('Context exceeds hard limit')
}
```

**Token 估算分层**：
1. **精确计数**：API 返回的 usage.input_tokens
2. **估算计数**：tokenEstimation.ts 的本地估算（JSON 序列化 + 系数）
3. **预算扣减**：snipTokensFreed 修正避免误判

### 5.4 上下文压缩触发策略

**自动压缩触发条件**：

```typescript
function getAutoCompactThreshold(contextWindow, reservedOutput) {
  const effectiveWindow = contextWindow - reservedOutput
  return effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS  // 13,000 安全余量
}

// 触发判定
const shouldAutoCompact = 
  currentTokenCount > autoCompactThreshold
  && consecutiveFailures < MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES  // 3
```

**响应式压缩**：

```typescript
// API 返回 max_output_tokens 错误时
if (apiError === 'max_output_tokens' && !hasAttemptedReactiveCompact) {
  hasAttemptedReactiveCompact = true
  // 尝试压缩后重试 API 调用
}
```

**压缩后恢复**：

```typescript
// POST_COMPACT_MAX_FILES_TO_RESTORE = 5
// 恢复预压缩上下文中的 top 5 文件
// 重新注入技能（每技能 5KB 预算）
// 总预算：50KB 文件 + 25KB 技能
```

---

## 6. 关键数据结构图解

### 6.1 消息类型体系

```
Message (联合类型)
  ├── UserMessage
  │     ├── role: 'user'
  │     └── content: string | ContentBlockParam[]
  │           ├── TextBlockParam
  │           ├── ImageBlockParam
  │           └── ToolResultBlockParam
  │
  ├── AssistantMessage
  │     ├── role: 'assistant'
  │     ├── content: ContentBlock[]
  │     │     ├── TextBlock
  │     │     ├── ThinkingBlock
  │     │     └── ToolUseBlock { id, name, input }
  │     ├── model: string
  │     ├── stop_reason: StopReason
  │     └── usage: Usage
  │
  └── SystemMessage (内部)
        ├── 边界消息 (compact/snip 标记)
        ├── 通知消息 (task-notification)
        └── 控制消息 (model-switch 等)
```

### 6.2 工具结果类型

```typescript
type ToolResult<Output = unknown> = {
  type: 'tool_result'
  tool_use_id: string
  content: ToolResultContent     // 文本/图片/错误
  is_error?: boolean
  output?: Output                // 类型安全的结构化输出
  metadata?: {
    fileChanges?: FileChange[]   // 文件变更记录
    tokensUsed?: number          // 工具消耗的 token
  }
}
```

### 6.3 AppState 完整结构

```typescript
type AppState = DeepImmutable<{
  // 设置与配置
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting
  
  // UI 状态
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  selectedIPAgentIndex: number
  
  // 权限上下文
  toolPermissionContext: ToolPermissionContext
  
  // 预测执行状态
  speculation: SpeculationState
  speculationSessionTimeSavedMs: number
  
  // 任务管理
  tasks: Record<AgentId, TaskState>
  viewingAgentTaskId: AgentId | undefined
  
  // 消息与通知
  messages: Message[]
  notifications: NotificationState
  
  // 元数据
  attributionState: AttributionState
  denialTrackingState: DenialTrackingState
}>
```

### 6.4 ToolPermissionContext 结构

```typescript
type ToolPermissionContext = {
  mode: PermissionMode
  
  // 规则集（按来源分层）
  allowRules: PermissionRule[]     // 允许规则
  denyRules: PermissionRule[]      // 拒绝规则
  
  // 状态
  isBypassPermissionsModeAvailable: boolean
  trustedDirectories: Set<string>  // 可信目录
  
  // 会话级临时规则
  sessionRules: PermissionRule[]
}
```

---

## 7. 错误处理与容错架构

### 7.1 API 错误分类与处理策略

```typescript
categorizeRetryableAPIError(error):
  ├── 'rate_limited' (429)
  │     └── 延迟等待 retry-after 头 → 重试
  │
  ├── 'overloaded' (529)
  │     ├── 前台请求：限 3 次重试
  │     ├── 后台请求：无限重试 + 心跳
  │     └── 可选：fallback 到备用模型
  │
  ├── 'context_too_long'
  │     └── 触发响应式压缩 → 重试
  │
  └── 'other' (不可重试)
        └── 传播错误到上层
```

### 7.2 重试策略矩阵

| 错误类型 | 最大重试 | 退避策略 | 特殊处理 |
|---------|---------|---------|---------|
| 429 Rate Limit | 10 | 指数退避 (500ms base) | 前台/后台分别处理 |
| 529 Overload | 3 (前台) | 指数退避 + 抖动 | Fast mode 冷却 |
| ECONNRESET | 10 | 立即重试 | 陈旧连接检测 |
| EPIPE | 10 | 立即重试 | 写管道断裂 |
| prompt-too-long | 1 | — | 响应式压缩 |
| max_output_tokens | 3 | — | 恢复循环 |

### 7.3 Fallback 模型切换

```typescript
// withRetry.ts
if (consecutiveOverloadErrors >= FALLBACK_THRESHOLD) {
  if (fallbackModel) {
    throw new FallbackTriggeredError(fallbackModel)
  }
}

// 调用方捕获
try {
  yield* queryWithModel(primaryModel)
} catch (e) {
  if (e instanceof FallbackTriggeredError) {
    yield* queryWithModel(e.fallbackModel)
  }
}
```

### 7.4 熔断器模式

```
自动压缩熔断器
  ├── MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
  ├── 连续失败计数 → 超过阈值停止重试
  └── 成功后重置计数

Fast Mode 冷却
  ├── 跟踪 fast mode 拒绝次数
  ├── 指数退避冷却期
  └── 超过阈值降级为标准模式

Token Refresh 熔断器
  ├── MAX_REFRESH_FAILURES = 3
  ├── 连续刷新失败 → 停止调度
  └── 成功后重置
```

---

## 8. 性能优化关键路径

### 8.1 Prompt Cache 稳定性设计

```typescript
// tools.ts: assembleToolPool()
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
return uniqBy(
  [...builtInTools].sort(byName)         // 内置工具排序为连续前缀
    .concat(allowedMcpTools.sort(byName)), // MCP 工具追加排序
  'name'  // 内置工具名优先
)
```

**设计目的**：内置工具保持稳定排序作为 prompt 前缀，MCP 工具增减不影响前缀的 cache key。

### 8.2 投机执行 (Speculation)

```typescript
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      messagesRef: { current: Message[] }        // 可变引用（效率）
      writtenPathsRef: { current: Set<string> }  // 投机写入路径
      boundary: CompletionBoundary | null        // 投机停止点
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
    }
```

**工作原理**：
1. 用户输入时，预测可能的完整提示
2. 提前启动 API 调用
3. 如果预测正确，直接使用结果（节省等待时间）
4. 如果预测错误，abort 并重新请求

### 8.3 ToolSearch 延迟加载

```typescript
// 工具数量 > 阈值时启用
if (isToolSearchEnabledOptimistic()) {
  // 标记 shouldDefer=true 的工具不注入完整描述
  // 模型通过 ToolSearchTool 按需搜索
  // 减少 system prompt token 消耗
}

// 工具分类
Tool { shouldDefer: true,  alwaysLoad: false }  → 需 ToolSearch 才可调用
Tool { shouldDefer: false, alwaysLoad: true  }  → 始终在 prompt 中
```

### 8.4 并行初始化

```typescript
// preAction Hook: 并行加载
await Promise.all([
  ensureMdmSettingsLoaded(),        // MDM 设置
  ensureKeychainPrefetchCompleted() // 凭证预取
])

// 异步后台（不阻塞启动）
void loadRemoteManagedSettings()    // 远程设置
void loadPolicyLimits()             // 策略限制
```

### 8.5 文件状态 COW (Copy-on-Write) 缓存

```typescript
// QueryEngine 级别
private readFileState: FileStateCache

// 子智能体继承父级缓存的快照
// 子智能体修改不影响父级
// 避免重复读取已知文件
```

---

## 关联文档

- [整体框架设计](./ARCHITECTURE.md) — 架构分层与技术栈
- [模块详细设计](./MODULES.md) — 按目录逐一分析
- [服务层关键分析](./SERVICES_ANALYSIS.md) — 21+ 服务模块深度分析
- [Agent 设计](./AGENT_DESIGN.md) — 智能体架构与多智能体协调
- [框架设计深度解读](./DESIGN_DEEP_DIVE.md) — 设计哲学与工程权衡

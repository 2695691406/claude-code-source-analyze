# Claude Code 框架设计深度解读

> 从架构师视角剖析 Claude Code 整体框架设计的精髓、模式选择与工程权衡
>
> 基于 `@anthropic-ai/claude-code` v2.1.88 源码映射还原分析

---

## 目录

- [1. 架构哲学：为什么选择 React+Ink 终端 UI](#1-架构哲学为什么选择-reactink-终端-ui)
- [2. QueryEngine 核心循环设计](#2-queryengine-核心循环设计)
- [3. AsyncGenerator 管道：流式处理的精髓](#3-asyncgenerator-管道流式处理的精髓)
- [4. 工具抽象体系](#4-工具抽象体系)
- [5. 分层权限系统](#5-分层权限系统)
- [6. Feature Gate 编译期优化](#6-feature-gate-编译期优化)
- [7. 状态管理架构](#7-状态管理架构)
- [8. 上下文压缩策略](#8-上下文压缩策略)
- [9. 扩展性三件套：Plugin / Skill / MCP](#9-扩展性三件套plugin--skill--mcp)
- [10. 多模式运行架构](#10-多模式运行架构)
- [11. 优劣势总评](#11-优劣势总评)

---

## 1. 架构哲学：为什么选择 React+Ink 终端 UI

### 1.1 技术选型

| 技术 | 选择 | 备选方案 |
|------|------|----------|
| UI 框架 | React + Ink | Blessed, Vorpal, Commander.js |
| 运行时 | Node.js ≥18 | Deno, Go, Rust |
| 构建工具 | Bun | esbuild, Webpack, tsc |
| 类型系统 | TypeScript + Zod | TypeScript only, io-ts |
| 模块格式 | ES Module | CommonJS |

### 1.2 React+Ink 的深层原因

传统 CLI 工具通常使用 Commander.js/Yargs 等库，而 Claude Code 选择了 **React+Ink** 这一非常规方案。分析其原因：

**（1）声明式 UI = 复杂状态的自然表达**

Claude Code 不是一个简单的命令行工具，它的 UI 需要同时展示：
- 多个并行 Agent 的进度条
- 流式文本输出（打字机效果）
- 工具执行的实时状态（spinning, completed, failed）
- 权限确认对话框
- Diff 预览
- 虚拟滚动的历史记录

使用命令式 API（如 `process.stdout.write`）管理这些状态的组合爆炸是极其困难的。React 的声明式模型让 UI = f(state) 成为可能：

```typescript
// 声明式：状态变化自动触发 UI 更新
<Box flexDirection="column">
  {tasks.map(task => (
    <TaskProgress key={task.id} status={task.status} />
  ))}
  <StreamingText text={currentResponse} />
  {showPermission && <PermissionDialog tool={pendingTool} />}
</Box>
```

**（2）组件复用 = 一致的 UI 体验**

80+ React 组件和 80+ Hooks 构成了丰富的 UI 组件库。同一个 `DiffView` 组件可以在交互模式、IDE 集成和 SDK 中复用，保证了一致的视觉体验。

**（3）Hooks = 复杂逻辑的优雅封装**

```typescript
// useVirtualScroll - 终端虚拟滚动
// useVimInput - Vim 键绑定
// useArrowKeyHistory - 命令历史导航
// useIDEIntegration - IDE 双向通信
// useVoice - 语音输入
```

每个 Hook 封装了一个独立的交互逻辑维度，可以自由组合而不产生耦合。

**（4）代价与权衡**

| 优势 | 代价 |
|------|------|
| 声明式 UI 管理 | 运行时开销（React reconciler） |
| 组件/Hook 复用 | 包体积增大（React + Ink 依赖） |
| 生态系统（TypeScript 类型推导） | 终端渲染的性能上限 |
| 测试友好（组件快照测试） | 调试复杂度（React DevTools 不可用） |

> **设计洞察**：Claude Code 实质上是一个**终端应用**而非简单 CLI。React+Ink 的选择反映了"将终端作为 GUI 平台"的思维转变。

### 1.3 Bun 构建 + Feature Gate 的组合拳

选择 Bun 而非 esbuild/Webpack，核心原因是其 **宏系统** 支持编译期代码消除：

```typescript
// 编译期求值，而非运行时 if-else
const snipModule = feature('HISTORY_SNIP')
  ? (require('./services/compact/snipCompact.js') as typeof import('./services/compact/snipCompact.js'))
  : null
```

Bun 的 `bun:bundle` 在打包时将 `feature()` 求值为常量布尔值，Terser 随后消除不可达分支。这比运行时 Feature Flag（如 LaunchDarkly）更彻底——**死代码连字符串字面量都不会进入 bundle**。

---

## 2. QueryEngine 核心循环设计

### 2.1 一个类，一个对话

```typescript
export class QueryEngine {
  constructor(config: QueryEngineConfig)
  async *submitMessage(prompt, options?): AsyncGenerator<SDKMessage>
  interrupt(): void
  getMessages(): readonly Message[]
  setModel(model: string): void
}
```

**设计决策**：一个 QueryEngine 实例 = 一个对话会话。状态（消息、用量、文件缓存）在多次 `submitMessage()` 调用间持久化，但跨会话通过重新构建来隔离。

**与 LangChain 的对比**：

| 维度 | Claude Code QueryEngine | LangChain Agent |
|------|------------------------|-----------------|
| 生命周期 | 长生命周期，跨多轮 | 通常单次调用 |
| 状态管理 | 内置消息数组 + 文件缓存 | 依赖外部 Memory 类 |
| 工具执行 | 内联于查询循环 | 通过 AgentExecutor 调度 |
| 流式支持 | 原生 AsyncGenerator | 回调/Stream 适配器 |
| 错误恢复 | 内置 Fallback 模型 + 重试 | 需自行实现 |

### 2.2 七阶段生命周期

```
submitMessage() 的完整生命周期：

Phase 1: 构造（一次性）
  └─ 消息数组、AbortController、文件缓存初始化

Phase 2: 初始化（每次调用）
  ├─ 2a. 权限包装（wrappedCanUseTool 装饰器）
  ├─ 2b. 系统提示组装（customPrompt + memoryMechanics + appendPrompt）
  ├─ 2c. 两阶段 ProcessUserInputContext（可变→不可变）
  ├─ 2d. 孤立权限处理（前会话遗留的权限请求）
  ├─ 2e. 用户输入处理（斜杠命令、附件、模型切换）
  └─ 2f. 会话记录（转录写入，bare 模式 fire-and-forget）

Phase 3: 本地命令短路
  └─ shouldQuery === false → 跳过 API 调用，直接返回

Phase 4: 文件历史快照
  └─ 为每条用户消息创建 Git 快照

Phase 5: 查询循环（核心消息泵）
  └─ for await (const message of query({...}))
      ├─ 消息类型分发（assistant/progress/user/stream_event/attachment/system）
      ├─ 用量追踪（message_start/message_delta/message_stop 三阶段累计）
      ├─ 转录记录（assistant fire-and-forget / 其他 await）
      └─ 预算守卫（USD 上限 / 结构化输出重试限制）

Phase 6: 结果验证与最终化
  ├─ isResultSuccessful 判定（assistant+text+end_turn）
  ├─ 水印式错误收集（仅包含本轮新增错误）
  └─ 结果 yield（success/error_during_execution/error_max_turns/...）

Phase 7: 清理与会话刷新
  └─ Desktop/Cowork 模式立即刷新；bare 模式延迟刷新
```

### 2.3 巧妙设计：两阶段 ProcessUserInputContext

**问题**：斜杠命令（如 `/model claude-3.5-sonnet`）需要修改消息数组和模型设置。但后续代码假设状态已冻结。

**解决方案**：

```typescript
// 阶段一：允许突变
processUserInputContext = {
  setMessages: fn => { this.mutableMessages = fn(this.mutableMessages) },
  // ...
}

// 处理斜杠命令（可能调用 setMessages）
const { messages, shouldQuery, model } = await processUserInput({
  context: processUserInputContext, ...
})

// 阶段二：冻结
processUserInputContext = {
  setMessages: () => {},  // no-op
  // ...
}
```

**精妙之处**：不是用 `Object.freeze()` 或 `readonly` 运行时检查，而是通过**配置级别的行为切换**——同一个接口，不同的实现。编译器无法捕捉这种时序约束，但这种设计让意图在代码中自文档化。

### 2.4 巧妙设计：水印式错误范围限定

**问题**：`getInMemoryErrors()` 是一个全局环形缓冲区（100 条）。长时间运行的会话中，旧错误（如上一小时的 ripgrep 超时）会污染当前轮次的错误报告。

**解决方案**：

```typescript
// 轮次开始时，记录当前"水位线"
const errorLogWatermark = getInMemoryErrors().at(-1)

// 轮次结束时，只取水位线之后的新错误
const all = getInMemoryErrors()
const start = errorLogWatermark
  ? all.lastIndexOf(errorLogWatermark) + 1
  : 0
errors: [...all.slice(start)]
```

**精妙之处**：
- 使用**对象引用**而非数组索引作为水位线——即使环形缓冲区旋转多圈，`lastIndexOf` 仍能定位
- 水位线不存在（已被挤出缓冲区）时，`lastIndexOf` 返回 -1，`-1 + 1 = 0`，回退到"包含全部"——安全的降级行为
- 零额外内存开销（不复制错误数组）

---

## 3. AsyncGenerator 管道：流式处理的精髓

### 3.1 管道架构

```
用户输入
  ↓
processUserInput() [本地处理]
  ├─ 解析斜杠命令
  ├─ 验证附件
  ├─ 更新工具/模型
  → yield 消息
  ↓
[可选: 本地命令短路返回]
  ↓
query() [API 循环 - AsyncGenerator]
  ├─ 系统提示组装
  ├─ 消息发送至 API
  ├─ 流式响应解析 (content_block_start/delta/stop, message_start/delta/stop)
  ├─ 工具调用 (可嵌套：Agent 工具触发递归 query())
  └─ yield 控制流消息 (progress, attachment, tombstone)
  ↓
submitMessage() [调度器 - AsyncGenerator]
  ├─ 包装 query() 输出
  ├─ 规范化消息类型
  ├─ 追踪状态 (用量、权限)
  ├─ 持久化到磁盘 (转录)
  ├─ 应用预算守卫
  └─ yield 给调用方
  ↓
调用方 (ask() 包装器或 SDK 消费者)
  ├─ 逐条处理消息
  ├─ 可通过 abortController 取消
  └─ 消费直到 result 消息
```

### 3.2 为什么选择 AsyncGenerator 而非 EventEmitter/Observable

| 维度 | AsyncGenerator | EventEmitter | RxJS Observable |
|------|---------------|-------------|-----------------|
| **背压** | ✅ 天然支持（caller 控制迭代速度） | ❌ 无背压，可能溢出 | ✅ 支持但需 operator |
| **取消** | ✅ AbortController + return() | ⚠️ removeListener 易泄漏 | ✅ unsubscribe |
| **类型安全** | ✅ `AsyncGenerator<Yield, Return, Next>` | ❌ 字符串事件名，弱类型 | ✅ `Observable<T>` |
| **组合** | ⚠️ 手动（for-await + yield*） | ❌ 回调地狱 | ✅ 管道 operator |
| **错误传播** | ✅ try-catch 自然传播 | ⚠️ 'error' 事件可能未处理 | ✅ error() 回调 |
| **懒执行** | ✅ 直到 .next() 才执行 | ❌ 注册即触发 | ✅ subscribe 才执行 |
| **内存效率** | ✅ 逐条处理，无缓冲 | ⚠️ 队列可能增长 | ⚠️ 取决于 operator |
| **零依赖** | ✅ 语言内置 | ✅ Node.js 内置 | ❌ 需 rxjs 包 |
| **调试** | ✅ 堆栈追踪清晰 | ❌ 异步回调堆栈混乱 | ⚠️ operator 链模糊 |

**Claude Code 的关键需求**是：
1. **流式传输** — API 响应逐 token 到达，需要立即传递给 UI
2. **背压控制** — UI 渲染速度可能慢于 API 传输速度
3. **中途取消** — 用户随时可以 Ctrl+C
4. **嵌套组合** — Agent 工具需要递归调用 `query()`，内层 generator 的消息需要向外传播

AsyncGenerator 的 `yield*` 语法完美支持嵌套传播，这是 EventEmitter 和 Observable 都难以优雅实现的：

```typescript
// query() 内部：Agent 工具触发嵌套 query
async function* query(params) {
  // ...
  for await (const msg of nestedQuery(agentParams)) {
    yield msg  // 内层消息自然向外传播
  }
}

// submitMessage() 内部：包装 query()
async *submitMessage(prompt) {
  for await (const msg of query({...})) {
    yield* normalizeMessage(msg)  // 一条消息可能展开为多条 SDK 消息
  }
}
```

### 3.3 巧妙设计：Fire-and-Forget 转录写入

**问题**：转录写入会阻塞 generator 的下一次迭代。如果 generator 等待磁盘 I/O 完成，`message_delta` 事件就无法及时处理，导致流式延迟。

**解决方案**：

```typescript
if (message.type === 'assistant') {
  void recordTranscript(messages)  // Fire-and-forget
} else {
  await recordTranscript(messages)  // 阻塞等待
}
```

**为什么安全**：
- `message_delta` 会原地修改最后一条 assistant 消息的 usage/stop_reason
- 写入队列已经以 100ms 间隔延迟字符串化
- 多次快速修改合并为一次写入
- `enqueueWrite` 保证顺序，不会丢失
- **效果**：流式响应的逐 block 延迟控制在 <1ms

**Bare 模式进一步优化**：

```typescript
if (isBareMode()) {
  void transcriptPromise  // 脚本模式：连用户消息的记录都 fire-and-forget
} else {
  await transcriptPromise  // 交互模式：必须等待（支持 resume）
}
```

**影响**：bare 模式每轮次节省 ~4ms（5-10 轮次累计 20-40ms）。

### 3.4 巧妙设计：三阶段用量追踪

**问题**：一条 API 消息被拆分为 5+ 个流事件（message_start, content_block_delta×N, message_delta, message_stop）。如何正确累计用量？

```typescript
let currentMessageUsage = EMPTY_USAGE

// message_start: 初始用量（prompt tokens）
if (event.type === 'message_start') {
  currentMessageUsage = updateUsage(currentMessageUsage, event.message.usage)
}

// message_delta: 增量用量（output tokens, stop_reason）
if (event.type === 'message_delta') {
  currentMessageUsage = updateUsage(currentMessageUsage, event.usage)
}

// message_stop: 消息完成，累加到总计
if (event.type === 'message_stop') {
  this.totalUsage = accumulateUsage(this.totalUsage, currentMessageUsage)
  currentMessageUsage = EMPTY_USAGE  // 重置
}
```

**不变量**：`totalUsage` 始终是已完成消息的精确总和，绝不包含部分消息的用量。

---

## 4. 工具抽象体系

### 4.1 Tool 类型定义

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 身份与元数据
  readonly name: string
  aliases?: string[]
  searchHint?: string           // 3-10 词，供 ToolSearch 关键词匹配

  // Schema 与验证
  readonly inputSchema: Input   // Zod Schema
  outputSchema?: z.ZodType<unknown>

  // 行为谓词
  isConcurrencySafe(input): boolean    // 可否并行执行
  isReadOnly(input): boolean           // 是否只读
  isDestructive?(input): boolean       // 是否具破坏性
  isEnabled(): boolean                 // 特性开关

  // 执行
  async call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>

  // 权限
  async checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>

  // 渲染
  async description(input, options): Promise<string>
  userFacingName(input): string
  renderToolUseProgressMessage?(...): React.ReactNode
  renderToolResultMessage?(...): React.ReactNode
}
```

### 4.2 Zod Schema 的双重保障

Claude Code 对工具输入使用 **Zod Schema** 而非简单的 TypeScript 类型或 JSON Schema：

```typescript
const inputSchema = z.object({
  command: z.string().describe('Bash command to execute'),
  timeout: z.number().optional().describe('Timeout in milliseconds'),
})
```

**双重保障机制**：

| 层次 | 保障 | 时机 |
|------|------|------|
| TypeScript 类型推导 | `z.infer<typeof inputSchema>` | 编译期 |
| Zod 运行时验证 | `inputSchema.parse(input)` | 运行时（API 返回数据） |

**为什么需要运行时验证**：Claude API 返回的 tool_use 参数是 JSON，TypeScript 类型在运行时消失。Zod 提供了：
- 自动类型收窄（parse 后的值是强类型的）
- 描述性错误消息（哪个字段、什么类型不匹配）
- `.describe()` 注解直接作为 Claude 的工具参数描述

**验证流程**：

```
Claude API 返回 tool_use
  → Zod parse (运行时类型检查)
    → validateInput() (业务逻辑检查)
      → checkPermissions() (权限检查)
        → call() (实际执行)
```

**精妙之处**：无效输入在权限检查**之前**被拒绝——权限系统永远不会看到畸形数据。

### 4.3 Fail-Closed 默认值

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,    // 默认不安全 → 串行执行
  isReadOnly: () => false,            // 默认可写 → 需要权限
  isDestructive: () => false,
  checkPermissions: () => ({ behavior: 'allow', updatedInput: input }),
}
```

**设计哲学**：**Fail-Closed（默认关闭）**。新工具如果忘记声明 `isConcurrencySafe`，将被串行执行（安全但慢），而非并行执行（快但可能出错）。同样，忘记声明 `isReadOnly` 的工具将触发权限检查。

这是安全系统设计的黄金法则：**安全的默认行为，明确的放宽声明**。

### 4.4 工具搜索与延迟加载

**问题**：43+ 工具的描述注入系统提示会消耗大量 token。

**解决方案**：`shouldDefer` + `ToolSearchTool`

```typescript
// 工具标记为可延迟
{ name: 'NotebookEdit', shouldDefer: true, searchHint: 'edit jupyter notebook cells' }

// ToolSearchTool 在需要时按关键词搜索
ToolSearchTool.call({ query: 'notebook' })
// → 返回 NotebookEdit 的完整描述
```

**效果**：只有 `alwaysLoad: true` 的核心工具（Bash, FileRead, FileEdit, Grep, Glob 等）出现在初始系统提示中。其余工具按需加载，节省 token 预算。

---

## 5. 分层权限系统

### 5.1 四层决策架构

Claude Code 的权限系统是一个 **4 层过滤管道**，每层逐步收窄决策空间：

```
                     ┌─────────────────────────────────┐
                     │  LAYER 1: 拒绝规则 & 工具级检查  │
                     │  (即使 bypassPermissions 也生效)  │
                     │                                   │
                     │  1a. 整体工具拒绝规则             │
                     │  1b. 整体工具询问规则             │
                     │  1c. Zod 验证 + checkPermissions()│
                     │  1d. 工具返回 deny → 立即拒绝     │
                     │  1e. 需要用户交互 → 必须询问      │
                     │  1f. 内容特定询问规则             │
                     │  1g. 安全检查 (.git/, .claude/)   │
                     └──────────────┬────────────────────┘
                                    │
                     ┌──────────────▼────────────────────┐
                     │  LAYER 2: 模式决策                │
                     │                                   │
                     │  2a. bypassPermissions → 允许      │
                     │  2b. 整体工具允许规则 → 允许       │
                     └──────────────┬────────────────────┘
                                    │
                     ┌──────────────▼────────────────────┐
                     │  LAYER 3: Passthrough → Ask 转换  │
                     │                                   │
                     │  工具返回 passthrough → 转为 ask   │
                     └──────────────┬────────────────────┘
                                    │
                     ┌──────────────▼────────────────────┐
                     │  LAYER 4: 后处理 & 分类器         │
                     │                                   │
                     │  • dontAsk: ask → deny             │
                     │  • auto: YOLO 分类器 (Claude API)  │
                     │  • Hooks: 外部拦截                 │
                     │  • Bash 分类器: 危险命令检测        │
                     └──────────────────────────────────┘
```

### 5.2 权限模式

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `default` | 每次工具使用都提示用户 | 标准交互模式 |
| `plan` | 暂停执行（仅规划） | 代码审查前 |
| `acceptEdits` | 自动允许安全编辑（FileEdit, FileRead） | 快速迭代 |
| `bypassPermissions` | 自动允许所有工具 | 受信环境 |
| `dontAsk` | 自动拒绝所有（ask → deny） | 锁定模式 |
| `auto` | AI 安全分类器自动判断（仅内部） | 自主 Agent |

### 5.3 Bash 分类器：危险命令检测

对于 Bash 工具，权限系统有额外的**两阶段分类**：

```
第一阶段：模式匹配（快速）
  ├─ 拒绝描述匹配（rm -rf, curl | sh, ...）
  ├─ 询问描述匹配（docker rm, git push --force, ...）
  ├─ 允许描述匹配（ls, cat, grep, ...）
  └─ 危险模式正则（dangerousPatterns.ts）

第二阶段：YOLO 分类器（昂贵，Claude API 调用）
  ├─ 仅在模式匹配无法判断时触发
  ├─ 发送独立 API 请求（sideQuery）
  │   ├─ 系统提示（auto_mode_system_prompt.txt）
  │   ├─ 权限规则（permissions_external.txt）
  │   ├─ 动作描述："Tool: Bash, Command: ..."
  │   └─ 对话历史（提供上下文）
  └─ Claude 响应解析为 allow/deny/ask 决策
```

**精妙之处**：
- **赛跑模式**：Bash 分类器与 UI 提示并行执行。如果分类器在用户点击之前返回 `allow`，直接跳过提示
- **连续拒绝追踪**：如果 auto 模式连续拒绝 N 次，自动回退到用户提示模式（防止 Agent 卡死）
- **成本优化**：sideQuery 使用 prompt caching 减少开销

### 5.4 安全检查的旁路免疫

Layer 1g 中的安全检查是**旁路免疫**的——即使 `bypassPermissions` 模式也无法跳过：

```typescript
// 这些路径的操作始终需要用户确认
.git/        → Git 数据库直接操作
.claude/     → Claude Code 配置修改
.vscode/     → IDE 设置修改
```

**设计哲学**：**没有任何模式可以绕过对关键资源的保护**。这类似于操作系统中 Ring 0 的概念——某些操作即使 root 用户也需要额外确认。

---

## 6. Feature Gate 编译期优化

### 6.1 机制

```typescript
// 定义（构建脚本中）
export function feature(name: FeatureFlag): boolean {
  return ENABLED_FEATURES.includes(name)
}

// 使用
if (feature('COORDINATOR_MODE')) {
  // 编译期求值为 true/false
  // Bun bundler 消除不可达分支
}
```

### 6.2 20+ 特性标识

基于源码中出现的 `feature()` 调用统计：

| 类别 | 示例 |
|------|------|
| 智能体功能 | `COORDINATOR_MODE`, `AGENT_SWARMS`, `FORK_SUBAGENT` |
| 压缩策略 | `HISTORY_SNIP`, `CONTEXT_COLLAPSE`, `MICROCOMPACT` |
| 安全功能 | `AUTO_MODE`, `BASH_CLASSIFIER`, `YOLO_CLASSIFIER` |
| UI 特性 | `VIRTUAL_SCROLL`, `DIFF_IN_IDE`, `BUDDY_AI` |
| 集成 | `BRIDGE_V2`, `KAIROS`, `VOICE_INPUT` |

### 6.3 与运行时 Feature Flag 的对比

| 维度 | 编译期 Feature Gate | 运行时 Feature Flag (LaunchDarkly/GrowthBook) |
|------|-------------------|----------------------------------------------|
| 代码消除 | ✅ 死代码完全移除 | ❌ 代码始终存在 |
| 字符串泄漏 | ✅ 不泄漏 | ❌ 错误消息、日志中可能泄漏 |
| 包体积 | ✅ 更小 | ❌ 无影响 |
| 灵活性 | ❌ 需要重新构建 | ✅ 动态切换 |
| A/B 测试 | ❌ 不支持 | ✅ 原生支持 |
| 回滚速度 | ❌ 需要重新部署 | ✅ 秒级回滚 |

**Claude Code 的策略**：编译期 Gate 用于**大功能开关**（是否包含 Coordinator 模式的全部代码），运行时 Flag（GrowthBook `getFeatureValue`）用于**行为调优**（投机执行的阈值、轮询间隔等）。两者互补。

---

## 7. 状态管理架构

### 7.1 AppState 的类 Zustand 设计

```typescript
// 状态读取
getAppState: () => AppState

// 状态更新（函数式不可变更新）
setAppState: (f: (prev: AppState) => AppState) => void
```

**为什么不直接用 Redux/Zustand**？

1. **终端环境限制**：Redux DevTools 不可用；Zustand 的 React 集成依赖 `useSyncExternalStore`，在 Ink 中行为不一致
2. **SDK 模式需求**：SDK 调用者需要注入自己的 `getAppState/setAppState` 实现（如存储在数据库中）
3. **极简设计**：Claude Code 的状态更新模式非常一致（函数式不可变更新），不需要 reducer/action/middleware 这些抽象

**不可变更新模式**：

```typescript
setAppState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    alwaysAllowRules: {
      ...prev.toolPermissionContext.alwaysAllowRules,
      command: allowedTools
    }
  }
}))
```

**优势**：
- 安全的并发读取（前一个版本不受后续修改影响）
- 可测试（`setAppState` 可以 mock）
- SDK 可注入自定义实现

**劣势**：
- 深层嵌套更新冗长（没有 Immer 的语法糖）
- 没有 middleware 机制（需手动实现日志/持久化）

### 7.2 React Context 集成

在交互模式中，AppState 通过 React Context 注入到组件树：

```typescript
// 顶层 Provider
<AppStateContext.Provider value={{ getAppState, setAppState }}>
  <ChatInterface />
</AppStateContext.Provider>

// 组件消费
function PermissionDialog() {
  const { getAppState } = useContext(AppStateContext)
  const mode = getAppState().toolPermissionContext.mode
  // ...
}
```

**精妙之处**：`getAppState` 是一个**函数引用**而非状态值——组件不会因为无关的状态变更而重新渲染。只有当组件调用 `getAppState()` 并使用返回值时，才会读取最新状态。这是一种手动的"选择性订阅"模式。

---

## 8. 上下文压缩策略

### 8.1 五层压缩架构

Claude Code 在 `query()` 循环中实现了 **5 层递进式上下文压缩**：

```
长对话的消息数组
  ↓
Layer 1: Tool Result Budget（工具结果预算）
  ├─ 单条消息中工具结果的聚合大小超过阈值
  └─ 截断/汇总过大的工具结果
  ↓
Layer 2: History Snip（历史截断）
  ├─ snipCompactIfNeeded()
  └─ 将远端历史压缩为摘要，保留近端完整消息
  ↓
Layer 3: Microcompact（微压缩）
  ├─ 缓存感知的逐消息压缩
  └─ 保留 API prompt cache 命中
  ↓
Layer 4: Context Collapse（上下文折叠）
  ├─ 将归档上下文折叠为紧凑表示
  └─ 跳过已确认的中间结果
  ↓
Layer 5: Autocompact（自动压缩）
  ├─ 完整的消息历史摘要（调用 Claude API）
  └─ 当 token 使用接近上下文窗口限制时触发
  ↓
压缩后的消息数组 → 发送给 Claude API
```

### 8.2 精妙之处：缓存感知压缩

Microcompact 的设计考虑了 **Claude API 的 prompt caching** 机制：

- Prompt caching 要求消息前缀保持不变
- 如果压缩修改了消息前缀，缓存失效
- Microcompact 只压缩**不在缓存窗口中**的消息
- 效果：减少 token 使用的同时保持缓存命中率

### 8.3 Snip 边界重放

```typescript
const snipResult = this.config.snipReplay?.(message, this.mutableMessages)

if (snipResult?.executed) {
  // 压缩模块处理了消息替换
  this.mutableMessages.length = 0
  this.mutableMessages.push(...snipResult.messages)

  // 释放压缩前的消息以便 GC 回收
  this.mutableMessages.splice(0, boundaryIdx)
}
```

**精妙之处**：
- Feature-gated 回调设计让排除的字符串不进入 QueryEngine
- 重放发生在我们自己的存储上（不只是信号）
- `splice` 移除边界前的消息以便 GC
- 防止僵尸消息重新触发压缩

---

## 9. 扩展性三件套：Plugin / Skill / MCP

### 9.1 三种扩展机制对比

| 维度 | Plugin | Skill | MCP |
|------|--------|-------|-----|
| **定义方式** | JS/TS 模块 | `CLAUDE.md` + 斜杠命令 | 独立进程/服务 |
| **执行环境** | 进程内 | Bash/工具组合 | 独立进程（stdio/SSE） |
| **能力范围** | 工具注册、Hook、UI | 工具调用序列 | 工具、资源、提示 |
| **安全隔离** | ❌ 共享进程空间 | ✅ 通过工具权限系统 | ✅ 进程隔离 + 权限 |
| **发现方式** | 配置文件注册 | CLAUDE.md 自动发现 | 服务连接配置 |
| **热重载** | ⚠️ 需要缓存刷新 | ✅ 每次调用重新读取 | ✅ 重连即可 |
| **适用场景** | 深度框架扩展 | 工作流自动化 | 外部系统集成 |
| **复杂度** | 高 | 低 | 中 |

### 9.2 MCP 集成的深度设计

MCP (Model Context Protocol) 是最灵活的扩展方式。Claude Code 的 MCP 集成有几个精妙设计：

**（1）MCP 工具升格为一等公民**

```typescript
// MCP 工具与内置工具享有相同的类型签名
const mcpTool: Tool = {
  name: `mcp__${serverName}__${toolName}`,
  isMcp: true,
  mcpInfo: { serverName, toolName },
  inputSchema: zodSchemaFromJsonSchema(tool.inputSchema),
  // ... 完整的 Tool 接口实现
}
```

**（2）跨 Agent MCP 共享**

```typescript
// Agent 定义中引用共享 MCP 服务器
mcpServers: ['slack']  // 引用已有服务器，使用缓存连接

// 或定义 Agent 专属服务器
mcpServers: [{ mydb: { command: 'npx', args: ['@mcp/db'] } }]
// Agent 专属服务器在 Agent 结束时清理
```

**（3）权限继承**

MCP 工具继承宿主权限系统的全部 4 层检查，没有绕过路径。

### 9.3 Skill 系统的轻量设计

Skill 是最轻量的扩展方式——本质上是 `CLAUDE.md` 中定义的命名工具调用序列：

```markdown
<!-- CLAUDE.md -->
## Skills

### /deploy
1. Run `npm run build`
2. Run `npm test`
3. Run `./scripts/deploy.sh`
```

**精妙之处**：Skill 不需要任何代码——它们是**自然语言指令**，由 Claude 解释执行。这使得非开发者也能扩展 Claude Code 的能力。

---

## 10. 多模式运行架构

### 10.1 八种运行模式

```
                    ┌──────────────────────────────────┐
                    │          统一入口点               │
                    │     QueryEngine + query()        │
                    └──────────┬───────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   ┌────▼────┐           ┌────▼────┐           ┌────▼────┐
   │  CLI    │           │  SDK   │           │ Bridge │
   │ (交互式) │           │ (编程式)│           │ (远程)  │
   └────┬────┘           └────┬────┘           └────┬────┘
        │                      │                      │
   ┌────▼────┐           ┌────▼────┐           ┌────▼────┐
   │非交互式 │           │Coordinator│          │ Daemon │
   │ (管道)  │           │ (多Agent)│          │ (后台)  │
   └─────────┘           └────┬────┘           └─────────┘
                               │
                    ┌────▼────┐    ┌────▼────┐
                    │ KAIROS │    │ Voice  │
                    │ (助手)  │    │ (语音)  │
                    └─────────┘    └─────────┘
```

### 10.2 一个 QueryEngine 如何支撑八种模式

**关键**：QueryEngine 通过**配置注入**而非**继承/子类**来适配不同模式。

| 配置项 | CLI | SDK | Bridge | Coordinator |
|--------|-----|-----|--------|-------------|
| `getAppState/setAppState` | React Context | 调用方注入 | Bridge State | 共享 |
| `canUseTool` | UI 提示 | 自动允许/拒绝 | 远程权限 | Worker 策略 |
| `tools` | 完整工具集 | 过滤后工具集 | 远程工具集 | Worker 工具集 |
| `includePartialMessages` | false | 可选 true | false | false |
| `maxTurns` | 无限制 | 调用方指定 | 服务端指定 | 无限制 |
| `customSystemPrompt` | null | 调用方指定 | null | Coordinator 提示 |
| `abortController` | Ctrl+C | 调用方控制 | WebSocket 关闭 | 父 Agent 控制 |

**设计洞察**：这是一种"**配置即策略**"的架构模式。不同模式不需要不同的类层次结构——只需要不同的配置对象。这极大地简化了代码库，避免了继承层次爆炸。

### 10.3 模式判断的决策树

```
环境变量 / CLI 参数
  ├─ --sdk / 编程式调用 → SDK 模式
  ├─ CLAUDE_CODE_IS_BRIDGE → Bridge 模式
  ├─ CLAUDE_CODE_COORDINATOR_MODE → Coordinator 模式
  ├─ CLAUDE_CODE_IS_KAIROS → KAIROS 助手模式
  ├─ CLAUDE_CODE_IS_DAEMON → Daemon 后台模式
  ├─ CLAUDE_CODE_VOICE → Voice 语音模式
  ├─ 管道输入 (stdin.isTTY === false) → 非交互模式
  └─ 默认 → CLI 交互模式
```

---

## 11. 优劣势总评

### 11.1 架构优势

| 优势 | 说明 |
|------|------|
| **AsyncGenerator 管道** | 流式、背压、取消、嵌套组合——一个原语解决四个问题 |
| **配置即策略** | 一个 QueryEngine 支撑 8 种模式，避免继承爆炸 |
| **Fail-Closed 安全默认** | 新工具默认需要权限，忘记声明 = 安全但慢 |
| **编译期 Feature Gate** | 代码消除比运行时 Flag 更彻底 |
| **Zod 双重验证** | 编译期类型推导 + 运行时 Schema 验证 |
| **分层权限** | 4 层过滤管道 + 旁路免疫安全检查 |
| **水印式错误范围** | 零开销的轮次级错误隔离 |
| **Copy-on-Write 缓存** | Agent 间文件缓存隔离，零初始开销 |

### 11.2 潜在改进空间

| 方面 | 现状 | 潜在改进 |
|------|------|----------|
| **深层嵌套状态更新** | 手动展开运算符 | 引入 Immer 或 lens 库简化 |
| **测试复杂度** | AsyncGenerator 测试需要完整迭代 | 可考虑提供测试辅助工具 |
| **包体积** | React + Ink + 43 工具 = 较大 | Tree-shaking 或按需加载 |
| **错误堆栈** | AsyncGenerator 链中堆栈可能不完整 | 可考虑添加 AsyncContext 追踪 |
| **Middleware 缺失** | 无 Redux-style middleware | 可考虑添加状态变更 Hook |
| **工具注册全局性** | 所有工具在启动时注册 | 可考虑延迟注册减少启动时间 |
| **Coordinator 提示长度** | ~370 行系统提示 | 可考虑拆分为独立提示模块 |

### 11.3 设计模式总结

| 模式 | 应用位置 | 解决的问题 |
|------|----------|-----------|
| AsyncGenerator Pipeline | QueryEngine → query() → submitMessage() | 流式、背压、嵌套 |
| Dependency Injection | QueryEngineConfig | 模式适配、测试 |
| Decorator | wrappedCanUseTool | 权限追踪透明注入 |
| Feature Gate | feature() + Bun bundle | 编译期代码消除 |
| Fail-Closed Default | TOOL_DEFAULTS | 安全默认行为 |
| Copy-on-Write | cloneFileStateCache | 隔离无初始开销 |
| Watermark | errorLogWatermark | 轮次级错误范围 |
| Two-Phase Config | ProcessUserInputContext | 可变→不可变状态转换 |
| Fire-and-Forget | recordTranscript | 异步 I/O 不阻塞流 |
| Lazy Import | messageSelector() | 按需加载重量级模块 |
| Strategy | 消息类型 switch | 可扩展的类型分发 |
| Builder | buildTool() | 安全默认 + 用户覆盖 |

---

> **结语**：Claude Code 的框架设计展现了一种"**务实的优雅**"——不追求理论上的完美抽象，而是在具体约束（终端 UI、流式 API、多模式运行、安全需求）下做出最佳权衡。AsyncGenerator 管道、Feature Gate 编译优化、4 层权限系统这些设计都是对具体问题的精准回应，而非泛用框架的堆砌。

# Claude Code Agent 设计深度解读

> 从 AI Agent 系统设计的角度深入剖析多智能体架构的精髓与权衡
>
> 基于 `@anthropic-ai/claude-code` v2.1.88 源码映射还原分析

---

## 目录

- [1. Agent 即一等公民的设计哲学](#1-agent-即一等公民的设计哲学)
- [2. 递归嵌套智能体](#2-递归嵌套智能体)
- [3. 四种执行模式深度对比](#3-四种执行模式深度对比)
- [4. Coordinator 多智能体编排](#4-coordinator-多智能体编排)
- [5. 五路通信机制设计](#5-五路通信机制设计)
- [6. Agent 记忆与上下文](#6-agent-记忆与上下文)
- [7. 任务框架设计](#7-任务框架设计)
- [8. Bridge 远程控制架构](#8-bridge-远程控制架构)
- [9. 安全与权限的精细控制](#9-安全与权限的精细控制)
- [10. 设计模式总结与优劣评析](#10-设计模式总结与优劣评析)

---

## 1. Agent 即一等公民的设计哲学

### 1.1 核心设计决策：每个 Agent 独立 query() 执行作用域

Claude Code 的 Agent 系统基于一个关键决策：**每个 Agent 获得独立的 `query()` 执行作用域**，而非共享单一的对话上下文。

```
父 QueryEngine（主对话）
  messages: [user, assistant, tool_use:Agent]
  ├─ AgentTool("research-agent")
  │   └─ 独立的 query() 循环
  │       ├─ 独立消息数组
  │       ├─ 独立文件缓存（cloneFileStateCache）
  │       ├─ 独立权限追踪
  │       ├─ 独立用量累计
  │       └─ 可调用任何可用工具
  │       → 返回 tool_result 给父
  └─ 继续父 query() 循环
```

**代码实现**（`runAgent.ts`）：

```typescript
// 每个 Agent 获得独立的文件缓存
const agentReadFileState = forkContextMessages !== undefined
  ? cloneFileStateCache(toolUseContext.readFileState)  // Clone if fork
  : createFileStateCacheWithSizeLimit(READ_FILE_STATE_CACHE_SIZE)

// 每个 Agent 获得独立的工具池和权限模式
const workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits'
}
const workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools)
```

### 1.2 与其他框架的对比

| 维度 | Claude Code | AutoGPT | CrewAI | LangGraph |
|------|------------|---------|--------|-----------|
| **Agent 隔离** | 独立 query() + 文件缓存 + 权限 | 共享 workspace | 共享 memory | 共享 state |
| **嵌套能力** | 无限递归嵌套 | 不支持 | 1 级委托 | 通过子图 |
| **通信方式** | 5 种路径（详见第 5 节） | 共享文件系统 | 直接函数调用 | State channel |
| **工具隔离** | 每 Agent 独立工具池 | 共享工具集 | 每 Agent 配置 | 每 Node 配置 |
| **执行模式** | 同步/异步/远程/进程内 | 仅同步 | 同步+委托 | 状态机 |
| **VCS 隔离** | Git Worktree | 无 | 无 | 无 |

**Claude Code 的独特优势**：

1. **隔离即安全**：Agent 间的文件缓存隔离防止了"脏读"——一个 Agent 的文件修改不会影响另一个 Agent 的缓存视图，直到显式同步
2. **权限继承可控**：子 Agent 可以有比父 Agent **更严格**的权限（如 `acceptEdits` → `plan`），但不能拥有比父 Agent 更宽松的权限
3. **VCS 隔离是杀手锏**：通过 `git worktree` 让 Agent 在独立分支上工作，完全避免了并行 Agent 间的文件冲突

### 1.3 隔离的代价

| 优势 | 代价 |
|------|------|
| 安全的并行执行 | 每个 Agent 消耗独立的上下文窗口 token |
| 无脏读/脏写 | 文件缓存克隆的内存开销 |
| 独立的权限控制 | 跨 Agent 信息共享需要显式通信 |
| 可独立取消/恢复 | 每个 Agent 的 API 调用是独立的（无法共享 prompt cache） |

**例外——Fork 路径的缓存优化**：

```typescript
// Fork 子 Agent 继承父 Agent 的完整对话上下文
const contextMessages = forkContextMessages
  ? filterIncompleteToolCalls(forkContextMessages)
  : []

// 所有 Fork 子 Agent 共享字节级相同的消息前缀 → prompt cache 命中
[
  ...history,                          // 相同
  assistant(all_tool_uses, thinking),   // 相同
  user(placeholder_results, directive)  // 仅最后的 directive 不同
]
```

这个设计让 Fork 分支的 Agent 们在享受隔离的同时，复用了父 Agent 的 prompt cache——**隔离性与经济性的巧妙平衡**。

---

## 2. 递归嵌套智能体

### 2.1 无限递归支持

Claude Code 支持任意深度的 Agent 嵌套：

```
用户 → 主 Agent
        ├─ 子 Agent A（研究）
        │   ├─ 孙 Agent A1（搜索代码）
        │   └─ 孙 Agent A2（搜索文档）
        ├─ 子 Agent B（实现）
        │   └─ 孙 Agent B1（测试）
        └─ 子 Agent C（审查）
```

**实现机制**：AgentTool 本身就是一个 Tool，其 `call()` 方法内部调用 `query()`。由于 `query()` 可以执行任何 Tool（包括 AgentTool），自然形成递归。

```typescript
// AgentTool.call() 内部：
const agentIterator = runAgent({...})[Symbol.asyncIterator]()

// runAgent() 内部调用 query()
for await (const msg of query({
  tools: workerTools,  // 工具池中包含 AgentTool
  // ...
})) {
  // Agent 可以再次调用 AgentTool → 递归
}
```

### 2.2 递归深度控制

虽然支持无限递归，但有多重机制防止失控：

| 机制 | 控制方式 |
|------|----------|
| **Token 预算** | `taskBudget: { total: N }` 限制每个 Agent 的 token 消耗 |
| **USD 预算** | `maxBudgetUsd` 限制总花费 |
| **轮次限制** | `maxTurns` 限制每个 Agent 的对话轮次 |
| **工具池过滤** | 子 Agent 的工具池可以不包含 AgentTool（阻止递归） |
| **权限模式** | `dontAsk` 模式下 Agent 无法获得权限，自然终止 |

### 2.3 递归 vs 扁平的权衡

**递归的优势**：
- 自然分解复杂任务（"研究→实现→测试"可以递归嵌套）
- 每层有独立的上下文窗口（不会因任务太大而溢出）
- 失败隔离（孙 Agent 失败不影响其他分支）

**递归的风险**：
- Token 消耗可能指数级增长
- 信息在层级间传递会丢失细节
- 调试困难（需要追踪多层嵌套的转录）

**Claude Code 的缓解策略**：

```typescript
// 一次性 Agent（Explore, Plan）不返回 agentId/SendMessage 信息
// 节省 ~135 字符 × 3400 万次/周
export const ONE_SHOT_BUILTIN_AGENT_TYPES: ReadonlySet<string> = new Set([
  'Explore',
  'Plan',
])
```

对于常见的轻量级嵌套（主 Agent → Explore Agent 搜索代码），Claude Code 优化了协议开销。

---

## 3. 四种执行模式深度对比

### 3.1 模式概览

```
                     ┌─────────────────────────────┐
                     │          AgentTool           │
                     └──────────┬──────────────────┘
                                │
        ┌───────────┬───────────┼───────────┬──────────────┐
        │           │           │           │              │
   ┌────▼────┐ ┌────▼────┐ ┌───▼────┐ ┌────▼─────┐  ┌────▼────┐
   │ 同步    │ │ 异步    │ │ 远程   │ │进程内队友│  │  Fork   │
   │Sync     │ │Async    │ │Remote  │ │Teammate  │  │SubAgent │
   └─────────┘ └─────────┘ └────────┘ └──────────┘  └─────────┘
```

### 3.2 同步执行（Sync）

```typescript
// 父 Agent 阻塞等待子 Agent 完成
const agentIterator = runAgent({...})[Symbol.asyncIterator]()
while (true) {
  const result = await agentIterator.next()
  if (result.done) break
  // 处理消息...
}
return { status: 'completed', content: [...] }
```

| 特性 | 值 |
|------|-----|
| 父 Agent 行为 | 阻塞等待 |
| 结果获取 | 内联返回 |
| 适用场景 | 简单的信息收集、代码搜索 |
| 优势 | 简单可靠，结果直接可用 |
| 劣势 | 无法并行，父 Agent 空闲等待 |

### 3.3 异步执行（Async）

```typescript
// 子 Agent 转入后台，父 Agent 立即继续
const backgroundPromise = new Promise(resolve => {
  onBackgroundSignal = resolve
})

const raceResult = await Promise.race([
  agentIterator.next().then(r => ({ type: 'message', result: r })),
  backgroundPromise.then(() => ({ type: 'background' }))
])

if (raceResult.type === 'background') {
  // 后台运行生命周期
  runAsyncAgentLifecycle(agentIterator, ...)
  return { status: 'async_launched', agentId, outputFile }
}
```

| 特性 | 值 |
|------|-----|
| 父 Agent 行为 | 立即继续（fire-and-forget） |
| 结果获取 | 任务通知（task-notification） |
| 适用场景 | 独立的长时间任务 |
| 优势 | 并行执行，不阻塞父 Agent |
| 劣势 | 结果需要异步轮询/通知 |

**通知机制**：

```typescript
enqueueAgentNotification({
  taskId: backgroundedTaskId,
  description,
  status: 'completed',
  finalMessage,
  usage: { totalTokens, toolUses, durationMs },
  worktreePath,
})
// → 作为 user-role 消息注入父 Agent 的下一轮对话
```

### 3.4 远程执行（Remote）

```typescript
// 在 Cloud CCR (Claude Code Runner) 环境中执行
return {
  status: 'remote_launched',
  taskId,
  sessionUrl,  // 可在 Web UI 中查看
}
```

| 特性 | 值 |
|------|-----|
| 执行环境 | 云端 CCR 容器 |
| 隔离级别 | 完全隔离（独立 VM） |
| 适用场景 | ultraplan, ultrareview, autofix-pr |
| 优势 | 资源无限，不影响本地 |
| 劣势 | 网络延迟，需要认证 |

### 3.5 进程内队友（In-Process Teammate）

```typescript
// 同一 Node.js 进程中运行，通过 AsyncLocalStorage 隔离
const teammateIdentity: TeammateIdentity = {
  agentId: 'researcher@my-team',
  agentName: 'researcher',
  teamName: 'my-team',
  planModeRequired: false,
  parentSessionId: 'parent-session-uuid',
  color: 'blue',
}
```

| 特性 | 值 |
|------|-----|
| 执行环境 | 同一 Node.js 进程 |
| 隔离方式 | AsyncLocalStorage 上下文隔离 |
| 通信方式 | 文件邮箱 + 消息队列 |
| 适用场景 | Agent Swarms（团队协作） |
| 优势 | 零网络开销，共享 MCP 连接 |
| 劣势 | 内存共享同一进程，RSS 增长 |

**内存管理的精妙设计**：

```typescript
// BigQuery 分析显示：~20MB RSS/agent，292 agents → 36.8GB 峰值
// 解决方案：UI 消息上限
const TEAMMATE_MESSAGES_UI_CAP = 50

// appendCappedMessage() 保持不可变性（始终返回新数组）
function appendCappedMessage(messages, newMsg) {
  const updated = [...messages, newMsg]
  return updated.length > TEAMMATE_MESSAGES_UI_CAP
    ? updated.slice(-TEAMMATE_MESSAGES_UI_CAP)
    : updated
}
```

### 3.6 Fork 子 Agent

Fork 是一种特殊的同步执行模式，子 Agent **继承父 Agent 的完整对话上下文**：

```typescript
// 子 Agent 接收：
[
  ...parentHistory,                      // 完整的父对话历史
  assistant(all_tool_uses, thinking),     // 父的最后一条消息
  user(placeholder_result_1, ..., directive)  // 占位结果 + 子任务指令
]
```

**精妙之处**：所有 Fork 子 Agent 的消息前缀完全相同（字节级别），只有最后的 `directive` 不同。这确保了 **prompt cache 命中率最大化**——N 个 Fork 子 Agent 只需要 1 次 cache miss。

---

## 4. Coordinator 多智能体编排

### 4.1 Coordinator 模式的本质

Coordinator 不是一个新的类或框架——它是通过 **系统提示注入** 将普通 Claude 转变为多 Agent 编排者：

```typescript
// coordinatorMode.ts
export function getCoordinatorSystemPrompt(): string {
  return `
    你是 Claude Code，一个编排多个 Worker 的 AI 助手。
    
    你的职责：
    1. 将任务分解为可并行的子任务
    2. 派遣 Worker 执行子任务
    3. 综合 Worker 的结果
    4. 直接回答简单问题
    
    关键规则：
    - Worker 结果是内部信号，不是对话伙伴
    - 永远不要感谢或确认 Worker 结果
    - 永远不要预测 Worker 结果
  `
}
```

**这是一个极其重要的设计决策**：Coordinator 不需要特殊的代码路径——它使用和普通 Agent 完全相同的 `QueryEngine + query()` 循环，区别仅在于系统提示。

### 4.2 四阶段工作流

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Research    │     │  Synthesis   │     │Implementation│     │ Verification │
│  研究阶段    │────→│  综合阶段    │────→│  实现阶段    │────→│  验证阶段    │
│              │     │              │     │              │     │              │
│ Workers 并行 │     │ Coordinator  │     │ Workers 执行 │     │ Workers 验证 │
│ 调查、搜索   │     │ 理解、规划   │     │ 编码、提交   │     │ 测试、检查   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

### 4.3 Worker 结果通知格式

```xml
<task-notification>
  <task-id>agent-uuid-1234</task-id>
  <status>completed|failed|killed</status>
  <summary>Research auth bug in validate.ts</summary>
  <result>Found null pointer at line 42...</result>
  <usage>
    <total_tokens>12500</total_tokens>
    <tool_uses>8</tool_uses>
    <duration_ms>5000</duration_ms>
  </usage>
</task-notification>
```

**为什么用 XML 而非 JSON**？
- XML 的标签边界更清晰（不会被 JSON 的引号/括号混淆）
- Claude 模型对 XML 标签的解析更可靠
- 嵌套内容（如代码片段）在 XML CDATA 中更安全

### 4.4 Continue vs. Spawn 决策矩阵

这是 Coordinator 系统中最精妙的设计之一——一个**显式的决策矩阵**，教导 AI 何时复用现有 Worker、何时创建新 Worker：

| 场景 | 决策 | 理由 |
|------|------|------|
| 研究者找到了确切实现文件 | **Continue** | 保留文件上下文，添加实现规范 |
| 广泛研究 → 精确实现 | **Spawn fresh** | 避免探索噪声污染实现 |
| 修正失败或扩展工作 | **Continue** | Worker 有错误上下文 |
| 验证其他 Worker 的代码 | **Spawn fresh** | 新视角，避免实现偏见 |
| 完全错误的方向 | **Spawn fresh** | 重置心智，避免锚定效应 |
| 不相关的任务 | **Spawn fresh** | 无有用的上下文重叠 |

**设计洞察**：这不是代码层面的决策——是**提示工程层面的认知引导**。通过在系统提示中提供决策矩阵，Coordinator 学会了基于上下文重叠度做出合理判断。

### 4.5 综合阶段的反偷懒设计

系统提示中有一个极具创意的**反偷懒守卫**：

```
❌ 错误做法: "Based on your findings, fix the auth bug"
  → 将理解责任委托给 Worker

✅ 正确做法: "Fix the null pointer in src/auth/validate.ts:42. 
   The user field is undefined when sessions expire. 
   Add null check before user.id access — if null, return 401 
   with 'Session expired'. Commit and report hash."
  → Coordinator 展示了对问题的理解
```

**为什么这很重要**？

如果 Coordinator 只是转发 Worker 的研究结果，信息会在层级传递中**逐渐退化**（类似于传话游戏）。强制 Coordinator 综合信息并写出具体规范，确保了信息的**精确传递**。

### 4.6 Scratchpad 跨 Worker 知识共享

```typescript
if (scratchpadDir && isScratchpadGateEnabled()) {
  content += `\nScratchpad directory: ${scratchpadDir}\n
    Workers can read and write here without permission prompts.
    Use this for durable cross-worker knowledge...`
}
```

**问题**：Workers 之间无法看到彼此的对话上下文。研究 Worker 发现的代码库结构、决策树等知识如何传递给实现 Worker？

**解决方案**：Scratchpad 目录是一个**共享文件系统**，Workers 可以在其中持久化知识。它绕过了权限系统（不需要提示），使得跨 Worker 知识传递变得轻量。

**精妙之处**：这是对"共享内存"概念的文件系统实现——没有引入任何新的进程间通信机制，只是利用了已有的文件读写工具。

### 4.7 与其他多 Agent 框架的对比

| 维度 | Claude Code Coordinator | Autogen | Semantic Kernel |
|------|----------------------|---------|-----------------|
| **编排机制** | 系统提示注入 | Python 代码定义 | Planner + Kernel |
| **通信协议** | XML task-notification | 函数调用/消息 | 函数调用 |
| **决策方式** | AI 自主决策 + 决策矩阵 | 代码逻辑 | Planner AI |
| **综合阶段** | 强制综合（反偷懒守卫） | 无特殊要求 | 无特殊要求 |
| **知识共享** | Scratchpad 文件系统 | 共享 Memory 对象 | Shared State |
| **扩展性** | 配置即策略 | 代码级扩展 | 插件系统 |

**Claude Code 的独特之处**：它把多 Agent 编排完全交给了 **AI 自身**，而非用代码预定义工作流。这带来了极高的灵活性——Coordinator 可以根据任务性质动态决定并行策略、Worker 数量和阶段划分。

---

## 5. 五路通信机制设计

### 5.1 通信路径全景

```
                    ┌────────────────────────────┐
                    │      SendMessageTool       │
                    │    统一消息发送入口         │
                    └─────────┬──────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
     ┌────────▼──────┐ ┌─────▼──────┐ ┌──────▼──────┐
     │ 地址解析器    │ │ 广播路由   │ │ 结构化消息  │
     │ parseAddress()│ │ to = "*"   │ │ 验证器     │
     └────────┬──────┘ └─────┬──────┘ └──────┬──────┘
              │               │               │
   ┌──────┬──▼──┬──────┐     │               │
   │      │     │      │     │               │
┌──▼──┐┌──▼─┐┌─▼──┐┌──▼──┐ ├───────────────┘
│队友 ││UDS ││Bridge││进程内│ │
│邮箱 ││Socket││远程 ││子Agent│ │
└─────┘└────┘└────┘└─────┘ │
                              │
                    ┌─────────▼──────┐
                    │  文件邮箱广播   │
                    └────────────────┘
```

### 5.2 路径一：文件邮箱（In-Team Messaging）

**适用场景**：同一团队内的 Agent 通信

```
~/.claude/teams/{team_name}/inboxes/
  ├─ agent_1.json          ← 消息数组
  ├─ agent_1.json.lock     ← 文件锁
  ├─ agent_2.json
  └─ agent_2.json.lock
```

**并发安全设计**：

```typescript
// 使用 proper-lockfile 库
const LOCK_OPTIONS = {
  retries: { retries: 10, minTimeout: 5, maxTimeout: 100 }
}

async function writeToMailbox(recipient, message, team) {
  // 1. 确保目录存在
  await ensureInboxDir(team)

  // 2. 创建收件箱文件（如果不存在）
  try {
    await writeFile(inboxPath, '[]', { flag: 'wx' })  // wx = write exclusive
  } catch (e) {
    if (e.code !== 'EEXIST') throw e  // 已存在则忽略
  }

  // 3. 加锁
  let release = await lockfile.lock(inboxPath, { lockfilePath, ...LOCK_OPTIONS })
  try {
    // 4. 重新读取（获取其他写者的最新状态）
    const messages = await readMailbox(recipient, team)
    // 5. 追加
    messages.push({ ...message, read: false })
    // 6. 原子写入
    await writeFile(inboxPath, JSON.stringify(messages), 'utf-8')
  } finally {
    // 7. 释放锁
    await release()
  }
}
```

**精妙之处**：使用**文件系统**而非内存队列作为消息通道。这让邮箱自动持久化——即使进程崩溃，消息也不会丢失。文件锁保证了多 Agent 并发写入的安全性。

### 5.3 路径二：UDS Socket（本地跨会话）

```typescript
// 地址格式：uds:/path/to/socket 或 /path/to/socket（兼容旧格式）
if (addr.scheme === 'uds') {
  const { sendToUdsSocket } = require('../../utils/udsClient.js')
  await sendToUdsSocket(addr.target, input.message)
}
```

**适用场景**：同一机器上不同 Claude Code 会话之间的通信。比文件邮箱更快（无磁盘 I/O），但仅限本地。

### 5.4 路径三：Bridge 远程通信

```typescript
// 地址格式：bridge:session_<session_id>
if (addr.scheme === 'bridge') {
  const { postInterClaudeMessage } = require('../../bridge/peerSessions.js')
  const result = await postInterClaudeMessage(addr.target, input.message)
}
```

**安全限制**：
- 结构化消息（shutdown_request 等）**不能**通过 Bridge 发送
- 只能发送纯文本消息
- 需要 Bridge 连接处于活跃状态

### 5.5 路径四：进程内消息队列

```typescript
// 查找同进程中的子 Agent
const registered = appState.agentNameRegistry.get(input.to)
const agentId = registered ?? toAgentId(input.to)

if (task.status === 'running') {
  // 运行中：直接入队
  queuePendingMessage(agentId, input.message, setAppState)
} else {
  // 已停止：自动恢复并发送
  await resumeAgentBackground({ agentId, prompt: input.message, ... })
}
```

**精妙之处**：`resumeAgentBackground` 实现了**透明恢复**——如果目标 Agent 已经停止（甚至已被从内存中逐出），发送消息会自动从磁盘转录中恢复 Agent 上下文并重启。调用方无需关心目标 Agent 的状态。

### 5.6 路径五：广播

```typescript
async function handleBroadcast(content, summary, context) {
  const teamFile = await readTeamFileAsync(teamName)
  const recipients = teamFile.members
    .filter(m => m.name !== senderName)
    .map(m => m.name)

  for (const recipientName of recipients) {
    await writeToMailbox(recipientName, message, teamName)
  }

  return { success: true, recipients, ... }
}
```

### 5.7 消息路由的统一设计

所有五种通信路径共享同一个入口——`SendMessageTool`。地址解析是自动的：

```
输入: to = "research-agent"
  → parseAddress()
    ├─ "uds:/path" → UDS Socket
    ├─ "bridge:session_xxx" → Bridge 远程
    ├─ "/path/to.sock" → UDS（兼容）
    └─ 其他 → 进程内查找 → 文件邮箱
```

**设计洞察**：调用方不需要知道目标 Agent 的物理位置——是在本进程、本机还是远程。统一的 `SendMessage(to, message)` 接口隐藏了所有路由复杂度。这类似于 gRPC 的服务发现——**位置透明性**。

### 5.8 结构化消息协议

除了纯文本，SendMessageTool 支持结构化消息类型：

```typescript
type StructuredMessage =
  | { type: 'shutdown_request', reason?: string }
  | { type: 'shutdown_response', request_id: string, approve: boolean, reason?: string }
  | { type: 'plan_approval_response', request_id: string, approve: boolean, feedback?: string }
```

这些消息实现了 Agent 间的**协议级通信**——不只是传递信息，还可以请求操作（关机）和审批流程（计划确认）。

---

## 6. Agent 记忆与上下文

### 6.1 三层记忆架构

```
┌────────────────────────────────────────────────────┐
│                    长期记忆                         │
│                                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐│
│  │ User 级别   │  │ Project 级别 │  │ Local 级别 ││
│  │ ~/.claude/   │  │ ./.claude/   │  │ ./.claude/ ││
│  │ agent-memory/│  │ agent-memory/│  │ agent-     ││
│  │              │  │              │  │ memory-    ││
│  │ 跨项目通用  │  │ 项目特定     │  │ local/     ││
│  │ 学习        │  │ 知识         │  │ 本机特定   ││
│  └─────────────┘  └──────────────┘  └────────────┘│
│                                                    │
├────────────────────────────────────────────────────┤
│                    中期记忆                         │
│                                                    │
│  ┌───────────────────────────────────────────────┐ │
│  │ CLAUDE.md (项目记忆)                          │ │
│  │ - 编码规范、架构决策、工作流程                 │ │
│  │ - 每次会话自动加载                             │ │
│  │ - 可嵌套（子目录的 CLAUDE.md）                 │ │
│  └───────────────────────────────────────────────┘ │
│                                                    │
├────────────────────────────────────────────────────┤
│                    短期记忆                         │
│                                                    │
│  ┌───────────────────────────────────────────────┐ │
│  │ 对话上下文 (mutableMessages[])                │ │
│  │ - 当前会话的消息历史                          │ │
│  │ - 可通过 compact 压缩                         │ │
│  │ - 会话结束后保存为转录                        │ │
│  └───────────────────────────────────────────────┘ │
│                                                    │
│  ┌───────────────────────────────────────────────┐ │
│  │ 文件状态缓存 (FileStateCache)                 │ │
│  │ - Agent 已读文件的内容缓存                    │ │
│  │ - Copy-on-Write 隔离                          │ │
│  └───────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

### 6.2 Agent 记忆的持久化

```typescript
export type AgentMemoryScope = 'user' | 'project' | 'local'

function getAgentMemoryDir(agentType: string, scope: AgentMemoryScope): string {
  const dirName = sanitizeAgentTypeForPath(agentType)
  switch (scope) {
    case 'project': return join(getCwd(), '.claude', 'agent-memory', dirName)
    case 'local':   return getLocalAgentMemoryDir(dirName)
    case 'user':    return join(getMemoryBaseDir(), 'agent-memory', dirName)
  }
}
```

**精妙之处**：Agent 记忆使用**与代码仓库相同的文件系统**进行存储。`project` 级别的记忆可以被 Git 跟踪——团队成员可以共享 Agent 学到的项目知识。

### 6.3 记忆快照同步

```typescript
// 检查快照是否比本地记忆更新
const action = await checkAgentMemorySnapshot(agentType, scope)
// 返回: 'initialize' | 'prompt-update' | 'none'

if (action === 'initialize') {
  await initializeFromSnapshot(agentType, scope, timestamp)
} else if (action === 'prompt-update') {
  await replaceFromSnapshot(agentType, scope, timestamp)
}
```

**用途**：团队工作流中，一个成员训练 Agent 后，记忆快照可以通过版本控制同步到其他成员。

### 6.4 DreamTask：自动记忆整合

```typescript
type DreamTaskState = {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]
  turns: DreamTurn[]        // 最多 30 轮
  priorMtime: number        // 用于回滚锁
}
```

DreamTask 在会话结束后异步运行，自动整合会话中的学习成果到长期记忆。这类似于人类睡眠时的记忆整合过程（因此命名为 "Dream"）。

**生命周期**：

```
会话结束
  → registerDreamTask() 创建任务
    → Agent 审查会话转录
      → 提取有价值的学习
        → 写入 Agent 记忆文件
          → completeDreamTask()

如果被中断：
  → kill() → rollbackConsolidationLock(priorMtime)
  → 下次会话可以重试
```

---

## 7. 任务框架设计

### 7.1 七种任务类型

```typescript
type TaskState =
  | LocalShellTaskState        // Bash 命令执行
  | LocalAgentTaskState        // 本地 Agent（前台/后台）
  | RemoteAgentTaskState       // 远程 Agent（CCR）
  | InProcessTeammateTaskState // 进程内队友（Swarm）
  | DreamTaskState             // 梦境记忆整合
  | LocalWorkflowTaskState     // 本地工作流
  // + 隐式的 MonitorTask
```

### 7.2 任务状态机

```
pending → running → completed
                  → failed
                  → killed
```

**进度追踪**：

```typescript
type ProgressTracker = {
  toolUseCount: number
  latestInputTokens: number         // 最新值（API 是累计的）
  cumulativeOutputTokens: number    // 跨轮次累加
  recentActivities: ToolActivity[]  // 最近 5 个活动
}
```

**精妙之处**：`latestInputTokens` 保存最新值而非累加——因为 Claude API 的 input_tokens 已经是累计值（包含前序消息），重复累加会导致数字膨胀。

### 7.3 投机执行（Speculation）

```typescript
// 安全约束
WRITE_TOOLS = ['Edit', 'Write', 'NotebookEdit']
SAFE_READ_ONLY_TOOLS = ['Read', 'Glob', 'Grep', 'ToolSearch', 'LSP', 'TaskGet', 'TaskList']
```

**投机执行**是 Claude Code 中最创新的设计之一：

```
用户阅读当前响应的同时：
  → Fork Agent 开始推测下一轮的执行
    → 在 Overlay 文件系统中执行写操作
    → 如果用户输入与推测匹配 → 接受（合并 Overlay）
    → 如果不匹配 → 丢弃 Overlay

边界检测（终止推测的条件）：
  • Bash 命令 → 停止（可能有副作用）
  • Edit/Write 工具 → 停止（需要确认）
  • 权限被拒绝 → 停止
  • 自然完成 → 停止
```

**精妙之处**：
- **Overlay 文件系统**：推测写入不直接修改真实文件，而是写入临时目录（`$TEMP/speculation/{pid}/{id}`）
- **接受时合并**：`copyOverlayToMain()` 将推测结果复制回主目录
- **消息过滤**：注入推测结果前，清理 thinking blocks、未完成的 tool_use 等

### 7.4 远程 Agent 的轮询架构

```typescript
// RemoteAgentTask 的轮询循环
pollRemoteSessionEvents()
  → 从 /v1/sessions/{id}/events 获取事件
    → 提取 <ultraplan> 或 <remote-review> 标签
    → 运行完成检查器（每种任务类型有独立的检查逻辑）
    → 30 分钟超时（从 pollStartedAt 计算，非 spawn 时间）
    → 持久化元数据（fire-and-forget）
```

**恢复机制**：

```typescript
// --resume 恢复远程任务
扫描 .claude/sessions/remote-agents/ 边车文件
  → 获取 CCR 会话状态
    → 404: 删除任务和边车文件
    → 401: 可恢复（CCR 仍在运行）
    → 200: 重启轮询
```

---

## 8. Bridge 远程控制架构

### 8.1 Bridge 的分布式系统设计

Bridge 是 Claude Code 的远程控制层，展现了经典的分布式系统设计模式：

```
                    ┌──────────────┐
                    │  Web UI/API  │
                    │  (客户端)    │
                    └──────┬───────┘
                           │ WebSocket / HTTP
                    ┌──────▼───────┐
                    │  Bridge 服务  │
                    │  (中间层)    │
                    └──────┬───────┘
                           │ stdio / UDS
                    ┌──────▼───────┐
                    │ Claude Code  │
                    │  (本地进程)  │
                    └──────────────┘
```

### 8.2 JWT 双令牌架构

**问题**：需要同时支持 WebSocket 认证和 API 调用认证，但两者的生命周期和刷新策略不同。

**解决方案**：双令牌系统

| 令牌 | 用途 | 前缀 | 生命周期 | 刷新策略 |
|------|------|------|----------|----------|
| Session Ingress JWT | WebSocket 认证 | `sk-ant-si-` | 可变 | 提前 5 分钟刷新 |
| OAuth Token | API 调用 | - | 3h55m | 提前刷新 |

**精妙设计：代际淘汰**

```typescript
const generations = new Map<string, number>()

async function doRefresh(sessionId, gen) {
  if (generations.get(sessionId) !== gen) return  // 过期回调，静默丢弃
  // ... 执行刷新 ...
}
```

**问题**：JWT 刷新是异步的——刷新 A 的回调可能在会话 A 已经被取消后才触发。

**解决方案**：每个会话有一个代际计数器。调度刷新时记录当前代际，回调触发时检查代际是否匹配。不匹配则静默丢弃——比 AbortController 更轻量（无需创建 Signal 对象）。

### 8.3 容量感知轮询

```
at_capacity + heartbeat:
  → 进入心跳循环（不轮询新任务）
    → 心跳间隔: non_exclusive_heartbeat_interval_ms (GrowthBook)
    → 退出条件:
        • 容量释放 (capacityWake.signal())
        • 认证失败 (401/403)
        • 截止时间到达 (pollDeadline)
        • 中止信号
  → 恢复轮询
```

**精妙设计：容量唤醒信号**

```typescript
// 当一个会话完成时
capacityWake.signal()  // 创建 AbortSignal

// at-capacity 的睡眠可以提前唤醒
await sleep(interval, capacityWakeSignal)  // 而非等待完整间隔
```

**效果**：满载时不做无用轮询（节省 API 调用），但一旦有空闲立即响应（低延迟）。

### 8.4 回声消除

```typescript
// BoundedUUIDSet: 固定容量的循环缓冲区
const recentOutbound = new BoundedUUIDSet(capacity)

// 发送消息时记录 UUID
recentOutbound.add(msg.uuid)

// 接收消息时检查是否是回声
if (recentOutbound.has(msg.uuid)) {
  // 跳过——这是自己发出的消息的回声
  continue
}
```

**问题**：WebSocket 服务端可能回放历史消息（重连时），或者回显客户端自己发送的消息。

**解决方案**：固定容量的循环缓冲区存储最近发送的消息 UUID。O(capacity) 内存，FIFO 淘汰。

### 8.5 控制请求/响应协议

```
服务端 → 客户端:
  control_request:
    ├─ initialize: 会话初始化
    ├─ set_model: 切换模型
    ├─ set_max_thinking_tokens: 调整思考预算
    ├─ set_permission_mode: 切换权限模式
    └─ interrupt: 取消当前轮次

  超时: ~10-14s（无响应则断开 WebSocket）

客户端 → 服务端:
  control_response:
    ├─ request_id: 关联请求
    └─ success | error
```

### 8.6 会话归档设计

```typescript
// Bridge 关闭前
makeResultMessage() → 构建 SDKResultSuccess
→ 发送给服务端
→ 触发服务端会话归档
→ 关闭 WebSocket
```

**为什么需要**：防止 Web UI 中出现"悬空"的活跃会话（实际已关闭但服务端不知道）。

---

## 9. 安全与权限的精细控制

### 9.1 Agent 权限继承模型

```
父 Agent (mode: bypassPermissions)
  ├─ 子 Agent A (mode: acceptEdits)     ← 更严格 ✅
  │   └─ 孙 Agent A1 (mode: default)    ← 更严格 ✅
  ├─ 子 Agent B (mode: bypassPermissions) ← 不能超过父 ❌
  │   → 实际: mode: acceptEdits           ← 降级为父级别
  └─ 子 Agent C (mode: plan)             ← 更严格 ✅
```

**规则**：子 Agent 的权限模式可以比父 Agent **更严格**（从 bypass → acceptEdits → default → plan → dontAsk），但不能**更宽松**。

### 9.2 工具池过滤

```typescript
// 不同 Agent 类型的工具限制
ALL_AGENT_DISALLOWED_TOOLS     // 所有 Agent 禁用（如 AgentTool 递归守卫）
CUSTOM_AGENT_DISALLOWED_TOOLS  // 自定义 Agent 禁用（如 spawnTeammate）
ASYNC_AGENT_ALLOWED_TOOLS      // 异步 Agent 白名单
INTERNAL_WORKER_TOOLS          // 仅 Coordinator 可用
```

**过滤逻辑**：

```typescript
// Coordinator Workers 的工具过滤
const workerTools = isSimpleMode
  ? ['Bash', 'FileRead', 'FileEdit']    // 简化模式
  : Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
      .filter(name => !INTERNAL_WORKER_TOOLS.has(name))  // 排除内部工具
```

### 9.3 异步 Agent 的权限困境

**问题**：异步 Agent 没有 UI，无法显示权限提示对话框。

**解决方案**：

```typescript
const shouldAvoidPrompts = !canShowPermissionPrompts || isAsync

// 异步 Agent 的策略：
// 1. shouldAvoidPermissionPrompts = true → 跳过所有需要 UI 的权限
// 2. 使用 acceptEdits 或 bypassPermissions 模式
// 3. 如果遇到需要确认的操作 → 自动拒绝
```

### 9.4 MCP 工具的安全边界

```typescript
// MCP 工具继承完整的 4 层权限系统
const mcpTool: Tool = {
  checkPermissions: async (input, context) => {
    // MCP 工具没有特殊豁免
    // 经过与内置工具相同的权限检查流程
    return { behavior: 'passthrough' }  // → Layer 3 转为 'ask'
  },
}
```

**设计决策**：MCP 工具来自外部进程，安全风险更高。Claude Code 没有给 MCP 工具任何权限快车道——它们经过与内置工具**完全相同**的 4 层权限检查。

### 9.5 Worktree 隔离的安全价值

```typescript
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${agentId.slice(0, 8)}`
  worktreeInfo = await createAgentWorktree(slug)
  // Agent 在独立的 Git Worktree 中操作
}

// 清理逻辑：
if (hasWorktreeChanges(worktreePath, headCommit)) {
  // 有变更 → 保留 Worktree（等待人工审查）
  return { worktreePath, worktreeBranch }
} else {
  // 无变更 → 自动清理
  await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot)
  return {}
}
```

**安全价值**：
1. **写隔离**：Agent 的文件修改在独立分支上，不影响主分支
2. **可审查**：有变更的 Worktree 被保留，等待人工审查
3. **自动清理**：无变更的 Worktree 自动删除，避免磁盘泄漏
4. **冲突免疫**：多个并行 Agent 不会产生文件冲突

---

## 10. 设计模式总结与优劣评析

### 10.1 七大核心设计模式

#### 模式一：AsyncGenerator 管道

```
query() → submitMessage() → SDK 消费者
  ↑                              |
  └─── 嵌套 query()（Agent）───┘
```

| 优势 | 劣势 |
|------|------|
| 流式、背压、取消原生支持 | 调试时堆栈追踪可能不完整 |
| 嵌套组合（yield*） | 错误传播路径长 |
| 零依赖（语言内置） | 测试需要完整迭代 |

#### 模式二：隔离执行

```
每个 Agent:
  ├─ 独立 query() 作用域
  ├─ 独立文件缓存 (COW)
  ├─ 独立权限追踪
  └─ 独立消息数组
```

| 优势 | 劣势 |
|------|------|
| 安全的并行执行 | 跨 Agent 信息共享成本高 |
| 失败隔离 | 每 Agent 消耗独立上下文窗口 |
| 可独立取消/恢复 | 内存占用线性增长 |

#### 模式三：系统提示即策略

```
普通 Agent: 标准系统提示
Coordinator: + 多 Agent 编排提示
Worker: + Worker 行为指南
KAIROS: + 助手模式提示
```

| 优势 | 劣势 |
|------|------|
| 零代码路径差异 | 提示长度影响 token 预算 |
| 极高灵活性 | AI 可能不完全遵循提示 |
| 易于迭代调优 | 难以做单元测试 |

#### 模式四：配置即模式

```typescript
QueryEngineConfig = {
  canUseTool,    // 不同模式不同实现
  getAppState,   // 不同模式不同来源
  tools,         // 不同模式不同工具集
  maxTurns,      // 不同模式不同限制
}
```

| 优势 | 劣势 |
|------|------|
| 避免继承层次爆炸 | 配置项数量可能失控 (20+) |
| 可测试（mock 配置） | 类型安全依赖开发者纪律 |
| SDK 可注入自定义实现 | 新模式需要添加新配置项 |

#### 模式五：文件系统即基础设施

```
消息通信: 文件邮箱 (~/.claude/teams/)
记忆存储: Agent 记忆目录 (.claude/agent-memory/)
会话恢复: 转录文件 (.claude/sessions/)
知识共享: Scratchpad 目录
```

| 优势 | 劣势 |
|------|------|
| 自然持久化（无需额外存储） | I/O 延迟 |
| 可 Git 跟踪（项目级记忆） | 并发控制复杂（文件锁） |
| 零依赖（无需数据库） | 跨机器共享困难 |

#### 模式六：反偷懒认知守卫

```
❌ "Based on your findings, fix the bug"
✅ "Fix null pointer at line 42: add check before user.id"
```

| 优势 | 劣势 |
|------|------|
| 防止信息在层级间退化 | 增加系统提示长度 |
| 强制 Coordinator 理解后综合 | 依赖 AI 遵循指令 |
| 改善最终输出质量 | 可能增加 Coordinator 延迟 |

#### 模式七：Feature Gate 编译消除

```typescript
if (feature('COORDINATOR_MODE')) { ... }
// Bun 打包时求值为 true/false
// Terser 消除不可达分支
```

| 优势 | 劣势 |
|------|------|
| 死代码完全移除 | 需要重新构建才能切换 |
| 字符串不泄漏 | 无法运行时 A/B 测试 |
| 包体积更小 | 每种配置需要独立 bundle |

### 10.2 整体架构优势

1. **统一抽象**：一个 `QueryEngine + query()` 支撑所有 Agent 类型和运行模式
2. **安全优先**：Fail-Closed 默认值、4 层权限、旁路免疫安全检查、Worktree 隔离
3. **经济效率**：Fork prompt cache、shouldDefer 工具延迟加载、投机执行
4. **韧性设计**：自动恢复（resumeAgentBackground）、水印式错误隔离、DreamTask 锁回滚
5. **位置透明**：SendMessage 统一接口隐藏物理位置差异

### 10.3 潜在改进空间

| 方面 | 现状 | 潜在改进 |
|------|------|----------|
| **Agent 间缓存共享** | 每 Agent 独立缓存 | 可考虑只读共享层 + 写隔离层 |
| **Coordinator 测试** | 提示驱动，难以单元测试 | 可考虑提取决策逻辑为可测试的函数 |
| **进程内队友内存** | 线性增长（~20MB/agent） | 可考虑 Agent 池化或按需加载 |
| **文件邮箱性能** | 文件锁 + JSON 序列化 | 高频场景可考虑 SQLite 或内存队列 |
| **投机执行覆盖率** | 仅覆盖只读工具 | 可考虑扩展到安全的写操作 |
| **远程任务恢复** | 依赖边车文件 | 可考虑集中式任务注册表 |
| **广播效率** | 逐一写邮箱 | 可考虑发布/订阅模式 |

### 10.4 与业界 Agent 框架的综合对比

```
                    灵活性
                      ↑
                      │
         Claude Code  ●
                      │         ● LangGraph
                      │
         ● CrewAI     │
                      │
                      │    ● Autogen
         ● AutoGPT    │
                      │
         ─────────────┼──────────────→ 安全性
                      │
```

- **Claude Code** 在灵活性和安全性上都处于领先——灵活性来自系统提示即策略的设计，安全性来自 4 层权限 + Worktree 隔离
- **LangGraph** 在灵活性上接近（状态机+子图），但安全性方面缺少 Claude Code 级别的权限系统
- **CrewAI/Autogen** 在灵活性上受限于代码级定义的工作流，但也因此更可预测
- **AutoGPT** 在两个维度上都较弱——共享 workspace 缺少隔离，循环执行缺少灵活编排

---

> **结语**：Claude Code 的 Agent 系统展现了"**AI 原生**"的设计哲学——它不是简单地在现有框架上加一层 AI 接口，而是从 AI Agent 的核心需求（隔离、通信、记忆、安全）出发，设计了一套完整的运行时。AsyncGenerator 管道、5 路通信、三层记忆、投机执行这些设计都是对 AI Agent 特有问题的原创回答。最令人印象深刻的是 Coordinator 的"系统提示即策略"设计——它证明了在 AI 时代，最好的编排代码可能不是代码，而是精心设计的提示。

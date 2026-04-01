# 通用 Agent 设计模式与工程实践

> 从 Claude Code 源码中提炼的 10 大 Agent 设计模式，可直接应用于任何 AI Agent 系统的设计与开发。

---

## 一、总览：Agent 设计模式全景

Claude Code 的源码展现了一套**经过生产验证的 Agent 设计模式体系**。这些模式不仅适用于编码助手，更是构建任何复杂 AI Agent 系统的通用方法论。

```
10 大设计模式
├── 模式 1：AsyncGenerator 流式管道模式
├── 模式 2：QueryEngine 会话引擎模式
├── 模式 3：工具抽象与注册模式
├── 模式 4：层次化权限决策链模式
├── 模式 5：递归嵌套 Agent 模式
├── 模式 6：多模式执行引擎模式
├── 模式 7：上下文窗口智能管理模式
├── 模式 8：Agent 记忆与学习模式
├── 模式 9：推测性执行模式
├── 模式 10：配置即策略模式
```

---

## 二、模式详解

### 模式 1：AsyncGenerator 流式管道模式

**问题**：AI Agent 需要处理流式响应、支持取消、管理背压，传统的回调或 Promise 模式难以优雅处理。

**解决方案**：使用 AsyncGenerator 作为所有层间通信的统一协议。

**架构图**：
```
User Layer          Query Layer           API Layer
    │                   │                    │
    │  for await ──→    │  for await ──→     │
    │  (messages)       │  (responses)       │  API Stream
    │                   │                    │
    │  ◄── yield        │  ◄── yield         │
    │     message       │     response       │
    │                   │                    │
    │  abort ──────→    │  abort ──────→     │
    │  (cancel)         │  (cancel)          │  Cancel
```

**关键实现要点**：

```typescript
// 核心模式：三层嵌套的 AsyncGenerator
async function* submitMessage(prompt: string): AsyncGenerator<Message> {
  // 第一层：会话管理
  const context = prepareContext(prompt);
  
  // 第二层：查询循环 - yield* 优雅传播
  yield* queryLoop(context);
  
  // 第三层：清理
  await cleanup();
}

async function* queryLoop(context: QueryContext): AsyncGenerator<Message> {
  while (true) {
    // API 调用返回流式响应
    for await (const chunk of apiStream(context)) {
      // 处理不同消息类型
      if (chunk.type === 'tool_use') {
        // 工具调用结果也是 AsyncGenerator
        yield* executeToolAndYield(chunk);
      } else {
        yield chunk;  // 直接传播给上层
      }
    }
    
    // 判断是否需要继续循环
    if (shouldStop(context)) break;
  }
}
```

**优势**：
| 特性 | AsyncGenerator | Callback | Observable |
|------|---------------|----------|------------|
| 背压控制 | ✅ 原生 | ❌ 需手动 | ⚠️ 需 operator |
| 取消支持 | ✅ return() | ⚠️ 需约定 | ✅ unsubscribe |
| 类型安全 | ✅ 泛型 | ⚠️ 弱 | ✅ 泛型 |
| 嵌套组合 | ✅ yield* | ❌ 回调地狱 | ⚠️ switchMap |
| 调试友好 | ✅ 栈追踪 | ❌ 丢失栈 | ⚠️ 调度器 |
| 懒求值 | ✅ 原生 | ❌ 立即 | ✅ 原生 |

**适用场景**：
- AI 对话系统的消息流
- 实时数据管道
- 多步骤工作流编排
- 流式 API 网关

---

### 模式 2：QueryEngine 会话引擎模式

**问题**：AI Agent 需要管理会话状态（消息历史、文件缓存、权限）、支持多种执行模式，且每个 Agent 实例需要独立的状态空间。

**解决方案**：每个 Agent = 一个 QueryEngine 实例，通过配置注入而非继承来支持多种模式。

**架构设计**：

```
QueryEngine 实例（每 Agent 一个）
├── 不可变配置
│   ├── model: 模型选择
│   ├── tools: 工具集（经过权限过滤）
│   ├── systemPrompt: 系统 Prompt
│   ├── maxTurns: 最大轮次
│   └── canUseTool: 权限判断函数
├── 可变状态
│   ├── messages: 消息历史
│   ├── fileCache: 文件缓存（COW）
│   ├── totalUsage: Token 使用量
│   └── permissionDenials: 权限拒绝记录
└── 生命周期
    ├── submitMessage(): 七阶段处理
    ├── abort(): 取消当前查询
    └── dispose(): 释放资源
```

**七阶段处理生命周期**：
```
1. Construct    → 创建 AbortController、初始化文件缓存
2. Initialize   → 权限包装、系统 Prompt 组装
3. Shortcut     → 斜杠命令短路（跳过 API）
4. Snapshot     → Git 快照（记录文件状态）
5. Query Loop   → 核心消息泵（API 调用 + 工具执行循环）
6. Validate     → 结果校验（结构化输出、成功条件）
7. Cleanup      → 使用量统计、记忆提取、会话刷新
```

**关键设计决策**：
- **配置注入而非继承**：不需要 CLIQueryEngine、SDKQueryEngine 等子类
- **COW 文件缓存**：Agent Fork 时共享缓存，修改时拷贝
- **七阶段明确分离**：每个阶段职责清晰，便于扩展和调试

**适用场景**：
- 多 Agent 系统中每个 Agent 的会话管理
- 需要支持多种执行模式的 AI 应用
- 需要精细状态管理的对话系统

---

### 模式 3：工具抽象与注册模式

**问题**：Agent 需要调用多种工具（文件操作、API 调用、Shell 执行等），需要统一的抽象来管理工具的发现、校验、执行和权限。

**解决方案**：统一的 Tool 接口 + Zod Schema 校验 + 动态注册。

**工具接口设计**：

```typescript
// 统一工具接口
interface Tool<TInput, TOutput> {
  // 元数据
  name: string;
  description: string | (() => string);  // 支持动态描述
  inputSchema: ZodSchema<TInput>;        // Zod 运行时校验
  
  // 执行
  call(input: TInput, context: ToolContext): Promise<TOutput>;
  
  // 权限
  checkPermissions?(input: TInput): PermissionResult;
  
  // 并发安全
  isConcurrencySafe: boolean;  // 默认 false（fail-closed）
  
  // 可见性
  isEnabled?(context: AppState): boolean;
  userFacingName?(): string;
}
```

**工具注册与发现**：

```
工具集聚合
├── 内置工具（编译时注册，40+）
├── MCP 工具（运行时发现，动态 Schema 翻译）
├── Skill 工具（运行时加载，零代码定义）
├── Plugin 工具（进程内，深度集成）
└── Feature Gate 工具（编译时裁剪）

工具搜索优化（ToolSearch）
├── 延迟加载：非必要工具的描述延迟注入
├── 按需发现：AI 请求时才加载完整 Schema
└── 缓存策略：工具元数据缓存，避免重复计算
```

**流式工具执行器**：
```
API 响应流 → StreamingToolExecutor
  ├── 工具调用入队（流式收集参数）
  ├── 权限预检（与 API 流并行）
  ├── 并发安全工具 → 并行执行
  ├── 非并发安全工具 → 串行执行
  └── 结果收集 → ToolResultBlock
```

**关键设计决策**：
- **Fail-Closed 默认**：新工具默认非并发安全（串行），显式声明安全才并行
- **Zod 运行时校验**：防止 AI 生成的错误参数到达工具逻辑
- **动态描述**：工具描述可根据上下文变化（如当前文件类型）

**适用场景**：
- 任何多工具 Agent 系统
- 需要动态工具发现的插件化架构
- 需要严格参数校验的自动化系统

---

### 模式 4：层次化权限决策链模式

**问题**：AI Agent 操作涉及安全敏感操作（文件修改、命令执行、数据访问），需要灵活且安全的权限控制。

**解决方案**：七层权限决策管道，从粗到细、从快到慢。

**决策管道**：

```
工具调用请求
  │
  ├─ Layer 0: 工具池过滤（该工具是否可用？）
  │   └── 编译时 Feature Gate + 运行时模式过滤
  │
  ├─ Layer 1: 输入校验（参数是否合法？）
  │   └── Zod Schema 校验
  │
  ├─ Layer 2: 权限模式快速路径
  │   ├── bypass → 允许（最快路径）
  │   ├── dontAsk → 拒绝（最快路径）
  │   └── 其他 → 继续下层
  │
  ├─ Layer 3: 工具级权限检查
  │   └── tool.checkPermissions(input) → allow/deny/ask
  │
  ├─ Layer 4: 规则引擎匹配
  │   ├── policy（组织策略）→ 最高优先级
  │   ├── user（用户规则）
  │   ├── project（项目规则）
  │   ├── local（本地规则）
  │   ├── flag（标记规则）
  │   └── session（会话规则）→ 最低优先级
  │
  ├─ Layer 5: PreToolUse Hooks
  │   └── 外部脚本可 allow/deny/ask + 修改输入
  │
  ├─ Layer 6: AI 安全分类器
  │   └── Bash 命令安全评估（两阶段竞速）
  │
  └─ Layer 7: 用户交互确认
      └── 终端对话框 / 自动批准
```

**规则 DSL 设计**：

```typescript
interface PermissionRule {
  tool: string;           // 工具名称（支持通配符）
  action: 'allow' | 'deny' | 'ask';
  pattern?: string;       // 参数匹配模式
  scope: 'policy' | 'user' | 'project' | 'local' | 'flag' | 'session';
  priority: number;       // 同 scope 内的优先级
  metadata?: {
    reason: string;       // 决策原因（审计用）
    expiry?: Date;        // 过期时间
  };
}
```

**适用场景**：
- 企业级 AI Agent 安全治理
- 多租户系统的操作权限管理
- DevOps 自动化的安全控制

---

### 模式 5：递归嵌套 Agent 模式

**问题**：复杂任务需要分解为子任务，由专业化的子 Agent 处理，且子任务可能进一步分解。

**解决方案**：支持无限嵌套的 Agent 递归模型，通过预算和工具池控制安全边界。

**递归架构**：

```
Main Agent（用户交互）
  ├── Research Agent（信息收集）
  │   ├── CodeSearch Agent（代码搜索）
  │   └── DocSearch Agent（文档搜索）
  ├── Implementation Agent（实现）
  │   ├── Frontend Agent（前端实现）
  │   └── Backend Agent（后端实现）
  │       └── DB Agent（数据库变更）
  └── Review Agent（审查）
      ├── Security Agent（安全审查）
      └── Performance Agent（性能审查）
```

**安全边界控制**：

```
资源预算：
├── Token Budget：子 Agent 的 Token 预算 ≤ 父 Agent 剩余预算
├── USD Budget：全局美元预算上限
├── Max Turns：每个 Agent 的最大轮次数
└── Time Budget：超时自动终止

工具池控制：
├── 继承模式：子 Agent 继承父 Agent 的工具子集
├── 排除模式：可排除 AgentTool 防止无限递归
├── 限制模式：轻量级 Agent（如 Explore）只有只读工具
└── 定制模式：YAML 定义精确的工具白名单

权限继承：
├── 规则：子 Agent 权限 ≤ 父 Agent 权限
├── 只能更严格，不能更宽松
└── dontAsk 模式防止子 Agent 弹出交互提示
```

**Fork 优化**：
```
父 Agent 消息历史 = [M1, M2, M3, ..., Mn]
Fork 子 Agent 继承 = [M1, M2, M3, ..., Mn, Fork指令]

优势：字节级相同的前缀 → Prompt Cache 命中
      所有 Fork 子 Agent 共享缓存 → 单次 Cache Miss
```

**适用场景**：
- 复杂任务的自动分解与执行
- 多专家协作系统
- 递归搜索与分析任务

---

### 模式 6：多模式执行引擎模式

**问题**：同一个 Agent 引擎需要支持多种执行模式（交互式、批处理、远程、协作），但不希望代码路径爆炸。

**解决方案**：通过配置注入实现 8 种模式，底层共用同一个 QueryEngine。

**八种执行模式**：

| 模式 | 配置差异 | 适用场景 |
|------|---------|---------|
| CLI Interactive | 标准工具集 + 用户审批 | 日常交互 |
| Non-Interactive Print | 输出到 stdout + 自动审批 | 脚本集成 |
| SDK Programmatic | API 输出 + 编程控制 | 应用嵌入 |
| Bridge Remote | 远程协议 + 代理认证 | 云端控制 |
| Coordinator | 完整工具 + Agent 编排 | 多 Agent 协作 |
| Daemon | 后台运行 + Cron 触发 | 定时任务 |
| KAIROS Assistant | 语义理解 + 助手工具 | 个人助理 |
| Voice | 语音输入 + 语音输出 | 语音交互 |

**配置注入模式**：

```typescript
// 不需要继承层次 — 只需配置对象
const engineConfig = {
  // 模型选择
  model: selectModel(mode),
  
  // 工具集（根据模式过滤）
  tools: assembleTools(mode, permissions),
  
  // 权限判断函数
  canUseTool: createPermissionChecker(mode),
  
  // 状态管理
  getAppState: mode === 'sdk' ? sdkStateGetter : reactStateGetter,
  setAppState: mode === 'sdk' ? sdkStateSetter : reactStateSetter,
  
  // 行为参数
  maxTurns: mode === 'daemon' ? 10 : Infinity,
  bare: mode === 'print',
  
  // 输出处理
  onMessage: mode === 'bridge' ? bridgeRelay : terminalRender,
};

// 同一个 QueryEngine，不同的配置
const engine = new QueryEngine(engineConfig);
```

**适用场景**：
- 多渠道 AI 服务（API/WebSocket/CLI/SDK）
- 多产品线共享 Agent 引擎
- 需要灵活部署模式的 Agent 系统

---

### 模式 7：上下文窗口智能管理模式

**问题**：AI 模型上下文窗口有限，长对话和大量工具结果会耗尽窗口，但简单截断会丢失关键信息。

**解决方案**：五层递进压缩 + 缓存感知策略。

**五层压缩管道**：

```
原始上下文 [500K tokens]
  │
  ├─ Layer 1: 工具结果预算
  │  └── 单次工具输出超 50K → 截断至摘要
  │  结果: [350K tokens]
  │
  ├─ Layer 2: 历史裁剪
  │  └── 老对话 → AI 生成摘要替代
  │  结果: [200K tokens]
  │
  ├─ Layer 3: 微压缩
  │  └── 每种工具类型独立策略
  │  │   FILE_READ: 保留文件名 + 关键行
  │  │   Bash: 保留命令 + 退出码 + 最后 N 行
  │  │   Search: 保留 Top K 结果
  │  └── 缓存感知：避免修改缓存前缀！
  │  结果: [120K tokens]
  │
  ├─ Layer 4: 上下文折叠
  │  └── 归档旧内容，读取时投影
  │  结果: [100K tokens]
  │
  └─ Layer 5: 自动压缩（最后手段）
     └── AI 驱动的完整对话重写
     └── 熔断机制：连续 3 次失败则停止
     结果: [80K tokens]
```

**缓存感知的关键意义**：

```
Prompt Cache 机制：
├── API 缓存前 N 个 Token（系统 Prompt + 历史前缀）
├── 后续请求如果前缀不变 → Cache Hit（便宜 90%）
├── 如果前缀变了 → Cache Miss（重新计算，全价）
└── 因此压缩时必须保护前缀不变！

微压缩策略：
├── 只压缩缓存窗口之后的内容
├── Fork 子 Agent 共享相同前缀 → 单次 Miss
└── snipTokensFreed 精确传递释放量
```

**适用场景**：
- 长对话 AI 应用（客服、编程助手）
- 大上下文 AI 分析任务
- 需要成本优化的 AI 服务

---

### 模式 8：Agent 记忆与学习模式

**问题**：AI Agent 每次对话都从零开始，无法积累经验和知识，团队知识无法在 Agent 间共享。

**解决方案**：三层记忆架构 + Dream 巩固机制 + 团队同步。

**三层记忆体系**：

```
长期记忆（跨会话持久化）
├── 用户级 ~/.claude/agent-memory/
│   └── 个人偏好、通用经验
├── 项目级 .claude/agent-memory/
│   └── 项目知识、架构约定
└── 本地级 .claude/local-agent-memory/
    └── 机器相关配置

中期记忆（项目级配置）
├── CLAUDE.md（根目录）
│   └── 项目规范、编码约定
├── 嵌套 CLAUDE.md（子目录）
│   └── 模块特定约定
└── 团队记忆（远程同步）
    └── 团队共享知识

短期记忆（会话内）
├── 消息历史
│   └── 当前对话上下文
├── 文件缓存（COW）
│   └── 已读文件的内存副本
└── 权限决策缓存
    └── 已授权操作的记忆
```

**Dream 巩固机制**：

```
会话结束 → DreamTask 触发
  │
  ├─ Gate 1: 时间间隔检查（距上次 Dream > N 小时？）
  ├─ Gate 2: 会话数量检查（累积会话 > N 次？）
  ├─ Gate 3: 文件锁检查（无其他 Dream 在运行？）
  │
  └─ 通过所有 Gate：
     ├── Fork 子 Agent 分析会话内容
     ├── 提取有价值的洞察和模式
     ├── 写入长期记忆文件
     ├── 冲突检测（回滚锁）
     └── 后台静默完成
```

**团队记忆同步**：

```
本地记忆 ←→ 远程服务器
  ├── Pull: 服务端优先（解决冲突）
  ├── Push: 内容哈希比较（增量同步）
  ├── 安全: 秘密扫描（上传前检测凭证）
  └── 约束: 250KB 上限 / 本地删除不传播
```

**适用场景**：
- 需要跨会话学习的 AI 助手
- 团队知识管理和共享
- 持久化 AI Agent（个人助理、运维 Agent）

---

### 模式 9：推测性执行模式

**问题**：AI Agent 的响应存在延迟，用户等待时间影响体验。

**解决方案**：在用户空闲时 Fork Agent 预测下一步操作，匹配则合并，不匹配则丢弃。

**架构设计**：

```
主线程                     推测线程
  │                          │
  ├─ AI 响应完成              │
  │                          │
  ├─ 用户阅读中... ──────→   Fork Agent 启动
  │                          ├─ 预测下一步操作
  │                          ├─ 执行操作
  │                          └─ 写入 Overlay FS
  │                              $TEMP/speculation/{pid}/{id}
  │                          
  ├─ 用户输入 ─────────→    匹配检查
  │                          ├─ 匹配 → 合并 Overlay（零拷贝）
  │                          └─ 不匹配 → 丢弃 Overlay
  │
  └─ 继续处理
```

**安全边界检测**：

```
推测执行在以下情况自动停止：
├── Bash 命令（副作用不可逆）
├── Edit/Write 操作（文件修改需确认）
├── 权限拒绝（需要用户授权）
├── 网络请求（外部影响不可控）
└── 自然完成（任务结束）
```

**性能模型**：
- 可预测工作流：~40% 感知延迟降低
- 不可预测工作流：~0% 改善（推测丢弃，仅消耗少量计算）
- 平均：~20% 感知延迟降低

**适用场景**：
- AI 编码助手的下一步预测
- 智能客服的回答预准备
- 数据分析的预加载

---

### 模式 10：配置即策略模式 (Config as Strategy)

**问题**：Agent 系统需要支持多种行为模式（调试模式、安全模式、快速模式等），传统的继承或策略模式导致代码爆炸。

**解决方案**：一个核心引擎 + 配置对象决定全部行为，无需子类或策略接口。

**设计精髓**：

```typescript
// 传统方式：N 种模式 = N 个子类或 N 个策略对象
class CLIAgent extends BaseAgent { ... }
class SDKAgent extends BaseAgent { ... }
class BridgeAgent extends BaseAgent { ... }
// 模式组合爆炸：CLI+Debug, SDK+Safe, Bridge+Fast...

// Claude Code 方式：一个配置对象
interface AgentConfig {
  model: ModelConfig;
  tools: Tool[];
  canUseTool: PermissionChecker;
  maxTurns: number;
  bare: boolean;
  getAppState: () => AppState;
  setAppState: (state: AppState) => void;
  onMessage: MessageHandler;
  systemPrompt: string;
  // ... 20+ 配置项
}

// 同一个引擎，不同的配置
const engine = new QueryEngine(config);
```

**与 Coordinator 的结合**：

```
普通 Agent = QueryEngine + 标准配置
Coordinator = QueryEngine + Coordinator系统Prompt
Worker = QueryEngine + 受限工具集 + dontAsk权限

零代码路径差异！仅系统 Prompt 不同。
```

**Anti-Pattern 对比**：

| 维度 | 继承层次 | 策略模式 | 配置即策略 |
|------|---------|---------|-----------|
| 代码重复 | 高（方法覆盖） | 中（接口实现） | 低（配置变化） |
| 组合爆炸 | 高（NxM子类） | 中（NxM策略） | 低（配置组合） |
| 运行时灵活 | 低（编译时绑定） | 中（构造时注入） | 高（随时修改） |
| 理解难度 | 高（继承链） | 中（多接口） | 低（单配置对象） |

**适用场景**：
- 多模式 Agent 系统
- 多租户 AI 服务
- 需要运行时行为调整的系统

---

## 三、工程实践总结

### 架构级实践

| 实践 | 说明 | Claude Code 示例 |
|------|------|-----------------|
| AsyncGenerator 统一通信 | 所有层间通信用同一协议 | submitMessage → query → runAgent |
| 配置即策略 | 用配置而非继承支持多模式 | 8 种模式共用 QueryEngine |
| Fail-Closed 默认 | 安全相关默认走最严格路径 | 工具默认非并发安全 |
| COW 状态共享 | Fork 时共享状态，修改时拷贝 | Agent 文件缓存 |
| 水位线错误隔离 | 对象引用作为循环缓冲区标记 | 仅暴露当前轮次的错误 |

### 性能优化实践

| 实践 | 效果 | Claude Code 示例 |
|------|------|-----------------|
| Prompt Cache 稳定性 | 避免 90% 的缓存 Miss | 内置工具排序稳定 |
| 推测性执行 | ~40% 延迟降低 | Fork Agent 预测 |
| 懒加载 | 减少启动开销 | Bedrock SDK 按需加载 |
| 并行初始化 | 减少启动时间 | MDM + Keychain 并行 |
| 流式工具执行 | 减少等待时间 | 权限检查与 API 流并行 |

### 安全实践

| 实践 | 说明 | Claude Code 示例 |
|------|------|-----------------|
| 七层权限决策 | 从粗到细的安全过滤 | 工具池 → 模式 → 规则 → 用户 |
| 两阶段命令检测 | 快速预筛 + 精确判断 | 正则 → AI 分类器 |
| 决策可追溯 | 记录每次权限决策的来源 | PermissionMetadata |
| 权限只严不松 | 子 Agent 不能超越父 Agent | 继承 + 过滤 |
| 熔断保护 | 连续失败后停止尝试 | autoCompact 3 次失败停止 |

### 可扩展性实践

| 实践 | 说明 | Claude Code 示例 |
|------|------|-----------------|
| MCP 协议标准化 | 统一的工具集成协议 | 5 种传输 + OAuth |
| Skill 零代码扩展 | 纯 Markdown 定义能力 | Skill 市场 |
| Plugin 深度集成 | 进程内代码扩展 | Plugin 市场 |
| Feature Gate 裁剪 | 编译时产品线分离 | 20+ Gate |
| YAML Agent 定义 | 声明式 Agent 配置 | .claude/agents/ |

---

## 四、最佳实践检查清单

在设计新的 Agent 系统时，参考以下检查清单：

- [ ] **通信模式**：是否使用了 AsyncGenerator 或类似的流式管道？
- [ ] **会话管理**：每个 Agent 是否有独立的状态空间？
- [ ] **工具抽象**：是否有统一的工具接口和 Schema 校验？
- [ ] **权限控制**：是否有层次化的权限决策链？
- [ ] **嵌套支持**：是否支持 Agent 递归调用和预算控制？
- [ ] **多模式支持**：是否通过配置而非继承支持多种模式？
- [ ] **上下文管理**：是否有递进式的上下文压缩策略？
- [ ] **记忆系统**：是否支持跨会话的学习和知识积累？
- [ ] **性能优化**：是否利用了缓存、推测执行、懒加载？
- [ ] **安全默认**：是否遵循 Fail-Closed 原则？
- [ ] **可扩展性**：是否支持标准化的工具/能力扩展？
- [ ] **可观测性**：是否有完善的追踪、计量和审计？

---

## 五、总结

Claude Code 的 10 大设计模式构成了一个**完整的 Agent 系统设计方法论**。这些模式不是孤立的，而是相互支撑、协同工作的：

```
AsyncGenerator（通信基础）
  ↓
QueryEngine（引擎核心）→ 配置即策略（多模式）
  ↓
工具抽象（能力层）→ MCP/Skill/Plugin（扩展）
  ↓
权限决策链（安全层）→ Bash分类器（AI判断）
  ↓
递归嵌套Agent（协作层）→ Coordinator（编排）
  ↓
上下文管理（效率层）→ 缓存感知压缩
  ↓
记忆系统（知识层）→ Dream巩固 + 团队同步
  ↓
推测性执行（体验层）→ Overlay合并
```

掌握这套方法论，就能设计出**安全、高效、可扩展**的 AI Agent 系统。

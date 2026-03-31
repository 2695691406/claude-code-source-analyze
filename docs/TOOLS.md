# Claude Code 工具详述

> 43+ 智能体工具逐一详述，按分类汇总

---

## 目录

- [工具系统设计](#工具系统设计)
- [分类汇总](#分类汇总)
- [核心文件操作工具](#1-核心文件操作工具)
- [代码搜索与导航工具](#2-代码搜索与导航工具)
- [Shell 执行工具](#3-shell-执行工具)
- [规划与设计工具](#4-规划与设计工具)
- [隔离与工作树工具](#5-隔离与工作树工具)
- [多智能体与通信工具](#6-多智能体与通信工具)
- [任务管理工具](#7-任务管理工具)
- [用户交互工具](#8-用户交互工具)
- [Web 访问工具](#9-web-访问工具)
- [Notebook 编辑工具](#10-notebook-编辑工具)
- [配置工具](#11-配置工具)
- [MCP 协议工具](#12-mcp-协议工具)
- [调度与触发器工具](#13-调度与触发器工具)
- [实用工具](#14-实用工具)
- [特性门控工具](#特性门控工具)

---

## 工具系统设计

### 工具抽象基类 (`Tool.ts`)

所有工具继承自统一的抽象基类，定义了标准接口：

```typescript
type Tool = {
  // 标识
  name: string                                      // 唯一标识符
  searchHint: string                                // 搜索发现提示

  // Schema
  inputSchema: ZodSchema                            // 输入参数 (Zod 验证)
  outputSchema: ZodSchema                           // 输出格式 (Zod 验证)

  // 描述
  description(input, context): Promise<string>      // 动态描述 (可基于输入生成)
  prompt(): Promise<string>                         // 使用说明 (注入系统提示)
  userFacingName(input): string                     // 用户可见名称
  getActivityDescription?(input): string | undefined // 活动描述 (用于进度追踪)

  // 执行
  call(input, toolUseContext): Promise<ToolResult>  // 核心执行逻辑
  checkPermissions(): PermissionResult              // 权限前置检查

  // 元数据
  maxResultSizeChars: number                        // 结果持久化大小限制
  isConcurrencySafe(): boolean                      // 是否支持并发执行
  isReadOnly(): boolean                             // 是否为只读工具
  shouldDefer: boolean                              // 是否支持延迟加载
}
```

### 工具集合管理 (`tools.ts`)

`getTools()` 函数汇总所有工具来源：
1. **内置工具** — 40+ 核心工具
2. **MCP 工具** — 通过 MCP 协议动态发现
3. **Skill 工具** — 项目自定义技能
4. **Feature Gate 工具** — 条件编译控制

### 工具调用循环

```
API 返回 ToolUseBlock
    │
    ▼
StreamingToolExecutor 选择工具
    │
    ▼
CanUseTool() 权限检查
    │ allow / deny / ask
    ▼
Tool.call(input, context)
    │
    ▼
ToolResultBlock 返回给 API 循环
```

---

## 分类汇总

| 分类 | 工具 | 数量 |
|------|------|------|
| 文件操作 | Read, Write, Edit, Glob, Grep | 5 |
| 代码导航 | LSP | 1 |
| Shell 执行 | Bash, PowerShell, REPL | 3 |
| 规划设计 | EnterPlanMode, ExitPlanMode | 2 |
| 工作树隔离 | EnterWorktree, ExitWorktree | 2 |
| 多智能体 | Agent, SendMessage, TeamCreate, TeamDelete, Skill | 5 |
| 任务管理 | TaskCreate, TaskGet, TaskList, TaskUpdate, TaskOutput, TaskStop, TodoWrite | 7 |
| 用户交互 | AskUserQuestion, SendUserMessage | 2 |
| Web 访问 | WebFetch, WebSearch | 2 |
| Notebook | NotebookEdit | 1 |
| 配置 | Config | 1 |
| MCP 协议 | ListMcpResources, ReadMcpResource, MCP, McpAuthTool | 4 |
| 调度触发 | CronCreate, CronDelete, CronList, RemoteTrigger | 4 |
| 实用工具 | ToolSearch, Sleep, StructuredOutput | 3 |
| **总计** | | **42+** |

---

## 1. 核心文件操作工具

### Read (FileReadTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Read` |
| **目录** | `tools/FileReadTool/` |
| **功能** | 读取文件、图片、PDF、Jupyter Notebook 的内容 |
| **只读** | ✅ 是 |
| **并发安全** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `path` | string | ✅ | 文件路径 |
| `range` | [number, number] | ❌ | 行号范围 `[start, end]` |
| `line_numbers` | boolean | ❌ | 是否显示行号 |

**行为说明**：
- 自动检测文件类型（文本、图片、PDF、Notebook）
- 图片返回 Base64 编码
- PDF 提取文本内容
- Notebook 解析 Cell 结构
- 支持行号范围读取以节省上下文

---

### Write (FileWriteTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Write` |
| **目录** | `tools/FileWriteTool/` |
| **功能** | 创建新文件或完全覆写现有文件 |
| **只读** | ❌ 否 |
| **并发安全** | ❌ 否 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file_path` | string | ✅ | 目标文件路径 |
| `content` | string | ✅ | 文件内容 |

**行为说明**：
- 自动创建父目录
- 完全覆写文件内容（非增量编辑）
- 需要文件写入权限

---

### Edit (FileEditTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Edit` |
| **目录** | `tools/FileEditTool/` |
| **功能** | 在现有文件中进行精确的查找-替换或行范围编辑 |
| **只读** | ❌ 否 |
| **并发安全** | ❌ 否 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file_path` | string | ✅ | 目标文件路径 |
| `old_string` | string | 条件 | 查找-替换模式: 要替换的原字符串 |
| `new_string` | string | 条件 | 查找-替换模式: 替换后的新字符串 |
| `line_start` | number | 条件 | 行范围模式: 起始行号 |
| `line_end` | number | 条件 | 行范围模式: 结束行号 |
| `content` | string | 条件 | 行范围模式: 替换内容 |

**行为说明**：
- 两种编辑模式：查找-替换 或 行范围替换
- 查找-替换要求 `old_string` 在文件中唯一匹配
- 返回编辑后的文件片段作为确认

---

### Glob (GlobTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Glob` |
| **目录** | `tools/GlobTool/` |
| **功能** | 使用 Glob 模式查找文件 |
| **只读** | ✅ 是 |
| **并发安全** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `pattern` | string | ✅ | Glob 模式 (如 `**/*.ts`) |
| `path` | string | ❌ | 搜索根目录 |

**行为说明**：
- 递归遍历目录树
- 尊重 `.gitignore` 规则
- 返回匹配的文件路径列表

---

### Grep (GrepTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Grep` |
| **目录** | `tools/GrepTool/` |
| **功能** | 使用正则表达式搜索文件内容 |
| **只读** | ✅ 是 |
| **并发安全** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `pattern` | string | ✅ | 正则表达式模式 |
| `path` | string | ❌ | 搜索目录 |
| `glob` | string | ❌ | 文件过滤 Glob |
| `output_mode` | string | ❌ | 输出模式: `content` / `files_with_matches` / `count` |
| `context_before` | number | ❌ | 匹配前上下文行数 |
| `context_after` | number | ❌ | 匹配后上下文行数 |

**行为说明**：
- 基于 ripgrep 的高性能搜索
- 支持多种输出模式
- 支持上下文行显示

---

## 2. 代码搜索与导航工具

### LSP (LSPTool)

| 属性 | 值 |
|------|-----|
| **名称** | `LSP` |
| **目录** | `tools/LSPTool/` |
| **功能** | Language Server Protocol 代码智能 — 定义跳转、引用查找、符号搜索 |
| **只读** | ✅ 是 |
| **并发安全** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `operation` | string | ✅ | 操作类型: `definitions` / `references` / `hover` / `documentSymbols` / `workspaceSymbols` |
| `filePath` | string | ✅ | 目标文件路径 |
| `line` | number | 条件 | 行号（定义/引用/悬停需要） |
| `character` | number | 条件 | 列号 |
| `query` | string | 条件 | 符号搜索查询 |

**行为说明**：
- 依赖 LSP 服务管理器 (`services/lsp/`)
- 自动启动/管理对应语言的 LSP 服务器
- 返回结构化的代码智能信息

---

## 3. Shell 执行工具

### Bash (BashTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Bash` |
| **目录** | `tools/BashTool/` |
| **功能** | 执行 Shell 命令 |
| **只读** | ❌ 否 |
| **并发安全** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `command` | string | ✅ | 要执行的 Shell 命令 |
| `description` | string | ❌ | 命令描述（用于权限提示） |
| `timeout` | number | ❌ | 超时时间 (毫秒) |

**行为说明**：
- 通过 `Shell` 抽象层执行
- 经过 `bashClassifier` 危险命令检测
- 支持超时控制
- 捕获 stdout 和 stderr
- 受权限系统控制

### PowerShell

| 属性 | 值 |
|------|-----|
| **名称** | `PowerShell` |
| **功能** | Windows PowerShell 命令执行 |
| **限制** | 仅 Windows 平台 |

### REPL (REPLTool)

| 属性 | 值 |
|------|-----|
| **名称** | `REPL` |
| **功能** | 交互式 VM 环境 (JavaScript) |
| **限制** | 特性门控: ant/PROACTIVE/KAIROS |

---

## 4. 规划与设计工具

### EnterPlanMode

| 属性 | 值 |
|------|-----|
| **名称** | `EnterPlanMode` |
| **目录** | `tools/EnterPlanModeTool/` |
| **功能** | 切换到计划模式，用于处理复杂任务 |
| **只读** | ✅ 是 |

**输入参数**：无

**行为说明**：
- 将智能体切换到计划模式
- 在计划模式下，Claude 只制定计划不执行
- 用户可审批计划后再执行
- 切换权限模式为 `plan`

### ExitPlanMode

| 属性 | 值 |
|------|-----|
| **名称** | `ExitPlanMode` |
| **目录** | `tools/ExitPlanModeTool/` |
| **功能** | 退出计划模式，携带已审批的实现计划 |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `allowedPrompts` | string[] | ❌ | 允许的提示词列表 |
| `plan` | string | ❌ | 审批通过的计划内容 |

---

## 5. 隔离与工作树工具

### EnterWorktree

| 属性 | 值 |
|------|-----|
| **名称** | `EnterWorktree` |
| **目录** | `tools/EnterWorktreeTool/` |
| **功能** | 创建隔离的 Git Worktree 环境 |
| **只读** | ❌ 否 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ❌ | Worktree 名称 |

**行为说明**：
- 创建新的 Git Worktree
- 切换工作目录到 Worktree
- 提供隔离的文件修改环境
- 适用于不想影响主分支的操作

### ExitWorktree

| 属性 | 值 |
|------|-----|
| **名称** | `ExitWorktree` |
| **目录** | `tools/ExitWorktreeTool/` |
| **功能** | 退出 Worktree 并清理 |
| **只读** | ❌ 否 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `action` | string | ✅ | `keep` (保留变更) 或 `remove` (删除 worktree) |
| `discard_changes` | boolean | ❌ | 是否丢弃未提交变更 |

---

## 6. 多智能体与通信工具

### Agent (AgentTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Agent` |
| **目录** | `tools/AgentTool/` |
| **功能** | 生成子智能体执行复杂任务 |
| **只读** | ❌ 否 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `description` | string | ✅ | 3-5 词简要描述 |
| `prompt` | string | ✅ | 详细任务指令 |
| `subagent_type` | string | ❌ | 智能体类型 (worker, explorer 等) |
| `model` | string | ❌ | 模型覆盖 (`sonnet`/`opus`/`haiku`) |
| `run_in_background` | boolean | ❌ | 是否后台运行 |
| `name` | string | ❌ | 智能体名称（用于 SendMessage 寻址） |
| `team_name` | string | ❌ | 所属团队名称 |
| `mode` | string | ❌ | 权限模式 (`plan`/`normal`) |
| `isolation` | string | ❌ | 隔离类型 (`worktree`/`remote`) |
| `cwd` | string | ❌ | 工作目录覆盖 |

**执行路径**：
- **同步模式**：智能体在当前会话完成，返回 `status: 'completed'` + 结果
- **异步模式**：智能体在后台运行（`LocalAgentTask`），返回 `status: 'async_launched'` + agentId
- **远程模式**：智能体在云端 CCR 环境运行（`RemoteAgentTask`）

> 详细架构见 [Agent 设计文档](./AGENT_DESIGN.md)

---

### SendMessage (SendMessageTool)

| 属性 | 值 |
|------|-----|
| **名称** | `SendMessage` |
| **目录** | `tools/SendMessageTool/` |
| **功能** | 智能体间消息传递 |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `to` | string | ✅ | 接收方：智能体名称 / `*`(广播) / `uds:<path>` / `bridge:<id>` |
| `summary` | string | ❌ | 5-10 词消息摘要 |
| `message` | string / StructuredMessage | ✅ | 消息内容 |

**结构化消息类型**：
- `shutdown_request` — 请求队友停止
- `shutdown_response` — 审批/拒绝停止请求
- `plan_approval_response` — 审批/拒绝计划

---

### TeamCreate (TeamCreateTool)

| 属性 | 值 |
|------|-----|
| **名称** | `TeamCreate` |
| **目录** | `tools/TeamCreateTool/` |
| **功能** | 创建智能体团队（Swarm） |
| **标记** | 内部工具（对用户隐藏） |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `team_name` | string | ✅ | 团队名称 |
| `description` | string | ❌ | 团队描述 |
| `agent_type` | string | ❌ | 默认智能体类型 |

---

### TeamDelete (TeamDeleteTool)

| 属性 | 值 |
|------|-----|
| **名称** | `TeamDelete` |
| **目录** | `tools/TeamDeleteTool/` |
| **功能** | 解散智能体团队 |
| **标记** | 内部工具（对用户隐藏） |

---

### Skill (SkillTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Skill` |
| **目录** | `tools/SkillTool/` |
| **功能** | 执行项目自定义技能 |
| **只读** | 取决于技能定义 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `command_name` | string | ✅ | 技能名称 |
| (其他) | 动态 | 取决于技能 | 技能定义的自定义参数 |

---

## 7. 任务管理工具

### TaskCreate

| 属性 | 值 |
|------|-----|
| **名称** | `TaskCreate` |
| **功能** | 创建新任务 (v2 Todo 系统) |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `subject` | string | ✅ | 任务标题 |
| `description` | string | ❌ | 任务描述 |
| `activeForm` | string | ❌ | 活动表单 |
| `metadata` | object | ❌ | 附加元数据 |

### TaskGet

| 属性 | 值 |
|------|-----|
| **名称** | `TaskGet` |
| **功能** | 按 ID 获取任务详情 |

**输入参数**：`taskId: string`

### TaskList

| 属性 | 值 |
|------|-----|
| **名称** | `TaskList` |
| **功能** | 列出所有任务 |

**输入参数**：无

### TaskUpdate

| 属性 | 值 |
|------|-----|
| **名称** | `TaskUpdate` |
| **功能** | 更新任务状态和内容 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | ✅ | 任务 ID |
| `subject` | string | ❌ | 更新标题 |
| `description` | string | ❌ | 更新描述 |
| `status` | string | ❌ | 更新状态 |
| `blocks` | array | ❌ | 阻塞关系 |
| `owner` | string | ❌ | 负责人 |
| `metadata` | object | ❌ | 更新元数据 |

### TaskOutput

| 属性 | 值 |
|------|-----|
| **名称** | `TaskOutput` |
| **目录** | `tools/TaskOutputTool/` |
| **功能** | 读取任务输出结果 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_id` | string | ✅ | 任务 ID |
| `block` | boolean | ❌ | 是否阻塞等待 |
| `timeout` | number | ❌ | 等待超时 |

### TaskStop

| 属性 | 值 |
|------|-----|
| **名称** | `TaskStop` |
| **目录** | `tools/TaskStopTool/` |
| **功能** | 终止运行中的任务 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_id` | string | ✅ | 任务 ID |

### TodoWrite (v1 — 已废弃)

| 属性 | 值 |
|------|-----|
| **名称** | `TodoWrite` |
| **目录** | `tools/TodoWriteTool/` |
| **功能** | 管理 v1 Todo 清单（已废弃，建议使用 TaskCreate/TaskUpdate） |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `todos` | array | ✅ | Todo 项数组 |

---

## 8. 用户交互工具

### AskUserQuestion

| 属性 | 值 |
|------|-----|
| **名称** | `AskUserQuestion` |
| **目录** | `tools/AskUserQuestionTool/` |
| **功能** | 向用户展示多选题提示 |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `questions` | array | ✅ | 问题数组 |
| `questions[].text` | string | ✅ | 问题文本 |
| `questions[].options` | string[] | ✅ | 选项列表 |
| `questions[].multiSelect` | boolean | ❌ | 是否允许多选 |

### SendUserMessage

| 属性 | 值 |
|------|-----|
| **功能** | 向用户发送聊天消息 |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `message` | string | ✅ | 消息内容 |
| `attachments` | array | ❌ | 附件列表 |
| `status` | string | ❌ | `normal` 或 `proactive` |

---

## 9. Web 访问工具

### WebFetch

| 属性 | 值 |
|------|-----|
| **名称** | `WebFetch` |
| **目录** | `tools/WebFetchTool/` |
| **功能** | 获取 URL 内容并提取信息 |
| **只读** | ✅ 是 |
| **并发安全** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `url` | string | ✅ | 目标 URL |
| `prompt` | string | ❌ | 信息提取提示（指导从页面中提取什么） |

### WebSearch

| 属性 | 值 |
|------|-----|
| **名称** | `WebSearch` |
| **目录** | `tools/WebSearchTool/` |
| **功能** | 执行 Web 搜索 |
| **只读** | ✅ 是 |
| **并发安全** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | ✅ | 搜索查询 |
| `allowed_domains` | string[] | ❌ | 白名单域名 |
| `blocked_domains` | string[] | ❌ | 黑名单域名 |

---

## 10. Notebook 编辑工具

### NotebookEdit

| 属性 | 值 |
|------|-----|
| **名称** | `NotebookEdit` |
| **目录** | `tools/NotebookEditTool/` |
| **功能** | 编辑 Jupyter Notebook Cell |
| **只读** | ❌ 否 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `notebook_path` | string | ✅ | Notebook 文件路径 |
| `cell_id` | string | ✅ | Cell 标识 |
| `new_source` | string | ✅ | 新的 Cell 源码 |
| `cell_type` | string | ❌ | Cell 类型 (`code`/`markdown`) |
| `edit_mode` | string | ❌ | 编辑模式 |

---

## 11. 配置工具

### Config

| 属性 | 值 |
|------|-----|
| **名称** | `Config` |
| **功能** | 获取或设置配置项 |
| **只读** | 视操作而定 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `setting` | string | ✅ | 配置键名 |
| `value` | any | ❌ | 设置值（省略则读取） |

---

## 12. MCP 协议工具

### ListMcpResources

| 属性 | 值 |
|------|-----|
| **名称** | `ListMcpResources` |
| **目录** | `tools/ListMcpResourcesTool/` |
| **功能** | 列出 MCP 服务器暴露的资源 |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `server` | string | ❌ | 指定 MCP 服务器名称（省略列出所有） |

### ReadMcpResource

| 属性 | 值 |
|------|-----|
| **名称** | `ReadMcpResource` |
| **目录** | `tools/ReadMcpResourceTool/` |
| **功能** | 读取 MCP 资源内容 |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `server` | string | ✅ | MCP 服务器名称 |
| `uri` | string | ✅ | 资源 URI |

### MCP (MCPTool)

| 属性 | 值 |
|------|-----|
| **名称** | 动态（per-tool） |
| **目录** | `tools/MCPTool/` |
| **功能** | MCP 服务器暴露的通用工具模板 |

**行为说明**：
- 每个 MCP 服务器工具生成一个 MCPTool 实例
- 输入 Schema 由 MCP 服务器定义
- 执行时代理调用 MCP 服务器的对应工具

### McpAuthTool

| 属性 | 值 |
|------|-----|
| **功能** | MCP 服务器的 OAuth 认证 |
| **行为** | 动态生成 per-server 认证流程 |

---

## 13. 调度与触发器工具

### CronCreate

| 属性 | 值 |
|------|-----|
| **功能** | 创建计划触发器 |
| **特性门控** | KAIROS |

**输入参数**：包含 Cron 表达式和任务提示

### CronDelete

| 属性 | 值 |
|------|-----|
| **功能** | 删除计划触发器 |

### CronList

| 属性 | 值 |
|------|-----|
| **功能** | 列出所有计划触发器 |

### RemoteTrigger

| 属性 | 值 |
|------|-----|
| **功能** | 管理远程触发器 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `action` | string | ✅ | `list` / `get` / `create` / `update` / `run` |
| `trigger_id` | string | 条件 | 触发器 ID |
| `body` | object | 条件 | 请求体 |

---

## 14. 实用工具

### ToolSearch

| 属性 | 值 |
|------|-----|
| **功能** | 搜索延迟加载的工具 |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | ✅ | 搜索查询 |
| `max_results` | number | ❌ | 最大返回数量 |

### Sleep (SleepTool)

| 属性 | 值 |
|------|-----|
| **名称** | `Sleep` |
| **目录** | `tools/SleepTool/` |
| **功能** | 延迟执行（用于主动/后台任务） |
| **只读** | ✅ 是 |

**输入参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `duration` | number | ✅ | 延迟秒数 |

### StructuredOutput

| 属性 | 值 |
|------|-----|
| **功能** | 返回结构化 JSON 输出 |
| **限制** | 仅 SDK/CLI 模式可用 |

---

## 特性门控工具

以下工具需要特定 Feature Flag 启用：

| 工具 | 特性门控 | 说明 |
|------|---------|------|
| **PowerShell** | Windows 平台 | Windows Shell 执行 |
| **REPL** | ant/PROACTIVE/KAIROS | 交互式 JS 虚拟机 |
| **WebBrowserTool** | 特性门控 | Web 浏览器自动化 |
| **CtxInspectTool** | CONTEXT_COLLAPSE | 上下文检查 |
| **TerminalCaptureTool** | TERMINAL_PANEL | 终端面板截图 |
| **WorkflowTool** | WORKFLOW_SCRIPTS | 工作流脚本执行 |
| **VerifyPlanExecutionTool** | 环境变量 | 计划验证 |
| **ListPeersTool** | UDS_INBOX | UDS Peer 列表 |
| **MonitorTool** | 特性门控 | MCP 服务器监控 |
| **PushNotificationTool** | KAIROS | 推送通知 |

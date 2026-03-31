# Claude Code 命令详述

> 86+ CLI 命令逐一详述，按分类汇总

---

## 目录

- [命令系统设计](#命令系统设计)
- [分类汇总](#分类汇总)
- [公共命令详述](#公共命令详述)
  - [会话与上下文管理](#1-会话与上下文管理)
  - [AI 模型与配置](#2-ai-模型与配置)
  - [会话信息与状态](#3-会话信息与状态)
  - [设置与配置](#4-设置与配置)
  - [认证与账户](#5-认证与账户)
  - [订阅与限额](#6-订阅与限额)
  - [集成与安装](#7-集成与安装)
  - [开发工具](#8-开发工具)
  - [代码审查与质量](#9-代码审查与质量)
  - [沟通与工具](#10-沟通与工具)
  - [远程与协作](#11-远程与协作)
  - [项目设置](#12-项目设置)
  - [发布与更新](#13-发布与更新)
  - [其他](#14-其他)
- [内部命令详述](#内部命令详述)
- [特性门控命令](#特性门控命令)

---

## 命令系统设计

### 命令类型

| 类型 | 说明 |
|------|------|
| `local` | 本地执行，返回纯文本结果（无 UI） |
| `local-jsx` | 本地执行，渲染 React Ink UI |
| `prompt` | 作为提示词发送给 Claude 模型处理 |
| `proactive` | 自主型主动任务 |

### 命令属性

```typescript
type Command = {
  name: string              // 命令标识符 (如 "context")
  description: string       // 用户可见的帮助文本
  aliases?: string[]        // 别名 (如 "reset", "new")
  argumentHint?: string     // 用法提示 (如 "[low|medium|high]")
  immediate?: boolean       // 立即执行（不排队）
  isEnabled?: () => boolean // 运行时可用性检查
  isHidden?: boolean        // 从 /help 隐藏但仍可用
  supportsNonInteractive?: boolean // 非交互模式支持
  availability?: string     // 平台限制 (claude-ai, console 等)
}
```

### 命令注册系统

`COMMANDS()` 函数（记忆化）汇总以下来源的命令：
1. 核心内置命令 (`commands/` 子目录)
2. 根级命令文件 (`commands/*.ts`)
3. Skill 技能目录命令
4. Plugin 插件命令
5. 内置技能 (`skills/bundled/`)
6. 工作流命令 (`WORKFLOW_SCRIPTS` 特性)
7. 动态技能（运行时发现）
8. MCP 技能

### 过滤与可见性

- `meetsAvailabilityRequirement()` — Provider 级别过滤
- `isCommandEnabled()` — 用户可用性检查
- `isHidden` — UI 不可见但命令仍可用

---

## 分类汇总

| 分类 | 命令数 | 典型命令 |
|------|--------|---------|
| 会话与上下文管理 | 10 | `/clear`, `/compact`, `/context`, `/plan`, `/resume` |
| AI 模型与配置 | 7 | `/model`, `/effort`, `/fast`, `/advisor`, `/theme` |
| 会话信息与状态 | 4 | `/cost`, `/status`, `/usage`, `/version` |
| 设置与配置 | 6 | `/config`, `/help`, `/memory`, `/permissions`, `/vim` |
| 认证与账户 | 3 | `/login`, `/logout`, `/oauth-refresh` |
| 订阅与限额 | 5 | `/upgrade`, `/passes`, `/extra-usage`, `/privacy-settings` |
| 集成与安装 | 6 | `/install-github-app`, `/install-slack-app`, `/ide`, `/desktop` |
| 开发工具 | 7 | `/doctor`, `/plugin`, `/skills`, `/hooks`, `/mcp` |
| 代码审查 | 3 | `/review`, `/ultrareview`, `/security-review` |
| 沟通与工具 | 4 | `/feedback`, `/btw`, `/copy`, `/tasks` |
| 远程与协作 | 4 | `/branch`, `/remote-control`, `/session`, `/remote-env` |
| 项目设置 | 2 | `/init`, `/init-verifiers` |
| 发布与更新 | 1 | `/release-notes` |
| 其他 | 3 | `/stickers`, `/thinkback`, `/exit` |
| **内部命令** | **16+** | `/commit`, `/autofix-pr`, `/bughunter`, `/teleport` |
| **特性门控** | **10+** | `/voice`, `/buddy`, `/assistant`, `/workflows` |

---

## 公共命令详述

### 1. 会话与上下文管理

#### `/add-dir`
- **功能**：将新的工作目录添加到当前会话的上下文中
- **类型**：`local`
- **参数**：目录路径
- **用途**：在多目录项目中扩展文件访问范围

#### `/clear`
- **功能**：清除对话历史，释放上下文空间
- **类型**：`local`
- **别名**：`/reset`, `/new`
- **行为**：完全重置消息历史，不保留摘要

#### `/compact`
- **功能**：清除对话历史但保留上下文摘要
- **类型**：`local`
- **行为**：使用 Claude 生成对话摘要，然后清除历史但注入摘要作为上下文
- **参数**：可选的自定义摘要提示
- **关键实现**：调用 `services/compact/` 的压缩算法，支持自动压缩、微压缩、会话记忆压缩等

#### `/context`
- **功能**：可视化当前上下文使用情况
- **类型**：`local-jsx`
- **输出**：彩色网格显示上下文窗口使用率（消息、工具结果、系统提示等占比）

#### `/diff`
- **功能**：查看未提交的变更和每轮对话的差异
- **类型**：`local-jsx`
- **输出**：Git diff 格式的变更展示

#### `/export`
- **功能**：将当前对话导出到文件或剪贴板
- **类型**：`local`
- **参数**：输出文件路径（可选）
- **格式**：Markdown 格式的对话记录

#### `/plan`
- **功能**：启用计划模式或查看当前会话计划
- **类型**：`local`
- **行为**：切换到计划模式时，Claude 将先制定计划再执行，需要用户审批

#### `/rename`
- **功能**：重命名当前对话
- **类型**：`local`
- **标记**：`immediate`（立即执行）
- **参数**：新名称

#### `/resume`
- **功能**：恢复之前的对话
- **类型**：`local-jsx`
- **别名**：`/continue`
- **参数**：搜索关键词（可选）
- **行为**：显示历史对话列表，支持搜索和选择

#### `/tag`
- **功能**：在当前会话上切换可搜索标签
- **类型**：`local`
- **限制**：仅 Ant 内部用户可用

---

### 2. AI 模型与配置

#### `/effort`
- **功能**：设置 Claude 的努力程度
- **类型**：`local`
- **标记**：`immediate`
- **参数**：`[low|medium|high|max|auto]`
- **效果**：影响 Claude 的思考深度和响应质量

#### `/fast`
- **功能**：切换快速模式（仅使用快速模型）
- **类型**：`local`
- **标记**：`immediate`
- **效果**：在快速和默认模型之间切换

#### `/model`
- **功能**：设置 AI 模型
- **类型**：`local`
- **标记**：`immediate`
- **参数**：模型名称（如 `claude-3-7-sonnet`）
- **描述**：动态显示当前模型名称

#### `/advisor`
- **功能**：配置顾问模型，用于二次分析
- **类型**：`local`
- **用途**：设置辅助模型进行交叉验证

#### `/color`
- **功能**：设置会话的提示栏颜色
- **类型**：`local`
- **标记**：`immediate`

#### `/theme`
- **功能**：更改终端主题
- **类型**：`local-jsx`
- **输出**：主题选择界面

#### `/output-style`
- **功能**：已废弃 — 使用 `/config` 替代
- **标记**：`hidden`

---

### 3. 会话信息与状态

#### `/cost`
- **功能**：显示会话总费用和持续时间
- **类型**：`local`
- **输出**：Token 使用量、API 调用次数、预估费用

#### `/status`
- **功能**：显示 Claude Code 当前状态
- **类型**：`local`
- **标记**：`immediate`
- **输出**：版本号、当前模型、账户信息、API 状态、可用工具列表

#### `/usage`
- **功能**：显示 Claude AI 订阅者的计划使用限额
- **类型**：`local-jsx`
- **限制**：仅 Claude AI 订阅用户可见

#### `/version`
- **功能**：打印当前会话版本信息
- **类型**：`local`
- **限制**：仅 Ant 内部用户可用

---

### 4. 设置与配置

#### `/config`
- **功能**：打开配置面板
- **类型**：`local-jsx`
- **别名**：`/settings`
- **输出**：交互式配置界面，管理模型、权限、MCP 等设置

#### `/help`
- **功能**：显示帮助信息和可用命令
- **类型**：`local-jsx`
- **输出**：命令列表、快捷键、使用说明

#### `/keybindings`
- **功能**：创建或编辑键盘绑定配置
- **类型**：`local`
- **行为**：打开键盘绑定配置文件进行编辑

#### `/memory`
- **功能**：编辑 Claude 记忆文件
- **类型**：`local`
- **行为**：打开 `CLAUDE.md` 或其他记忆文件
- **用途**：管理项目级和用户级的上下文记忆

#### `/permissions`
- **功能**：管理工具权限规则
- **类型**：`local-jsx`
- **别名**：`/allowed-tools`
- **输出**：权限规则列表，支持添加/删除/修改

#### `/vim`
- **功能**：切换 Vim/Normal 编辑模式
- **类型**：`local`
- **效果**：在 Vim 模式和普通模式之间切换输入行为

---

### 5. 认证与账户

#### `/login`
- **功能**：登录或切换 Anthropic 账户
- **类型**：`local-jsx`
- **标记**：`immediate`
- **流程**：触发 OAuth 2.0 PKCE 认证流程

#### `/logout`
- **功能**：登出当前账户
- **类型**：`local`
- **行为**：清除 OAuth Token 和认证状态

---

### 6. 订阅与限额

#### `/upgrade`
- **功能**：升级到 Max 套餐（更高限额 + 更多 Opus）
- **类型**：`local`
- **输出**：升级选项和链接

#### `/passes`
- **功能**：分享 Claude Code 免费体验周（赚取额外用量）
- **类型**：`local`

#### `/extra-usage`
- **功能**：配置限额用尽后的额外用量设置
- **类型**：`local`

#### `/rate-limit-options`
- **功能**：在触发限速时显示可用选项
- **类型**：`local`
- **标记**：`hidden`

#### `/privacy-settings`
- **功能**：查看/更新隐私设置
- **类型**：`local`
- **限制**：仅订阅用户可见

---

### 7. 集成与安装

#### `/install-github-app`
- **功能**：设置 Claude GitHub Actions 集成
- **类型**：`local-jsx`
- **行为**：引导用户安装 GitHub App

#### `/install-slack-app`
- **功能**：安装 Claude Slack 应用
- **类型**：`local-jsx`

#### `/ide`
- **功能**：管理 IDE 集成
- **类型**：`local-jsx`
- **支持**：VSCode、JetBrains

#### `/desktop`
- **功能**：在 Claude Desktop 中继续当前会话
- **类型**：`local`
- **别名**：`/app`
- **限制**：macOS 和 Win64

#### `/mobile`
- **功能**：显示移动端应用二维码
- **类型**：`local`
- **别名**：`/ios`, `/android`

#### `/chrome`
- **功能**：Chrome 浏览器中的 Claude（Beta）设置
- **类型**：`local`

---

### 8. 开发工具

#### `/doctor`
- **功能**：诊断安装和设置问题
- **类型**：`local-jsx`
- **输出**：环境检查报告（Node.js、Git、权限、API 连接等）

#### `/plugin`
- **功能**：管理插件
- **类型**：`local-jsx`
- **别名**：`/plugins`, `/marketplace`
- **标记**：`immediate`
- **行为**：浏览/安装/卸载插件

#### `/reload-plugins`
- **功能**：激活挂起的插件变更
- **类型**：`local`

#### `/skills`
- **功能**：列出可用技能
- **类型**：`local`
- **输出**：所有已加载技能及其描述

#### `/hooks`
- **功能**：查看当前的钩子配置
- **类型**：`local`
- **标记**：`immediate`

#### `/mcp`
- **功能**：管理 MCP 服务器
- **类型**：`local`
- **标记**：`immediate`
- **参数**：`[enable|disable server-name]`
- **行为**：启用/禁用 MCP 服务器连接

#### `/files`
- **功能**：列出上下文中的所有文件
- **类型**：`local`
- **限制**：仅 Ant 内部用户可用

---

### 9. 代码审查与质量

#### `/review`
- **功能**：本地 Pull Request 审查分析
- **类型**：`prompt`
- **参数**：PR 编号或分支名
- **行为**：获取 PR diff 并进行代码审查

#### `/ultrareview`
- **功能**：深度远程 Bug 搜索（10-20 分钟）
- **类型**：`local-jsx`
- **行为**：使用 Claude Code on Web 进行深度代码分析
- **特点**：远程执行，超时 10-20 分钟

#### `/security-review`
- **功能**：安全性聚焦的代码审查
- **类型**：`prompt`
- **行为**：针对待提交变更进行安全审查

---

### 10. 沟通与工具

#### `/feedback`
- **功能**：提交关于 Claude Code 的反馈
- **类型**：`local-jsx`
- **别名**：`/bug`
- **行为**：收集并发送用户反馈

#### `/btw`
- **功能**：发送快速的旁白问题（不中断当前工作）
- **类型**：`prompt`
- **标记**：`immediate`
- **用途**：在不影响主对话流的情况下提问

#### `/copy`
- **功能**：复制 Claude 的最近回复
- **类型**：`local`
- **参数**：`/copy N` 复制倒数第 N 个回复

#### `/tasks`
- **功能**：列出和管理后台任务
- **类型**：`local-jsx`
- **别名**：`/bashes`
- **输出**：后台运行的 Bash 任务和 Agent 任务列表

---

### 11. 远程与协作

#### `/branch`
- **功能**：创建当前对话的分支
- **类型**：`local`
- **别名**：`/fork`（如果 `/fork` 未被 `FORK_SUBAGENT` 特性占用）
- **行为**：复制当前对话状态到新分支

#### `/remote-control`
- **功能**：连接远程控制会话
- **类型**：`local`
- **别名**：`/rc`
- **标记**：`immediate`
- **行为**：建立远程控制连接

#### `/session`
- **功能**：显示远程会话 URL 和二维码
- **类型**：`local`
- **别名**：`/remote`
- **输出**：远程连接信息

#### `/remote-env`
- **功能**：配置远程传送的默认环境
- **类型**：`local`

---

### 12. 项目设置

#### `/init`
- **功能**：使用 CLAUDE.md 初始化项目
- **类型**：`prompt`
- **行为**：创建项目级 CLAUDE.md 配置文件

#### `/init-verifiers`
- **功能**：创建验证器技能用于自动化代码验证
- **类型**：`prompt`
- **行为**：在 `.claude/skills/` 下生成验证器定义

---

### 13. 发布与更新

#### `/release-notes`
- **功能**：查看发布说明
- **类型**：`local`
- **输出**：当前版本的更新内容

---

### 14. 其他

#### `/stickers`
- **功能**：订购 Claude Code 贴纸
- **类型**：`local`

#### `/thinkback`
- **功能**：2025 年 Claude Code 年度回顾
- **类型**：`local-jsx`
- **特性门控**：需要特定 Feature Flag

#### `/exit`
- **功能**：退出 REPL
- **类型**：`local`
- **别名**：`/quit`
- **标记**：`immediate`

---

## 内部命令详述

以下命令仅在 `USER_TYPE === 'ant'` 且非 Demo 模式下可用：

#### `/ant-trace`
- **功能**：性能和执行追踪
- **用途**：内部性能分析

#### `/autofix-pr`
- **功能**：自动修复并创建 PR
- **行为**：分析代码问题并自动创建修复 PR

#### `/backfill-sessions`
- **功能**：回填会话数据
- **用途**：数据补充

#### `/break-cache`
- **功能**：使缓存失效
- **用途**：测试缓存机制

#### `/bridge-kick`
- **功能**：注入 Bridge 失败以测试恢复
- **用途**：Bridge 模式容错测试

#### `/bughunter`
- **功能**：查找并报告 Bug
- **行为**：自动化 Bug 搜索

#### `/commit`
- **功能**：AI 辅助创建 Git Commit
- **行为**：分析变更、生成 commit message、执行提交

#### `/commit-push-pr`
- **功能**：完整的 Commit → Push → PR 工作流
- **行为**：一键完成从提交到 PR 的全流程

#### `/ctx_viz`
- **功能**：上下文可视化
- **用途**：调试上下文窗口使用情况

#### `/debug-tool-call`
- **功能**：调试工具调用执行
- **用途**：开发调试

#### `/good-claude`
- **功能**：内部验证测试
- **用途**：质量验证

#### `/issue`
- **功能**：创建/管理 Issues
- **行为**：AI 辅助创建 GitHub/GitLab Issues

#### `/mock-limits`
- **功能**：模拟速率限制
- **用途**：限速功能测试

#### `/oauth-refresh`
- **功能**：刷新 OAuth Token
- **行为**：手动触发 Token 刷新

#### `/perf-issue`
- **功能**：性能问题追踪
- **用途**：性能分析

#### `/reset-limits`
- **功能**：重置速率限制
- **用途**：测试恢复

#### `/onboarding`
- **功能**：项目新手引导流程
- **行为**：交互式项目配置向导

#### `/share`
- **功能**：分享当前会话
- **用途**：内部会话共享

#### `/summary`
- **功能**：生成会话摘要
- **行为**：调用 Claude 总结对话内容

#### `/teleport`
- **功能**：传送到远程环境
- **行为**：将当前会话迁移到远程 CCR 环境

#### `/ultraplan`
- **功能**：多智能体规划（约 30 分钟超时）
- **特性门控**：`ULTRAPLAN`
- **行为**：使用多个智能体进行复杂项目规划

---

## 特性门控命令

以下命令需要特定 Feature Flag 启用：

| 命令 | 特性门控 | 说明 |
|------|---------|------|
| `/brief` | `KAIROS` / `KAIROS_BRIEF` | 简报生成 |
| `/assistant` | `KAIROS` | 助手智能体 |
| `/bridge` | `BRIDGE_MODE` | 远程控制桥接 |
| `/remote-control-server` | `DAEMON` + `BRIDGE_MODE` | 远程服务器 |
| `/voice` | `VOICE_MODE` | 语音模式切换 |
| `/workflows` | `WORKFLOW_SCRIPTS` | 工作流脚本管理 |
| `/remote-setup` | `CCR_REMOTE_SETUP` | 远程配置 |
| `/torch` | `TORCH` | Torch 模式 |
| `/peers` | `UDS_INBOX` | Peer/Inbox 管理 |
| `/fork` | `FORK_SUBAGENT` | 分叉子智能体 |
| `/buddy` | `BUDDY` | Buddy 伴侣智能体 |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | GitHub PR Webhook 订阅 |

---

## 使用示例

```bash
# 状态与信息
/help                       # 显示命令帮助
/status                     # 检查 Claude Code 状态
/cost                       # 显示会话费用

# 配置管理
/model claude-3-7-sonnet    # 切换模型
/effort max                 # 设置努力程度
/theme dark                 # 更改主题
/config                     # 打开配置面板

# 会话管理
/clear                      # 清除历史
/compact                    # 压缩并保留摘要
/plan                       # 启用计划模式
/resume search-term         # 恢复对话
/export output.md           # 导出对话

# 开发工作
/review <pr-number>         # 审查 GitHub PR
/security-review            # 安全审查
/commit                     # AI 辅助 Git 提交

# 集成与扩展
/skills                     # 列出技能
/plugin                     # 浏览插件
/mcp enable server-name     # 启用 MCP 服务器
```

# claude-code-sourcemap

[![linux.do](https://img.shields.io/badge/linux.do-huo0-blue?logo=linux&logoColor=white)](https://linux.do)

> [!WARNING]
> This repository is **unofficial** and is reconstructed from the public npm package and source map analysis, **for research purposes only**.
> It does **not** represent the original internal development repository structure.
>
> 本仓库为**非官方**整理版，基于公开 npm 发布包与 source map 分析还原，**仅供研究使用**。
> **不代表**官方原始内部开发仓库结构。
> 一切基于L站"飘然与我同"的情报提供

## 分析报告预览

![Claude Code v2.1.88 源码工程设计分析](https://github.com/user-attachments/assets/92570375-ab1a-461c-a14a-8aed6779e734)

## 概述

本仓库通过 npm 发布包（`@anthropic-ai/claude-code`）内附带的 source map（`cli.js.map`）还原的 TypeScript 源码，版本为 `2.1.88`。

## 来源

- npm 包：[@anthropic-ai/claude-code](https://www.npmjs.com/package/@anthropic-ai/claude-code)
- 还原版本：`2.1.88`
- 还原文件数：**4756 个**（含 1884 个 `.ts`/`.tsx` 源文件）
- 还原方式：提取 `cli.js.map` 中的 `sourcesContent` 字段

## 目录结构

```
claude-code-source-analyze/
├── restored-src/src/         # 还原的 TypeScript 源码
│   ├── main.tsx              # CLI 入口
│   ├── tools/                # 工具实现（Bash、FileEdit、Grep、MCP 等 43+ 个）
│   ├── commands/             # 命令实现（commit、review、config 等 86+ 个）
│   ├── services/             # API、MCP、分析等服务（21+ 个核心服务）
│   ├── utils/                # 工具函数（git、model、auth、env 等）
│   ├── context/              # React Context
│   ├── coordinator/          # 多 Agent 协调模式
│   ├── assistant/            # 助手模式（KAIROS）
│   ├── buddy/                # AI 伴侣 UI
│   ├── remote/               # 远程会话
│   ├── plugins/              # 插件系统
│   ├── skills/               # 技能系统
│   ├── voice/                # 语音交互
│   └── vim/                  # Vim 模式（8 种运行模式）
│
├── docs/                     # 架构与设计分析文档（Markdown）
│   ├── ARCHITECTURE.md       # 整体架构概览
│   ├── MODULES.md            # 各目录模块详解
│   ├── COMMANDS.md           # 86+ 条命令分析
│   ├── TOOLS.md              # 43+ 个 Agent 工具分析
│   ├── CORE_FLOW_ANALYSIS.md # 核心流程分析
│   ├── SERVICES_ANALYSIS.md  # 服务层分析
│   ├── AGENT_DESIGN.md       # Agent 设计模式
│   ├── AGENT_DESIGN_DEEP_DIVE.md  # Agent 深度剖析
│   └── DESIGN_DEEP_DIVE.md   # 整体设计深度剖析
│
└── reports/                  # 可视化 HTML 分析报告
    ├── index.html            # 总览（技术栈、统计数据、系统架构）
    ├── architecture.html     # 架构层次详细报告
    ├── tools-commands.html   # 工具与命令报告
    ├── agent-design.html     # Agent 设计报告
    ├── prompt-context.html   # Prompt 与上下文报告
    └── deep-analysis.html    # 深度分析报告
```

## 声明

- 源码版权归 [Anthropic](https://www.anthropic.com) 所有
- 本仓库仅用于技术研究与学习，请勿用于商业用途
- 如有侵权，请联系删除

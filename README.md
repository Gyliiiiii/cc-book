# Claude Code 源码解析

> Claude Code 是 Anthropic 开发的基于终端的 AI 代理编程工具（Agentic Coding Tool）。本书深入解析其内部架构与实现细节。

## 项目定位

Claude Code 运行在用户终端中，通过自然语言理解代码库并辅助完成软件开发任务。核心设计思想是将 LLM 推理能力与系统级工具访问相结合，通过工具调用架构实现代码编辑、命令执行、Git 操作等自动化工作流。

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Extension Layer                          │
│         Plugins · Skills · Hooks · MCP Servers              │
├─────────────────────────────────────────────────────────────┤
│                    Tool System                              │
│    BashTool · FileRead/Write/Edit · WebFetch · TaskTool     │
│              Permission Layer (allow/ask/deny)              │
├─────────────────────────────────────────────────────────────┤
│                    Agent Core                               │
│      Main Agent · Subagents · Context Manager · Compaction  │
├─────────────────────────────────────────────────────────────┤
│                    Session Layer                            │
│     Session Store · Transcript (JSONL) · Resume · Recovery  │
├─────────────────────────────────────────────────────────────┤
│                    Entry Layer                              │
│          CLI Binary · REPL · Slash Commands · Flags         │
└─────────────────────────────────────────────────────────────┘
```

## 八大核心子系统

| # | 子系统 | 描述 |
|---|--------|------|
| 1 | [代理系统](03-agent-system.md) | 主代理/子代理编排、协调器模式、6 种任务类型 |
| 2 | [工具系统](04-tool-system.md) | 40+ 工具注册、执行与 7 种权限模式 |
| 3 | [上下文管理](05-context-management.md) | Token 追踪、五层压缩策略、自动压缩 |
| 4 | [钩子系统](06-hook-system.md) | 27 个生命周期事件、4 种钩子类型 |
| 5 | [MCP 集成](07-mcp-integration.md) | 6 种传输协议、7 个配置范围、OAuth 2.0 |
| 6 | [插件系统](08-plugin-system.md) | 7 种组件、7 种安装源、防冒充保护 |
| 7 | [技能系统](09-skill-system.md) | SKILL.md 格式、3 级渐进式发现 |
| 8 | [沙箱环境](10-sandbox.md) | BashTool 进程隔离、网络/文件系统控制 |

## 文档目录

| 文档 | 内容 |
|------|------|
| [00-overview.md](00-overview.md) | 项目架构总览 |
| [01-entry-layer.md](01-entry-layer.md) | 入口层：CLI/REPL/命令解析 |
| [02-session-management.md](02-session-management.md) | 会话管理：持久化/恢复/存储 |
| [03-agent-system.md](03-agent-system.md) | 代理系统：主代理/子代理/调度 |
| [04-tool-system.md](04-tool-system.md) | 工具系统：注册/执行/权限 |
| [05-context-management.md](05-context-management.md) | 上下文管理：Token 追踪/压缩 |
| [06-hook-system.md](06-hook-system.md) | 钩子系统：生命周期事件 |
| [07-mcp-integration.md](07-mcp-integration.md) | MCP 集成：外部工具服务器 |
| [08-plugin-system.md](08-plugin-system.md) | 插件系统：发现/加载/管理 |
| [09-skill-system.md](09-skill-system.md) | 技能系统：知识包管理 |
| [10-sandbox.md](10-sandbox.md) | 沙箱环境：进程隔离 |
| [11-ui-terminal.md](11-ui-terminal.md) | UI/终端：渲染/输入/集成 |
| [12-configuration.md](12-configuration.md) | 配置管理：层叠设置体系 |
| [13-github-automation.md](13-github-automation.md) | GitHub 自动化：CI/CD 集成 |
| [14-query-engine.md](14-query-engine.md) | 查询引擎：模型调用链 |

## 技术栈

| 领域 | 技术选型 |
|------|----------|
| 语言 | TypeScript / TSX |
| 运行时 | Node.js |
| 终端 UI | React + Ink (自定义终端渲染引擎) |
| AI 模型 | Claude 3.5 / 4.5 / 4.6 (Sonnet / Opus) |
| 协议 | MCP (Model Context Protocol) |
| 原生模块 | N-API (Rust/C++ via vendor/) |
| 持久化 | JSONL (会话) / JSON (配置) / Markdown (记忆) |

## 两大使用场景

1. **交互式开发工具** — 开发者在终端中与 AI 对话，完成编码、调试、重构等任务
2. **GitHub 自动化平台** — 通过 GitHub Actions 实现 Issue 分类、去重、生命周期管理、CI 集成

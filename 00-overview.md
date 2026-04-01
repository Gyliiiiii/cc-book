# Claude Code 项目架构总览

## 1. 项目定位

Claude Code 是 Anthropic 开发的基于终端的 AI 代理编程工具（Agentic Coding Tool）。它运行在用户终端中，通过自然语言理解代码库并辅助完成软件开发任务。核心设计思想是将 LLM 推理能力与系统级工具访问相结合，通过工具调用架构实现代码编辑、命令执行、Git 操作等自动化工作流。

## 2. 完整系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     用户交互界面                                             │
│                                                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌───────────┐  ┌─────────────┐  │
│  │ CLI 终端  │  │ VS Code  │  │ JetBrains│  │  Web App   │  │ Desktop   │  │ Deep Link   │  │
│  │ (REPL)   │  │ Extension│  │ Extension│  │ claude.ai  │  │ Mac/Win   │  │claude-cli://│  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘  └─────┬─────┘  └──────┬──────┘  │
│       └──────────────┴──────────────┴──────────────┴──────────────┴───────────────┘          │
└───────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────────────────────────────────────────┐
│                              入口层 (Entry Layer)                          [01-entry-layer]  │
│                                                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ cli.tsx     │  │ init.ts      │  │ setup.ts      │  │commands.ts│  │ replLauncher.tsx │  │
│  │ 入口分发    │─▶│ 环境初始化   │─▶│ 认证/配置     │─▶│ 命令路由  │─▶│ REPL 启动       │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  └──────────┘  └──────────────────┘  │
│                                                                                             │
│  6 种交互模式: 交互式REPL │ 单次执行(-p) │ 管道模式 │ MCP服务器 │ SDK模式 │ 指引模式         │
│  快速路径分发: version/mcp/print → 跳过初始化直接执行                                        │
└───────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────────────────────────────────────────┐
│                           会话层 (Session Layer)                    [02-session-management]  │
│                                                                                             │
│  ┌─────────────────┐  ┌─────────────────────┐  ┌────────────────┐  ┌─────────────────────┐ │
│  │ 会话存储        │  │ 写入缓冲            │  │ 崩溃恢复       │  │ 会话发现/恢复       │ │
│  │ ~/.claude/      │  │ 100ms 刷新          │  │ PID 追踪       │  │ 64KB lite 加载      │ │
│  │ projects/       │  │ 100MB 分块          │  │ 3 种中断类型   │  │ --resume / --continue│ │
│  │ <cwd>/<uuid>    │  │ 有序 JSONL          │  │ 自动恢复       │  │ 会话选择器 Picker   │ │
│  │ .jsonl          │  └─────────────────────┘  └────────────────┘  └─────────────────────┘ │
│  └─────────────────┘                                                                       │
└───────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────────────────────────────────────────┐
│                          代理核心 (Agent Core)                                               │
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │                        查询引擎 (QueryEngine)                      [14-query-engine]    │ │
│  │                                                                                         │ │
│  │  submitMessage() ──▶ query() ──▶ queryLoop() { while(true) }                            │ │
│  │                                                                                         │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │ │
│  │  │ 技能预取 │─▶│ 多层压缩 │─▶│ API 调用 │─▶│ 流式处理 │─▶│ 工具执行 │─▶│ Stop Hooks │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │ │
│  │                                                                                         │ │
│  │  恢复策略: 413→排空折叠 │ 8K→64K升级 │ Fallback切换 │ <90%预算→继续 │ 流式中止→消费    │ │
│  │  依赖注入: callModel │ microcompact │ autocompact │ uuid                                │ │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                             │
│  ┌──────────────────────────────────┐  ┌───────────────────────────────────────────────────┐ │
│  │   代理系统 [03-agent-system]     │  │  上下文管理 [05-context-management]                │ │
│  │                                  │  │                                                   │ │
│  │  ┌────────────┐ ┌─────────────┐  │  │  ┌─────────────┐  ┌───────────────────────────┐  │ │
│  │  │ 主代理     │ │ 子代理(Fork)│  │  │  │ Token 追踪  │  │ 五层压缩策略              │  │ │
│  │  │ Main Agent │ │ 缓存优化    │  │  │  │ API实际值   │  │ Micro → TimeBased →       │  │ │
│  │  └────────────┘ └─────────────┘  │  │  │ + 字符估算  │  │ APISide → SessionMemory → │  │ │
│  │  ┌────────────┐ ┌─────────────┐  │  │  └─────────────┘  │ Full                      │  │ │
│  │  │ 协调器模式 │ │ 6 种任务类型│  │  │  ┌─────────────┐  └───────────────────────────┘  │ │
│  │  │ Coordinator│ │ local_bash  │  │  │  │ 自动压缩    │                                 │ │
│  │  │ 4 阶段流程 │ │ local_agent │  │  │  │ ≈93% 触发   │  9 节摘要模板                   │ │
│  │  └────────────┘ │ remote_agent│  │  │  │ effective-  │  压缩后状态保留:                │ │
│  │                 │ in_process  │  │  │  │ 13K 阈值    │  files/skills/plan/hooks        │ │
│  │  Agent 定义:    │ workflow    │  │  │  └─────────────┘                                 │ │
│  │  YAML + MD      │ monitor_mcp│  │  │                                                   │ │
│  │  6 级加载优先级 └─────────────┘  │  │  Token 预算: 90%阈值 + 递减回报检测               │ │
│  └──────────────────────────────────┘  └───────────────────────────────────────────────────┘ │
└───────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────────────────────────────────────────┐
│                            工具系统 (Tool System)                       [04-tool-system]     │
│                                                                                             │
│  ┌──── 文件操作 ────┐  ┌──── Shell ─────┐  ┌──── 搜索 ─────┐  ┌──── 代理/任务 ───────────┐ │
│  │ FileReadTool     │  │ BashTool       │  │ GlobTool      │  │ AgentTool               │ │
│  │ FileWriteTool    │  │  ┌───────────┐ │  │ GrepTool      │  │ TaskTool (Create/Update) │ │
│  │ FileEditTool     │  │  │ 沙箱隔离  │ │  │ WebSearchTool │  │ BackgroundTaskTool      │ │
│  │ NotebookEditTool │  │  │[10-sandbox]│ │  │ WebFetchTool  │  └─────────────────────────┘ │
│  └──────────────────┘  │  └───────────┘ │  └───────────────┘                               │
│                        └────────────────┘                                                   │
│  ┌──── 版本控制 ────┐  ┌──── 内存/计划 ──┐  ┌──── MCP ──────┐  ┌──── 特殊 ──────────────┐ │
│  │ GitTool          │  │ MemoryReadTool  │  │ mcp__*        │  │ EnterPlanMode          │ │
│  │                  │  │ MemoryWriteTool │  │ 动态注册      │  │ ExitPlanMode           │ │
│  │                  │  │ TodoRead/Write  │  │ 6 种传输类型  │  │ AskUserQuestion        │ │
│  └──────────────────┘  └────────────────┘  └───────────────┘  │ SkillTool              │ │
│                                                                └────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                          权限系统 (Permission Layer)                                  │  │
│  │                                                                                      │  │
│  │  7 种模式: default │ plan │ acceptEdits │ bypassPermissions │ dontAsk │ auto │ --     │  │
│  │  6 个规则源: managed → user → project → local → SDK → runtime                        │  │
│  │  规则语法: ToolName(argument_pattern)   例: Bash(git *), WebFetch(domain:*.npm.org)   │  │
│  │  3 种结果: allow (静默放行) │ ask (用户确认) │ deny (静默拒绝)                         │  │
│  └──────────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────────────────────────────────────────┐
│                          扩展层 (Extension Layer)                                            │
│                                                                                             │
│  ┌────────────────────────┐  ┌───────────────────────────┐  ┌────────────────────────────┐  │
│  │ 钩子系统               │  │ MCP 集成                  │  │ 插件系统                   │  │
│  │ [06-hook-system]       │  │ [07-mcp-integration]      │  │ [08-plugin-system]         │  │
│  │                        │  │                           │  │                            │  │
│  │ 27 个生命周期事件:     │  │ 6 种传输:                 │  │ 7 种组件:                  │  │
│  │ PreToolUse             │  │ stdio │ SSE │ HTTP        │  │ commands │ agents          │  │
│  │ PostToolUse            │  │ WebSocket │ SDK           │  │ skills │ hooks             │  │
│  │ SessionStart/Stop      │  │ claude.ai proxy           │  │ output-styles              │  │
│  │ Notification           │  │                           │  │ MCP servers │ LSP          │  │
│  │ PostSampling           │  │ 7 个配置范围:             │  │                            │  │
│  │ TaskCompleted          │  │ project │ user │ managed  │  │ 7 种安装源:                │  │
│  │ ...                    │  │ inline │ plugin │ flag    │  │ Marketplace (GitHub/npm/   │  │
│  │                        │  │ session                   │  │ URL/local) │ Git           │  │
│  │ 4 种钩子类型:          │  │                           │  │ session-only │ built-in    │  │
│  │ command (shell)        │  │ 工具命名:                 │  │                            │  │
│  │ prompt (单次LLM)       │  │ mcp__<server>__<tool>     │  │ 防冒充保护                 │  │
│  │ agent (多轮)           │  │                           │  │ 企业策略控制               │  │
│  │ HTTP (POST)            │  │ OAuth 2.0 + PKCE          │  │ 自动下架检测               │  │
│  └────────────────────────┘  └───────────────────────────┘  └────────────────────────────┘  │
│                                                                                             │
│  ┌────────────────────────┐  ┌───────────────────────────────────────────────────────────┐  │
│  │ 技能系统               │  │ 沙箱环境                                                  │  │
│  │ [09-skill-system]      │  │ [10-sandbox]                                              │  │
│  │                        │  │                                                           │  │
│  │ SKILL.md 格式:         │  │ 作用范围: 仅 BashTool                                     │  │
│  │ YAML 前置 + Markdown   │  │                                                           │  │
│  │ 15+ 前置字段           │  │ ┌─────────┐  ┌──────────────┐  ┌───────────────────────┐  │  │
│  │ 变量替换               │  │ │ macOS   │  │ Linux        │  │ 安全硬编码            │  │  │
│  │                        │  │ │sandbox- │  │ bubblewrap + │  │ 拒写: settings.json   │  │  │
│  │ 3 级渐进式发现:        │  │ │exec     │  │ seccomp      │  │ 拒写: managed-settings│  │  │
│  │ Discovery (1%预算)     │  │ └─────────┘  └──────────────┘  │ 拒写: .claude/skills  │  │  │
│  │  → Invocation          │  │                               │ 拒写: Git 哨兵文件    │  │  │
│  │   → Fork Execution    │  │ 网络隔离: 域名白名单           └───────────────────────┘  │  │
│  │                        │  │ 文件系统: 可读/可写路径控制                                │  │
│  │ 7 个加载源             │  │ 排除命令: 便利功能(非安全边界)                             │  │
│  │ realpath 去重          │  │                                                           │  │
│  └────────────────────────┘  └───────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────────────────────────────────────────┐
│                         基础设施层 (Infrastructure)                                          │
│                                                                                             │
│  ┌─────────────────────────┐  ┌──────────────────────────┐  ┌────────────────────────────┐  │
│  │ UI/终端渲染             │  │ 配置管理                 │  │ GitHub 自动化              │  │
│  │ [11-ui-terminal]        │  │ [12-configuration]       │  │ [13-github-automation]     │  │
│  │                         │  │                          │  │                            │  │
│  │ React + Ink 自定义引擎  │  │ 六层层叠配置:            │  │ 六大子系统:                │  │
│  │ ConcurrentRoot 并发渲染 │  │                          │  │                            │  │
│  │ Yoga Flex 布局          │  │ ┌────────────────────┐   │  │ AI Issue 分类              │  │
│  │                         │  │ │ 6. 企业托管(策略)  │   │  │ 语义去重 (5并行Agent)      │  │
│  │ 17 个键绑定上下文       │  │ │  Remote > MDM >    │   │  │ 生命周期管理               │  │
│  │ 100+ 默认快捷键         │  │ │  File > HKCU      │   │  │ @Claude 提及               │  │
│  │ Vim 完整模式            │  │ ├────────────────────┤   │  │ 跨仓库事件分发             │  │
│  │  Normal/Insert/Visual   │  │ │ 5. CLI 标志/SDK    │   │  │ 事件日志(Statsig)          │  │
│  │  motions+operators+     │  │ ├────────────────────┤   │  │                            │  │
│  │  text objects           │  │ │ 4. local (.local)  │   │  │ claude-code-action@v1      │  │
│  │                         │  │ ├────────────────────┤   │  │ 安全 CLI 包装 (gh.sh)      │  │
│  │ 状态栏:                 │  │ │ 3. project         │   │  │ 防跨仓库查询               │  │
│  │  模型│模式│CWD│         │  │ ├────────────────────┤   │  │ 3 天去重宽限期             │  │
│  │  上下文│费用│Vim        │  │ │ 2. user (~/.claude)│   │  │                            │  │
│  │                         │  │ ├────────────────────┤   │  │                            │  │
│  │ VS Code / IDE 桥接      │  │ │ 1. plugin          │   │  │                            │  │
│  │  via MCP SDK            │  │ └────────────────────┘   │  │                            │  │
│  │                         │  │                          │  │                            │  │
│  │ Deep Link 协议          │  │ Zod Schema 验证          │  │                            │  │
│  │ @ 文件自动补全          │  │ 实时文件监控              │  │                            │  │
│  └─────────────────────────┘  └──────────────────────────┘  └────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                              外部依赖与集成                                                  │
│                                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Anthropic API│  │ GitHub API   │  │ MCP Servers  │  │ 企业 MDM     │  │ Statsig      │  │
│  │ Claude Models│  │ Actions/CLI  │  │ (外部)       │  │ plist / HKLM │  │ 分析平台     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 架构图说明

- **用户交互界面** — 6 种接入方式，统一汇入入口层
- **入口层 → 会话层 → 代理核心** — 自上而下的请求生命周期
- **查询引擎** — 核心执行循环，驱动模型调用 → 工具执行 → 压缩 → 恢复的闭环
- **工具系统** — 40+ 工具 + 权限层，所有工具调用必经权限检查
- **扩展层** — 钩子/MCP/插件/技能/沙箱五大扩展机制，提供可定制性
- **基础设施层** — 终端渲染、配置管理、GitHub 自动化三大支撑系统
- **外部依赖** — Anthropic API、GitHub、MCP 外部服务器、企业管理、分析平台

## 3. 分层架构

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

### 各层职责

| 层级 | 职责 | 核心文件/目录 |
|------|------|---------------|
| Entry Layer | CLI 入口、参数解析、路由到子命令或交互会话 | `src/entrypoints/`, `src/cli/`, `src/commands.ts` |
| Session Layer | 对话持久化、会话恢复、JSONL 存储 | `src/history.ts`, `src/services/` |
| Agent Core | 多代理编排、上下文管理、模型调用 | `src/query.ts`, `src/QueryEngine.ts`, `src/Task.ts`, `src/context/` |
| Tool System | 工具定义、注册、执行、权限验证 | `src/Tool.ts`, `src/tools.ts`, `src/tools/` |
| Extension Layer | 插件、技能、钩子、MCP 外部工具服务器 | `src/plugins/`, `src/skills/`, `src/hooks/`, `src/bridge/` |

## 3. 核心数据流

```
用户输入
  │
  ▼
InputParser (src/cli/)
  │
  ▼
SessionContext (src/context/)
  │
  ▼
AgentExecutor (src/QueryEngine.ts)
  │
  ├──► 模型推理 → 选择工具
  │
  ▼
PermissionChecker (managed → user → project → local)
  │
  ├── deny → 拒绝执行
  ├── ask  → 提示用户授权
  └── allow → 放行
        │
        ▼
  PreToolUse Hook (src/hooks/)
        │
        ▼
  工具执行 (src/tools/)
        │
        ▼
  PostToolUse Hook
        │
        ▼
  ContextManager (token 检查)
        │
        ├── ≥98% → 触发自动压缩
        └── <98% → 正常继续
              │
              ▼
        响应输出 → 用户
```

## 4. 八大核心子系统

| # | 子系统 | 对应文档 | 核心源码 |
|---|--------|----------|----------|
| 1 | 代理系统 | [03-agent-system.md](03-agent-system.md) | `src/Task.ts`, `src/QueryEngine.ts`, `src/coordinator/` |
| 2 | 工具系统 | [04-tool-system.md](04-tool-system.md) | `src/Tool.ts`, `src/tools.ts`, `src/tools/` (184 files) |
| 3 | 上下文管理 | [05-context-management.md](05-context-management.md) | `src/context/` (9 files), `src/query.ts` |
| 4 | 钩子系统 | [06-hook-system.md](06-hook-system.md) | `src/hooks/` (104 files) |
| 5 | MCP 集成 | [07-mcp-integration.md](07-mcp-integration.md) | `src/bridge/` (31 files) |
| 6 | 插件系统 | [08-plugin-system.md](08-plugin-system.md) | `src/plugins/` (2 files), `src/commands/` |
| 7 | 技能系统 | [09-skill-system.md](09-skill-system.md) | `src/skills/` (20 files) |
| 8 | 沙箱环境 | [10-sandbox.md](10-sandbox.md) | `src/tools/` (BashTool 相关) |

## 5. 源码目录结构

```
claudecode/
├── src/                          # 主源码 (TypeScript/TSX)
│   ├── entrypoints/     (8)      # CLI 入口点
│   ├── cli/            (19)      # 命令行解析与路由
│   ├── commands/      (207)      # 命令实现（斜杠命令等）
│   ├── components/    (389)      # UI 组件 (React/Ink)
│   ├── tools/         (184)      # 工具定义与实现
│   ├── hooks/         (104)      # 钩子系统
│   ├── services/      (130)      # 核心服务层（API、认证等）
│   ├── utils/         (564)      # 工具函数库
│   ├── ink/            (96)      # 终端渲染引擎
│   ├── bridge/         (31)      # MCP 桥接层
│   ├── constants/      (21)      # 常量定义
│   ├── skills/         (20)      # 技能加载与管理
│   ├── keybindings/    (14)      # 快捷键系统
│   ├── tasks/          (12)      # 任务管理
│   ├── types/          (11)      # TypeScript 类型定义
│   ├── migrations/     (11)      # 数据迁移
│   ├── context/         (9)      # 上下文窗口管理
│   ├── memdir/          (8)      # 记忆目录管理
│   ├── buddy/           (6)      # Buddy 功能
│   ├── state/           (6)      # 状态管理
│   ├── vim/             (5)      # Vim 模式
│   ├── query/           (4)      # 查询子模块
│   ├── remote/          (4)      # 远程控制
│   ├── native-ts/       (4)      # 原生 TS 模块
│   ├── screens/         (3)      # 屏幕/页面
│   ├── server/          (3)      # 服务端
│   ├── plugins/         (2)      # 插件系统入口
│   ├── upstreamproxy/   (2)      # 上游代理
│   ├── coordinator/     (1)      # 协调器
│   ├── assistant/       (1)      # 助手模块
│   ├── bootstrap/       (1)      # 引导启动
│   ├── moreright/       (1)      # 扩展权限
│   ├── outputStyles/    (1)      # 输出风格
│   ├── schemas/         (1)      # Schema 定义
│   └── voice/           (1)      # 语音支持
│   ├── main.tsx               # 主入口 (804KB)
│   ├── Tool.ts                # 工具基类
│   ├── Task.ts                # 任务/子代理基类
│   ├── QueryEngine.ts         # 查询引擎核心
│   ├── query.ts               # 查询逻辑
│   ├── commands.ts            # 命令注册
│   ├── context.ts             # 上下文构建
│   ├── tools.ts               # 工具注册表
│   ├── history.ts             # 历史记录
│   ├── setup.ts               # 初始化设置
│   ├── interactiveHelpers.tsx # 交互辅助 (57KB)
│   ├── dialogLaunchers.tsx    # 对话启动器
│   ├── replLauncher.tsx       # REPL 启动器
│   ├── cost-tracker.ts        # 费用追踪
│   ├── costHook.ts            # 费用钩子
│   ├── ink.ts                 # Ink 渲染入口
│   ├── tasks.ts               # 任务集合
│   └── projectOnboardingState.ts # 项目入门状态
├── vendor/                     # 原生依赖
│   ├── audio-capture-src/     # 音频捕获 (语音功能)
│   ├── image-processor-src/   # 图片处理
│   ├── modifiers-napi-src/    # Native 键盘修饰符
│   └── url-handler-src/       # URL 协议处理 (deep link)
```

## 6. 技术栈

| 领域 | 技术选型 |
|------|----------|
| 语言 | TypeScript / TSX |
| 运行时 | Node.js |
| 终端 UI | React + Ink (自定义终端渲染引擎) |
| AI 模型 | Claude 3.5 / 4.5 / 4.6 (Sonnet / Opus) |
| 协议 | MCP (Model Context Protocol) |
| 原生模块 | N-API (Rust/C++ via vendor/) |
| 持久化 | JSONL (会话) / JSON (配置) / Markdown (记忆) |

## 7. 两大使用场景

1. **交互式开发工具** — 开发者在终端中与 AI 对话，完成编码、调试、重构等任务
2. **GitHub 自动化平台** — 通过 GitHub Actions 实现 Issue 分类、去重、生命周期管理、CI 集成

## 8. 文档索引

| 文档 | 描述 |
|------|------|
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

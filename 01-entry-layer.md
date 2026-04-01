# 01 - 入口层（Entry Layer）

## 1. 模块概述

入口层是 Claude Code 的最外层，负责接收用户的 CLI 调用，完成参数解析、环境初始化，并路由到对应的执行模式（交互式 REPL、MCP 服务器、桥接模式等）。设计核心原则是**延迟加载**——快速路径（如 `--version`）零模块加载即可返回，重量级模块仅在实际需要时动态导入。

## 2. 核心文件结构

```
src/
├── entrypoints/
│   ├── cli.tsx              # (39KB) 真正的入口，main() 函数
│   ├── init.ts              # (14KB) 初始化编排器（memoized）
│   ├── mcp.ts               # (6KB)  MCP 服务器入口
│   ├── agentSdkTypes.ts     # (13KB) Agent SDK 公共 API 类型
│   ├── sandboxTypes.ts      # (6KB)  沙箱配置 Zod Schema
│   └── sdk/
│       ├── controlSchemas.ts  # (20KB) SDK 控制协议 Schema
│       ├── coreSchemas.ts     # (56KB) 核心序列化 Schema
│       └── coreTypes.ts       # (1.5KB) 核心类型定义
├── cli/
│   ├── exit.ts              # CLI 退出辅助函数
│   ├── print.ts             # (213KB) 终端渲染与输出格式化
│   ├── structuredIO.ts      # (29KB) 结构化 I/O 基类
│   ├── remoteIO.ts          # (10KB) 远程环境 I/O
│   ├── update.ts            # (14KB) 自更新逻辑
│   ├── ndjsonSafeStringify.ts # NDJSON 安全序列化
│   ├── handlers/
│   │   ├── agents.ts        # Agent 相关 CLI 处理器
│   │   ├── auth.ts          # 认证流处理器
│   │   ├── autoMode.ts      # 自动/非交互模式
│   │   ├── mcp.tsx          # (56KB) MCP 子命令处理器
│   │   ├── plugins.ts       # (31KB) 插件管理处理器
│   │   └── util.tsx         # 共享工具函数
│   └── transports/
│       ├── ccrClient.ts     # (34KB) Claude Code Remote 客户端
│       ├── HybridTransport.ts # 混合传输层
│       ├── SSETransport.ts   # (24KB) SSE 传输
│       ├── WebSocketTransport.ts # (28KB) WebSocket 传输
│       ├── SerialBatchEventUploader.ts # 批量事件上传
│       ├── WorkerStateUploader.ts # Worker 状态上传
│       └── transportUtils.ts # 传输选择工具
├── commands.ts              # (25KB) 命令注册表
├── replLauncher.tsx         # (3.5KB) REPL 启动器
└── setup.ts                 # (21KB) 会话级初始化
```

## 3. 启动流程

```
用户输入 "claude [args]"
       │
       ▼
  entrypoints/cli.tsx :: main()
       │
       ├─ 快速路径检查（零加载即返回）
       │   ├── --version / -v        → 打印版本号，立即退出
       │   ├── --dump-system-prompt  → 渲染并打印 system prompt
       │   ├── --claude-in-chrome-mcp → 启动 Chrome MCP 服务器模式
       │   ├── --chrome-native-host  → 启动 Chrome 原生宿主
       │   ├── --computer-use-mcp    → 启动 Computer Use MCP（feature-gated）
       │   ├── --daemon-worker=<kind> → 启动精简 Daemon Worker
       │   └── remote-control/rc/bridge → 桥接模式（feature-gated）
       │
       ▼ （无快速路径匹配）
  entrypoints/init.ts :: init()     ← memoized，全局仅执行一次
       │
       ├── enableConfigs()                    — 验证并启用配置系统
       ├── applySafeConfigEnvironmentVariables() — 安全环境变量
       ├── applyExtraCACertsFromConfig()      — TLS 证书（必须在首次 TLS 之前）
       ├── setupGracefulShutdown()            — 注册退出处理器
       ├── ┌ 异步后台任务（非阻塞）：
       │   ├── initialize1PEventLogging()     — 一方分析
       │   ├── populateOAuthAccountInfoIfNeeded() — OAuth 信息缓存
       │   ├── initJetBrainsDetection()       — IDE 检测
       │   ├── detectCurrentRepository()      — GitHub 仓库检测
       │   ├── initializeRemoteManagedSettingsLoadingPromise()
       │   └── initializePolicyLimitsLoadingPromise()
       ├── recordFirstStartTime()             — 记录首次启动时间
       ├── configureGlobalMTLS()              — 双向 TLS
       └── configureGlobalAgents()            — HTTP 代理配置
       │
       ▼
  setup.ts :: setup()               ← 每次会话执行
       │
       ├── Node.js 版本检查（≥ 18）
       ├── 自定义 Session ID 赋值
       ├── UDS (Unix Domain Socket) 消息服务器启动
       ├── Worktree 创建 + tmux 会话管理
       ├── 项目根目录检测 + Git 根目录发现
       ├── 会话记忆初始化
       ├── Hook 配置快照
       ├── 文件变更监视器初始化
       ├── Release Notes 检查
       ├── API Key 预获取
       └── 分析 Sink 初始化
       │
       ▼
  commands.ts :: getCommands()      ← 命令注册表
       │
       ▼
  replLauncher.tsx :: launchRepl()  ← React/Ink 挂载
       │
       ├── 动态导入 App 组件
       ├── 动态导入 REPL 组件
       └── 渲染 <App><REPL /></App> 到 Ink Root
       │
       ▼
  用户进入交互式 REPL
```

## 4. 核心组件详解

### 4.1 cli.tsx — 真正的入口

**文件**: `src/entrypoints/cli.tsx` (39KB)

这是 `claude` 二进制文件执行的第一个模块。核心设计是**快速路径分发模式**：

```typescript
async function main(): Promise<void> {
  // 快速路径：检查 process.argv 中的特殊标志
  // 匹配则短路返回，不加载任何重量级模块

  if (hasFlag('--version')) {
    console.log(MACRO.VERSION);
    return;
  }

  // ... 其他快速路径

  // 所有导入均为动态导入 await import(...)
  // 最大化减少非必要模块的求值开销
}
```

**关键设计决策**:
- 所有模块导入均为 `await import(...)` 动态导入
- 顶层仅设置环境变量（`COREPACK_ENABLE_AUTO_PIN=0`、Node 堆大小等）
- 快速路径命中率高的分支排在最前

### 4.2 init.ts — 初始化编排器

**文件**: `src/entrypoints/init.ts` (14KB)

使用 `memoize` 包装确保全局只执行一次。连接几乎所有子系统：分析、认证、配置、代理、TLS、遥测、优雅退出。

**关键特性**:
- 初始化顺序有严格依赖：TLS 证书必须在首次网络请求之前配置
- 后台任务（OAuth、IDE 检测等）采用 fire-and-forget 模式，不阻塞主流程

### 4.3 commands.ts — 命令注册表

**文件**: `src/commands.ts` (25KB)

所有斜杠命令的**中央注册中心**。导入约 50+ 命令模块：

```
始终可用的命令:
  /clear, /compact, /config, /commit, /help, /login, /logout,
  /mcp, /review, /share, /status, /tasks, /memory, /model ...

Feature-gated 命令 (通过 bun:bundle 的 feature() 控制):
  PROACTIVE / KAIROS    → /proactive, /brief, /assistant
  BRIDGE_MODE           → /bridge, /remoteControlServer
  VOICE_MODE            → /voice
  HISTORY_SNIP          → /forceSnip
  WORKFLOW_SCRIPTS      → /workflows
  CCR_REMOTE_SETUP      → /web
  EXPERIMENTAL_SKILL_SEARCH → /clearSkillIndexCache
```

### 4.4 replLauncher.tsx — REPL 启动器

**文件**: `src/replLauncher.tsx` (3.5KB)

极简的桥接模块，职责单一：

```typescript
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js');    // 延迟加载
  const { REPL } = await import('./screens/REPL.js');     // 延迟加载
  renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>);
}
```

延迟导入确保非交互式路径不会加载 React/Ink 渲染树。

### 4.5 setup.ts — 会话级初始化

**文件**: `src/setup.ts` (21KB)

在 `init()` 之后、REPL 启动之前运行。每个会话执行一次，处理：
- Worktree 和 tmux 会话管理
- 项目根目录和 Git 根目录发现
- Hook 配置快照保存
- 文件变更监视器启动

## 5. 交互模式

| 模式 | 触发方式 | 描述 |
|------|----------|------|
| **交互式 REPL** | `claude`（默认） | 流式输出 + 工具调用 + 自动模式 |
| **Bash 模式** | `!` 前缀 | 直接执行 shell 命令 |
| **Print/Headless** | `-p` 标志 | CI/CD 非交互模式，可与 `--bare` 组合 |
| **Remote Control** | `remote-control` 子命令 | 桥接到 claude.ai/code Web 界面 |
| **MCP 服务器** | `--mcp` 标志 | 将 Claude Code 工具暴露为 MCP 服务 |
| **Daemon Worker** | `--daemon-worker=<kind>` | 精简后台工作进程 |

## 6. I/O 与传输层

`src/cli/` 目录提供了灵活的 I/O 抽象：

```
              StructuredIO (基类)
                  │
          ┌───────┴───────┐
          │               │
    标准 stdin/stdout   RemoteIO
                          │
                    ┌─────┼─────────┐
                    │     │         │
                  SSE  WebSocket   CCR
```

- **StructuredIO** — NDJSON 解析、消息分发、权限提示转发、Hook 执行转发
- **RemoteIO** — 继承 StructuredIO，增加动态 Token 刷新、Keep-alive、传输选择
- **传输层** — SSE / WebSocket / CCR (Claude Code Remote) 三种传输协议

## 7. 设计要点

1. **零开销快速路径**: `--version` 等命令不加载任何模块
2. **动态导入**: 所有重量级模块使用 `await import()`
3. **Memoized 初始化**: `init()` 保证全局只执行一次
4. **Feature Gate**: 实验性功能通过编译时特性开关控制
5. **关注点分离**: 全局初始化(init) / 会话初始化(setup) / UI 挂载(replLauncher) 三阶段清晰分离

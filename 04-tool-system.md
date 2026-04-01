# 04 - 工具系统与权限（Tool System & Permissions）

## 1. 模块概述

工具系统是 Claude Code 与外部世界交互的唯一通道。所有操作（文件读写、命令执行、网络请求、子代理生成）都通过类型化的工具接口完成，并经过多层权限验证。系统采用 "Tool 接口 + 注册表 + 权限三层体系" 的设计，兼顾灵活性与安全性。

## 2. 核心文件结构

```
src/
├── Tool.ts                          # (29KB) Tool 接口定义 + buildTool 工厂
├── tools.ts                         # (17KB) 工具注册表
├── tools/                           # (184 files) 工具实现
│   ├── BashTool/                    # Shell 执行
│   ├── FileReadTool/                # 文件读取
│   ├── FileEditTool/                # 文件编辑
│   ├── FileWriteTool/               # 文件写入
│   ├── GlobTool/                    # 文件模式匹配
│   ├── GrepTool/                    # 内容搜索
│   ├── WebFetchTool/                # Web 抓取
│   ├── WebSearchTool/               # Web 搜索
│   ├── AgentTool/                   # 子代理生成
│   ├── MCPTool/                     # MCP 工具包装器
│   ├── AskUserQuestionTool/         # 用户交互
│   ├── EnterPlanModeTool/           # Plan 模式
│   ├── ScheduleCronTool/            # 定时任务
│   ├── shared/                      # 共享工具函数
│   └── utils.ts                     # 工具辅助函数
├── utils/permissions/               # (23 files) 权限系统
│   ├── permissions.ts               # 权限决策核心
│   ├── permissionRuleParser.ts      # 规则解析器
│   ├── PermissionMode.ts            # 权限模式定义
│   ├── dangerousPatterns.ts         # 危险命令模式
│   ├── filesystem.ts                # 文件系统保护
│   ├── denialTracking.ts            # 拒绝追踪
│   └── bashClassifier.ts            # Bash 分类器
└── types/
    └── permissions.ts               # 权限类型定义
```

## 3. Tool 接口

**文件**: `src/Tool.ts`

Tool 是一个泛型接口（非类），参数化三个类型变量：

```typescript
type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
>
```

### 3.1 核心属性

| 属性 | 类型 | 描述 |
|------|------|------|
| `name` | `string` (readonly) | 主要工具标识符 |
| `aliases` | `string[]?` | 向后兼容的旧名称 |
| `searchHint` | `string?` | 3-10 词短语，用于 ToolSearch 关键词匹配 |
| `inputSchema` | `ZodSchema` (readonly) | Zod 输入验证 Schema |
| `inputJSONSchema` | `object?` | JSON Schema 格式（MCP 工具使用） |
| `outputSchema` | `ZodSchema?` | 输出验证 Schema |
| `maxResultSizeChars` | `number` | 大结果持久化到磁盘的阈值（Read 等工具设为 `Infinity`） |
| `strict` | `boolean?` | 启用更严格的 API 指令遵守 |
| `shouldDefer` | `boolean?` | 需要先调用 ToolSearch 才能使用 |
| `alwaysLoad` | `boolean?` | 永不延迟——始终在初始 prompt 中 |
| `isMcp` / `isLsp` | `boolean?` | MCP/LSP 工具标志 |

### 3.2 核心方法

| 方法 | 参数 | 描述 |
|------|------|------|
| `call()` | `args, context, canUseTool, parentMessage, onProgress?` | 执行工具 |
| `description()` | `input, options` | 生成人类可读描述（权限提示中显示） |
| `validateInput()` | `input, context` | 执行前验证，返回 `{result: true}` 或 `{result: false, message}` |
| `checkPermissions()` | `input, context` | 工具特定权限逻辑，返回 `allow/deny/ask` |
| `preparePermissionMatcher()` | `input` | 创建 Hook `if` 条件模式匹配闭包 |
| `isReadOnly()` | `input` | 此次调用是否只读 |
| `isDestructive()` | `input` | 是否不可逆操作（删除、覆盖、发送） |
| `isConcurrencySafe()` | `input` | 是否可并发执行 |
| `isEnabled()` | — | 工具当前是否可用 |
| `interruptBehavior()` | — | 用户中断时行为：`'cancel'` 或 `'block'` |
| `prompt()` | `options` | 返回此工具注入的系统提示文本 |
| `toAutoClassifierInput()` | `input` | 用于 auto-mode 安全分类器的紧凑表示 |

### 3.3 buildTool 工厂

工具通过 `buildTool(def: ToolDef)` 工厂函数构造，接受普通定义对象，产出冻结的 `Tool` 实例。

## 4. 工具注册表

**文件**: `src/tools.ts`

### 4.1 注册机制

三种注册模式：

```
1. 静态导入 — 始终可用的工具
   import { BashTool } from './tools/BashTool/...'

2. Feature-gated 条件加载 — 通过 feature() 或环境变量控制
   if (feature('VOICE_MODE')) require('./tools/REPLTool/...')

3. 延迟加载 — 通过工厂函数打破循环依赖
   getTeamCreateTool(), getSendMessageTool()
```

### 4.2 核心函数

| 函数 | 描述 |
|------|------|
| `getAllBaseTools()` | 返回当前环境中所有可能的工具 |
| `getTools(permissionContext)` | 按 deny 规则和 `isEnabled()` 过滤 |
| `assembleToolPool(ctx, mcpTools)` | 合并内置 + MCP 工具，去重（内置优先） |
| `filterToolsByDenyRules(tools, ctx)` | 在模型看到之前移除被 deny 的工具 |

## 5. 全部工具分类

### 5.1 核心文件操作

| 工具 | 目录 | 描述 |
|------|------|------|
| `FileReadTool` | `tools/FileReadTool/` | 紧凑行号格式读取，去重未变更的重复读取 |
| `FileEditTool` | `tools/FileEditTool/` | 基于行的编辑，带验证 |
| `FileWriteTool` | `tools/FileWriteTool/` | 文件写入，保留 CRLF/LF，符号链接检查 |
| `NotebookEditTool` | `tools/NotebookEditTool/` | Jupyter Notebook 编辑 |

### 5.2 Shell 执行

| 工具 | 描述 |
|------|------|
| `BashTool` | 主要 shell 执行，支持沙箱 |
| `PowerShellTool` | Windows opt-in 预览，危险命令检测 |
| `REPLTool` | 交互式 REPL（ant-only） |

### 5.3 搜索与发现

| 工具 | 描述 |
|------|------|
| `GlobTool` | 文件模式匹配，按修改时间排序 |
| `GrepTool` | 基于 ripgrep 的内容搜索 |
| `ToolSearchTool` | 延迟加载工具的按需发现 |
| `LSPTool` | Language Server Protocol 集成 |

### 5.4 Web 与网络

| 工具 | 描述 |
|------|------|
| `WebFetchTool` | URL 内容抓取 + AI 处理 |
| `WebSearchTool` | Web 搜索 + 结果摘要 |

### 5.5 代理与任务管理

| 工具 | 描述 |
|------|------|
| `AgentTool` | 生成子代理 |
| `SendMessageTool` | 向现有代理发送消息 |
| `TaskOutputTool` | 获取任务输出 |
| `TaskStopTool` | 停止运行中的任务 |
| `TaskCreateTool` | 创建结构化任务 |
| `TaskGetTool` | 获取任务详情 |
| `TaskUpdateTool` | 更新任务状态 |
| `TaskListTool` | 列出所有任务 |
| `TeamCreateTool` | 创建代理团队 |
| `TeamDeleteTool` | 删除代理团队 |

### 5.6 规划与模式控制

| 工具 | 描述 |
|------|------|
| `EnterPlanModeTool` | 进入规划模式 |
| `ExitPlanModeV2Tool` | 退出规划模式（含审批） |
| `TodoWriteTool` | 写入待办事项 |

### 5.7 MCP 集成

| 工具 | 描述 |
|------|------|
| `MCPTool` | 动态 MCP 工具包装器 |
| `ListMcpResourcesTool` | 列出 MCP 资源 |
| `ReadMcpResourceTool` | 读取 MCP 资源 |
| `McpAuthTool` | MCP 认证 |

### 5.8 用户交互

| 工具 | 描述 |
|------|------|
| `AskUserQuestionTool` | 向用户提问 |
| `SkillTool` | 调用技能 |
| `ConfigTool` | 配置管理（ant-only） |

### 5.9 Worktree 管理

| 工具 | 描述 |
|------|------|
| `EnterWorktreeTool` | 创建并进入 worktree |
| `ExitWorktreeTool` | 退出 worktree |

### 5.10 定时与触发器（feature-gated）

| 工具 | 描述 |
|------|------|
| `CronCreateTool` | 创建定时任务 |
| `CronDeleteTool` | 删除定时任务 |
| `CronListTool` | 列出定时任务 |
| `RemoteTriggerTool` | 远程触发器管理 |
| `SleepTool` | 休眠等待 |

### 5.11 特殊/内部工具

| 工具 | 条件 |
|------|------|
| `WebBrowserTool` | feature-gated |
| `SnipTool` | feature-gated |
| `MonitorTool` | feature-gated |
| `TerminalCaptureTool` | feature-gated |
| `PushNotificationTool` | KAIROS |
| `SubscribePRTool` | KAIROS |

## 6. 权限系统

### 6.1 权限模式（全局行为预设）

| 模式 | 行为 |
|------|------|
| `default` | 正常模式——非只读操作需要权限 |
| `plan` | 规划模式——只读，写入和执行需审批 |
| `acceptEdits` | 自动接受文件编辑，命令仍需询问 |
| `bypassPermissions` | 跳过所有权限提示（危险） |
| `dontAsk` | 自动拒绝需要提示的操作 |
| `auto` | 基于 transcript 的 AI 分类器自动审批（feature-gated） |
| `bubble` | 权限决策冒泡到父代理（内部） |

### 6.2 三层规则体系

```typescript
// src/Tool.ts - ToolPermissionContext
{
  alwaysAllowRules: ToolPermissionRulesBySource,  // 自动放行
  alwaysDenyRules:  ToolPermissionRulesBySource,  // 自动拒绝
  alwaysAskRules:   ToolPermissionRulesBySource,  // 始终询问
}
```

### 6.3 规则来源优先级

| 来源 | 标识 | 描述 |
|------|------|------|
| `policySettings` | 企业策略 | `managed-settings.json`（最高优先级） |
| `flagSettings` | Feature Flag | 远程配置 |
| `userSettings` | 用户设置 | `~/.claude/settings.json` |
| `projectSettings` | 项目设置 | `.claude/settings.json` |
| `localSettings` | 本地设置 | `settings.local.json` |
| `cliArg` | CLI 参数 | `--allowedTools` / `--disallowedTools` |
| `command` | 斜杠命令 | 会话中的命令授权 |
| `session` | 运行时授权 | 用户点击 "Allow" 产生 |

### 6.4 规则语法

格式：`ToolName` 或 `ToolName(argument_pattern)`

```
Bash                          # 整个 Bash 工具
Bash(npm install)             # 仅匹配 npm install 命令
Bash(git *)                   # git 开头的所有命令
Bash(python -c "print\\(1\\)")  # 转义括号
mcp__server1                  # 整个 MCP 服务器的所有工具
mcp__server1__*               # 同上，通配符形式
Agent(Explore)                # 仅 Explore 代理类型
WebFetch domain:github.com    # 域名级匹配
```

括号内的内容用反斜杠转义：`\\(`、`\\)`。解析器使用 `findFirstUnescapedChar` 定位边界。

### 6.5 权限决策流程

```
1. validateInput()
   └── 工具特定输入验证（拒绝畸形输入）

2. 规则匹配
   ├── toolAlwaysAllowedRule()    → 整工具级放行检查
   ├── getDenyRuleForTool()      → 整工具级拒绝检查
   ├── getAskRuleForTool()       → 整工具级询问检查
   └── getRuleByContentsForTool() → 内容模式匹配
                                    (如 Bash(git *))

3. checkPermissions()
   └── 工具特定权限逻辑
       (如 BashTool 分解复合命令逐一检查)
       返回：allow / deny / ask

4. Hooks
   └── executePermissionRequestHooks()
       PreToolUse 钩子可覆盖决策

5. Classifier（仅 auto 模式）
   └── classifyYoloAction()
       AI 模型评估上下文安全性

6. 用户提示（如果 ask）
   └── 显示权限对话框
       选项：Allow Once / Always Allow / Deny
```

### 6.6 权限决策结果

```typescript
type PermissionAllowDecision = { behavior: 'allow'; updatedInput?; userModified? }
type PermissionDenyDecision  = { behavior: 'deny'; ... }
type PermissionAskDecision   = { behavior: 'ask'; decisionReason?; ... }
```

决策原因：`classifier` / `hook` / `rule` / `subcommandResults` / `mode` / `sandboxOverride` / `workingDir` / `safetyCheck` / `asyncAgent` / `permissionPromptTool` / `other`

## 7. 安全防护

### 7.1 危险命令模式

**文件**: `src/utils/permissions/dangerousPatterns.ts`

即使有 allow 规则也不能完全放行的命令：

| 类别 | 命令 |
|------|------|
| 解释器 | `python`, `python3`, `node`, `deno`, `ruby`, `perl`, `php`, `lua` |
| 包运行器 | `npx`, `bunx`, `npm run`, `yarn run`, `pnpm run`, `bun run` |
| Shell | `bash`, `sh`, `zsh`, `fish` |
| 权限提升 | `sudo`, `eval`, `exec`, `env`, `xargs` |
| 网络（ant-only） | `ssh`, `curl`, `wget`, `gh`, `gh api` |
| 云（ant-only） | `kubectl`, `aws`, `gcloud`, `gsutil` |
| Git（ant-only） | `git`（因 hooks/config 可执行代码） |

### 7.2 受保护文件

**文件**: `src/utils/permissions/filesystem.ts`

| 类型 | 文件/目录 |
|------|----------|
| 文件 | `.gitconfig`, `.gitmodules`, `.bashrc`, `.bash_profile`, `.zshrc`, `.zprofile`, `.profile`, `.ripgreprc`, `.mcp.json`, `.claude.json` |
| 目录 | `.git`, `.vscode`, `.idea`, `.claude` |

### 7.3 拒绝追踪

**文件**: `src/utils/permissions/denialTracking.ts`

追踪重复拒绝次数。超过 `DENIAL_LIMITS` 后回退到提示用户，防止模型陷入拒绝循环。

### 7.4 分类器

| 分类器 | 文件 | 描述 |
|--------|------|------|
| Bash 分类器 | `bashClassifier.ts` | 按 allow/deny 描述分类单条命令 |
| YOLO/Transcript 分类器 | `yoloClassifier.ts` | 全 transcript 感知，AI 模型评估上下文安全性（驱动 `auto` 模式） |

### 7.5 模型级工具过滤

`filterToolsByDenyRules()` 在模型看到工具列表**之前**移除被 deny 的工具，模型甚至无法尝试调用被禁止的工具。

## 8. 设计要点

1. **接口而非类**: Tool 采用泛型接口设计，通过 `buildTool` 工厂冻结实例
2. **三种注册模式**: 静态/feature-gated/延迟加载，平衡启动性能与功能覆盖
3. **六层权限来源**: 企业策略 → 用户 → 项目 → 本地 → CLI → 运行时
4. **深度防御**: 输入验证 → 规则匹配 → 工具权限 → Hook → 分类器 → 用户提示
5. **MCP 命名空间**: `mcp__server__tool` 格式避免工具名冲突
6. **延迟加载**: `shouldDefer` 机制减少初始 prompt 中的工具描述体积
7. **拒绝追踪**: 防止模型陷入无限拒绝循环

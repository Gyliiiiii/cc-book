# 03 - 代理系统（Agent System & Subagents）

## 1. 模块概述

代理系统是 Claude Code 的核心编排层，实现了层级化的任务分解与多代理协作。主代理（Main Agent）接收用户指令，可通过 `AgentTool` 生成子代理（Subagent）来并行处理独立任务。系统支持多种代理类型、隔离模式、执行模式，以及专用的协调器（Coordinator）编排模式。

## 2. 核心文件结构

```
src/
├── Task.ts                              # 任务基类与类型定义
├── tasks.ts                             # 任务注册表
├── tasks/
│   ├── types.ts                         # 任务类型联合
│   ├── LocalAgentTask/
│   │   └── LocalAgentTask.tsx           # 本地代理任务实现
│   ├── InProcessTeammateTask/
│   │   └── types.ts                     # 进程内队友任务类型
│   ├── RemoteAgentTask/
│   │   └── RemoteAgentTask.tsx          # 远程代理任务
│   └── stopTask.ts                      # 任务停止逻辑
├── tools/AgentTool/
│   ├── AgentTool.tsx                    # AgentTool 入口（生成子代理）
│   ├── runAgent.ts                      # 运行代理核心逻辑
│   ├── forkSubagent.ts                  # Fork 子代理（继承父上下文）
│   ├── resumeAgent.ts                   # 恢复/继续代理
│   ├── prompt.ts                        # 代理 Prompt 构建
│   ├── loadAgentsDir.ts                 # 加载代理定义
│   ├── builtInAgents.ts                 # 内置代理注册
│   └── built-in/
│       ├── generalPurposeAgent.ts       # 通用代理
│       ├── exploreAgent.ts              # 探索代理（只读）
│       └── planAgent.ts                 # 规划代理（只读）
├── coordinator/
│   └── coordinatorMode.ts               # 协调器模式
├── buddy/                               # 虚拟伙伴系统（装饰性）
│   ├── companion.ts                     # 伙伴生成
│   ├── types.ts                         # 伙伴类型定义
│   ├── prompt.ts                        # 伙伴 Prompt 注入
│   ├── CompanionSprite.tsx              # UI 渲染
│   └── useBuddyNotification.tsx         # 通知钩子
└── utils/
    └── worktree.ts                      # Worktree 隔离工具
```

## 3. 任务类型与生命周期

### 3.1 TaskType — 六种任务类型

| 类型 | 前缀 | 描述 |
|------|------|------|
| `local_bash` | `b` | Shell 命令执行 |
| `local_agent` | `a` | 本地子代理 |
| `remote_agent` | `r` | 远程代理（CCR 环境） |
| `in_process_teammate` | `t` | 进程内队友（团队/Swarm） |
| `local_workflow` | — | 工作流脚本（feature-gated） |
| `monitor_mcp` | — | MCP 服务器监控（feature-gated） |
| `dream` | — | 后台 "dreaming" 任务 |

### 3.2 Task ID 生成

使用类型前缀 + 8 位随机字符（36 字符字母表：数字 + 小写字母），约 2.8 万亿种组合：

```
a963d1f2  → local_agent 类型
b4f7e8a1  → local_bash 类型
```

### 3.3 生命周期状态机

```
pending (创建)
    │
    ▼
 running (执行中)
    │
    ├──────► completed (成功完成)
    │
    ├──────► failed (执行失败)
    │
    └──────► killed (手动停止 / ESC / killAsyncAgent)
```

- **pending**: 已注册但未开始执行
- **running**: query 循环活跃中
- **completed**: 成功完成，结果已存储
- **failed**: 遇到错误，错误信息已存储
- **killed**: 手动终止，abort controller 触发，发送部分结果通知

终态检查：`isTerminalTaskStatus()` 对 `completed/failed/killed` 返回 true，防止向已终止代理注入消息。

## 4. AgentTool — 子代理生成入口

**文件**: `src/tools/AgentTool/AgentTool.tsx`

### 4.1 参数

| 参数 | 类型 | 描述 |
|------|------|------|
| `description` | `string` | 3-5 字简要描述 |
| `prompt` | `string` | 完整任务指令 |
| `subagent_type` | `string?` | 代理类型（默认通用型/Fork） |
| `model` | `enum?` | 模型覆盖：`sonnet` / `opus` / `haiku` |
| `run_in_background` | `boolean?` | 后台异步执行 |
| `name` | `string?` | 可寻址名称（用于 SendMessage） |
| `team_name` | `string?` | 团队上下文 |
| `mode` | `string?` | 权限模式 |
| `isolation` | `enum?` | 隔离模式：`worktree` / `remote` |
| `cwd` | `string?` | 工作目录覆盖（feature-gated） |

### 4.2 自动后台化

运行超过 120 秒的前台代理可被自动转为后台执行（需 `CLAUDE_AUTO_BACKGROUND_TASKS` 环境变量或 GrowthBook gate `tengu_auto_background_agents`）。

## 5. 代理运行核心流程

**文件**: `src/tools/AgentTool/runAgent.ts`

`runAgent()` 是一个异步生成器（async generator），核心流程：

```
1. 解析代理模型、ID、权限模式

2. 初始化代理专属 MCP 服务器（累加到父代理的 MCP 客户端）

3. 构建初始消息
   ├── 上下文消息 (context messages)
   ├── Prompt 消息
   ├── Hook 上下文
   └── 预加载技能

4. 创建子代理上下文 (createSubagentContext)
   ├── 同步代理：共享 setAppState + abortController
   └── 异步代理：完全隔离，新 AbortController

5. 循环调用 query()，yield 消息回调用方

6. 记录 sidechain transcript 到磁盘（支持恢复/查看）

7. 清理
   ├── MCP 服务器
   ├── Session hooks
   ├── Prompt cache 追踪
   ├── 文件状态缓存
   ├── Perfetto tracing
   ├── Todo 条目
   └── 代理生成的后台 bash 任务
```

### 性能优化

| 优化 | 适用对象 | 效果 |
|------|----------|------|
| `omitClaudeMd: true` | Explore、Plan 代理 | 跳过 CLAUDE.md 注入，节省 ~5-15 Gtok/周 |
| 跳过 `gitStatus` | Explore、Plan 代理 | 减少系统上下文体积 |
| 继承 thinking config | Fork 子代理 | 提高 prompt cache 命中率 |
| 继承精确 tool pool | Fork 子代理 | API 前缀字节一致，共享缓存 |
| `effort` 级别覆盖 | 自定义代理 | 调整推理深度 |

## 6. Fork 子代理

**文件**: `src/tools/AgentTool/forkSubagent.ts`

省略 `subagent_type` 参数时触发 Fork 模式——子代理**继承父代理的完整对话上下文和系统提示**：

```
Fork 特征：
├── tools: ['*'] + useExactTools    — 继承父代理精确工具集
├── permissionMode: 'bubble'        — 权限提示冒泡到父终端
├── model: 'inherit'                — 与父代理相同模型
├── buildForkedMessages()           — 构建字节级一致的前缀
└── 注入自我描述："You are a forked worker process"
    ├── 不可嵌套 fork（isInForkChild 检测）
    ├── 不发起对话
    ├── 提交变更
    └── 报告 <500 字
```

**关键设计**: `buildForkedMessages()` 确保所有 Fork 子代理的 API 请求前缀字节一致，从而共享 prompt cache。

## 7. 代理定义格式

### 7.1 三种来源

| 来源 | 类型 | 描述 |
|------|------|------|
| `built-in` | `BuiltInAgentDefinition` | 内置代理，动态 `getSystemPrompt()` |
| 文件系统 | `CustomAgentDefinition` | `.claude/agents/*.md` 或设置中定义 |
| 插件 | `PluginAgentDefinition` | 从已安装插件加载 |

### 7.2 Markdown Frontmatter 格式

```yaml
---
name: my-agent                    # 必填：代理类型标识
description: 何时使用此代理        # 必填：触发条件描述
tools: [Bash, Read, Edit]         # 可选：工具白名单
disallowedTools: [Agent]          # 可选：工具黑名单
model: sonnet|opus|haiku|inherit  # 可选：模型选择
effort: low|medium|high|<int>     # 可选：推理深度
permissionMode: default|plan|bypassPermissions|bubble  # 可选
maxTurns: 200                     # 可选：最大轮次
color: blue                       # 可选：UI 颜色
background: true                  # 可选：始终后台运行
isolation: worktree|remote        # 可选：隔离模式
memory: user|project|local        # 可选：持久记忆范围
skills: [commit, verify]          # 可选：预加载技能
initialPrompt: "..."              # 可选：首轮前置 prompt
mcpServers:                       # 可选：代理专属 MCP 服务器
  - "slack"                       # 按名称引用
  - myServer: { command: ... }    # 内联定义
hooks:                            # 可选：会话作用域钩子
  SubagentStart: ...
---
系统提示内容作为 Markdown 正文。
```

### 7.3 加载优先级

后定义覆盖先定义（按 `agentType`）：

```
1. Built-in agents          (最低优先级)
2. Plugin agents
3. User settings agents     (~/.claude/agents/)
4. Project settings agents  (.claude/agents/)
5. Flag settings agents
6. Managed/policy agents    (最高优先级)
```

## 8. 内置代理

| 代理 | 模型 | 工具 | 特点 |
|------|------|------|------|
| `general-purpose` | 继承 | `['*']` | 默认代理，研究/搜索/多步任务 |
| `Explore` | Haiku(外部)/继承(内部) | 只读 | 快速文件搜索，禁止 Agent/Edit/Write |
| `Plan` | 继承 | 只读 | 架构规划，禁止 Agent/Edit/Write |
| `statusline-setup` | — | Read, Edit | 配置状态栏 |
| `claude-code-guide` | — | — | 帮助/指南代理（非 SDK） |
| `verification` | — | — | 验证代理（feature-gated） |

## 9. 隔离模式

### 9.1 默认（无隔离）

子代理与父代理共享工作目录和 Git checkout。文件编辑对所有代理立即可见。

### 9.2 Worktree 隔离

**文件**: `src/utils/worktree.ts`

```
设置 isolation: "worktree" 时：
├── 创建临时 git worktree → .claude/worktrees/<slug>/
├── slug 验证：防止路径穿越（字母数字+点/下划线/短横线，最大 64 字符）
├── 大目录（node_modules）从主仓库符号链接，避免磁盘膨胀
├── buildWorktreeNotice() 注入路径转换指引
├── 有变更 → 返回 worktree 路径和分支名
└── 无变更 → 自动清理 worktree
```

### 9.3 远程隔离

`isolation: "remote"` 在远程 CCR 环境中启动，始终后台运行。

## 10. 前台 vs 后台执行

| 特性 | 前台（默认） | 后台 |
|------|-------------|------|
| 父代理 | 阻塞等待结果 | 立即继续 |
| AbortController | 共享父代理的 | 独立的 |
| 权限提示 | 显示在终端 | 默认抑制（除非 `bubble`） |
| 取消行为 | 取消父 = 取消子 | 独立运行 |
| 结果交付 | 直接返回上下文 | `<task-notification>` XML 消息 |
| 超时自动化 | — | 120 秒后可自动后台化 |

## 11. 代理间通信

### 11.1 完成通知

后台代理完成时，通过 `enqueueAgentNotification()` 发送 XML 通知：

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>Agent "description" completed</summary>
  <result>{最终文本响应}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

以 **user-role 消息** 入队到父代理对话中。

### 11.2 继续代理（SendMessage）

通过 `SendMessageTool` 使用代理 ID 或名称作为 `to` 字段继续代理：

```
resumeAgentBackground()
├── 从磁盘读取 sidechain transcript
├── 重建对话状态
├── 注入新 prompt 作为用户消息
└── 恢复 query() 循环
```

### 11.3 待处理消息队列

代理执行中收到的 `SendMessage` 消息排队到 `pendingMessages`，在工具轮次边界通过 `drainPendingMessages()` 排空。

### 11.4 进度追踪

`ProgressTracker` 监控：
- `toolUseCount` — 工具调用次数
- `latestInputTokens` / `cumulativeOutputTokens` — Token 使用量
- `recentActivities` — 最近 5 次工具调用描述
- 输出到 UI 后台任务指示器

## 12. 协调器模式（Coordinator Mode）

**文件**: `src/coordinator/coordinatorMode.ts`

通过 `CLAUDE_CODE_COORDINATOR_MODE=1`（feature flag `COORDINATOR_MODE`）启用的替代编排范式。

### 12.1 协调器角色

- 作为编排者指导 "workers"（子代理）研究、实施和验证代码变更
- 综合结果并与用户沟通
- 能直接回答的问题不委托

### 12.2 协调器专属工具

| 工具 | 描述 |
|------|------|
| `Agent` | 生成新 worker |
| `SendMessage` | 继续现有 worker |
| `TaskStop` | 停止运行中 worker |
| `subscribe_pr_activity` | 订阅 GitHub PR 事件 |
| `unsubscribe_pr_activity` | 取消订阅 |

### 12.3 任务工作流阶段

```
1. Research   — worker 并行调查代码库
2. Synthesis  — 协调器阅读发现，编制实施规范
3. Implementation — worker 按规范定向修改
4. Verification   — worker 测试并证明变更有效
```

### 12.4 关键规则

- Worker 看不到协调器对话 → 每个 prompt 必须自包含
- 不使用 "based on your findings" → 始终自己综合发现
- 强调并行："Launch independent workers concurrently whenever possible"
- 只读任务自由并行；写入密集型任务按文件集序列化
- 上下文重叠高 → `SendMessage` 继续；重叠低 → 新建 worker

## 13. 进程内队友系统（InProcessTeammate）

用于多代理 "Swarm"/"Team" 模式，队友作为进程内代理共享终端。

### 关键状态

| 字段 | 描述 |
|------|------|
| `identity` | `TeammateIdentity`：agentId、agentName、teamName、color |
| `awaitingPlanApproval` | 是否等待 plan 审批 |
| `permissionMode` | 通过 Shift+Tab 独立切换 |
| `messages` | 上限 50 条（`TEAMMATE_MESSAGES_UI_CAP`）用于 UI |
| `pendingUserMessages` | 待交付消息队列 |
| `isIdle` / `shutdownRequested` | 生命周期标志 |
| `onIdleCallbacks` | 高效等待回调（无轮询） |

**50 条上限原因**: 大型会话可在 2 分钟内生成 292 个代理，每个代理约 20MB RSS，最高达 36.8GB 内存。

## 14. Buddy 系统（虚拟伙伴）

与代理团队**无关**，是装饰性的虚拟宠物陪伴功能：

- 基于 `hash(userId + salt)` 确定性生成
- 18 种物种、5 种稀有度、6 种眼型、8 种帽子、5 项属性
- 稀有度分布：Common 60% / Uncommon 25% / Rare 10% / Epic 4% / Legendary 1%
- "骨骼"（视觉特征）从 hash 重新生成，"灵魂"（名字、性格）持久化

## 15. 设计要点

1. **层级分解**: 主代理 → 子代理 → 子代理的子代理，支持递归编排
2. **Fork 缓存优化**: 字节一致的 API 前缀确保 prompt cache 共享
3. **自动后台化**: 长时间运行的前台任务自动转后台，不阻塞用户
4. **隔离灵活性**: 无隔离 / Worktree / 远程三种级别按需选择
5. **异步生成器**: `runAgent()` 使用 async generator 实现流式消息传递
6. **50 条 UI 上限**: 防止大量代理导致内存爆炸
7. **协调器模式**: 替代编排范式，适合大规模并行任务

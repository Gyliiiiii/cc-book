# 06 - 钩子系统（Hook System）

## 1. 模块概述

钩子系统是 Claude Code 的生命周期扩展机制，支持在 27 个不同的生命周期事件点注入自定义行为。钩子可以是 Shell 命令、LLM Prompt、多轮 Agent 查询或 HTTP 请求，支持条件过滤、热重载、安全控制和异步执行。

## 2. 核心文件结构

```
src/
├── hooks/                               # (104 files) 钩子系统主目录
├── schemas/
│   └── hooks.ts                         # HookCommandSchema, HookMatcherSchema, IfConditionSchema
├── types/
│   └── hooks.ts                         # Zod 输出 Schema, HookResult, AggregatedHookResult
├── utils/
│   ├── hooks.ts                         # 主执行引擎：spawn, 解析输出, pre/post 工具钩子
│   └── hooks/
│       ├── hooksSettings.ts             # getAllHooks(), 发现/优先级排序
│       ├── hooksConfigManager.ts        # 事件元数据, groupHooksByEventAndMatcher()
│       ├── hooksConfigSnapshot.ts       # 安全控制: disableAllHooks, allowManagedHooksOnly
│       ├── sessionHooks.ts              # 内存会话钩子, 函数钩子
│       ├── hookEvents.ts               # 广播钩子执行事件
│       ├── AsyncHookRegistry.ts         # 后台异步钩子追踪
│       ├── execAgentHook.ts             # Agent 钩子执行器
│       ├── execPromptHook.ts            # Prompt 钩子执行器
│       └── execHttpHook.ts              # HTTP 钩子执行器
├── utils/plugins/
│   └── loadPluginHooks.ts              # 插件钩子加载 + 热重载
└── services/tools/
    └── toolHooks.ts                     # Pre/PostToolUse 钩子编排
```

## 3. 全部 27 个生命周期事件

### 3.1 工具相关事件

| 事件 | 触发时机 | 匹配字段 | 退出码语义 |
|------|----------|----------|-----------|
| `PreToolUse` | 工具执行前 | `tool_name` | 0=安静, 2=阻止+stderr 给模型 |
| `PostToolUse` | 工具执行后（成功） | `tool_name` | 0=stdout 入 transcript, 2=stderr 给模型 |
| `PostToolUseFailure` | 工具执行后（失败） | `tool_name` | 0=stdout 入 transcript, 2=stderr 给模型 |
| `PermissionDenied` | auto 模式分类器拒绝工具 | `tool_name` | 0=stdout 入 transcript, 支持 `retry:true` |
| `PermissionRequest` | 显示权限对话框 | `tool_name` | 0=使用钩子决策 |

### 3.2 会话生命周期事件

| 事件 | 触发时机 | 匹配字段 | 退出码语义 |
|------|----------|----------|-----------|
| `SessionStart` | 新会话开始 | `source` (startup/resume/clear/compact) | 0=stdout 给 Claude |
| `SessionEnd` | 会话结束 | `reason` (clear/logout/exit/other) | 0=成功, 1.5s 超时 |
| `Stop` | Claude 结束响应前 | — | 0=安静, 2=stderr 给模型+继续 |
| `StopFailure` | 因 API 错误结束 | `error` (rate_limit/auth/billing) | fire-and-forget |

### 3.3 代理相关事件

| 事件 | 触发时机 | 匹配字段 | 退出码语义 |
|------|----------|----------|-----------|
| `SubagentStart` | 子代理启动 | `agent_type` | 0=stdout 给子代理 |
| `SubagentStop` | 子代理结束前 | `agent_type` | 0=安静, 2=stderr 给子代理+继续 |
| `TeammateIdle` | 队友即将空闲 | — | 0=安静, 2=stderr+阻止空闲 |
| `TaskCreated` | 任务创建中 | — | 0=安静, 2=stderr+阻止创建 |
| `TaskCompleted` | 任务完成中 | — | 0=安静, 2=stderr+阻止完成 |

### 3.4 压缩事件

| 事件 | 触发时机 | 匹配字段 | 退出码语义 |
|------|----------|----------|-----------|
| `PreCompact` | 对话压缩前 | `trigger` (manual/auto) | 0=stdout 作为压缩指令, 2=阻止压缩 |
| `PostCompact` | 对话压缩后 | `trigger` (manual/auto) | 0=stdout 给用户 |

### 3.5 配置与系统事件

| 事件 | 触发时机 | 匹配字段 | 退出码语义 |
|------|----------|----------|-----------|
| `Setup` | 仓库初始化/维护 | `trigger` (init/maintenance) | 0=stdout 给 Claude |
| `ConfigChange` | 会话中配置文件变化 | `source` (user/project/local/policy/skills) | 0=允许, 2=阻止 |
| `InstructionsLoaded` | 指令文件加载 | `load_reason` | 仅观察性，不支持阻止 |
| `CwdChanged` | 工作目录变化后 | — | 0=成功, 可返回 `watchPaths` |
| `FileChanged` | 监视文件变化 | 文件名模式 | 0=成功, 可返回 `watchPaths` |

### 3.6 MCP 相关事件

| 事件 | 触发时机 | 匹配字段 | 退出码语义 |
|------|----------|----------|-----------|
| `Elicitation` | MCP 服务器请求用户输入 | `mcp_server_name` | 0=使用钩子响应, 2=拒绝 |
| `ElicitationResult` | 用户响应 MCP 输入请求后 | `mcp_server_name` | 0=使用钩子响应, 2=阻止 |

### 3.7 环境事件

| 事件 | 触发时机 | 匹配字段 | 退出码语义 |
|------|----------|----------|-----------|
| `Notification` | 发送通知时 | `notification_type` | 0=安静 |
| `UserPromptSubmit` | 用户提交 prompt 时 | — | 0=stdout 给 Claude, 2=阻止+清除 |
| `WorktreeCreate` | 创建隔离 worktree | — | 0=stdout 作为路径 |
| `WorktreeRemove` | 移除 worktree | — | 0=成功 |

## 4. 钩子协议

### 4.1 输入（stdin JSON）

每个钩子通过 stdin 接收 JSON，包含基础输入：

```json
{
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "permission_mode": "...",
  "agent_id": "...",       // 如在子代理中
  "agent_type": "..."      // 如在子代理中
}
```

加上事件特定字段。如 `PreToolUse` 额外包含 `tool_name`、`tool_input`、`tool_use_id`。

### 4.2 输出（stdout JSON）

同步响应：

```json
{
  "continue": true,              // Claude 是否继续（默认 true）
  "suppressOutput": false,       // 隐藏 stdout
  "stopReason": "...",           // continue=false 时的消息
  "decision": "approve|block",  // 传统格式
  "reason": "...",
  "systemMessage": "...",        // 显示给用户的警告
  "hookSpecificOutput": { ... } // 事件特定字段
}
```

异步响应（后台运行）：

```json
{
  "async": true,
  "asyncTimeout": 15000         // 超时毫秒（默认 15s）
}
```

### 4.3 退出码

| 退出码 | 含义 |
|--------|------|
| 0 | 成功。行为因事件而异（stdout 给 Claude/入 transcript/安静） |
| 2 | 阻塞错误。stderr 给模型，可阻止操作 |
| 其他 | 非阻塞错误。stderr 仅给用户，操作继续 |

如果 stdout 不是有效 JSON 或不以 `{` 开头，视为纯文本。

## 5. 四种钩子类型

### 5.1 Command（Shell 命令）

```json
{
  "type": "command",
  "command": "echo 'about to write'",
  "shell": "bash",              // bash(默认) 或 powershell
  "timeout": 30,
  "statusMessage": "检查中...",
  "once": false,                // 运行一次后移除
  "async": false,               // 后台运行
  "asyncRewake": false,         // 后台+退出码2时唤醒模型
  "if": "Write(*.py)"           // 条件过滤
}
```

### 5.2 Prompt（单次 LLM 评估）

```json
{
  "type": "prompt",
  "prompt": "Is this write safe? $ARGUMENTS",
  "model": "claude-sonnet-4-6",  // 默认小快模型
  "timeout": 30,                // 默认 30s
  "statusMessage": "评估中...",
  "once": false,
  "if": "..."
}
```

LLM 返回 `{ok: true}` 或 `{ok: false, reason: "..."}`。

### 5.3 Agent（多轮 Agent 查询）

```json
{
  "type": "agent",
  "prompt": "Verify unit tests pass. $ARGUMENTS",
  "model": "...",
  "timeout": 60,                // 默认 60s
  "statusMessage": "验证中...",
  "once": false,
  "if": "..."
}
```

运行完整的 Agent 循环，可使用工具进行验证任务。

### 5.4 HTTP（HTTP POST）

```json
{
  "type": "http",
  "url": "https://my-server.com/hooks/pre-write",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 30,
  "statusMessage": "调用 webhook...",
  "once": false,
  "if": "..."
}
```

安全防护：SSRF 保护（`ssrfGuardedLookup()`）、CRLF 注入防护、策略限制 URL（`allowedHttpHookUrls`）。

## 6. 钩子发现与加载

### 6.1 来源优先级

`getAllHooks()` 按以下顺序汇集钩子：

```
1. userSettings       — ~/.claude/settings.json
2. projectSettings    — .claude/settings.json
3. localSettings      — .claude/settings.local.json
4. policySettings     — 企业/托管策略（特殊权限）
5. 插件钩子           — 插件 hooks.json（最低排序优先级 999）
6. 会话钩子           — 内存中，由技能/frontmatter 添加
7. 已注册钩子         — 内置回调（归因、文件访问等）
```

### 6.2 分组与匹配

`groupHooksByEventAndMatcher()` 组织为：

```
Record<HookEvent, Record<matcherString, IndividualHookConfig[]>>
```

Matchers 按来源优先级排序（`sortMatchersByPriority()`）。

## 7. 条件过滤（if 字段）

使用**权限规则语法**：

```json
{ "if": "Bash(git *)" }
{ "if": "Read(*.ts)" }
{ "if": "Write(src/**/*.py)" }
```

在钩子进程生成前评估，不匹配则完全跳过（无子进程）。包含 `if` 的相同命令被视为不同钩子。

## 8. PreToolUse 钩子详解

`hookSpecificOutput` 结构：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "...",
    "updatedInput": { ... },
    "additionalContext": "..."
  }
}
```

### 8.1 permissionDecision 解析

| 决策 | 行为 |
|------|------|
| `allow` | 绕过交互式权限提示。但**不能**绕过 settings.json 的 deny/ask 规则（深度防御） |
| `deny` | 阻止工具调用（最终决策） |
| `ask` | 强制显示权限对话框，附带自定义消息 |

### 8.2 updatedInput

替换工具输入参数。可与 `permissionDecision` 同时返回或单独返回。单独返回时修改后的输入正常走权限流程。

对于需要用户交互的工具（`requiresUserInteraction`），`allow` + `updatedInput` 可满足交互要求。

## 9. PostToolUse 钩子详解

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "...",
    "updatedMCPToolOutput": { ... }
  }
}
```

| 字段 | 描述 |
|------|------|
| `additionalContext` | 作为系统消息注入，模型可见 |
| `updatedMCPToolOutput` | 仅替换 MCP 工具的输出 |
| `continue: false` | 停止对话 |
| `decision: "block"` | 创建阻塞错误 |

## 10. 热重载

### 插件钩子

```
settingsChangeDetector 监控 enabledPlugins 变化
    ↓
检测到变化：比较 lastPluginSettingsSnapshot
    ↓
清除缓存：clearPluginCache() + clearRegisteredPluginHooks()
    ↓
重新加载：loadPluginHooks()（memoized，缓存清除后强制刷新）
```

### 设置钩子

```
captureHooksConfigSnapshot()          — 启动时捕获一次
updateHooksConfigSnapshot()           — 设置修改时调用
    ├── resetSettingsCache()          — 强制从磁盘重读
    └── 重新捕获快照
```

## 11. 安全控制

### 11.1 策略级控制

| 控制 | 位置 | 效果 |
|------|------|------|
| `disableAllHooks` | `policySettings` | 禁用**所有**钩子（包括托管） |
| `disableAllHooks` | 非托管设置 | 仅禁用非托管钩子，托管钩子仍运行 |
| `allowManagedHooksOnly` | `policySettings` | 仅运行策略定义的钩子 |
| `strictPluginOnlyCustomization` | 策略 | 阻止用户/项目/本地钩子 |

### 11.2 工作区信任

交互模式下所有钩子要求工作区信任（信任对话框已接受）。SDK 非交互模式下信任隐式授予。

### 11.3 HTTP 钩子限制

- `allowedHttpHookUrls` — URL 白名单模式匹配
- `httpHookAllowedEnvVars` — 环境变量插值白名单
- SSRF 防护（`ssrfGuardedLookup()`）
- CRLF 注入防护（`sanitizeHeaderValue()`）

## 12. 配置格式

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          { "type": "command", "command": "...", "if": "Write(*.py)" },
          { "type": "prompt", "prompt": "...", "model": "..." },
          { "type": "http", "url": "...", "headers": { "Auth": "$TOKEN" } },
          { "type": "agent", "prompt": "...", "timeout": 60 }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          { "type": "command", "command": "echo 'session started'" }
        ]
      }
    ]
  }
}
```

结构：`hooks → 事件名 → Array<{ matcher?: string, hooks: HookCommand[] }>`

配置可存在于：
- `~/.claude/settings.json`（用户）
- `.claude/settings.json`（项目）
- `.claude/settings.local.json`（本地）
- 企业策略设置
- 插件 `hooks.json`
- 内存会话钩子（仅编程方式）

## 13. 设计要点

1. **27 个事件点**: 覆盖完整生命周期，从会话开始到工具执行到压缩到结束
2. **四种执行器**: Command / Prompt / Agent / HTTP，满足不同集成需求
3. **深度防御**: 钩子 `allow` 不能绕过 settings.json 的 deny 规则
4. **条件过滤**: `if` 字段避免不必要的进程开销
5. **异步支持**: 钩子可后台运行，超时后自动完成
6. **热重载**: 插件和设置变更自动检测并重加载
7. **托管策略优先**: 企业钩子不受普通禁用影响

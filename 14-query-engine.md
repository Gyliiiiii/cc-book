# 14 - 查询引擎（Query Engine）

## 1. 模块概述

查询引擎是 Claude Code 的核心执行循环，驱动每一次对话轮次。`QueryEngine` 类管理会话状态和查询生命周期，`query()` 函数实现实际的模型调用 → 工具执行 → 继续循环。系统支持流式工具执行（模型流式输出时并行执行工具）、多层压缩、回退模型切换、Token 预算管理等高级特性。

## 2. 核心文件结构

```
src/
├── QueryEngine.ts                   # (47KB) QueryEngine 类：会话状态 + submitMessage()
├── query.ts                         # (69KB) query() + queryLoop()：主循环
├── query/
│   ├── config.ts                    # QueryConfig：不可变会话/环境快照
│   ├── deps.ts                      # QueryDeps：可注入 I/O 依赖
│   ├── tokenBudget.ts               # BudgetTracker：Token 预算管理
│   └── stopHooks.ts                 # handleStopHooks()：轮次结束钩子
├── services/compact/                # 压缩引擎（见 05-context-management.md）
└── tools/                           # 工具执行（见 04-tool-system.md）
```

## 3. QueryEngine 类

**文件**: `src/QueryEngine.ts`

每个对话一个 `QueryEngine` 实例，每次 `submitMessage()` 调用启动一个新轮次。

### 3.1 配置（QueryEngineConfig）

| 参数 | 描述 |
|------|------|
| `cwd` | 工作目录 |
| `tools` | 可用工具集 |
| `commands` | 斜杠命令集 |
| `mcpClients` | MCP 客户端 |
| `agents` | 代理定义 |
| `canUseTool` | 权限检查回调 |
| `getAppState` / `setAppState` | 应用状态读写 |
| `initialMessages` | 初始消息 |
| `readFileCache` | 文件内容缓存 |
| `userSpecifiedModel` / `fallbackModel` | 模型选择 |
| `thinkingConfig` | 思考模式配置 |
| `maxTurns` / `maxBudgetUsd` / `taskBudget` | 预算控制 |
| `jsonSchema` | 结构化输出 Schema |
| `snipReplay` | 历史修剪回调 |

### 3.2 关键状态

| 状态 | 描述 |
|------|------|
| `mutableMessages` | 对话历史（跨轮次持久） |
| `abortController` | 取消控制器 |
| `permissionDenials` | SDK 权限拒绝追踪 |
| `totalUsage` | 累计 API 使用量 |
| `readFileState` | 文件状态缓存 |
| `discoveredSkillNames` | 轮次级技能发现追踪（每次 submitMessage 重置） |

### 3.3 submitMessage() 流程

```
1. 解构配置，重置技能名，设置 CWD

2. 包装 canUseTool 以拦截和追踪权限拒绝

3. 解析模型 (parseUserSpecifiedModel 或 getMainLoopModel)

4. 配置思考模式（默认自适应）

5. fetchSystemPromptParts() 构建：
   ├── System Prompt
   ├── User Context
   └── System Context

6. 可选：注入记忆机制 Prompt

7. 可选：注册结构化输出强制（jsonSchema）

8. 构建 ProcessUserInputContext：
   ├── 消息管理
   ├── 工具选项
   ├── MCP 客户端
   ├── 代理定义
   ├── 预算
   └── 状态更新回调

9. 处理孤立权限（每引擎生命周期一次）

10. 委托给 query() → 异步生成器 yield SDKMessage
```

## 4. 主循环（queryLoop）

**文件**: `src/query.ts`

### 4.1 循环状态（State 结构体）

| 字段 | 描述 |
|------|------|
| `messages` | 当前消息数组 |
| `toolUseContext` | 工具使用上下文 |
| `autoCompactTracking` | 自动压缩追踪 |
| `maxOutputTokensRecoveryCount` | 最大输出恢复重试次数 |
| `hasAttemptedReactiveCompact` | 是否已尝试响应式压缩 |
| `maxOutputTokensOverride` | 输出 Token 覆盖 |
| `pendingToolUseSummary` | 待处理工具使用摘要 |
| `stopHookActive` | Stop Hook 是否活跃 |
| `turnCount` | 轮次计数 |
| `transition` | 上次循环重启原因 |

### 4.2 主循环流程

```
while (true) {
  ┌────────────────────────────────────────┐
  │ 1. 技能发现预取                         │
  │    async prefetch 与模型流式并行         │
  ├────────────────────────────────────────┤
  │ 2. Snip Compact (HISTORY_SNIP)         │
  │    修剪旧历史                           │
  ├────────────────────────────────────────┤
  │ 3. 工具结果预算                         │
  │    applyToolResultBudget() 限制大小     │
  ├────────────────────────────────────────┤
  │ 4. 微压缩                              │
  │    deps.microcompact() 压缩消息         │
  ├────────────────────────────────────────┤
  │ 5. 上下文折叠 (CONTEXT_COLLAPSE)       │
  │    applyCollapsesIfNeeded()            │
  │    在 autocompact 之前运行              │
  ├────────────────────────────────────────┤
  │ 6. 自动压缩                            │
  │    deps.autocompact()                  │
  │    成功 → yield 压缩后消息 + 重置追踪  │
  ├────────────────────────────────────────┤
  │ 7. 阻塞限制检查                        │
  │    auto-compact 禁用 + 超限 → 报错返回 │
  ├────────────────────────────────────────┤
  │ 8. API 调用                            │
  │    deps.callModel()                    │
  │    = queryModelWithStreaming()          │
  │    配置：模型, thinking, 工具, fast mode│
  │         tool choice, fallback, budget  │
  ├────────────────────────────────────────┤
  │ 9. 流式处理循环                        │
  │    ├── 收集 AssistantMessage            │
  │    ├── 检测 tool_use 块                │
  │    ├── StreamingToolExecutor 并行执行   │
  │    └── 暂扣可恢复错误                  │
  │        (prompt-too-long, max-output,   │
  │         media-size)                    │
  ├────────────────────────────────────────┤
  │ 10. 回退模型处理                       │
  │     FallbackTriggeredError →           │
  │     切换 fallback model,               │
  │     tombstone 孤立消息, 重置, 重试     │
  ├────────────────────────────────────────┤
  │ 11. Post-sampling Hooks                │
  │     executePostSamplingHooks()         │
  ├────────────────────────────────────────┤
  │ 12. 中止处理                           │
  │     消费剩余流, 发出中断消息            │
  ├────────────────────────────────────────┤
  │ 13. 恢复路径 (多个 continue 站点)      │
  │     ├── 上下文折叠排空 (413 错误)      │
  │     ├── 响应式压缩 (完整摘要回退)      │
  │     ├── 最大输出升级 (8K → 64K)        │
  │     ├── 最大输出恢复 (≤3 次重试)       │
  │     └── Token 预算继续 (<90% → 继续)   │
  ├────────────────────────────────────────┤
  │ 14. 工具执行                           │
  │     needsFollowUp → runTools()         │
  │     收集结果, continue 循环             │
  ├────────────────────────────────────────┤
  │ 15. Stop Hooks                         │
  │     handleStopHooks()                  │
  │     stop/task-completed/teammate-idle  │
  │     extract-memories/auto-dream        │
  └────────────────────────────────────────┘
}
```

### 4.3 终止原因

| 原因 | 描述 |
|------|------|
| `stop` | 模型正常结束 |
| `blocking_limit` | 达到阻塞限制 |
| `prompt_too_long` | Prompt 过长 |
| `image_error` | 图片处理错误 |
| `aborted_streaming` | 流式中止 |
| `model_error` | 模型错误 |
| `max_output_tokens_exhausted` | 最大输出 Token 耗尽 |
| `max_turns` | 达到最大轮次 |
| `token_budget_complete` | Token 预算完成 |

## 5. 依赖注入（QueryDeps）

**文件**: `src/query/deps.ts`

4 个核心 I/O 函数可注入，支持测试替换：

| 依赖 | 生产实现 | 描述 |
|------|----------|------|
| `callModel` | `queryModelWithStreaming` | API 调用 |
| `microcompact` | `microcompactMessages` | 微压缩 |
| `autocompact` | `autoCompactIfNeeded` | 自动压缩 |
| `uuid` | `randomUUID` | UUID 生成 |

```typescript
// 测试中注入 fake
const fakeDeps = {
  callModel: async function*() { yield mockResponse; },
  microcompact: (msgs) => msgs,
  autocompact: async () => undefined,
  uuid: () => 'test-uuid',
};
```

## 6. Token 预算管理

**文件**: `src/query/tokenBudget.ts`

`BudgetTracker` 管理 +500K Token 自动继续功能：

| 参数 | 值 | 描述 |
|------|-----|------|
| `COMPLETION_THRESHOLD` | 0.9 (90%) | 低于 90% 预算时继续 |
| 最小进展检测 | 500 tokens | 3+ 次继续后每次少于 500 tokens → 停止 |
| 递减回报检测 | — | 防止失控循环 |

## 7. 流式工具执行

```
模型流式输出中...
    │
    ├── 检测到 tool_use 块
    │
    ├── StreamingToolExecutor 立即开始执行
    │   （不等模型输出完毕）
    │
    ├── 模型继续输出其他 tool_use
    │
    └── 所有工具结果收集完毕 → 下一轮循环
```

工具在模型流式输出过程中**并行执行**，显著减少端到端延迟。

## 8. 恢复策略

| 错误 | 恢复策略 |
|------|----------|
| Prompt-too-long (413) | 排空上下文折叠 → 响应式压缩 |
| Max output tokens (8K) | 升级到 64K → 最多 3 次重试 + "resume directly" 提示 |
| Fallback model 触发 | 切换到 fallback model，tombstone 孤立消息 |
| Token 预算 <90% | 继续执行，附加 nudge 消息 |
| 流式中止 | 消费剩余结果，发出中断消息 |

## 9. Feature Gate 集成

通过 `bun:bundle` 的 `feature()` 实现编译时树摇：

```
feature('HISTORY_SNIP')      → Snip compact
feature('REACTIVE_COMPACT')  → 响应式压缩
feature('CONTEXT_COLLAPSE')  → 上下文折叠
```

外部构建中未启用的 feature 代码被完全消除。

## 10. 孤立工具结果处理

`yieldMissingToolResultBlocks()` 生成器：

- 流式中断或错误时，部分 `tool_use` 块可能没有对应的 `tool_result`
- 该生成器为每个孤立的 `tool_use` 发出合成的错误 `tool_result`
- 防止 API 校验失败

## 11. 设计要点

1. **依赖注入**: 4 个核心 I/O 函数可注入，模块级无 spy 测试
2. **状态机循环**: `State` 结构体 + `transition` 字段记录重启原因
3. **流式工具执行**: 模型流式输出中并行执行工具，减少延迟
4. **多层压缩**: Snip → Micro → Context Collapse → Auto → Reactive 五层策略
5. **Feature Gate**: 编译时树摇消除未启用功能的代码
6. **优雅降级**: Prompt-too-long / Max output / Fallback 多种恢复路径
7. **预算管理**: 90% 阈值 + 递减回报检测防止失控

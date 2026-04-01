# 05 - 上下文窗口与压缩（Context Window & Compaction）

## 1. 模块概述

上下文管理系统负责实时追踪 Token 使用量，在接近上下文窗口容量时自动触发对话压缩（Compaction），并通过多种优化策略节省上下文空间。系统采用"估算 + API 实际值"混合追踪、多层压缩策略（微压缩 → 会话记忆压缩 → 完整压缩），以及智能的状态保持机制确保压缩后对话连贯性。

## 2. 核心文件结构

```
src/
├── services/compact/                    # 核心压缩引擎 (11 files)
│   ├── autoCompact.ts                   # 自动压缩触发逻辑与阈值
│   ├── compact.ts                       # compactConversation() 编排器
│   ├── prompt.ts                        # 压缩摘要 Prompt 模板
│   ├── microCompact.ts                  # 预压缩工具结果清理
│   ├── apiMicrocompact.ts               # 服务端上下文管理 (clear_tool_uses/clear_thinking)
│   ├── grouping.ts                      # 按 API 轮次边界分组消息
│   ├── postCompactCleanup.ts            # 压缩后缓存/状态清理
│   ├── sessionMemoryCompact.ts          # 会话记忆压缩（轻量替代）
│   ├── compactWarningState.ts           # UI 警告状态管理
│   ├── compactWarningHook.ts            # React Hook
│   └── timeBasedMCConfig.ts             # 基于时间间隔的微压缩配置
├── utils/
│   ├── tokens.ts                        # Token 计数工具
│   └── context.ts                       # 上下文窗口大小解析
├── commands/compact/
│   └── compact.ts                       # /compact 斜杠命令处理
├── components/
│   └── TokenWarning.tsx                 # UI 警告组件
├── context/                             # React UI 上下文（非压缩系统）
│   ├── stats.tsx                        # 指标存储
│   └── ...                              # 模态/覆盖层/通知等 UI 上下文
└── context.ts                           # 系统上下文构建器（git status, CLAUDE.md）
```

## 3. Token 实时追踪

**文件**: `src/utils/tokens.ts`

### 3.1 混合追踪机制

`tokenCountWithEstimation()` 使用"API 实际值 + 字符估算"混合方式：

```
1. 从消息数组末尾向前查找最近一条有 API usage 数据的助手消息
   usage = input_tokens + cache_creation_input_tokens
         + cache_read_input_tokens + output_tokens

2. 对该消息之后新增的消息（尚未发送到 API）
   使用字符级粗略估算：roughTokenCountEstimationForMessages()

3. 对并行工具调用（共享同一 message.id 的多条助手记录）
   回退到第一个同组消息，避免漏计

4. 返回： API实际值 + 新消息估算值
```

### 3.2 辅助函数

| 函数 | 描述 |
|------|------|
| `tokenCountFromLastAPIResponse()` | 仅返回最近 API 响应的原始 Token 数（用于压缩后指标） |
| `roughTokenCountEstimationForMessages()` | 基于字符数的粗略估算 |

## 4. 上下文窗口计算

**文件**: `src/utils/context.ts` + `src/services/compact/autoCompact.ts`

### 4.1 原始上下文窗口

`getContextWindowForModel()` 的解析优先级：

```
1. CLAUDE_CODE_MAX_CONTEXT_TOKENS 环境变量（ant-only）
2. 模型名称含 [1m] 后缀 → 1,000,000
3. 模型能力注册表 getModelCapability().max_input_tokens
4. CONTEXT_1M_BETA_HEADER → 1,000,000（支持的模型）
5. Sonnet 1M 实验组
6. ant 专用模型注册表
7. 回退默认值：200,000
```

### 4.2 有效上下文窗口

```
effectiveWindow = rawContextWindow - min(maxOutputTokens, 20_000)
```

20,000 Token 预留基于压缩摘要输出的 p99.99 为 17,387 Token。

### 4.3 阈值体系（200K 模型示例）

| 阈值 | 计算公式 | 200K 模型值 | 含义 |
|------|----------|-------------|------|
| 有效窗口 | raw - 20K | ~180K | 可用上下文空间 |
| 警告阈值 | autoCompact - 20K | ~147K | 显示 UI 警告 |
| 自动压缩 | effective - 13K | ~167K | 触发自动压缩 |
| 阻塞限制 | effective - 3K | ~177K | 阻止输入直到压缩完成 |

## 5. 自动压缩触发

**文件**: `src/services/compact/autoCompact.ts`

### 5.1 触发条件

`shouldAutoCompact()` 检查：

```
✓ 不是递归调用（非 session_memory 或 compact querySource）
✓ 未被禁用（DISABLE_COMPACT / DISABLE_AUTO_COMPACT / 用户设置）
✓ 未被 reactive-compact-only 或 context-collapse 模式抑制
✓ Token 数超过自动压缩阈值
```

### 5.2 断路器

连续 **3 次失败**后，自动压缩在当前会话中停止重试，避免对 API 发送注定失败的请求。

## 6. 压缩流程

**文件**: `src/services/compact/compact.ts`

`compactConversation()` 完整流程：

```
1. 测量 ─ tokenCountWithEstimation(messages)

2. Hook ─ 执行 PreCompact hooks，合并自定义指令

3. 构建 Prompt ─ getCompactPrompt(customInstructions)
   │  9 节摘要模板：
   │  ├── Primary Request（主要需求）
   │  ├── Key Technical Concepts（关键技术概念）
   │  ├── Files/Code（文件/代码）
   │  ├── Errors/Fixes（错误/修复）
   │  ├── Problem Solving（问题解决）
   │  ├── All User Messages（所有用户消息）
   │  ├── Pending Tasks（待处理任务）
   │  ├── Current Work（当前工作）
   │  └── Optional Next Step（可选下一步）

4. 流式生成摘要 ─ streamCompactSummary()
   │  使用 queryModelWithStreaming()
   │  复用主对话的 prompt cache
   │  最大输出：20,000 tokens
   │  模型被显式告知 NO TOOLS —— 纯文本
   │  输出格式：<analysis>分析</analysis> + <summary>摘要</summary>

5. Prompt-too-long 重试 ─ 如果压缩请求本身过大
   │  truncateHeadForPTLRetry() 截断最旧的 API 轮次组
   │  最多重试 3 次

6. 解析摘要 ─ 提取 <summary> 块，丢弃 <analysis> 草稿

7. 构建压缩后附件（并行）
   ├── 文件重附加（最近读取的 5 个文件，每个 ≤5,000 tokens，总预算 50,000）
   ├── Plan 附件 + Plan 模式状态
   ├── 技能附件（每技能 ≤5,000 tokens，总预算 25,000）
   ├── 延迟工具/代理列表/MCP 指令 delta
   └── SessionStart hooks 重执行

8. 创建边界标记 ─ SystemCompactBoundaryMessage
   │  记录压缩前 Token 数、已发现工具名

9. 通知 Prompt Cache 系统 ─ notifyCompaction() 重置基线

10. 返回 CompactionResult
    { boundary, summaries, attachments, hookResults, tokenCounts }
```

## 7. 压缩后状态保持

### 7.1 保留的状态

| 状态 | 保持方式 |
|------|----------|
| 压缩边界标记 | `SystemCompactBoundaryMessage`，含压缩前 Token 数和工具名 |
| 对话摘要 | LLM 生成的摘要消息 |
| 最近读取的文件 | 最多 5 个文件重附加（总预算 50K tokens） |
| 技能内容 | 已调用技能重注入（总预算 25K tokens） |
| Plan 文件 | 当前 plan 文件内容 |
| Plan 模式 | 如果活跃则注入继续指令 |
| 延迟工具定义 | 从当前状态重宣告 |
| 代理列表 | 子代理列表重宣告 |
| MCP 指令 | MCP 服务器指令重宣告 |
| SessionStart hooks | 重执行并注入结果 |
| 已发现工具 | 存储在边界元数据中确保 schema 过滤正常 |

### 7.2 清除的状态

| 状态 | 原因 |
|------|------|
| `readFileState` 缓存 | 强制重新读取文件（可能已变） |
| `loadedNestedMemoryPaths` | 强制重新加载记忆文件 |
| 微压缩状态 | 重置工具结果清理追踪 |
| 分类器审批 | 清除 auto-mode 历史审批 |
| 文件内容缓存 | 清除归因追踪缓存 |
| 系统 prompt sections | 清除已缓存的 system prompt 片段 |

## 8. 多层压缩策略

### 8.1 微压缩（Microcompact）

**文件**: `src/services/compact/microCompact.ts`

在完整压缩之前和常规轮次中，对特定工具的结果进行清理：

```
可压缩工具（COMPACTABLE_TOOLS）：
Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write

策略：
├── 超过 Token 阈值的工具结果被清除
├── 图片块估算为 2,000 tokens
└── 仅在结果足够旧时清除
```

### 8.2 基于时间的微压缩

**文件**: `src/services/compact/timeBasedMCConfig.ts`

当距离最后一条助手消息的时间间隔超过阈值（默认 60 分钟，匹配服务端缓存 TTL），主动清除旧的工具结果——因为 prompt cache 已确定过期。

### 8.3 API 端上下文管理

**文件**: `src/services/compact/apiMicrocompact.ts`

服务端策略通过 `context_management` 配置发送：

| 策略 | 描述 |
|------|------|
| `clear_tool_uses_20250919` | 服务端清除旧的 tool_use/tool_result 对 |
| `clear_thinking_20251015` | 清除之前轮次的 thinking 块 |

### 8.4 会话记忆压缩

**文件**: `src/services/compact/sessionMemoryCompact.ts`

`/compact` 优先尝试的轻量压缩：使用运行中的会话记忆（增量提取的摘要）替代完整重总结。保留最近消息（默认 10K-40K tokens、至少 5 条文本块消息）。

### 8.5 完整压缩

最重量级的压缩方式，调用 LLM 生成完整摘要。作为回退策略使用。

## 9. 上下文节省优化

| 优化 | 节省效果 |
|------|----------|
| Read 去重 | 文件未变 → 返回 `[File content unchanged]` 存根 |
| 技能描述限制 | 初始 prompt 中每技能 250 字符 |
| MCP 延迟加载 | 工具描述超 10% 上下文时进入延迟模式，按需加载 |
| 图片/文档剥离 | 压缩 API 调用前替换为 `[image]`/`[document]` 标记 |
| 重注入附件剥离 | 技能发现/列表附件在摘要前过滤（反正会重注入） |
| `@` 文件原始字符串 | 不做 JSON 转义，减少 Token 开销 |
| 日期移出 system prompt | 提高 prompt cache 命中率 |

## 10. 手动压缩 (/compact)

**文件**: `src/commands/compact/compact.ts`

优先级链：

```
1. 会话记忆压缩（无自定义指令时优先）
   └── trySessionMemoryCompaction() — 轻量、快速

2. 响应式压缩（响应式模式下）
   └── 走响应式压缩路径

3. 完整压缩（回退）
   ├── 先执行微压缩 microcompactMessages()
   └── 再调用 compactConversation(isAutoCompact: false)
```

可传递自定义指令：`/compact focus on test output`

压缩后清理：
- 抑制上下文警告
- 清除所有相关缓存
- 强制重评估 CLAUDE.md
- 设置 bootstrap 状态标志

## 11. 设计要点

1. **混合追踪**: API 实际值 + 字符估算，平衡精度与性能
2. **多层压缩**: 微压缩 → 会话记忆 → 完整压缩，逐级加重
3. **断路器**: 3 次失败后停止重试，避免 API 浪费
4. **状态保持**: 压缩后精心重注入关键状态（文件、技能、plan、hooks）
5. **Prompt Cache 利用**: 压缩摘要复用主对话 prompt cache
6. **Prompt-too-long 容错**: 自动截断最旧轮次，最多重试 3 次
7. **20K Token 预留**: 基于 p99.99 统计数据的科学预留

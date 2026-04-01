# 02 - 会话管理（Session Management）

## 1. 模块概述

会话管理系统负责 Claude Code 对话的完整生命周期：创建、持久化、恢复、崩溃恢复。采用 JSONL（JSON Lines）格式实现高效的增量追加写入，支持跨项目会话查找、中断检测与自动恢复、并发会话 PID 追踪等高级特性。

## 2. 核心文件结构

```
src/
├── bootstrap/
│   └── state.ts                    # 全局状态 + Session ID 生成
├── utils/
│   ├── sessionStorage.ts           # JSONL 持久化、Project 类、写缓冲
│   ├── sessionStoragePortable.ts   # 可移植工具：lite 读取、字段提取
│   ├── sessionRestore.ts           # 恢复会话状态（文件历史、agent 等）
│   ├── conversationRecovery.ts     # 核心恢复逻辑、消息反序列化、中断检测
│   ├── listSessionsImpl.ts         # 会话列表、picker/SDK 用
│   ├── concurrentSessions.ts       # PID 文件注册、过期进程检测
│   ├── sessionActivity.ts          # 活跃操作引用计数
│   └── gracefulShutdown.ts         # 优雅退出、数据刷盘
├── commands/
│   └── resume/
│       ├── index.ts                # /resume 命令定义
│       └── resume.tsx              # 恢复 UI（picker 组件）
├── services/api/
│   ├── client.ts                   # X-Claude-Code-Session-Id header
│   └── sessionIngress.ts           # 远程会话日志持久化 + 409 恢复
├── history.ts                      # 输入历史（独立于会话 transcript）
└── context.ts                      # 系统/用户上下文生成
```

## 3. 会话创建

### 3.1 Session ID 生成

**文件**: `src/bootstrap/state.ts`

会话 ID 使用 `node:crypto` 的 `randomUUID()` 生成，类型为品牌类型 `SessionId`：

```typescript
// src/types/ids.ts
type SessionId = string & { __brand: 'SessionId' };

// src/bootstrap/state.ts
sessionId: randomUUID() as SessionId,
```

### 3.2 全局 State 对象

`State` 对象包含完整的会话元数据：

| 字段 | 类型 | 描述 |
|------|------|------|
| `sessionId` | `SessionId` | UUID 会话标识符 |
| `originalCwd` | `string` | 启动时工作目录（`realpathSync` 解析） |
| `projectRoot` | `string` | 项目根目录 |
| `startTime` | `number` | 会话启动时间戳 |
| `lastInteractionTime` | `number` | 最后交互时间 |
| `sessionSource` | `string` | 会话来源（CLI / SDK 等） |
| `clientType` | `string` | 默认 `'cli'` |
| `parentSessionId` | `SessionId?` | 父会话 ID（plan mode 等） |
| `sessionProjectDir` | `string?` | 会话 `.jsonl` 文件所在目录 |
| `mainThreadAgentType` | `string?` | 来自 `--agent` 标志或设置 |
| `teleportedSessionInfo` | `object?` | 远程迁移会话信息 |

### 3.3 会话切换

`switchSession()` 函数允许运行时切换活跃会话（用于 `/resume` 和 `--continue`）：

```typescript
export function switchSession(
  sessionId: SessionId,
  projectDir: string | null = null,
): void { ... }
```

切换时发出 `sessionSwitched` 信号，通知监听器（如 `concurrentSessions.ts`）更新 PID 文件。

## 4. 存储机制

### 4.1 目录结构

```
~/.claude/projects/
└── <sanitized-cwd>/                    # 按项目路径隔离
    ├── <session-uuid>.jsonl            # 主对话 transcript
    └── <session-uuid>/
        ├── subagents/
        │   └── agent-<agent-id>.jsonl  # 子代理 transcript
        └── remote-agents/
            └── remote-agent-<task-id>.meta.json
```

**路径生成逻辑**:
- `getProjectsDir()` → `~/.claude/projects/`
- `getProjectDir(cwd)` → `join(projectsDir, sanitizePath(cwd))`（memoized）
- `getTranscriptPath()` → `join(projectDir, sessionId + '.jsonl')`

### 4.2 JSONL 格式

每行为一个 JSON 序列化的 `Entry` 对象，支持以下类型：

| Entry 类型 | 描述 |
|-----------|------|
| `user` | 用户消息 |
| `assistant` | 助手回复 |
| `attachment` | 附件（文件、技能状态等） |
| `system` | 系统消息 |
| `customTitle` | 用户自定义标题 |
| `tag` | 会话标签 |
| `gitBranch` | Git 分支信息 |
| `lastPrompt` | 最后一条提示 |
| `summary` | 压缩摘要 |
| `worktreeSession` | 工作树会话标记 |

### 4.3 写缓冲机制

`Project` 类实现了高性能写缓冲：

```
写入请求 → enqueueWrite() → 每文件写入队列
                                    │
                          100ms 定时器触发 drain
                                    │
                          fsAppendFile() 批量写入
                          (mode: 0o600, 最大 100MB/chunk)
```

**关键参数**:
- `FLUSH_INTERVAL_MS = 100` — 刷盘间隔
- `MAX_CHUNK_BYTES = 100MB` — 单次最大写入量
- 目录创建延迟到首次写入失败时
- 关闭时 cleanup handler 刷盘 + 重追加元数据

### 4.4 元数据尾部重追加

关闭时，会话元数据（customTitle、tag 等）被重追加到 transcript 文件末尾。原因：`readSessionLite()` 只读取文件尾部 64KB 来快速构建 picker 列表，元数据必须在这个窗口内。

### 4.5 输入历史

输入历史独立存储，不与会话 transcript 混合：

```
~/.claude/history.jsonl    # 全局，跨项目共享
```

## 5. 会话恢复 (/resume)

### 5.1 触发方式

| 方式 | 行为 |
|------|------|
| `claude --resume` | 恢复最近的会话 |
| `/resume` | 打开会话选择器 picker |
| `/resume <id>` | 恢复指定 ID 的会话 |
| `claude-cli://open?q=...` | Deep link 恢复 |
| `/continue`（别名） | 等同于 `/resume` |

### 5.2 会话列表构建

**两轮扫描法**（`listSessionsImpl.ts`）:

```
第一轮：stat-only 扫描
  → 获取文件 mtime，按修改时间排序
  → 成本极低，仅 stat() 调用

第二轮：head/tail 读取
  → readSessionLite()：读取头部 + 尾部各 64KB
  → parseSessionInfoFromLite()：提取元数据
     (customTitle, aiTitle, firstPrompt, gitBranch, cwd, tag, createdAt, lastPrompt, summary)
  → 排除 sidechain 会话（检查第一行 "isSidechain":true）
```

### 5.3 Picker UI

**文件**: `src/commands/resume/resume.tsx`

1. 加载同仓库会话（`loadSameRepoMessageLogs()`）或所有项目（`loadAllProjectsMessageLogs()`）
2. 通过 `filterResumableSessions()` 过滤（排除 sidechain、当前会话）
3. 渲染 `LogSelector` 组件，支持：
   - Agent 搜索（`agenticSessionSearch`）
   - 跨项目检测（`checkCrossProjectResume`）
   - 同仓库 worktree 直接恢复
   - 不同项目会话 → 复制命令到剪贴板

### 5.4 会话加载流程

**文件**: `src/utils/conversationRecovery.ts` → `loadConversationForResume()`

```
1. 确定来源
   ├── undefined (--continue) → 加载最近会话，跳过活跃后台/daemon 会话
   ├── session ID string → 按 ID 查找
   ├── LogOption 对象 → 直接使用
   └── .jsonl 文件路径 → 直接加载

2. 加载完整消息
   └── 如果是 lite log → 加载完整 JSONL

3. 状态迁移与清理
   ├── copyPlanForResume()          — 携带 plan 文件
   ├── copyFileHistoryForResume()   — 携带文件历史
   └── restoreSkillStateFromMessages() — 从 invoked_skills 附件恢复技能状态

4. 消息反序列化 (deserializeMessagesWithInterruptDetection)
   ├── 迁移旧版附件类型
   ├── 过滤无效 permissionMode 值
   ├── 过滤未解决的 tool_use（无匹配 tool_result）
   ├── 过滤孤立的 thinking-only 消息
   ├── 过滤纯空白助手消息
   ├── 检测中断类型：
   │   ├── none           — 正常结束
   │   ├── interrupted_prompt — 用户提交了 prompt 但助手未开始响应
   │   └── interrupted_turn  — 助手正在执行工具调用时进程死亡
   └── 对于 interrupted_turn → 追加合成消息 "Continue from where you left off."

5. 运行 SessionStart hooks (带 resume 标记)

6. 返回消息 + 元数据
   (agent context, custom title, tag, worktree session, PR info 等)
```

### 5.5 压缩会话恢复

压缩会话通过 `parentUuid` 链自然处理：

- `SystemCompactBoundaryMessage` 标记压缩发生的位置
- `buildConversationChain()` 从叶节点沿 `parentUuid` 链回溯构建对话
- 压缩后新的摘要消息成为新的根，替换旧链
- 技能状态通过 `invoked_skills` 附件消息跨压缩保持

## 6. 崩溃恢复

### 6.1 中断检测

**三种中断类型**（`detectTurnInterruption()`）:

| 类型 | 条件 | 恢复行为 |
|------|------|----------|
| `none` | 正常结束 | 无特殊处理 |
| `interrupted_prompt` | 最后一条是用户 prompt | 原样恢复，用户 prompt 已在 |
| `interrupted_turn` | 最后一条用户消息含 `tool_result` | 追加 "Continue from where you left off." |

### 6.2 未解决工具调用过滤

`filterUnresolvedToolUses()` 移除有 `tool_use` 块但无匹配 `tool_result` 的助手消息，防止 API 报错。

### 6.3 优雅退出

**文件**: `src/utils/gracefulShutdown.ts`

- 刷盘所有待写入数据
- 清除 iTerm2 进度条和 OSC 21337 tab 状态
- 删除 PID 文件

### 6.4 安全限制

- `MAX_TRANSCRIPT_READ_BYTES = 50MB` — 防止读取超大会话文件导致 OOM
- `MAX_TOMBSTONE_REWRITE_BYTES` — 限制墓碑重写大小

## 7. 并发会话管理

**文件**: `src/utils/concurrentSessions.ts`

### PID 文件

每个活跃会话在 `~/.claude/sessions/` 下注册 PID 文件：

```json
{
  "pid": 12345,
  "sessionId": "abc-def-123",
  "cwd": "/Users/user/project",
  "startedAt": "2026-03-31T10:00:00Z",
  "kind": "interactive",
  "entrypoint": "cli"
}
```

- `registerSession()` — 写入 PID 文件
- `isProcessRunning()` — 检测 PID 是否存活，发现过期 PID 文件
- 退出时 cleanup handler 删除 PID 文件

### 后台会话排除

`--continue` 恢复时通过 UDS (`listAllLiveSessions()`) 发现活跃后台/daemon 会话并排除，防止误恢复正在运行的会话。

## 8. Session ID 在 API 中的传播

### HTTP Headers

每次 API 请求都携带会话标识：

```typescript
// src/services/api/client.ts
headers: {
  'X-Claude-Code-Session-Id': getSessionId(),
  'x-claude-remote-session-id': remoteSessionId,  // 远程会话
}
```

### API 请求元数据

```typescript
// src/services/api/claude.ts
metadata: {
  session_id: getSessionId(),
}
```

### 远程会话日志

**文件**: `src/services/api/sessionIngress.ts`

- 使用 `Last-Uuid` header 确保追加顺序
- 409 Conflict 恢复：检查 `x-last-uuid`，如果服务端已有该条目则更新本地 map，否则采用服务端 UUID 重试

## 9. 设计要点

1. **增量追加**: JSONL 格式天然支持追加写入，无需解析整个文件
2. **写缓冲**: 100ms 批量刷盘，平衡性能与数据安全
3. **Lite 读取**: 64KB head+tail 窗口快速构建会话列表
4. **元数据尾追加**: 确保元数据在 lite 读取窗口内
5. **中断容忍**: 三种中断类型检测 + 自动恢复
6. **PID 追踪**: 防止恢复活跃会话
7. **50MB 安全限制**: 防止超大 transcript OOM

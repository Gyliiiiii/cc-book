# 13 - GitHub 自动化（GitHub Automation）

## 1. 模块概述

Claude Code 的 GitHub 自动化系统基于 GitHub Actions 工作流，使用 `anthropics/claude-code-action@v1` 作为 AI 引擎，实现 Issue 自动分类、语义去重、生命周期管理、@Claude 提及响应、跨仓库事件分发等自动化功能。这是 Claude Code 项目的第二大使用场景（第一是交互式 CLI）。

## 2. 六大子系统

### 2.1 AI 驱动的 Issue 分类

**工作流**: `.github/workflows/claude-issue-triage.yml`

```
触发器: issues.opened / issue_comment.created
    ↓
claude-code-action@v1
    ↓
/triage-issue 命令 (.claude/commands/triage-issue.md)
    ↓
Claude Opus 标签分类
    ↓
scripts/edit-issue-labels.sh 应用标签
```

**关键特性**:
- **仅标签操作** — 明确禁止发布评论，只能添加/移除标签
- **安全包装** — `scripts/gh.sh`（97 行）包装 GitHub CLI，验证并限定所有调用范围
- **防跨仓库** — 拒绝包含 `repo:`、`org:`、`user:` 限定符的搜索查询
- **标签类别** — `bug`、`enhancement`、`documentation`、`model` 等
- **并发控制** — `cancel-in-progress: true`，按 Issue 限制并发

### 2.2 Issue 语义去重

**工作流**: `.github/workflows/claude-dedupe-issues.yml`

```
触发器: issues.opened / workflow_dispatch (手动)
    ↓
/dedupe 命令
    ↓
5 个并行搜索 Agent 扫描现有 Issue
    ↓
误报过滤阶段
    ↓
发现重复 → 发布评论 + 应用 duplicate 标签
    ↓
auto-close-duplicates.yml (每日定时)
    ↓
scripts/auto-close-duplicates.ts
    ↓
3 天等待期后关闭确认的重复
```

**关键特性**:
- **多代理编排** — 5 个并行搜索代理语义匹配
- **两阶段验证** — 搜索 + 误报过滤
- **3 天宽限期** — 给报告者争议时间
- **回填系统** — `backfill-duplicate-comments.yml` 处理历史 Issue

### 2.3 Issue 生命周期管理

**工作流**: `.github/workflows/sweep.yml`

```
触发器: 定时 Cron + workflow_dispatch
    ↓
scripts/sweep.ts (Bun 运行)
    ↓
markStale() → 标记不活跃 Issue
    ↓
scripts/lifecycle-comment.ts → 发布生命周期评论
```

**关键特性**:
- **并发限制** — `daily-issue-sweep` 组，同时只运行一个
- **权限** — `issues: write`
- **单一真相源** — `issue-lifecycle.ts` 定义标签超时配置

**生命周期标签**:

| 标签 | 描述 |
|------|------|
| `needs-repro` | 需要复现步骤 |
| `needs-info` | 需要更多信息 |
| `stale` | 不活跃标记 |
| `autoclose` | 自动关闭标记 |
| `invalid` | 无效 Issue |

**相关工作流**:
- `lock-closed-issues.yml`（每日）— 锁定旧的已关闭 Issue
- `remove-autoclose-label.yml` — 社区活动可以"拯救" Issue

### 2.4 @Claude 提及

**工作流**: `.github/workflows/claude.yml`

```
触发器: Issue/PR 评论中 @claude 提及
    ↓
claude-code-action@v1
    ↓
通用 AI 响应（自由形式对话）
```

与分类系统不同 — 这是自由形式的 Claude 交互，不是结构化标签分类。

### 2.5 跨仓库事件分发

**工作流**: `.github/workflows/issue-opened-dispatch.yml`

```
触发器: issues.opened
    ↓
gh api POST /repos/{target}/dispatches
    ↓
repository_dispatch 事件发送到目标仓库
    ↓
传递元数据: issue_number, repo, author
```

**关键特性**:
- 目标仓库通过 `ISSUE_OPENED_DISPATCH_TARGET_REPO` 配置
- 静默失败（exit 0 on failure）— 不在 Actions 日志中产生噪音
- 1 分钟超时快速失败

**用途**:
- 触发私有基础设施仓库的工作流
- 同步到外部追踪系统
- 跨仓库自动化管道

### 2.6 事件日志与分析

**工作流**: `.github/workflows/log-issue-events.yml`

```
触发器: Issue 事件
    ↓
Statsig 集成
    ↓
记录事件 + 元数据
```

记录的元数据：`issue_number`、`triggered_by`（如 `issues`、`workflow_dispatch`）、`workflow_run_id`。

去重工作流也独立记录到 Statsig。

## 3. 执行模型

所有 AI 工作流使用统一的 Action 接口：

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    command: "/triage-issue"  # 或自由文本 prompt
```

## 4. CLI 集成

CLI 中的 `WorkflowMultiselectDialog` 组件提供两个可安装的工作流：

| 工作流 | 描述 |
|--------|------|
| @Claude Code | 在 Issue/PR 评论中标记 @claude |
| Claude Code Review | 新 PR 自动代码审查 |

更多示例：`github.com/anthropics/claude-code-action/blob/main/examples/`

## 5. 安全考量

| 措施 | 描述 |
|------|------|
| `scripts/gh.sh` 包装 | 验证并限定 GitHub CLI 调用范围 |
| 防跨仓库查询 | 拒绝 `repo:`/`org:`/`user:` 限定符 |
| 最小权限 | 每个工作流只请求必要权限（`issues: write`） |
| 并发控制 | `cancel-in-progress` 防止重复执行 |
| 静默失败 | 跨仓库分发不阻塞主流程 |

## 6. 设计要点

1. **claude-code-action**: 统一的 GitHub Action 作为 AI 引擎
2. **多代理去重**: 5 个并行代理 + 误报过滤提高准确率
3. **3 天宽限期**: 去重后不立即关闭，给报告者争议机会
4. **最小权限**: 分类只改标签不发评论
5. **安全 CLI 包装**: 防止 AI 执行越权 GitHub 操作
6. **Statsig 分析**: 完整的事件追踪和分析支持

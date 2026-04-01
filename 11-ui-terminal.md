# 11 - UI/UX 与终端集成（UI/UX & Terminal Integration）

## 1. 模块概述

Claude Code 使用深度定制的 React + Ink 框架实现终端 UI 渲染。系统包含自定义 React Reconciler、基于 Yoga 的 Flex 布局、17 个键绑定上下文、Vim 模式、可配置状态栏，以及 VS Code / IDE 集成。

## 2. 核心文件结构

```
src/
├── ink.ts                           # 公共 API：render(), createRoot()
├── ink/                             # (96 files) 自定义 Ink 渲染引擎
│   ├── ink.tsx                      # Ink 类：渲染管线核心
│   ├── reconciler.ts                # React Reconciler (ConcurrentRoot)
│   └── ...                          # 布局、输出缓冲、差异、鼠标等
├── components/                      # (389 files) UI 组件库
│   ├── App.tsx                      # 顶层 React 组件
│   ├── StatusLine.tsx               # 状态栏
│   ├── TokenWarning.tsx             # Token 警告
│   ├── VimTextInput.tsx             # Vim 输入组件
│   ├── ContextSuggestions.tsx       # 文件自动补全
│   ├── IdeAutoConnectDialog.tsx     # IDE 连接对话框
│   └── ...                          # 150+ 组件
├── keybindings/                     # (14 files) 快捷键系统
│   ├── defaultBindings.ts           # 100+ 默认绑定
│   ├── schema.ts                    # Schema + 动作注册表
│   ├── KeybindingContext.tsx         # React 上下文提供器
│   ├── KeybindingProviderSetup.tsx   # 提供器设置
│   ├── useKeybinding.ts             # 消费 Hook
│   ├── loadUserBindings.ts          # 加载用户覆盖
│   ├── parser.ts                    # 键击模式解析器
│   └── validate.ts                  # 验证逻辑
├── vim/                             # (5 files) Vim 模式
│   ├── types.ts                     # 类型定义
│   ├── transitions.ts               # 状态机：模式转换
│   ├── motions.ts                   # 移动命令 (w, b, e, 0, $)
│   ├── operators.ts                 # 操作命令 (d, c, y)
│   └── textObjects.ts              # 文本对象 (iw, aw, i", a")
├── screens/                         # 屏幕/页面
│   └── REPL.tsx                     # 主 REPL 屏幕
└── outputStyles/
    └── ...                          # 输出风格
```

## 3. 渲染架构

```
┌─────────────────────────────────────────────┐
│             React Component Tree             │
│  App → FpsMetricsProvider → StatsProvider    │
│      → AppStateProvider → REPL              │
├─────────────────────────────────────────────┤
│           React Reconciler                   │
│  react-reconciler + ConcurrentRoot           │
│  虚拟 DOM: DOMElement / TextNode             │
├─────────────────────────────────────────────┤
│           Ink 渲染引擎                       │
│  Yoga Flex Layout → Output Buffer            │
│  → Screen Diff → Terminal Write              │
├─────────────────────────────────────────────┤
│           终端层                             │
│  Alt-screen · 鼠标追踪 · Kitty 键盘协议     │
│  帧率节流 (FRAME_INTERVAL_MS)               │
└─────────────────────────────────────────────┘
```

### 3.1 React Reconciler

**文件**: `src/ink/reconciler.ts`

- 使用 `react-reconciler` 库 + `ConcurrentRoot` 模式
- 维护终端元素的虚拟 DOM（`DOMElement`、`TextNode`）
- 基于 Yoga 的 Flex 布局
- 操作：`createNode`、`createTextNode`、`appendChildNode`、`removeChildNode`、`setAttribute`、`setStyle`

### 3.2 Ink 引擎

**文件**: `src/ink/ink.tsx`

- `Ink` 类管理渲染管线
- 流程：React Fiber Tree → Yoga 布局 → 节点渲染到 Output 缓冲 → 与上一帧 diff → 写入终端
- 支持 alt-screen 模式、鼠标追踪（启用/禁用）、Kitty 键盘协议

### 3.3 公共 API

**文件**: `src/ink.ts`

导出：
- `render()`、`createRoot()` — 包裹在 `ThemeProvider` 中
- 主题组件：`Box`、`Text`（设计系统）
- 基础组件：`BaseBox`、`BaseText`、`Button`、`Link`、`Newline`、`Spacer`
- Hooks：`useInput`、`useApp`、`useStdin`、`useSelection`、`useTerminalViewport`
- 事件：`ClickEvent`、`InputEvent`、`TerminalFocusEvent`

### 3.4 高级终端特性

| 特性 | 描述 |
|------|------|
| 文本选择 | 选中区域 + 复制 |
| 搜索高亮 | 匹配结果高亮显示 |
| 鼠标命中测试 | 点击/悬停 hit-testing |
| 滚动处理 | 滚动事件处理 |
| 超链接 | 终端超链接支持 |
| 双向文本 | BiDi 文本支持 |
| Tab Stop | Tab 位处理 |
| 焦点事件 | 终端焦点切换检测 |

## 4. 输入系统

### 4.1 17 个键绑定上下文

```
Global · Chat · Autocomplete · Confirmation · Help
Transcript · HistorySearch · Task · ThemePicker · Settings
Tabs · Attachments · Footer · MessageSelector · MessageActions
DiffDialog · ModelPicker · Select · Plugin
```

### 4.2 默认绑定

**文件**: `src/keybindings/defaultBindings.ts`

100+ 默认绑定，按上下文组织。平台感知：

| 操作 | macOS/Linux | Windows |
|------|-------------|---------|
| 图片粘贴 | `ctrl+v` | `alt+v`（`ctrl+v` 是系统粘贴） |
| 模式切换 | `shift+tab`（VT 模式） | `meta+m`（无 VT） |

VT 模式检测：Bun ≥ 1.2.23 或 Node ≥ 22.17.0 / 24.2.0

### 4.3 核心快捷键

| 快捷键 | 动作 |
|--------|------|
| `Enter` | 提交 |
| `Shift+Enter` / `Meta+Enter` | 换行 |
| `Tab` | 接受自动补全 |
| `Ctrl+C` | 取消（保留，不可重绑） |
| `Ctrl+D` | 退出（保留，不可重绑） |
| `Up/Down` | 历史导航 |
| `Ctrl+F` | 后台化当前任务 |
| `Escape` | 中断当前操作 |

### 4.4 用户自定义

通过 `~/.claude/keybindings.json` 覆盖：

```json
{
  "bindings": {
    "chat:submit": "ctrl+enter",
    "command:help": "ctrl+h",
    "app:interrupt": null  // 解绑
  }
}
```

绑定目标：内置动作（`app:interrupt`）、斜杠命令（`command:help`）、`null`（解绑）。

## 5. Vim 模式

**文件**: `src/vim/`

完整的 Vim 文本编辑模式实现：

| 文件 | 描述 |
|------|------|
| `types.ts` | Normal/Insert/Visual 模式类型 |
| `transitions.ts` | 模式转换状态机 |
| `motions.ts` | 移动命令：`w`、`b`、`e`、`0`、`$`、`f`、`t` 等 |
| `operators.ts` | 操作命令：`d`(删除)、`c`(修改)、`y`(复制) |
| `textObjects.ts` | 文本对象：`iw`、`aw`、`i"`、`a"`、`i(`、`a(` 等 |

`VimTextInput` 组件（`src/components/VimTextInput.tsx`）集成到 REPL 输入。

## 6. 状态栏

**文件**: `src/components/StatusLine.tsx`

可配置的持久底部状态栏，显示：

| 信息 | 描述 |
|------|------|
| 模型名称 | 当前使用的模型 |
| 权限模式 | default/plan/auto 等 |
| 工作目录 | 当前 CWD |
| 会话名称 | 自定义或自动标题 |
| 上下文使用 | `context_window.used_percentage` / `remaining_percentage` |
| 费用/Token | 费用和 Token 统计 |
| 速率限制 | 当前速率限制状态 |
| Vim 模式 | Normal/Insert/Visual 指示器 |
| 输出风格 | 当前输出风格 |
| Worktree | Worktree 会话信息 |

支持通过 settings 中的 `statusLine` 命令进行 Hook 自定义。

Kairos/assistant 模式下隐藏（状态栏反映的是 REPL 进程而非 agent 子进程）。

## 7. VS Code 集成

通过 MCP 的 SDK 连接实现：

| 文件 | 描述 |
|------|------|
| `services/mcp/vscodeSdkMcp.ts` | `notifyVscodeFileUpdated()` — BashTool/EditTool/WriteTool 修改文件后调用 |
| `components/IdeAutoConnectDialog.tsx` | IDE 连接状态对话框 |
| `components/IdeOnboardingDialog.tsx` | IDE 入门引导 |
| `components/IdeStatusIndicator.tsx` | IDE 连接状态指示器 |

## 8. Deep Linking

**文件**: `src/utils/deepLink/registerProtocol.ts`

协议：`claude-cli://open?q=...`

- 支持最多 5,000 字符
- 长 prompt 显示 "scroll to review" 安全警告
- 尊重用户首选终端

## 9. 文件自动补全（@ 提及）

- `@` 触发文件路径补全
- 通过 `fileSuggestion` 设置自定义补全命令
- 排除 `.jj`（Jujutsu）和 `.sl`（Sapling）元数据目录
- 使用原始字符串内容（无 JSON 转义）减少 Token 开销
- UI 组件：`src/components/ContextSuggestions.tsx`

## 10. 设计要点

1. **自定义 Ink**: 深度定制的 React-for-Terminal 框架，非标准 Ink
2. **ConcurrentRoot**: 使用 React 并发模式提升渲染性能
3. **17 个上下文**: 精细的键绑定上下文隔离
4. **平台感知绑定**: macOS/Windows/Linux 差异化快捷键
5. **完整 Vim**: 不是简化版，支持 motions + operators + text objects
6. **帧率节流**: 避免过度渲染导致终端闪烁
7. **IDE 桥接**: 通过 MCP SDK 连接而非直接 API

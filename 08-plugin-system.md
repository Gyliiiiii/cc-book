# 08 - 插件系统（Plugin System）

## 1. 模块概述

插件系统是 Claude Code 的分发与扩展平台，支持通过 Marketplace、GitHub 仓库、npm 包、本地路径等多种来源安装插件。每个插件可包含命令、代理、技能、钩子、MCP 服务器、LSP 服务器等多种组件类型。系统内置完整的信任与安全模型，包括反冒名、企业策略、自动下架检测。

## 2. 核心文件结构

```
src/
├── plugins/
│   ├── builtinPlugins.ts                # 内置插件注册表
│   └── bundled/
│       └── index.ts                     # 内置插件初始化
├── types/
│   └── plugin.ts                        # BuiltinPluginDefinition, LoadedPlugin, PluginComponent
├── utils/plugins/
│   ├── pluginLoader.ts                  # 插件发现、加载、验证
│   ├── pluginDirectories.ts             # 插件目录配置
│   ├── pluginPolicy.ts                  # 企业策略执行
│   ├── pluginBlocklist.ts               # 下架检测 + 自动卸载
│   ├── marketplaceManager.ts            # Marketplace 管理
│   ├── officialMarketplace.ts           # 官方 Marketplace 常量
│   ├── schemas.ts                       # 名称验证、反冒名模式
│   ├── pluginInstallationHelpers.ts     # 安装辅助函数
│   ├── loadPluginCommands.ts            # 加载命令/技能
│   └── loadPluginHooks.ts              # 加载钩子 + 热重载
├── commands/plugin/
│   ├── index.tsx                        # /plugin 命令入口
│   ├── BrowseMarketplace.tsx            # 浏览可用插件
│   ├── ManagePlugins.tsx                # 管理已安装插件
│   ├── ManageMarketplaces.tsx           # 管理 Marketplace 源
│   ├── AddMarketplace.tsx               # 添加新 Marketplace
│   ├── DiscoverPlugins.tsx              # 插件发现
│   ├── PluginSettings.tsx               # 插件设置
│   ├── PluginOptionsFlow.tsx            # 配置流程
│   ├── ValidatePlugin.tsx               # 插件验证
│   ├── PluginTrustWarning.tsx           # 信任警告 UI
│   └── PluginErrors.tsx                 # 错误显示
└── commands/reload-plugins/
    └── reload-plugins.ts                # /reload-plugins 命令
```

## 3. 插件发现与优先级

### 3.1 发现顺序

```
1. Marketplace 插件    — plugin@marketplace 格式
2. 会话专属插件        — --plugin-dir CLI 标志 / SDK plugins 选项
3. 内置插件            — {name}@builtin 格式
```

### 3.2 插件目录

| 目录 | 条件 |
|------|------|
| `~/.claude/plugins/` | 标准模式 |
| `~/.claude/cowork_plugins/` | `--cowork` 标志 / `CLAUDE_CODE_USE_COWORK_PLUGINS` |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | 环境变量覆盖 |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | 预烘焙容器镜像（支持多目录，PATH 风格分隔） |

### 3.3 Marketplace 文件结构

```
~/.claude/
├── plugins/
│   ├── known_marketplaces.json      # 所有已知 Marketplace 配置
│   ├── marketplaces/                # 缓存的 Marketplace 数据
│   │   ├── my-marketplace.json      # URL 来源的缓存
│   │   └── github-marketplace/      # 克隆的 GitHub 仓库
│   ├── cache/                       # 已安装插件缓存
│   └── data/                        # 持久化的插件数据（跨更新保留）
```

## 4. 七种组件类型

| 组件 | 描述 | 来源 |
|------|------|------|
| `commands` | 自定义斜杠命令（`.md` 文件，旧称，现为 skills） | `commands/` 目录 |
| `agents` | 自定义 AI 代理定义 | `agents/` 目录 |
| `skills` | 可复用技能包（SKILL.md 格式） | `skills/` 目录 |
| `hooks` | 钩子配置（`HooksSettings`） | `hooks/hooks.json` |
| `output-styles` | 自定义输出格式 | 配置文件 |
| `MCP servers` | MCP 服务器定义 | `plugin.json` 或 `.mcp.json` |
| `LSP servers` | Language Server 集成 | 配置文件 |

### 插件目录结构

```
my-plugin/
├── plugin.json          # 可选清单：元数据、设置选项
├── commands/            # 自定义斜杠命令
├── agents/              # 自定义 AI 代理
├── skills/              # SKILL.md 技能包
├── hooks/               # 钩子配置 (hooks.json)
└── .mcp.json            # MCP 服务器配置
```

## 5. 安装来源

| 来源 | 描述 |
|------|------|
| Marketplace (GitHub) | 主要模型。官方 Marketplace: `anthropics/claude-plugins-official` |
| Marketplace (URL) | 从 URL 获取的 JSON 清单 |
| Marketplace (npm) | 通过 Marketplace 条目引用的 npm 包 |
| Marketplace (本地) | 本地文件系统路径 |
| Git 仓库 | 直接 git clone |
| 会话专属 | `--plugin-dir` 加载，仅当前会话有效 |
| 内置 | 编译进 CLI 二进制文件 |

## 6. 信任与安全模型

### 6.1 信任警告

每次插件 UI 交互显示：

> "Make sure you trust a plugin before installing, updating, or using it. Anthropic does not control what MCP servers, files, or other software are included in plugins and cannot verify that they will work as intended or that they won't change."

### 6.2 反冒名保护

**文件**: `src/utils/plugins/schemas.ts`

| 保护 | 描述 |
|------|------|
| 保留名称 | `claude-code-marketplace`, `claude-plugins-official`, `anthropic-marketplace` 等 |
| 名称模式阻断 | `BLOCKED_OFFICIAL_NAME_PATTERN` 阻止 "official-claude-plugins" 等 |
| Unicode 检测 | 阻止非 ASCII 字符防止同形字攻击（如西里尔字母） |
| 官方验证 | 保留名称必须来自 `anthropics` GitHub 组织 |

### 6.3 企业策略

**文件**: `src/utils/plugins/pluginPolicy.ts`

| 控制 | 效果 |
|------|------|
| `isPluginBlockedByPolicy()` | 检查 managed settings 中的强制禁用 |
| `isRestrictedToPluginOnly('skills')` | 锁定技能仅来自插件 |
| `getStrictKnownMarketplaces()` | Marketplace 白名单 |
| `getBlockedMarketplaces()` | Marketplace 黑名单 |

策略阻断的插件无法在任何作用域安装或启用。

### 6.4 下架检测

**文件**: `src/utils/plugins/pluginBlocklist.ts`

```
detectDelistedPlugins()
├── 比较已安装插件与 Marketplace 清单
├── 已下架插件自动卸载（用户可控作用域）
└── 标记防止重新安装
```

### 6.5 启用状态

优先级：用户偏好 > 插件 `defaultEnabled` > `true`

存储位置：`settings.enabledPlugins[pluginId]`

## 7. 管理命令

| 命令 | 别名 | 描述 |
|------|------|------|
| `/plugin` | `/plugins`, `/marketplace` | 主插件管理 UI |
| `/reload-plugins` | — | 重新加载所有插件，报告计数 |
| `/plugin enable <name>` | — | 启用插件 |
| `/plugin disable <name>` | — | 禁用插件 |
| `/plugin uninstall <name>` | — | 卸载插件 |

## 8. 官方插件生态

| 类别 | 插件 | 描述 |
|------|------|------|
| **开发 (6)** | `agent-sdk-dev` | Agent SDK 开发辅助 |
| | `claude-opus-4-5-migration` | Opus 4.5 迁移工具 |
| | `feature-dev` | 7 阶段引导式功能开发 |
| | `frontend-design` | 前端设计（BOLD 美学方向） |
| | `plugin-dev` | 插件开发套件 |
| | `hookify` | 钩子开发辅助 |
| **效率 (4)** | `code-review` | 多代理 PR 审查（9 步流程） |
| | `commit-commands` | 提交命令 |
| | `pr-review-toolkit` | PR 审查工具包 |
| | `ralph-wiggum` | 迭代自引用循环 |
| **学习 (2)** | `explanatory-output-style` | 教育性解释输出 |
| | `learning-output-style` | 交互式学习输出 |
| **安全 (1)** | `security-guidance` | 安全指导 |

### 重点插件

**Code Review**（`/code-review`）:
- 多代理 PR 审查，9 步流程
- 4 个并行代理（2 Sonnet CLAUDE.md 合规 + 2 Opus Bug 检测）
- 两阶段验证（检测 → 验证）消除误报
- 发布带有可提交建议的 GitHub 行内评论

**Feature Dev**（`/feature-dev`）:
- 7 阶段引导式工作流
- 3 个专用代理并行运行：`code-explorer`、`code-architect`、`code-reviewer`

**Ralph Wiggum**（`/ralph-loop`）:
- 使用 Stop hook 重注入 prompt 的迭代循环
- Claude 通过文件系统/git 看到之前的工作持续迭代
- 适用于 TDD 和自主自我纠正

## 9. 设计要点

1. **多来源**: Marketplace / GitHub / npm / 本地 / Git / 内置 七种安装来源
2. **七种组件**: 命令/代理/技能/钩子/MCP/LSP/输出风格 全覆盖
3. **深度安全**: 反冒名 + Unicode 检测 + 企业策略 + 自动下架
4. **信任模型**: 含可执行组件的插件需显式用户批准
5. **热重载**: 插件设置变化自动检测并重加载
6. **会话专属**: `--plugin-dir` 支持不影响全局的临时插件

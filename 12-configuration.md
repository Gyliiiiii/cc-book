# 12 - 配置管理（Configuration Management）

## 1. 模块概述

Claude Code 使用 5 层（实际 6 层含插件）层叠配置体系，从插件设置到企业托管策略逐层覆盖。配置通过 Zod Schema 验证，支持实时文件监控、远程托管设置拉取、MDM/plist/注册表读取，以及 JSON Schema 发布提供编辑器自动补全。

## 2. 核心文件结构

```
src/utils/settings/
├── settings.ts                      # 核心加载、合并、缓存逻辑
├── types.ts                         # Zod SettingsSchema 定义
├── constants.ts                     # 来源定义、显示名、启用来源逻辑
├── managedPath.ts                   # 平台专用托管设置路径
├── validation.ts                    # 验证与错误格式化
├── settingsCache.ts                 # 会话级缓存
├── changeDetector.ts                # 文件变更检测（实时更新）
└── mdm/
    └── settings.ts                  # MDM (plist/注册表) 设置读取
src/services/remoteManagedSettings/
├── index.ts                         # 远程托管设置获取器
└── types.ts                         # 远程设置响应 Schema
```

## 3. 六层配置层叠

按优先级从低到高：

```
┌──────────────────────────────────────────────┐
│ 6. policySettings (企业托管)    ← 最高优先级  │
│    ├── Remote (API 拉取)        ← 策略内最高  │
│    ├── MDM/plist/HKLM (系统级)               │
│    ├── File (managed-settings.json + .d/)     │
│    └── HKCU (Windows 用户注册表) ← 策略内最低 │
├──────────────────────────────────────────────┤
│ 5. flagSettings (CLI 标志 / SDK 内联)         │
├──────────────────────────────────────────────┤
│ 4. localSettings (.claude/settings.local.json)│
├──────────────────────────────────────────────┤
│ 3. projectSettings (.claude/settings.json)    │
├──────────────────────────────────────────────┤
│ 2. userSettings (~/.claude/settings.json)     │
├──────────────────────────────────────────────┤
│ 1. Plugin Settings (插件注入)   ← 最低优先级  │
└──────────────────────────────────────────────┘
```

### 合并策略

使用 `lodash mergeWith` + 自定义合并器：
- **数组**: 拼接 + 去重
- **对象**: 深度合并
- **标量**: 高优先级覆盖低优先级

## 4. 设置文件详解

### 4.1 用户设置

```
~/.claude/settings.json              # 标准模式
~/.claude/cowork_settings.json       # Cowork 模式
```

### 4.2 项目设置

```
$PROJECT/.claude/settings.json       # 提交到版本控制
$PROJECT/.claude/settings.local.json # 自动加入 .gitignore
```

### 4.3 托管设置

| 平台 | 路径 |
|------|------|
| macOS | `/Library/Application Support/ClaudeCode/managed-settings.json` |
| Windows | `C:\Program Files\ClaudeCode\managed-settings.json` |
| Linux | `/etc/claude-code/managed-settings.json` |

Drop-in 目录：`managed-settings.d/*.json`，按字母排序加载（systemd/sudoers 约定）。

### 4.4 远程托管设置

- Console 用户（API Key）：所有合格用户
- OAuth 用户（Claude.ai）：仅 Enterprise/C4E 和 Team 订阅
- 每小时轮询，基于 checksum 的缓存

## 5. 设置 Schema 关键字段

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",

  "permissions": {
    "allow": ["Bash(git *)", "Read"],
    "deny": ["WebSearch"],
    "ask": ["Bash(rm *)"],
    "defaultMode": "default",
    "disableBypassPermissionsMode": "disable",
    "additionalDirectories": ["/shared/lib"]
  },

  "sandbox": { "enabled": true, "..." : "..." },

  "hooks": {
    "PreToolUse": [...],
    "SessionStart": [...]
  },

  "env": {
    "MY_VAR": "value"
  },

  "model": "claude-sonnet-4-6",
  "availableModels": ["claude-sonnet-4-6", "claude-opus-4-6"],
  "modelOverrides": { "fast": "claude-haiku-4-5" },

  "apiKeyHelper": "security find-generic-password ...",
  "attribution": { "commitMessage": "..." },
  "cleanupPeriodDays": 30,
  "fileSuggestion": { "type": "command", "command": "..." },
  "respectGitignore": true
}
```

### 向后兼容策略

- **允许**: 添加可选字段、新枚举值
- **禁止**: 删除字段、可选→必填、类型收紧

Schema 使用 `.passthrough()` + `.optional()` 确保兼容。

## 6. 权限模式

| 模式 | 描述 |
|------|------|
| `default` | 正常模式 |
| `plan` | 规划模式（只读） |
| `acceptEdits` | 自动接受编辑 |
| `bypassPermissions` | 跳过权限（危险） |
| `dontAsk` | 自动拒绝 |
| `auto` | AI 分类器（feature-gated） |

`disableBypassPermissionsMode: "disable"` 阻止 `--dangerously-skip-permissions`。
`disableAutoMode` 可禁用 auto 模式。

## 7. 环境变量

| 变量 | 描述 |
|------|------|
| `CLAUDE_CODE_MANAGED_SETTINGS_PATH` | 覆盖托管设置目录（ant-only） |
| `CLAUDE_CODE_USE_COWORK_PLUGINS` | 使用 cowork_settings.json |
| `CLAUDE_CODE_ENABLE_XAA` | 启用 XAA IdP 设置 |
| settings 中的 `env` 字段 | 注入到 Claude Code 会话的环境变量 |

## 8. 缓存与实时更新

```
会话级缓存 (settingsCache.ts)
    ↓
文件变更检测 (changeDetector.ts)
    ↓ 检测到变更
resetSettingsCache() → 重新从磁盘读取
    ↓
沙箱动态更新 → BaseSandboxManager.updateConfig()
```

## 9. CLI 来源过滤

`--setting-sources` 标志接受逗号分隔的来源名：`user,project,local`

**始终包含**: `policySettings` 和 `flagSettings`（不受此标志影响）。

## 10. 设计要点

1. **六层层叠**: 从插件到企业策略的完整覆盖链
2. **策略内四级**: 远程 > MDM > 文件 > HKCU 的精细企业控制
3. **Zod 验证**: 类型安全 + JSON Schema 发布 + 编辑器支持
4. **实时更新**: 文件监控 + 缓存失效 + 沙箱动态配置
5. **向后兼容**: 严格的兼容策略确保升级不破坏
6. **Drop-in 目录**: systemd 风格的 `managed-settings.d/` 模块化配置

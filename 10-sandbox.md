# 10 - 沙箱环境（Sandbox Environment）

## 1. 模块概述

沙箱环境为 BashTool 提供进程级隔离，通过 macOS `sandbox-exec`、Linux `bubblewrap/seccomp` 实现命令执行的网络隔离和文件系统限制。沙箱**仅作用于 BashTool**——文件工具（Read/Write/Edit）、WebFetch、WebSearch、MCP 服务器、Hook、斜杠命令均不受沙箱影响。

## 2. 核心文件结构

```
src/
├── utils/sandbox/
│   ├── sandbox-adapter.ts           # 主适配器：包装 @anthropic-ai/sandbox-runtime
│   └── sandbox-ui-utils.ts          # UI 工具：清理沙箱违规标签
├── tools/BashTool/
│   ├── shouldUseSandbox.ts          # 决策逻辑：是否沙箱化
│   └── bashPermissions.ts           # Bash 权限规则匹配
└── entrypoints/
    └── sandboxTypes.ts              # Zod Schema：沙箱配置（SDK/设置验证单一真相源）
```

## 3. 作用范围

```
受沙箱保护：
  ✓ BashTool（Shell 命令执行）

不受沙箱保护：
  ✗ FileReadTool / FileWriteTool / FileEditTool
  ✗ WebFetchTool / WebSearchTool
  ✗ MCP 服务器
  ✗ Hook 执行
  ✗ 斜杠命令
```

## 4. 配置选项

通过 settings JSON 的 `sandbox` 字段配置：

### 4.1 基础配置

| 选项 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `enabled` | boolean | `false` | 主开关 |
| `failIfUnavailable` | boolean | `false` | 沙箱不可用时启动失败（企业硬性要求） |
| `autoAllowBashIfSandboxed` | boolean | `true` | 沙箱启用时自动放行 Bash 命令 |
| `allowUnsandboxedCommands` | boolean | `true` | 是否允许 `dangerouslyDisableSandbox` 参数 |
| `enabledPlatforms` | string[] | — | 限制沙箱到特定平台（如 `["macos"]`） |
| `enableWeakerNestedSandbox` | boolean | — | 弱化嵌套沙箱约束（兼容性） |
| `enableWeakerNetworkIsolation` | boolean | — | 弱化网络隔离（开放 macOS trustd，Go TLS 验证） |

### 4.2 网络隔离

| 选项 | 类型 | 描述 |
|------|------|------|
| `network.allowedDomains` | string[] | 域名白名单 |
| `network.allowManagedDomainsOnly` | boolean | 仅使用托管设置的域名 |
| `network.allowUnixSockets` | string[] | 允许的 Unix 套接字路径（macOS） |
| `network.allowAllUnixSockets` | boolean | 禁用 Unix 套接字阻止 |
| `network.allowLocalBinding` | boolean | 允许本地端口绑定 |
| `network.httpProxyPort` | number | HTTP 代理端口 |
| `network.socksProxyPort` | number | SOCKS 代理端口 |

**域名白名单填充逻辑**：

```
来源 1: sandbox.network.allowedDomains 数组
来源 2: WebFetch(domain:...) allow 规则中的域名

当 allowManagedDomainsOnly = true:
  → 仅使用 policySettings 中的规则
  → 忽略 user/project/local 的域名

拒绝域名: 始终从所有源的 permissions.deny WebFetch 规则提取
```

### 4.3 文件系统限制

| 选项 | 描述 |
|------|------|
| `filesystem.allowWrite` | 额外可写路径。始终包含 `.`(cwd) 和 Claude 临时目录 |
| `filesystem.denyWrite` | 额外拒写路径。始终包含所有 settings.json 路径、`managed-settings.d/`、`.claude/skills` |
| `filesystem.allowRead` | 额外可读路径 |
| `filesystem.denyRead` | 额外拒读路径 |
| `filesystem.ignoreViolations` | 工具到模式的映射，抑制特定违规报告 |

**安全硬编码拒写**：
- 所有 settings.json 文件路径（防止沙箱逃逸）
- `managed-settings.d/` 目录
- `.claude/skills` 目录
- 裸 Git 仓库哨兵文件（`HEAD`、`objects`、`refs`、`hooks`、`config`）——防止 `core.fsmonitor` 逃逸攻击

### 4.4 排除命令

| 选项 | 描述 |
|------|------|
| `excludedCommands` | 绕过沙箱的命令模式 |

**注意**：排除命令是用户便利功能，**不是安全边界**。

匹配逻辑：
- `containsExcludedCommand()` 按 `&&`、`||`、`;` 拆分复合命令
- 逐一检查每个子命令
- 支持精确匹配、前缀匹配、通配符匹配

Anthropic 内部用户额外支持 `tengu_sandbox_disabled_commands` Feature Flag 动态配置。

## 5. 平台支持

| 平台 | 技术 | 状态 |
|------|------|------|
| macOS | `sandbox-exec` | 支持 |
| Linux | `bubblewrap` + `seccomp` | 支持 |
| WSL2+ | 同 Linux | 支持 |
| WSL1 | — | 明确不支持 |
| Windows | — | 不支持 |

`getSandboxUnavailableReason()` 在沙箱启用但无法运行时提供人类可读的诊断信息。

## 6. 配置示例

### 严格模式（企业）

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "network": {
      "allowedDomains": ["api.github.com", "registry.npmjs.org"],
      "allowManagedDomainsOnly": true
    }
  }
}
```

### 宽松模式

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": true,
    "excludedCommands": ["git", "npm", "docker"]
  }
}
```

## 7. 设计要点

1. **BashTool 专属**: 仅隔离 Shell 命令，不影响其他工具
2. **外部运行时**: 使用 `@anthropic-ai/sandbox-runtime` 包（非自建）
3. **防逃逸**: 硬编码拒写 settings 文件和 git 哨兵文件
4. **策略优先**: `allowManagedDomainsOnly` 确保企业控制域名白名单
5. **排除≠安全**: 排除命令是便利功能，非安全保证
6. **平台适配**: macOS/Linux/WSL2 三平台支持

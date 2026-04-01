# 07 - MCP 服务器集成（MCP Server Integration）

## 1. 模块概述

MCP（Model Context Protocol）集成系统允许 Claude Code 连接外部工具服务器，扩展其能力边界。系统支持六种传输协议、七个配置作用域、完整的 OAuth 2.0 认证流程，以及智能的延迟加载机制来控制上下文占用。

## 2. 核心文件结构

```
src/services/mcp/                        # MCP 服务层 (20+ files)
├── client.ts                            # 主客户端：传输创建、工具获取、连接管理
├── config.ts                            # 配置加载、合并、策略执行
├── types.ts                             # Zod Schema（传输类型、配置范围、连接状态）
├── auth.ts                              # OAuth 2.0：PKCE、Token 刷新、Slack 兼容
├── envExpansion.ts                       # 环境变量展开 ${VAR} / ${VAR:-default}
├── MCPConnectionManager.tsx             # React 上下文提供器
├── useManageMCPConnections.ts           # 连接生命周期 React Hook
├── mcpStringUtils.ts                    # 工具命名：buildMcpToolName, mcpInfoFromString
├── normalization.ts                     # 名称规范化 → ^[a-zA-Z0-9_-]{1,64}$
├── elicitationHandler.ts                # MCP 输入请求处理
├── headersHelper.ts                     # 远程服务器自定义 Header
├── oauthPort.ts                         # OAuth 回调端口/重定向 URI
├── xaa.ts                               # 跨应用访问 Token 交换
├── xaaIdpLogin.ts                       # IdP 登录（OIDC 发现、Token 获取）
├── claudeai.ts                          # Claude.ai 代理连接器配置
├── channelNotification.ts               # 频道通知中继
├── channelPermissions.ts                # 频道权限控制
├── SdkControlTransport.ts              # SDK 控制传输（IDE 集成）
├── InProcessTransport.ts               # 进程内传输
├── vscodeSdkMcp.ts                      # VS Code SDK MCP 集成
├── officialRegistry.ts                  # 官方 MCP 服务器注册表
└── utils.ts                             # 共享工具函数
src/utils/
├── mcpWebSocketTransport.ts             # 自定义 WebSocket 传输
└── toolSearch.ts                        # 延迟工具发现（ToolSearchTool）
src/tools/
├── MCPTool/                             # MCP 工具包装器
├── ListMcpResourcesTool/                # 列出 MCP 资源
├── ReadMcpResourceTool/                 # 读取 MCP 资源
└── McpAuthTool/                         # MCP 认证工具
src/skills/
└── mcpSkillBuilders.ts                  # MCP 技能构建器
```

## 3. 三层架构

```
┌─────────────────────────────────────────────────────┐
│ Layer 3: Tool/Resource Exposure                      │
│   MCPTool 包装 · mcp__server__tool 命名              │
│   ToolSearchTool 延迟加载 · Resource 暴露             │
├─────────────────────────────────────────────────────┤
│ Layer 2: Connection Management                       │
│   传输实例创建 · 连接状态管理 · OAuth 协商             │
│   指数退避重连 (max 5次, 1s初始, 30s最大)             │
├─────────────────────────────────────────────────────┤
│ Layer 1: Configuration & Policy                      │
│   7 个配置作用域 · Zod 验证 · Allow/Deny 策略         │
│   去重（插件 vs 手动, claude.ai vs 手动）             │
└─────────────────────────────────────────────────────┘
```

## 4. 六种传输协议

| 协议 | 配置类型 | SDK 类 | 关键配置字段 |
|------|----------|--------|-------------|
| `stdio` | `McpStdioServerConfigSchema` | `StdioClientTransport` | `command`, `args[]`, `env{}` |
| `sse` | `McpSSEServerConfigSchema` | `SSEClientTransport` | `url`, `headers{}`, `oauth{}` |
| `http` | `McpHTTPServerConfigSchema` | `StreamableHTTPClientTransport` | `url`, `headers{}`, `oauth{}` |
| `ws` | `McpWebSocketServerConfigSchema` | 自定义 `WebSocketTransport` | `url`, `headers{}` |
| `sdk` | `McpSdkServerConfigSchema` | `SdkControlClientTransport` | `name` |
| `claude.ai proxy` | `McpClaudeAIProxyServerConfigSchema` | OAuth Bearer fetch 包装 | `url`, `id` |

变体：`sse-ide`（IDE 扩展 SSE），`ws-ide`（IDE 扩展 WebSocket，含 `authToken`）。

## 5. 七个配置作用域

| 作用域 | 来源 | 描述 |
|--------|------|------|
| `local` | `~/.claude/` 设置 | 全局用户级 MCP 配置 |
| `user` | 用户设置 | 用户级设置 |
| `project` | `.mcp.json`（项目根目录） | 项目特定 MCP 服务器 |
| `dynamic` | 运行时注入 | `--mcp-config` CLI 标志等 |
| `enterprise` | `managed-mcp.json` | 企业/组织托管服务器 |
| `claudeai` | Claude.ai API | claude.ai Web UI 配置的连接器 |
| `managed` | 托管设置路径 | 策略托管服务器 |

`getAllMcpConfigs()` 合并所有作用域，每个服务器配置通过 `addScopeToServers()` 标记来源。

### .mcp.json 文件格式

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

## 6. 工具命名规范

**文件**: `src/services/mcp/mcpStringUtils.ts`

### 命名模式

```
mcp__<normalizedServerName>__<normalizedToolName>
```

### 规范化规则

- 所有非 `[a-zA-Z0-9_-]` 字符替换为下划线
- claude.ai 服务器（`"claude.ai "` 前缀）：连续下划线折叠，首尾下划线去除
- 最大长度：64 字符

### 关键函数

| 函数 | 描述 |
|------|------|
| `buildMcpToolName(serverName, toolName)` | 构建完整工具名 |
| `mcpInfoFromString(toolString)` | 从工具名解析 serverName 和 toolName |
| `getMcpPrefix(serverName)` | 返回 `mcp__<server>__` 前缀 |
| `getToolNameForPermissionCheck()` | 确保权限检查使用完全限定名 |

**已知限制**: 服务器名包含 `__` 会被错误解析。

## 7. 延迟加载机制（MCPSearch / ToolSearch）

**文件**: `src/utils/toolSearch.ts`

当 MCP 工具定义超过模型上下文窗口的一定百分比（默认 10%），触发延迟加载：

```
1. 工具标记为 defer_loading: true（不发送完整定义）

2. 注入 ToolSearchTool 到工具集

3. 模型通过 ToolSearchTool 按名称/描述搜索工具

4. formatDeferredToolLine() 创建紧凑单行摘要

5. 匹配的工具完整定义被加载到上下文
```

### 控制方式

| 方式 | 值 | 效果 |
|------|-----|------|
| `ENABLE_TOOL_SEARCH` 环境变量 | truthy | 强制开启 |
| `ENABLE_TOOL_SEARCH` 环境变量 | `auto` / `auto:N` | 基于阈值（N%）自动 |
| `ENABLE_TOOL_SEARCH` 环境变量 | falsy | 禁用 |
| GrowthBook 特性标志 | — | 额外控制 |

## 8. OAuth 与认证

**文件**: `src/services/mcp/auth.ts`

### 8.1 标准 OAuth 2.0 + PKCE

```
1. ClaudeAuthProvider 实现 OAuthClientProvider
2. 启动本地 HTTP 服务器（可用端口）作为 OAuth 回调
3. 打开浏览器进行用户授权
4. 使用 PKCE (code_challenge/code_verifier)
5. 敏感参数 (state, nonce, code) 从所有日志中脱敏
6. 单次请求超时：30 秒
```

### 8.2 Token 存储

- 安全存储（macOS Keychain helpers）
- 认证缓存：`~/.claude/mcp-needs-auth-cache.json`（15 分钟 TTL）
- 锁文件机制（`MAX_LOCK_RETRIES = 5`）确保并发写入安全

### 8.3 Token 刷新

- 处理标准和非标准错误码
- Slack 兼容：`invalid_refresh_token` / `expired_refresh_token` / `token_expired` → 规范化为 `invalid_grant`
- Slack 返回 HTTP 200 错误的兼容：将 2xx 错误响应重写为 400

### 8.4 跨应用访问（XAA / SEP-990）

| 文件 | 描述 |
|------|------|
| `xaa.ts` | `performCrossAppAccess()` Token 交换 |
| `xaaIdpLogin.ts` | IdP 级 OIDC 发现、Token 获取/缓存 |

配置：每服务器 `oauth.xaa: true`，全局 `settings.xaaIdp`（issuer、clientId、callbackPort）。

### 8.5 连接状态

| 状态 | 描述 |
|------|------|
| `connected` | 连接正常 |
| `failed` | 连接失败 |
| `needs-auth` | 需要 OAuth 认证 |
| `pending` | 连接中 |
| `disabled` | 已禁用 |

## 9. 安全控制

### 9.1 Deny 列表

`deniedMcpServers` in settings，三种匹配策略：

| 策略 | 示例 | 描述 |
|------|------|------|
| 按名称 | `"my-server"` | 服务器名精确匹配 |
| 按命令 | `["npx", "bad-tool"]` | stdio 命令数组匹配 |
| 按 URL | `"https://*.example.com/*"` | URL 模式匹配（支持通配符） |

**Deny 列表始终优先于 Allow 列表。** 从所有设置源合并（用户始终可以为自己拒绝服务器）。

### 9.2 Allow 列表

`allowedMcpServers`，同样三种匹配策略。空列表 = 阻止所有服务器。

当 `allowManagedMcpServersOnly` 策略开启时，仅托管/策略设置控制允许列表。

### 9.3 策略函数

| 函数 | 描述 |
|------|------|
| `isMcpServerDenied()` | 检查名称、命令、URL 匹配 |
| `isMcpServerAllowedByPolicy()` | 先检查 deny，再检查 allow |
| `filterMcpServersByPolicy()` | 过滤完整服务器映射 |

### 9.4 去重

| 函数 | 规则 |
|------|------|
| `dedupPluginMcpServers()` | 手动配置 > 插件；先加载的插件 > 后加载 |
| `dedupClaudeAiMcpServers()` | 已启用手动 > claude.ai 连接器 |
| `getMcpServerSignature()` | 基于内容的去重键 |

## 10. 环境变量展开

**文件**: `src/services/mcp/envExpansion.ts`

```
${VAR}              → 简单展开
${VAR:-default}     → 带默认值展开
```

- `expandEnvVarsInString(value)` 返回 `{ expanded, missingVars }`
- 缺失变量保留原始 `${VAR}` 文本（便于调试）
- 用于展开 MCP 配置中的 URL、args、headers

## 11. 设计要点

1. **六种传输**: stdio / SSE / HTTP / WebSocket / SDK / claude.ai proxy 全覆盖
2. **七个作用域**: 从用户到企业的完整配置层叠
3. **延迟加载**: 超过 10% 上下文时自动延迟，按需搜索加载
4. **OAuth 兼容性**: 标准 PKCE + Slack 非标准兼容 + XAA 跨应用
5. **Deny 优先**: 安全策略中拒绝列表始终优先于允许列表
6. **智能去重**: 防止插件与手动配置产生重复工具
7. **环境变量**: 支持 `${VAR:-default}` 语法，缺失变量保留原文

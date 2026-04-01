# 09 - 技能系统（Skill System）

## 1. 模块概述

技能系统是 Claude Code 的知识注入层，通过模块化的 SKILL.md 文件将 Claude 从通用助手转变为领域专家。系统采用三层渐进式加载设计：元数据始终在 system prompt 中（~100 词），完整指令仅在意图匹配时加载，外部资源按需执行/读取。这最大限度减少了上下文窗口占用。

## 2. 核心文件结构

```
src/skills/
├── loadSkillsDir.ts             # (1087行) 核心加载、发现、去重
├── bundledSkills.ts             # (221行) 内置技能注册 + 文件提取
├── mcpSkillBuilders.ts          # MCP 技能构建器（打破循环依赖）
└── bundled/
    └── index.ts                 # 内置技能初始化
src/tools/SkillTool/
├── SkillTool.ts                 # (1109行) Skill 工具实现
├── prompt.ts                    # Prompt 和预算管理
└── UI.tsx                       # React UI 组件
```

## 3. SKILL.md 格式

### 3.1 Frontmatter 字段

```yaml
---
name: my-skill                    # 显示名（可选，默认用目录名）
description: 这个技能做什么        # 描述
when_to_use: 何时应调用此技能      # 触发指引
allowed-tools: [Bash, Read]       # 允许使用的工具
argument-hint: "文件路径"          # 参数提示
arguments: [file, format]         # 命名参数定义
user-invocable: true              # 用户可否直接调用（默认 true）
disable-model-invocation: false   # 禁止模型自动调用
model: opus|sonnet|inherit        # 模型覆盖
effort: low|medium|high|<int>     # 推理深度
context: fork                     # 在隔离子代理中执行
agent: my-agent                   # 使用的代理类型
hooks:                            # 钩子配置（HooksSchema）
  PreToolUse: [...]
paths: ["src/**/*.py"]            # Gitignore 风格路径模式（条件激活）
version: "1.0.0"                  # 版本号
shell:                            # Shell 配置
  command: bash
---
技能指令正文（Markdown 格式）
```

### 3.2 变量替换

| 变量 | 替换为 |
|------|--------|
| `${CLAUDE_SKILL_DIR}` | 技能自身目录路径 |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID |
| `$ARGUMENTS` | 调用时传入的参数 |
| 命名参数 | `substituteArguments()` 处理 |
| `` `!command` `` | 内联 Shell 执行结果（MCP 技能禁用） |

## 4. 三层渐进式加载

```
┌─────────────────────────────────────────────┐
│ Level 1: Discovery (系统 Prompt 列表)        │
│ • 占上下文窗口 ≤1%                           │
│ • 每条 ≤250 字符                             │
│ • 格式: "- skill-name: description"          │
│ • 内置技能不截断，非内置按比例截断            │
├─────────────────────────────────────────────┤
│ Level 2: Invocation (SkillTool 调用)         │
│ • 完整 SKILL.md 内容加载                     │
│ • 变量替换、内联 Shell 执行                  │
│ • 注入为用户消息                              │
├─────────────────────────────────────────────┤
│ Level 3: Execution (Fork 子代理)             │
│ • context: "fork" 的技能                     │
│ • 独立 Token 预算                            │
│ • 不消耗主上下文窗口                         │
└─────────────────────────────────────────────┘
```

### 预算控制

| 参数 | 值 | 描述 |
|------|-----|------|
| `SKILL_BUDGET_CONTEXT_PERCENT` | 0.01 (1%) | 技能列表占上下文比例 |
| `MAX_LISTING_DESC_CHARS` | 250 | 单条列表最大字符 |
| 默认回退 | 8,000 字符 | 200K Token 窗口的回退值 |

### 渐进式截断策略

```
1. 尝试: 所有技能完整描述
   ↓ 超预算
2. 尝试: 内置技能完整描述 + 非内置按比例截断
   ↓ 仍超预算
3. 回退: 内置技能保留描述 + 非内置仅显示名称
```

## 5. 技能包结构

```
my-skill/
├── SKILL.md              # 必需：frontmatter + markdown 指令
├── scripts/              # 可选：通过 ${CLAUDE_SKILL_DIR} 引用的脚本
├── schemas/              # 可选：模型可 Read/Grep 的参考文件
└── other-files/          # 任意支持资产
```

内置技能（编译进二进制）：
- `files` 属性声明需要提取的文件
- 首次调用时延迟提取到 `getBundledSkillsRoot()/<skillName>/`
- 使用 `O_NOFOLLOW|O_EXCL` 标志 + `0o600` 权限防止符号链接攻击

## 6. 技能加载来源

按优先级从高到低：

| 优先级 | 来源 | 路径 |
|--------|------|------|
| 1 | 托管（策略） | `<managed-settings>/.claude/skills/` |
| 2 | 用户 | `~/.claude/skills/` |
| 3 | 项目 | `.claude/skills/`（向上遍历到 home） |
| 4 | 附加目录 | `--add-dir` 标志路径 |
| 5 | 旧版命令 | `.claude/commands/` 目录（已弃用） |
| — | 内置 | 启动时编程注册 |
| — | 插件 | 已安装插件的 `skills/` 目录 |
| — | MCP | MCP 服务器元数据 |
| — | 动态 | 访问含 `.claude/skills/` 的子目录时延迟发现 |
| — | 条件 | `paths` frontmatter 匹配时激活 |

**去重**: 按解析后的文件路径（`realpath`）去重，处理符号链接和重叠目录。先出现的优先。

**Bare 模式**（`--bare`）: 跳过所有自动发现，仅加载 `--add-dir` 路径。

## 7. 技能调用流程

```
1. 模型调用 Skill 工具: { skill: "name", args: "..." }

2. validateInput()
   ├── 技能存在检查
   ├── 是否为 prompt-based 检查
   ├── 模型调用未被禁用检查
   └── 远程规范技能处理

3. checkPermissions()
   ├── deny/allow 规则检查
   ├── 仅含安全属性的技能 → 自动允许
   └── 否则 → 提示用户

4. call() 分发
   ├── 远程规范技能 (_canonical_<slug>)
   │   └── 从 AKI/GCS 加载，直接注入
   ├── Fork 技能 (context: "fork")
   │   └── executeForkedSkill() → 独立子代理
   └── 内联技能（默认）
       └── processPromptSlashCommand()
           ├── 加载完整 SKILL.md
           ├── 变量替换
           ├── 内联 Shell 执行
           └── 注入为用户消息 + contextModifier
```

## 8. 内置技能

| 技能 | 描述 |
|------|------|
| `update-config` | 更新配置（settings.json） |
| `keybindings` | 键绑定管理 |
| `verify` | 验证技能 |
| `debug` | 调试辅助 |
| `lorem-ipsum` | Lorem ipsum 生成 |
| `skillify` | 将当前会话转为可复用 SKILL.md |
| `remember` | 会话记忆 |
| `simplify` | 代码简化 |
| `batch` | 批量操作 |
| `stuck` | 卡住时求助 |
| `claude-in-chrome` | Chrome 集成（条件） |
| `dream` | 后台 dreaming（KAIROS） |
| `loop` | 循环执行（AGENT_TRIGGERS） |
| `schedule-remote-agents` | 远程代理调度（AGENT_TRIGGERS_REMOTE） |
| `claude-api` | Claude API 开发辅助（BUILDING_CLAUDE_APPS） |

## 9. 上下文优化策略

| 策略 | 效果 |
|------|------|
| 1% 预算 | 技能列表最多占上下文窗口 1% |
| 渐进截断 | 超预算时逐级降级（完整 → 截断 → 仅名称） |
| 延迟加载 | 完整内容仅在调用时加载 |
| Fork 执行 | `context: "fork"` 在独立 Token 预算中执行 |
| 条件激活 | `paths` frontmatter 匹配前不加载 |
| 压缩保持 | `addInvokedSkill()` 记录调用，压缩后恢复 |
| 压缩后预算 | 每技能 ≤5,000 tokens，总 ≤25,000 tokens |

## 10. 设计要点

1. **三层加载**: 发现 → 调用 → 执行，按需增量加载
2. **1% 硬预算**: 技能列表不会失控膨胀上下文
3. **多来源发现**: 托管/用户/项目/插件/MCP/动态/条件 七种来源
4. **条件激活**: `paths` 模式确保无关技能不占用上下文
5. **Fork 隔离**: 大型技能在独立子代理中执行
6. **变量系统**: `${CLAUDE_SKILL_DIR}` + `$ARGUMENTS` + 内联 Shell
7. **安全提取**: 内置技能文件使用 `O_NOFOLLOW|O_EXCL` 防止攻击

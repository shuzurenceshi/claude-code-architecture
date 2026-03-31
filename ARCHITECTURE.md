# Claude Code 源码架构分析

> 基于 2026-03-31 泄露的官方源码（通过 npm source map 暴露）
> 
> 源码位置：`/root/projects/myapp/claude-code-source/claude-code/`

---

## 📊 规模统计

| 指标 | 数值 |
|-----|-----|
| TypeScript 文件 | 3,768 个 |
| 代码行数 | 98,000+ 行 |
| 目录大小 | 53MB |
| 工具数量 | 43+ 个 |
| 命令数量 | 101+ 个 |

---

## 🏗️ 技术栈

| 类别 | 技术 |
|-----|-----|
| **运行时** | [Bun](https://bun.sh) |
| **语言** | TypeScript (strict mode) |
| **终端 UI** | React + [Ink](https://github.com/vadimdemedes/ink) |
| **CLI 解析** | [Commander.js](https://github.com/tj/commander.js) |
| **Schema 验证** | [Zod v4](https://zod.dev) |
| **代码搜索** | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| **协议** | MCP SDK, LSP |
| **API** | Anthropic SDK |
| **遥测** | OpenTelemetry + gRPC |
| **Feature Flags** | GrowthBook |
| **认证** | OAuth 2.0, JWT, macOS Keychain |

---

## 📁 核心目录结构

```
src/
├── main.tsx                 # 入口点 (Commander.js CLI + React/Ink 渲染)
├── commands.ts              # 命令注册中心 (~25K 行)
├── tools.ts                 # 工具注册中心 (~17K 行)
├── Tool.ts                  # 工具类型定义 (~29K 行)
├── QueryEngine.ts           # LLM 查询引擎 (~46K 行)
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # Token 成本追踪
│
├── commands/                # 斜杠命令实现 (~101 个)
├── tools/                   # Agent 工具实现 (~43 个)
├── components/              # Ink UI 组件 (~140 个)
├── hooks/                   # React Hooks
├── services/                # 外部服务集成
├── screens/                 # 全屏 UI (Doctor, REPL, Resume)
├── types/                   # TypeScript 类型定义
├── utils/                   # 工具函数
│
├── bridge/                  # IDE 和远程控制桥接
├── coordinator/             # 多 Agent 协调器
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── keybindings/             # 快捷键配置
├── vim/                     # Vim 模式
├── voice/                   # 语音输入
├── remote/                  # 远程会话
├── server/                  # 服务器模式
├── memdir/                  # 持久化记忆目录
├── tasks/                   # 任务管理
├── state/                   # 状态管理
└── migrations/              # 配置迁移
```

---

## 🔧 核心系统

### 1. Tool 系统 (`src/tools/`)

每个工具都是独立的模块，定义了：
- 输入 schema (Zod)
- 权限模型
- 执行逻辑
- 进度状态

**核心工具列表：**

| 工具 | 说明 |
|-----|-----|
| `BashTool` | Shell 命令执行 |
| `FileReadTool` | 文件读取（支持图片、PDF、Notebook） |
| `FileWriteTool` | 文件创建/覆盖 |
| `FileEditTool` | 部分文件修改（字符串替换） |
| `GlobTool` | 文件模式匹配搜索 |
| `GrepTool` | 基于 ripgrep 的内容搜索 |
| `WebFetchTool` | 获取 URL 内容 |
| `WebSearchTool` | 网页搜索 |
| `AgentTool` | 子 Agent 生成 |
| `SkillTool` | 技能执行 |
| `MCPTool` | MCP 服务器工具调用 |
| `LSPTool` | Language Server Protocol 集成 |
| `NotebookEditTool` | Jupyter notebook 编辑 |
| `TaskCreateTool/TaskUpdateTool` | 任务创建和管理 |
| `SendMessageTool` | Agent 间消息传递 |
| `TeamCreateTool/TeamDeleteTool` | 团队 Agent 管理 |
| `EnterPlanModeTool/ExitPlanModeTool` | 计划模式切换 |
| `EnterWorktreeTool/ExitWorktreeTool` | Git worktree 隔离 |
| `ToolSearchTool` | 延迟工具发现 |
| `CronCreateTool` | 定时触发创建 |
| `SleepTool` | 主动模式等待 |
| `SyntheticOutputTool` | 结构化输出生成 |

**工具类型定义 (`Tool.ts`)：**

```typescript
export type Tool<Input extends Record<string, unknown> = Record<string, unknown>> = {
  name: string
  description: string | ((input: Input, ctx?) => Promise<string>)
  inputSchema: z.ZodSchema<Input>
  isEnabled: () => boolean
  isReadOnly?: () => boolean
  
  // 权限检查
  checkPermissions?: (
    input: Input,
    ctx: ToolPermissionContext
  ) => Promise<PermissionResult>
  
  // 执行
  call: (
    input: Input,
    ctx: ToolUseContext
  ) => Promise<string | AsyncGenerator<ToolCallProgress>>
}
```

### 2. Command 系统 (`src/commands/`)

用户面向的斜杠命令，用 `/` 前缀调用。

**核心命令：**

| 命令 | 说明 |
|-----|-----|
| `/commit` | 创建 git commit |
| `/review` | 代码审查 |
| `/compact` | 上下文压缩 |
| `/mcp` | MCP 服务器管理 |
| `/config` | 设置管理 |
| `/doctor` | 环境诊断 |
| `/login` `/logout` | 认证 |
| `/memory` | 持久化记忆管理 |
| `/skills` | 技能管理 |
| `/tasks` | 任务管理 |
| `/vim` | Vim 模式切换 |
| `/diff` | 查看变更 |
| `/cost` | 检查使用成本 |
| `/theme` | 更改主题 |
| `/context` | 上下文可视化 |
| `/pr_comments` | 查看 PR 评论 |
| `/resume` | 恢复之前的会话 |
| `/share` | 分享会话 |
| `/desktop` | 桌面应用移交 |
| `/mobile` | 移动应用移交 |

### 3. Service 层 (`src/services/`)

| 服务 | 说明 |
|-----|-----|
| `api/` | Anthropic API 客户端、文件 API、引导程序 |
| `mcp/` | Model Context Protocol 服务器连接和管理 |
| `oauth/` | OAuth 2.0 认证流程 |
| `lsp/` | Language Server Protocol 管理器 |
| `analytics/` | 基于 GrowthBook 的功能标志和分析 |
| `plugins/` | 插件加载器 |
| `compact/` | 对话上下文压缩 |
| `policyLimits/` | 组织策略限制 |
| `remoteManagedSettings/` | 远程托管设置 |
| `extractMemories/` | 自动记忆提取 |
| `tokenEstimation.ts` | Token 计数估算 |
| `teamMemorySync/` | 团队记忆同步 |

### 4. Bridge 系统 (`src/bridge/`)

连接 IDE 扩展（VS Code、JetBrains）与 Claude Code CLI 的双向通信层。

- `bridgeMain.ts` — 桥接主循环
- `bridgeMessaging.ts` — 消息协议
- `bridgePermissionCallbacks.ts` — 权限回调
- `replBridge.ts` — REPL 会话桥接
- `jwtUtils.ts` — 基于 JWT 的认证
- `sessionRunner.ts` — 会话执行管理

### 5. Permission 系统 (`src/hooks/toolPermission/`)

每次工具调用时检查权限。可以：
- 提示用户批准/拒绝
- 根据配置的权限模式自动处理（`default`, `plan`, `bypassPermissions`, `auto` 等）

**权限模式：**

| 模式 | 说明 |
|-----|-----|
| `default` | 默认模式，敏感操作需确认 |
| `plan` | 计划模式，只读操作 |
| `bypassPermissions` | 跳过所有权限检查 |
| `auto` | 自动模式，使用分类器决定 |

### 6. QueryEngine (`QueryEngine.ts`)

**核心 LLM 调用引擎**，~46K 行代码。处理：
- 流式响应
- 工具调用循环
- 思考模式
- 重试逻辑
- Token 计数
- 上下文压缩
- 多轮对话管理

**关键方法：**

```typescript
class QueryEngine {
  constructor(config: QueryEngineConfig)
  
  // 提交消息，返回 AsyncGenerator
  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean }
  ): AsyncGenerator<SDKMessage, void, unknown>
}
```

---

## 🧠 关键设计模式

### 1. 并行预取 (Parallel Prefetch)

启动时间优化，在重型模块评估之前并行预取：

```typescript
// main.tsx — 作为副作用在其他导入之前触发
startMdmRawRead()
startKeychainPrefetch()
```

### 2. 延迟加载 (Lazy Loading)

重型模块（OpenTelemetry、gRPC、分析等）通过动态 `import()` 延迟到实际需要时：

```typescript
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null
```

### 3. Feature Flags (死代码消除)

通过 Bun 的 `bun:bundle` feature flags 实现编译时死代码消除：

```typescript
import { feature } from 'bun:bundle'

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

**主要 Feature Flags：**
- `PROACTIVE` — 主动模式
- `KAIROS` — 内部功能集
- `BRIDGE_MODE` — 桥接模式
- `DAEMON` — 守护进程模式
- `VOICE_MODE` — 语音模式
- `AGENT_TRIGGERS` — Agent 触发器
- `MONITOR_TOOL` — 监控工具
- `COORDINATOR_MODE` — 协调器模式
- `HISTORY_SNIP` — 历史片段
- `WORKFLOW_SCRIPTS` — 工作流脚本

### 4. Agent Swarms (Agent 群)

子 Agent 通过 `AgentTool` 生成，`coordinator/` 处理多 Agent 编排。`TeamCreateTool` 启用团队级并行工作。

### 5. Skill 系统

在 `skills/` 中定义的可复用工作流，通过 `SkillTool` 执行。用户可以添加自定义技能。

### 6. Plugin 架构

内置和第三方插件通过 `plugins/` 子系统加载。

---

## 🔐 权限系统详解

### 权限检查流程

```
Tool Call
    ↓
checkPermissions() — 工具自定义检查
    ↓
hasPermissionsToUseTool() — 全局规则检查
    ↓
┌─────────────────────────────────────┐
│ Permission Rules (按优先级)          │
│ 1. alwaysDenyRules  — 总是拒绝       │
│ 2. alwaysAllowRules — 总是允许       │
│ 3. alwaysAskRules   — 总是询问       │
└─────────────────────────────────────┘
    ↓
如果无匹配规则 → 使用权限模式
    ↓
┌─────────────────────────────────────┐
│ Permission Modes                     │
│ - default: 敏感操作弹窗确认          │
│ - plan: 只允许只读操作               │
│ - bypassPermissions: 全部自动允许    │
│ - auto: 分类器自动决定              │
└─────────────────────────────────────┘
    ↓
用户决策 (allow/deny + remember)
    ↓
执行工具 或 拒绝
```

### 权限规则来源

```typescript
export type ToolPermissionRulesBySource = {
  commandLine?: PermissionRule[]      // 命令行参数
  managedSettings?: PermissionRule[]  // 托管设置
  settings?: PermissionRule[]         // 本地设置
  memory?: PermissionRule[]           // 记忆
}
```

---

## 🔌 MCP (Model Context Protocol) 集成

### MCP 客户端架构

```typescript
// client.ts
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'

// MCP 工具动态注册
const mcpTool = new MCPTool(mcpClient, toolDefinition)
```

### MCP 工具调用流程

```
Claude 调用 MCP 工具
    ↓
MCPTool.call()
    ↓
MCPServerConnection.request('tools/call', { name, arguments })
    ↓
MCP Server (外部进程/HTTP)
    ↓
返回结果
    ↓
格式化后返回给 Claude
```

---

## 💾 记忆系统 (Memory System)

### Agent 记忆

```typescript
export type AgentMemoryScope = 'user' | 'project' | 'local'

// user scope: ~/.claude/agent-memory/<agentType>/
// project scope: <cwd>/.claude/agent-memory/<agentType>/
// local scope: <cwd>/.claude/agent-memory-local/<agentType>/
```

### 记忆提取

- `services/extractMemories/` — 自动从对话中提取记忆
- `memdir/` — 持久化记忆目录管理

---

## 🎯 关键实现细节

### 1. 工具 Prompt 构建

每个工具都有 `prompt.ts` 定义 LLM 看到的描述：

```typescript
// tools/BashTool/prompt.ts
export const BASH_TOOL_NAME = 'Bash'

export function getBashToolDescription(): string {
  return `Executes a bash command in a persistent shell session...

[详细的工具说明和使用指南]
`
}
```

### 2. 上下文收集 (`context.ts`)

收集系统上下文发送给 LLM：
- 工作目录
- Git 状态
- 文件结构
- 环境变量
- 项目配置

### 3. 成本追踪 (`cost-tracker.ts`)

```typescript
export function getTotalCost(): number
export function getTotalInputTokens(): number
export function getTotalOutputTokens(): number
export function getTotalCacheCreationInputTokens(): number
export function getTotalCacheReadInputTokens(): number
```

---

## 🚀 启动流程

```
main.tsx
    ↓
1. 并行预取 (MDM, Keychain)
    ↓
2. GrowthBook 初始化
    ↓
3. Commander.js CLI 解析
    ↓
4. 加载配置 (settings, permissions)
    ↓
5. 初始化 MCP 客户端
    ↓
6. 创建 QueryEngine
    ↓
7. React/Ink 渲染
    ↓
8. 进入主循环
```

---

## 📚 学习要点总结

### 1. 工具系统设计
- ✅ 每个工具独立模块
- ✅ Zod schema 验证输入
- ✅ 权限系统分层（规则 → 模式 → 用户决策）
- ✅ 进度状态支持长时间操作

### 2. CLI 架构
- ✅ Commander.js 解析命令
- ✅ React + Ink 构建终端 UI
- ✅ Feature flags 实现条件编译
- ✅ 延迟加载优化启动时间

### 3. LLM 交互
- ✅ QueryEngine 封装完整生命周期
- ✅ 流式响应处理
- ✅ 工具调用循环
- ✅ 上下文压缩

### 4. 扩展性
- ✅ MCP 协议支持外部工具
- ✅ 插件系统
- ✅ 技能系统
- ✅ 多 Agent 协调

### 5. 权限安全
- ✅ 多层权限检查
- ✅ 规则优先级
- ✅ 用户确认机制
- ✅ 审计日志

---

## 🔗 相关资源

- **官方文档**: https://docs.anthropic.com/claude/docs/claude-code
- **MCP 协议**: https://modelcontextprotocol.io
- **Ink 框架**: https://github.com/vadimdemedes/ink
- **Bun 运行时**: https://bun.sh

---

*生成时间: 2026-04-01*
*基于源码版本: 2026-03-31 leak*

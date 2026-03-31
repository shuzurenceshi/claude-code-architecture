# Claude Code 工具系统详解

> 基于 `src/tools/` 和 `src/Tool.ts` 源码分析

---

## 工具类型定义

### 核心 Tool 接口

```typescript
// Tool.ts
export type Tool<Input extends Record<string, unknown> = Record<string, unknown>> = {
  // 工具名称 (在 Claude 眼中唯一标识)
  name: string
  
  // 描述 (静态字符串或动态生成函数)
  description: string | ((input: Input, ctx?) => Promise<string>)
  
  // 输入 Schema (Zod 验证)
  inputSchema: z.ZodSchema<Input>
  
  // 是否启用
  isEnabled: () => boolean
  
  // 是否只读 (影响 plan mode)
  isReadOnly?: () => boolean
  
  // 权限检查 (可选)
  checkPermissions?: (
    input: Input,
    ctx: ToolPermissionContext
  ) => Promise<PermissionResult>
  
  // 执行函数
  call: (
    input: Input,
    ctx: ToolUseContext
  ) => Promise<string | AsyncGenerator<ToolCallProgress>>
}
```

### 工具上下文

```typescript
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
  }
  
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  
  // MCP elicitation 处理
  handleElicitation?: (
    serverName: string,
    params: ElicitRequestURLParams,
  ) => Promise<ElicitResult>
  
  // 更新 UI (React JSX)
  setToolJSX: SetToolJSXFn
}
```

---

## 工具注册

### 工具列表 (`tools.ts`)

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    FileEditTool,
    FileReadTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    NotebookEditTool,
    WebFetchTool,
    WebSearchTool,
    AskUserQuestionTool,
    LSPTool,
    ListMcpResourcesTool,
    ReadMcpResourceTool,
    ToolSearchTool,
    EnterPlanModeTool,
    ExitPlanModeV2Tool,
    EnterWorktreeTool,
    ExitWorktreeTool,
    ConfigTool,
    TaskCreateTool,
    TaskGetTool,
    TaskUpdateTool,
    TaskListTool,
    TaskStopTool,
    BriefTool,
    
    // 条件工具 (Feature flags)
    ...(feature('PROACTIVE') ? [SleepTool] : []),
    ...(feature('AGENT_TRIGGERS') ? [CronCreateTool, ...] : []),
    ...(feature('KAIROS') ? [SendUserFileTool, PushNotificationTool] : []),
    ...(feature('WEB_BROWSER_TOOL') ? [WebBrowserTool] : []),
    ...(feature('WORKFLOW_SCRIPTS') ? [WorkflowTool] : []),
    
    // 动态加载
    getPowerShellTool(),
    getTeamCreateTool(),
    getTeamDeleteTool(),
    getSendMessageTool(),
    getMcpTools(), // MCP 工具
  ].filter(Boolean)
}
```

---

## 工具实现示例

### 1. BashTool

**文件结构：**
```
tools/BashTool/
├── toolName.ts          # 工具名常量
├── prompt.ts            # LLM 看到的描述
├── bashPermissions.ts   # 权限检查
├── bashSecurity.ts      # 安全验证
├── commandSemantics.ts  # 命令语义 (grep 返回 1 不是错误)
├── destructiveCommandWarning.ts  # 危险命令警告
├── readOnlyValidation.ts  # 只读验证
├── sedEditParser.ts     # sed 命令解析
├── shouldUseSandbox.ts  # 沙箱判断
└── utils.ts             # 工具函数
```

**核心实现：**

```typescript
// BashTool
export const BashTool: Tool<BashInput> = {
  name: 'Bash',
  
  description: async (input, ctx) => {
    return getBashToolDescription()
  },
  
  inputSchema: z.object({
    command: z.string().describe('The command to execute'),
    timeout: z.number().optional(),
    workdir: z.string().optional(),
    // ...
  }),
  
  isEnabled: () => true,
  
  isReadOnly: () => false,
  
  checkPermissions: async (input, ctx) => {
    // 检查危险命令
    // 检查沙箱需求
    // 检查权限规则
    return { behavior: 'ask', message: '...' }
  },
  
  call: async function* (input, ctx) {
    const { command, timeout, workdir } = input
    
    // 1. 解析命令
    const parsed = parseCommand(command)
    
    // 2. 安全检查
    const security = checkBashSecurity(parsed)
    if (security.blocked) {
      return security.message
    }
    
    // 3. 执行命令
    const process = spawn(parsed.binary, parsed.args, {
      cwd: workdir ?? getCwd(),
      timeout: timeout ?? 120000,
      env: { ...process.env, ...getBashEnv() }
    })
    
    // 4. 流式输出进度
    for await (const chunk of process.stdout) {
      yield {
        type: 'progress',
        output: chunk.toString()
      }
    }
    
    // 5. 返回结果
    return formatBashOutput(process)
  }
}
```

### 2. FileEditTool

```typescript
export const FileEditTool: Tool<FileEditInput> = {
  name: 'Edit',
  
  description: `Performs exact string replacements in files...`,
  
  inputSchema: z.object({
    file_path: z.string(),
    old_string: z.string(),
    new_string: z.string(),
  }),
  
  call: async (input, ctx) => {
    const { file_path, old_string, new_string } = input
    
    // 1. 读取文件
    const content = await readFile(file_path, 'utf-8')
    
    // 2. 查找并替换
    if (!content.includes(old_string)) {
      throw new Error(`String not found in file: ${old_string}`)
    }
    
    const newContent = content.replace(old_string, new_string)
    
    // 3. 写入文件
    await writeFile(file_path, newContent)
    
    return `Successfully edited ${file_path}`
  }
}
```

### 3. AgentTool (子 Agent)

```typescript
export const AgentTool: Tool<AgentInput> = {
  name: 'Agent',
  
  description: `Spawns a sub-agent to handle a task...`,
  
  inputSchema: z.object({
    task: z.string().describe('The task for the sub-agent'),
    agent_type: z.string().optional(),
    // ...
  }),
  
  call: async function* (input, ctx) {
    const { task, agent_type } = input
    
    // 1. 创建子 Agent 上下文
    const subAgentCtx = createSubagentContext(ctx, agent_type)
    
    // 2. 启动子 Agent
    const engine = new QueryEngine({
      ...ctx.options,
      initialMessages: [],
      cwd: getCwd(),
    })
    
    // 3. 流式返回进度
    for await (const msg of engine.submitMessage(task)) {
      yield {
        type: 'agent_progress',
        message: msg
      }
    }
    
    // 4. 返回结果
    return subAgentCtx.getResult()
  }
}
```

### 4. MCPTool

```typescript
export class MCPTool implements Tool {
  constructor(
    private mcpClient: MCPServerConnection,
    private toolDef: MCPToolDefinition
  ) {
    this.name = `mcp__${mcpClient.name}__${toolDef.name}`
    this.inputSchema = convertMCPSchemaToZod(toolDef.inputSchema)
  }
  
  name: string
  description: string
  inputSchema: z.ZodSchema
  
  call: async (input, ctx) => {
    // 调用 MCP 服务器
    const result = await this.mcpClient.request('tools/call', {
      name: this.toolDef.name,
      arguments: input
    })
    
    return formatMCPResult(result)
  }
}
```

---

## 工具 Prompt 设计

每个工具都有专门给 LLM 看的描述，定义在 `prompt.ts` 中：

### TodoWriteTool Prompt 示例

```typescript
// tools/TodoWriteTool/prompt.ts
export const PROMPT = `Use this tool to create and manage a structured task list...

## When to Use This Tool
1. Complex multi-step tasks
2. Non-trivial and complex tasks
3. User explicitly requests todo list
4. User provides multiple tasks
5. After receiving new instructions
6. When you start working on a task
7. After completing a task

## When NOT to Use This Tool
1. Single, straightforward task
2. Trivial task
3. Less than 3 trivial steps
4. Purely conversational

<example>
User: I want to add a dark mode toggle...
Assistant: *Creates todo list with items:*
1. Creating dark mode toggle component
2. Adding dark mode state management
3. Implementing CSS styles
4. Running tests and build
</example>
`
```

### FileWriteTool Prompt 示例

```typescript
export function getWriteToolDescription(): string {
  return `Writes a file to the local filesystem.

Usage:
- This tool will overwrite existing file
- You MUST use Read tool first for existing files
- Prefer Edit tool for modifications
- NEVER create *.md files unless requested
- Avoid emojis unless requested`
}
```

---

## 权限系统

### 权限检查流程

```typescript
// 1. 工具自定义检查
const toolCheck = await tool.checkPermissions?.(input, ctx)

// 2. 全局规则检查
const ruleCheck = hasPermissionsToUseTool(tool, input, ctx)

// 3. 用户交互 (如果需要)
if (ruleCheck.behavior === 'ask') {
  const decision = await askUser({
    tool,
    input,
    message: ruleCheck.message
  })
  return decision
}
```

### 权限规则

```typescript
export type PermissionRule = {
  rule: string  // e.g., "Bash(npm *)"
  behavior: PermissionBehavior  // 'allow' | 'deny' | 'ask'
  source: PermissionRuleSource
}

// 按优先级检查：
// 1. alwaysDenyRules  — 总是拒绝
// 2. alwaysAllowRules — 总是允许  
// 3. alwaysAskRules   — 总是询问
```

### 权限模式

```typescript
export type PermissionMode = 
  | 'default'          // 敏感操作需确认
  | 'plan'             // 只读模式
  | 'bypassPermissions' // 跳过所有检查
  | 'auto'             // 分类器自动决定
```

---

## 进度状态

长时间运行的工具可以流式返回进度：

```typescript
export type ToolCallProgress = 
  | BashProgress
  | AgentToolProgress
  | MCPProgress
  | WebSearchProgress
  | TaskOutputProgress

export type BashProgress = {
  type: 'bash_progress'
  output: string
  is_error?: boolean
}

export type AgentToolProgress = {
  type: 'agent_progress'
  agent_id: string
  message: string
}
```

---

## 关键设计要点

### 1. 工具隔离
- 每个工具独立模块
- 不直接依赖其他工具实现
- 通过 context 访问共享状态

### 2. 输入验证
- Zod schema 严格验证
- 类型推导 (TypeScript)
- 动态描述 (基于输入)

### 3. 错误处理
- 区分可重试错误
- 提供用户友好消息
- 保留原始错误信息

### 4. 流式输出
- AsyncGenerator 支持长时间操作
- 实时进度反馈
- 可取消操作

### 5. 权限分层
- 工具级权限检查
- 全局规则系统
- 用户最终决策

---

## 最佳实践

### 1. 命名规范
- 使用 PascalCase: `FileReadTool`
- 工具名动词开头: `Read`, `Write`, `Edit`
- MCP 工具前缀: `mcp__server__tool`

### 2. 描述编写
- 清晰说明工具功能
- 提供使用场景
- 包含注意事项
- 避免模糊描述

### 3. Schema 设计
- 必填字段放前面
- 提供默认值
- 使用 `.describe()` 说明字段
- 复杂结构用 `z.object()`

### 4. 权限设计
- 评估安全风险
- 提供细粒度规则
- 支持沙箱隔离
- 记录敏感操作

---

*生成时间: 2026-04-01*

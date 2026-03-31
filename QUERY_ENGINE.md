# Claude Code QueryEngine 详解

> 基于 `src/QueryEngine.ts` 源码分析（~46K 行）

---

## 核心类定义

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  private discoveredSkillNames: Set<string>
  private loadedNestedMemoryPaths: Set<string>

  constructor(config: QueryEngineConfig) {
    this.config = config
    this.mutableMessages = config.initialMessages ?? []
    this.abortController = config.abortController ?? createAbortController()
    this.permissionDenials = []
    this.readFileState = config.readFileCache
    this.totalUsage = EMPTY_USAGE
  }
}
```

---

## 配置接口

```typescript
export type QueryEngineConfig = {
  // 基础配置
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  
  // 状态管理
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  
  // 会话数据
  initialMessages?: Message[]
  readFileCache: FileStateCache
  
  // 模型配置
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  
  // 限制配置
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  
  // 输出配置
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  includePartialMessages?: boolean
  
  // 特殊处理
  handleElicitation?: ToolUseContext['handleElicitation']
  orphanedPermission?: OrphanedPermission
  snipReplay?: (msg: Message, store: Message[]) => { messages: Message[]; executed: boolean } | undefined
}
```

---

## 核心方法：submitMessage

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean }
): AsyncGenerator<SDKMessage, void, unknown> {
  const {
    cwd, commands, tools, mcpClients,
    verbose, thinkingConfig, maxTurns,
    maxBudgetUsd, taskBudget, canUseTool,
    customSystemPrompt, appendSystemPrompt,
    userSpecifiedModel, fallbackModel,
    jsonSchema, getAppState, setAppState,
    agents, setSDKStatus, orphanedPermission
  } = this.config

  // 1. 清理状态
  this.discoveredSkillNames.clear()
  setCwd(cwd)
  
  // 2. 处理用户输入
  const userInput = await processUserInput(prompt, {
    readFileState: this.readFileState,
    // ...
  })
  
  // 3. 添加用户消息
  this.mutableMessages.push({
    role: 'user',
    content: userInput.content
  })
  
  // 4. 主循环
  let turn = 0
  while (!this.abortController.signal.aborted) {
    // 检查限制
    if (maxTurns && turn >= maxTurns) break
    if (maxBudgetUsd && this.totalUsage.cost > maxBudgetUsd) break
    
    // 调用 API
    const response = await this.callAPI({
      messages: this.mutableMessages,
      tools,
      systemPrompt: await this.buildSystemPrompt(),
      thinkingConfig,
      model: userSpecifiedModel ?? getMainLoopModel()
    })
    
    // 流式返回消息
    yield { type: 'assistant', content: response }
    
    // 检查是否需要工具调用
    const toolCalls = extractToolCalls(response)
    
    if (toolCalls.length === 0) {
      // 没有工具调用，结束
      break
    }
    
    // 处理工具调用
    for (const toolCall of toolCalls) {
      const result = await this.executeToolCall(toolCall)
      
      // 添加工具结果
      this.mutableMessages.push({
        role: 'user',
        content: [{
          type: 'tool_result',
          tool_use_id: toolCall.id,
          content: result
        }]
      })
      
      // 流式返回工具结果
      yield { type: 'tool_result', toolCall, result }
    }
    
    turn++
  }
  
  // 5. 持久化会话
  if (persistSession) {
    await flushSessionStorage(this.mutableMessages)
  }
}
```

---

## 系统提示构建

```typescript
private async buildSystemPrompt(): Promise<SystemPrompt[]> {
  const prompts: SystemPrompt[] = []
  
  // 1. 基础系统提示
  prompts.push(...await fetchSystemPromptParts())
  
  // 2. 工具定义
  const toolDefs = this.config.tools.map(tool => ({
    name: tool.name,
    description: await tool.description(input, ctx),
    input_schema: zodToJsonSchema(tool.inputSchema)
  }))
  prompts.push({
    type: 'tools',
    tools: toolDefs
  })
  
  // 3. 自定义提示
  if (this.config.customSystemPrompt) {
    prompts.push({
      type: 'text',
      text: this.config.customSystemPrompt
    })
  }
  
  // 4. 附加提示
  if (this.config.appendSystemPrompt) {
    prompts.push({
      type: 'text',
      text: this.config.appendSystemPrompt
    })
  }
  
  // 5. 记忆
  const memoryPrompt = await loadMemoryPrompt()
  if (memoryPrompt) {
    prompts.push({
      type: 'text',
      text: memoryPrompt
    })
  }
  
  // 6. 上下文
  const context = await collectContext()
  prompts.push({
    type: 'text',
    text: formatContext(context)
  })
  
  return prompts
}
```

---

## API 调用

```typescript
private async callAPI(params: {
  messages: Message[]
  tools: Tools
  systemPrompt: SystemPrompt[]
  thinkingConfig?: ThinkingConfig
  model: string
}): Promise<AssistantMessage> {
  const { messages, tools, systemPrompt, thinkingConfig, model } = params
  
  // 1. 准备请求
  const request: MessageParam = {
    model,
    max_tokens: 8192,
    system: systemPrompt,
    messages: messages.map(formatMessage),
    tools: formatTools(tools),
    stream: true,
    
    // Thinking 模式
    ...(thinkingConfig?.enabled && {
      thinking: {
        type: 'enabled',
        budget_tokens: thinkingConfig.budgetTokens
      }
    })
  }
  
  // 2. 流式调用
  const stream = await anthropic.messages.stream(request)
  
  // 3. 累积使用量
  stream.on('messageStart', (event) => {
    updateUsage(this.totalUsage, event.message.usage)
  })
  
  // 4. 返回完整消息
  const finalMessage = await stream.finalMessage()
  
  // 5. 记录转录
  recordTranscript(finalMessage)
  
  return finalMessage
}
```

---

## 工具执行

```typescript
private async executeToolCall(toolCall: ToolUseBlock): Promise<string> {
  const { name, id, input } = toolCall
  
  // 1. 查找工具
  const tool = this.config.tools.find(t => toolMatchesName(t, name))
  if (!tool) {
    return `Unknown tool: ${name}`
  }
  
  // 2. 验证输入
  const validation = tool.inputSchema.safeParse(input)
  if (!validation.success) {
    return `Invalid input: ${validation.error.message}`
  }
  
  // 3. 权限检查
  const permission = await this.config.canUseTool(
    tool,
    validation.data,
    this.createToolUseContext(),
    { role: 'assistant' },
    id
  )
  
  if (permission.behavior === 'deny') {
    return `Permission denied: ${permission.message}`
  }
  
  // 4. 执行工具
  try {
    const result = tool.call(validation.data, this.createToolUseContext())
    
    // 流式结果
    if (isAsyncGenerator(result)) {
      let output = ''
      for await (const progress of result) {
        output += formatProgress(progress)
        // 可以 yield 进度
      }
      return output
    }
    
    return await result
  } catch (error) {
    return `Tool error: ${error.message}`
  }
}
```

---

## 上下文压缩

```typescript
// 当上下文过长时触发
private async compactIfNeeded(): Promise<void> {
  const tokenCount = estimateTokens(this.mutableMessages)
  
  if (tokenCount > MAX_CONTEXT_TOKENS) {
    // 使用 compact 服务压缩历史
    const compacted = await compactMessages(this.mutableMessages, {
      keepRecent: 10,  // 保留最近 10 条
      keepToolResults: true
    })
    
    this.mutableMessages = compacted
    
    // 记录压缩事件
    yield { type: 'compact', message: 'Context compacted' }
  }
}
```

---

## 流式消息类型

```typescript
export type SDKMessage =
  | { type: 'assistant'; content: AssistantMessage }
  | { type: 'user'; content: UserMessage }
  | { type: 'tool_result'; toolCall: ToolUseBlock; result: string }
  | { type: 'progress'; data: ToolCallProgress }
  | { type: 'compact'; message: string }
  | { type: 'permission'; tool: string; input: unknown; decision: PermissionDecision }
  | { type: 'error'; error: Error }
  | { type: 'complete'; usage: Usage }
```

---

## 错误处理

### 可重试错误

```typescript
export function categorizeRetryableAPIError(error: unknown): {
  retryable: boolean
  delay?: number
} {
  if (error instanceof APIError) {
    // 速率限制
    if (error.status === 429) {
      return { retryable: true, delay: parseRetryAfter(error) }
    }
    
    // 服务器错误
    if (error.status >= 500) {
      return { retryable: true, delay: 1000 }
    }
    
    // 过载
    if (error.status === 529) {
      return { retryable: true, delay: 5000 }
    }
  }
  
  return { retryable: false }
}
```

### 重试逻辑

```typescript
private async callWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  let lastError: Error
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn()
    } catch (error) {
      const { retryable, delay } = categorizeRetryableAPIError(error)
      
      if (!retryable || i === maxRetries - 1) {
        throw error
      }
      
      lastError = error
      await sleep(delay ?? 1000 * (i + 1))
    }
  }
  
  throw lastError
}
```

---

## 记忆加载

```typescript
// 加载持久化记忆
private async loadMemories(): Promise<void> {
  // 1. 用户级记忆
  const userMemory = await loadMemoryPrompt()
  
  // 2. 项目级记忆
  const projectMemory = await loadProjectMemory()
  
  // 3. Agent 记忆
  if (this.config.agents.length > 0) {
    for (const agent of this.config.agents) {
      const agentMemory = await loadAgentMemory(agent.type)
      this.loadedNestedMemoryPaths.add(agentMemory.path)
    }
  }
}
```

---

## 消息处理

### 用户输入处理

```typescript
export type ProcessUserInputContext = {
  readFileState: FileStateCache
  cwd: string
  isNonInteractiveSession: boolean
}

export async function processUserInput(
  prompt: string | ContentBlockParam[],
  ctx: ProcessUserInputContext
): Promise<{ content: ContentBlockParam[] }> {
  // 字符串输入
  if (typeof prompt === 'string') {
    return {
      content: [{ type: 'text', text: prompt }]
    }
  }
  
  // 结构化输入（已包含图片等）
  return { content: prompt }
}
```

### 消息格式化

```typescript
function formatMessage(msg: Message): MessageParam {
  switch (msg.role) {
    case 'user':
      return {
        role: 'user',
        content: msg.content.map(formatContentBlock)
      }
      
    case 'assistant':
      return {
        role: 'assistant',
        content: msg.content.map(formatContentBlock)
      }
  }
}

function formatContentBlock(block: ContentBlock): ContentBlockParam {
  switch (block.type) {
    case 'text':
      return { type: 'text', text: block.text }
      
    case 'image':
      return {
        type: 'image',
        source: {
          type: 'base64',
          media_type: block.mediaType,
          data: block.data
        }
      }
      
    case 'tool_use':
      return {
        type: 'tool_use',
        id: block.id,
        name: block.name,
        input: block.input
      }
      
    case 'tool_result':
      return {
        type: 'tool_result',
        tool_use_id: block.toolUseId,
        content: block.content,
        is_error: block.isError
      }
  }
}
```

---

## 使用量追踪

```typescript
export type NonNullableUsage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
  cost: number
}

// 累积使用量
function updateUsage(total: NonNullableUsage, delta: Usage): void {
  total.input_tokens += delta.input_tokens ?? 0
  total.output_tokens += delta.output_tokens ?? 0
  total.cache_creation_input_tokens += delta.cache_creation_input_tokens ?? 0
  total.cache_read_input_tokens += delta.cache_read_input_tokens ?? 0
  
  // 重新计算成本
  total.cost = calculateCost(total)
}
```

---

## 关键设计要点

### 1. 单一职责
- QueryEngine 只负责 LLM 交互循环
- 工具执行委托给各工具
- 权限检查委托给 canUseTool

### 2. 流式设计
- AsyncGenerator 支持实时反馈
- 每个重要事件都 yield
- 支持中途取消

### 3. 状态隔离
- 每个会话一个 QueryEngine
- 消息历史独立
- 文件缓存共享

### 4. 可配置性
- 最大轮次
- 预算限制
- 自定义系统提示
- 模型选择

### 5. 错误恢复
- 可重试错误自动重试
- 保存会话状态
- 支持恢复会话

---

## 最佳实践

### 1. 会话管理
- 长会话定期压缩
- 重要消息保留
- 工具结果缓存

### 2. 资源管理
- 设置合理超时
- 监控 token 使用
- 及时清理状态

### 3. 错误处理
- 区分临时/永久错误
- 提供用户友好信息
- 记录详细日志

### 4. 性能优化
- 并行加载资源
- 缓存系统提示
- 延迟加载记忆

---

*生成时间: 2026-04-01*

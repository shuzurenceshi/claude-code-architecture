# Claude Code 设计模式总结

> 从源码中提炼的核心设计模式和最佳实践

---

## 1. 🏗️ 架构模式

### 1.1 工具注册模式 (Tool Registry Pattern)

**问题**: 需要管理大量工具，支持条件加载和动态发现

**解决方案**:
```typescript
// tools.ts - 集中注册中心
export function getAllBaseTools(): Tools {
  return [
    // 核心工具 - 始终可用
    BashTool,
    FileReadTool,
    FileEditTool,
    
    // 条件工具 - Feature flags 控制
    ...(feature('PROACTIVE') ? [SleepTool] : []),
    ...(feature('KAIROS') ? [SendUserFileTool] : []),
    
    // 动态工具 - 懒加载
    getPowerShellTool(),
    getMcpTools(),
  ].filter(Boolean)
}
```

**优点**:
- 单一真实来源
- 条件编译 (死代码消除)
- 易于扩展

---

### 1.2 查询引擎模式 (Query Engine Pattern)

**问题**: 需要管理复杂的 LLM 交互循环

**解决方案**:
```typescript
class QueryEngine {
  private messages: Message[]
  private abortController: AbortController
  private usage: Usage
  
  async *submitMessage(prompt): AsyncGenerator<SDKMessage> {
    // 1. 添加用户消息
    this.messages.push({ role: 'user', content: prompt })
    
    // 2. 主循环
    while (!this.aborted) {
      // 调用 API
      const response = await this.callAPI()
      yield { type: 'assistant', content: response }
      
      // 处理工具调用
      const toolCalls = extractToolCalls(response)
      if (toolCalls.length === 0) break
      
      for (const call of toolCalls) {
        const result = await this.executeTool(call)
        yield { type: 'tool_result', call, result }
      }
    }
  }
}
```

**优点**:
- 封装复杂状态
- 流式输出
- 可中断

---

### 1.3 插件架构模式 (Plugin Architecture)

**问题**: 需要支持扩展和第三方功能

**解决方案**:
```typescript
// 插件接口
type Plugin = {
  name: string
  version: string
  activate: (ctx: PluginContext) => Promise<void>
  deactivate?: () => Promise<void>
}

// 插件加载器
class PluginLoader {
  private plugins: Map<string, Plugin> = new Map()
  
  async load(pluginPath: string): Promise<void> {
    const plugin = await import(pluginPath)
    await plugin.activate(this.createContext())
    this.plugins.set(plugin.name, plugin)
  }
  
  async unload(name: string): Promise<void> {
    const plugin = this.plugins.get(name)
    await plugin.deactivate?.()
    this.plugins.delete(name)
  }
}
```

**优点**:
- 松耦合
- 热加载
- 隔离性

---

## 2. 🔐 安全模式

### 2.1 分层权限模式 (Layered Permission Pattern)

**问题**: 需要灵活的权限控制，支持多级决策

**解决方案**:
```typescript
async function checkPermission(tool, input, ctx): PermissionResult {
  // 1. 工具自定义检查
  const toolCheck = await tool.checkPermissions?.(input, ctx)
  if (toolCheck) return toolCheck
  
  // 2. 全局规则检查 (按优先级)
  if (matchesRules(alwaysDenyRules, tool, input)) {
    return { behavior: 'deny' }
  }
  if (matchesRules(alwaysAllowRules, tool, input)) {
    return { behavior: 'allow' }
  }
  
  // 3. 权限模式
  switch (ctx.mode) {
    case 'bypassPermissions':
      return { behavior: 'allow' }
    case 'plan':
      return tool.isReadOnly() 
        ? { behavior: 'allow' }
        : { behavior: 'deny' }
    case 'default':
      // 4. 用户决策
      return askUser(tool, input)
  }
}
```

**优点**:
- 多层防御
- 灵活配置
- 可审计

---

### 2.2 沙箱执行模式 (Sandbox Execution Pattern)

**问题**: 需要安全执行不可信代码

**解决方案**:
```typescript
async function executeBash(command: string, ctx: Context): Promise<string> {
  // 1. 判断是否需要沙箱
  if (shouldUseSandbox(command)) {
    // 在隔离环境中执行
    return sandbox.execute(command, {
      timeout: 120000,
      network: 'restricted',
      filesystem: 'readonly'
    })
  }
  
  // 2. 主机执行
  return host.execute(command)
}

function shouldUseSandbox(command: string): boolean {
  // 危险命令必须沙箱
  if (matchesDestructivePatterns(command)) return true
  
  // 用户配置强制沙箱
  if (config.forceSandbox) return true
  
  return false
}
```

**优点**:
- 隔离风险
- 最小权限
- 可配置

---

### 2.3 敏感数据保护模式 (Sensitive Data Protection Pattern)

**问题**: 需要在日志和输出中保护敏感信息

**解决方案**:
```typescript
// 脱敏工具
class SensitiveDataRedactor {
  private patterns = [
    /api[_-]?key[_-]?.*/gi,
    /password[_-]?.*/gi,
    /token[_-]?.*/gi,
    /[a-zA-Z0-9]{32,}/g  // 长字符串
  ]
  
  redact(text: string): string {
    return this.patterns.reduce(
      (t, p) => t.replace(p, '[REDACTED]'),
      text
    )
  }
}

// 自动应用
function log(message: string): void {
  console.log(redactor.redact(message))
}
```

**优点**:
- 自动保护
- 防止泄露
- 合规要求

---

## 3. ⚡ 性能模式

### 3.1 并行预取模式 (Parallel Prefetch Pattern)

**问题**: 启动时间过长

**解决方案**:
```typescript
// main.tsx - 入口文件顶部
// 在其他导入之前启动预取
startMdmRawRead()
startKeychainPrefetch()
preconnectAPI()

// 然后才导入重型模块
import { QueryEngine } from './QueryEngine.js'
import { loadPlugins } from './plugins.js'
```

**优点**:
- 减少启动时间
- 并行化 I/O
- 用户体验更好

---

### 3.2 延迟加载模式 (Lazy Loading Pattern)

**问题**: 一次性加载所有模块太慢

**解决方案**:
```typescript
// 静态导入 - 核心功能
import { BashTool } from './BashTool.js'

// 动态导入 - 按需加载
const getVoiceTool = () => import('./VoiceTool.js').then(m => m.VoiceTool)

// 条件导入 - Feature flags
const SleepTool = feature('PROACTIVE')
  ? require('./SleepTool.js').SleepTool
  : null
```

**优点**:
- 减少初始包大小
- 按需加载
- 条件编译

---

### 3.3 缓存策略模式 (Caching Strategy Pattern)

**问题**: 重复计算和 I/O 开销

**解决方案**:
```typescript
// 文件状态缓存
class FileStateCache {
  private cache = new Map<string, { content: string, mtime: number }>()
  
  async read(path: string): Promise<string> {
    const stat = await fs.stat(path)
    const cached = this.cache.get(path)
    
    // 缓存有效
    if (cached && cached.mtime === stat.mtime) {
      return cached.content
    }
    
    // 读取并缓存
    const content = await fs.readFile(path, 'utf-8')
    this.cache.set(path, { content, mtime: stat.mtime })
    return content
  }
}

// 系统提示缓存
class SystemPromptCache {
  private static cached: string | null = null
  
  static async get(): Promise<string> {
    if (this.cached) return this.cached
    
    this.cached = await buildSystemPrompt()
    return this.cached
  }
  
  static invalidate(): void {
    this.cached = null
  }
}
```

**优点**:
- 减少重复计算
- 降低 I/O
- 提高响应速度

---

### 3.4 流式处理模式 (Streaming Processing Pattern)

**问题**: 长时间操作阻塞 UI

**解决方案**:
```typescript
async function* executeLongOperation(): AsyncGenerator<Progress> {
  yield { type: 'start', message: 'Starting...' }
  
  for (const item of items) {
    const result = await process(item)
    yield { type: 'progress', item, result }
  }
  
  yield { type: 'complete', message: 'Done!' }
}

// 使用
for await (const progress of executeLongOperation()) {
  updateUI(progress)
}
```

**优点**:
- 实时反馈
- 可中断
- 更好的 UX

---

## 4. 🔄 状态管理模式

### 4.1 不可变状态模式 (Immutable State Pattern)

**问题**: 状态变化难以追踪和调试

**解决方案**:
```typescript
type AppState = {
  messages: Message[]
  tools: Tools
  config: Config
}

class StateManager {
  private state: AppState = initialState
  
  getState(): Readonly<AppState> {
    return this.state
  }
  
  setState(updater: (prev: AppState) => AppState): void {
    this.state = updater(this.state)
    this.notifyListeners()
  }
}

// 使用
stateManager.setState(prev => ({
  ...prev,
  messages: [...prev.messages, newMessage]
}))
```

**优点**:
- 可预测
- 易于调试
- 支持撤销

---

### 4.2 事件驱动模式 (Event-Driven Pattern)

**问题**: 组件间通信复杂

**解决方案**:
```typescript
type Event = 
  | { type: 'tool_start'; tool: string }
  | { type: 'tool_end'; tool: string; result: string }
  | { type: 'error'; error: Error }

class EventBus {
  private listeners = new Map<string, Set<Function>>()
  
  on(event: string, handler: Function): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set())
    }
    this.listeners.get(event)!.add(handler)
    
    // 返回取消订阅函数
    return () => this.listeners.get(event)?.delete(handler)
  }
  
  emit(event: string, data: unknown): void {
    this.listeners.get(event)?.forEach(h => h(data))
  }
}

// 使用
eventBus.on('tool_start', ({ tool }) => {
  showSpinner(`Running ${tool}...`)
})
```

**优点**:
- 松耦合
- 可扩展
- 易于测试

---

### 4.3 会话持久化模式 (Session Persistence Pattern)

**问题**: 会话状态需要在重启后恢复

**解决方案**:
```typescript
class SessionManager {
  async save(session: Session): Promise<void> {
    const data = {
      id: session.id,
      messages: session.messages,
      state: session.state,
      timestamp: Date.now()
    }
    
    await fs.writeFile(
      this.getSessionPath(session.id),
      JSON.stringify(data)
    )
  }
  
  async load(id: string): Promise<Session> {
    const data = await fs.readFile(this.getSessionPath(id), 'utf-8')
    return JSON.parse(data)
  }
  
  async list(): Promise<SessionMeta[]> {
    const files = await fs.readdir(this.sessionsDir)
    return files.map(f => this.parseSessionMeta(f))
  }
}
```

**优点**:
- 持久化
- 可恢复
- 支持多会话

---

## 5. 🛠️ 错误处理模式

### 5.1 分类重试模式 (Categorized Retry Pattern)

**问题**: 不同错误需要不同的重试策略

**解决方案**:
```typescript
type RetryCategory = 'retryable' | 'permanent' | 'rate_limited'

function categorizeError(error: Error): RetryCategory {
  if (error instanceof RateLimitError) return 'rate_limited'
  if (error instanceof NetworkError) return 'retryable'
  if (error instanceof ValidationError) return 'permanent'
  return 'retryable'
}

async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn()
    } catch (error) {
      const category = categorizeError(error)
      
      if (category === 'permanent') throw error
      
      const delay = category === 'rate_limited'
        ? parseRetryAfter(error)
        : 1000 * Math.pow(2, i)
      
      await sleep(delay)
    }
  }
  
  throw new Error('Max retries exceeded')
}
```

**优点**:
- 智能重试
- 节省时间
- 避免无意义重试

---

### 5.2 错误边界模式 (Error Boundary Pattern)

**问题**: 单个工具失败不应影响整个系统

**解决方案**:
```typescript
async function executeToolSafely(
  tool: Tool,
  input: unknown,
  ctx: Context
): Promise<string> {
  try {
    return await tool.call(input, ctx)
  } catch (error) {
    // 记录错误
    logError(tool.name, error)
    
    // 返回用户友好消息
    return formatToolError(tool.name, error)
  }
}

function formatToolError(name: string, error: Error): string {
  if (error instanceof PermissionError) {
    return `Permission denied for ${name}. Reason: ${error.message}`
  }
  
  if (error instanceof TimeoutError) {
    return `${name} timed out. Try increasing the timeout.`
  }
  
  return `${name} failed: ${error.message}`
}
```

**优点**:
- 隔离错误
- 用户友好
- 可恢复

---

### 5.3 优雅降级模式 (Graceful Degradation Pattern)

**问题**: 功能不可用时应继续工作

**解决方案**:
```typescript
async function getTools(): Promise<Tools> {
  const tools: Tools = []
  
  // 核心工具 - 必须成功
  tools.push(...getCoreTools())
  
  // MCP 工具 - 可选
  try {
    const mcpTools = await loadMCPTools()
    tools.push(...mcpTools)
  } catch (error) {
    log.warning('MCP tools unavailable:', error)
    // 继续运行，只是没有 MCP 工具
  }
  
  // 插件工具 - 可选
  try {
    const pluginTools = await loadPluginTools()
    tools.push(...pluginTools)
  } catch (error) {
    log.warning('Plugin tools unavailable:', error)
  }
  
  return tools
}
```

**优点**:
- 部分可用
- 更稳定
- 更好的 UX

---

## 6. 🧪 测试模式

### 6.1 工具 Mock 模式 (Tool Mocking Pattern)

**问题**: 测试时不应调用真实 API

**解决方案**:
```typescript
// 生产工具
export const BashTool: Tool = {
  name: 'Bash',
  call: async (input) => {
    return execSync(input.command).toString()
  }
}

// Mock 工具
export const MockBashTool: Tool = {
  name: 'Bash',
  call: async (input) => {
    // 返回预设响应
    return mockResponses.get(input.command) ?? 'mock output'
  }
}

// 测试配置
const testContext: ToolUseContext = {
  options: {
    tools: [MockBashTool, MockFileTool, ...],
    // ...
  }
}
```

---

### 6.2 集成测试模式 (Integration Test Pattern)

**问题**: 需要测试完整流程

**解决方案**:
```typescript
describe('QueryEngine', () => {
  it('should execute tool and return result', async () => {
    // 1. 准备
    const engine = new QueryEngine({
      tools: [MockBashTool],
      // ...
    })
    
    // 2. 执行
    const messages = []
    for await (const msg of engine.submitMessage('List files')) {
      messages.push(msg)
    }
    
    // 3. 验证
    expect(messages).toHaveLength(3)
    expect(messages[0].type).toBe('assistant')
    expect(messages[1].type).toBe('tool_result')
    expect(messages[2].type).toBe('assistant')
  })
})
```

---

## 7. 📦 代码组织模式

### 7.1 功能模块化 (Feature Modularity)

```
tools/
├── BashTool/
│   ├── index.ts           # 导出
│   ├── prompt.ts          # LLM 描述
│   ├── permissions.ts     # 权限检查
│   ├── security.ts        # 安全验证
│   ├── utils.ts           # 工具函数
│   └── __tests__/         # 测试
```

**优点**:
- 高内聚
- 低耦合
- 易于维护

---

### 7.2 类型集中管理 (Centralized Types)

```typescript
// types/tools.ts - 集中定义
export type Tool<Input = Record<string, unknown>> = { ... }
export type ToolUseContext = { ... }
export type ToolCallProgress = { ... }

// 工具文件中导入
import type { Tool, ToolUseContext } from '../types/tools.js'
```

**优点**:
- 单一来源
- 避免循环依赖
- 类型安全

---

## 总结

| 模式 | 解决的问题 | 关键技术 |
|-----|----------|---------|
| 工具注册 | 管理大量工具 | Feature flags, 懒加载 |
| 查询引擎 | LLM 交互循环 | AsyncGenerator, 状态机 |
| 分层权限 | 灵活权限控制 | 规则引擎, 模式匹配 |
| 并行预取 | 启动性能 | Promise.all, 预加载 |
| 流式处理 | 长时间操作 | AsyncGenerator |
| 错误边界 | 故障隔离 | try-catch, 优雅降级 |
| 事件驱动 | 组件通信 | 发布订阅 |

这些模式共同构成了 Claude Code 稳定、高效、安全的架构基础。

---

*生成时间: 2026-04-01*

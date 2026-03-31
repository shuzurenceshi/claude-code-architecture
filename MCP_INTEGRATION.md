# Claude Code MCP 集成详解

> 基于 `src/services/mcp/` 源码分析

---

## MCP 协议概述

**Model Context Protocol (MCP)** 是 Anthropic 推出的标准协议，用于连接 AI 模型与外部工具/数据源。

### 核心概念

```
┌─────────────────────────────────────────────────┐
│                  Claude Code                     │
│                   (MCP Client)                   │
├─────────────────────────────────────────────────┤
│  - 连接多个 MCP Server                           │
│  - 发现并调用工具                                 │
│  - 读取资源                                      │
│  - 获取提示模板                                  │
└───────────────────────┬─────────────────────────┘
                        │ MCP Protocol
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ Server A│    │ Server B│    │ Server C│
   │ (stdio) │    │  (SSE)  │    │ (HTTP)  │
   └─────────┘    └─────────┘    └─────────┘
```

---

## MCP 客户端架构

### 传输层

```typescript
// MCP SDK 提供三种传输

// 1. Stdio 传输 - 本地进程
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'

const stdioTransport = new StdioClientTransport({
  command: 'npx',
  args: ['-y', '@anthropic-ai/mcp-server-filesystem']
})

// 2. SSE 传输 - HTTP Server-Sent Events
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'

const sseTransport = new SSEClientTransport({
  url: 'https://mcp-server.example.com/sse'
})

// 3. Streamable HTTP 传输
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'

const httpTransport = new StreamableHTTPClientTransport({
  url: 'https://mcp-server.example.com/mcp'
})
```

### 客户端初始化

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js'

class MCPClientManager {
  async connectServer(config: MCPServerConfig): Promise<MCPServerConnection> {
    // 1. 创建传输
    const transport = this.createTransport(config)
    
    // 2. 创建客户端
    const client = new Client({
      name: 'claude-code',
      version: '1.0.0'
    }, {
      capabilities: {
        tools: {},
        resources: {},
        prompts: {},
        roots: { listChanged: true }
      }
    })
    
    // 3. 连接
    await client.connect(transport)
    
    // 4. 获取能力
    const { tools } = await client.request(
      { method: 'tools/list' },
      ListToolsResultSchema
    )
    
    // 5. 创建工具代理
    const mcpTools = tools.map(tool => new MCPTool(client, tool))
    
    return {
      name: config.name,
      client,
      tools: mcpTools,
      status: 'connected'
    }
  }
}
```

---

## MCPTool 实现

### 工具定义

```typescript
export class MCPTool implements Tool {
  constructor(
    private connection: MCPServerConnection,
    private toolDef: MCPToolDefinition
  ) {
    this.name = `mcp__${connection.name}__${toolDef.name}`
  }
  
  name: string
  
  get description(): string {
    return this.toolDef.description
  }
  
  get inputSchema(): z.ZodSchema {
    // 将 JSON Schema 转换为 Zod
    return convertMCPSchemaToZod(this.toolDef.inputSchema)
  }
  
  isEnabled = () => true
  
  async call(input: Record<string, unknown>, ctx: ToolUseContext): Promise<string> {
    try {
      const result = await this.connection.client.request(
        {
          method: 'tools/call',
          params: {
            name: this.toolDef.name,
            arguments: input
          }
        },
        CallToolResultSchema
      )
      
      return this.formatResult(result)
    } catch (error) {
      // 处理 URL elicitation 错误
      if (error.code === -32042 && ctx.handleElicitation) {
        const elicitation = await ctx.handleElicitation(
          this.connection.name,
          error.params
        )
        // 用户完成授权后重试
        return this.call(input, ctx)
      }
      
      throw error
    }
  }
  
  private formatResult(result: CallToolResult): string {
    return result.content
      .map(block => {
        if (block.type === 'text') return block.text
        if (block.type === 'image') return `[Image: ${block.mimeType}]`
        if (block.type === 'resource') return formatResource(block.resource)
        return ''
      })
      .join('\n')
  }
}
```

### Schema 转换

```typescript
function convertMCPSchemaToZod(schema: JSONSchema): z.ZodSchema {
  if (!schema) return z.object({})
  
  const shape: Record<string, z.ZodTypeAny> = {}
  
  for (const [key, prop] of Object.entries(schema.properties ?? {})) {
    let field: z.ZodTypeAny
    
    switch (prop.type) {
      case 'string':
        field = z.string()
        break
      case 'number':
      case 'integer':
        field = z.number()
        break
      case 'boolean':
        field = z.boolean()
        break
      case 'array':
        field = z.array(convertMCPSchemaToZod(prop.items))
        break
      case 'object':
        field = convertMCPSchemaToZod(prop)
        break
      default:
        field = z.any()
    }
    
    // 必填检查
    if (!schema.required?.includes(key)) {
      field = field.optional()
    }
    
    // 描述
    if (prop.description) {
      field = field.describe(prop.description)
    }
    
    shape[key] = field
  }
  
  return z.object(shape)
}
```

---

## MCP 服务器配置

### 配置文件格式

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-filesystem", "/path/to/dir"],
      "env": {
        "NODE_ENV": "production"
      }
    },
    "github": {
      "type": "sse",
      "url": "https://mcp.github.com/sse",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "uvx",
      "args": ["mcp-server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

### 服务器类型

```typescript
export type MCPServerConfig = 
  | StdioServerConfig
  | SSEServerConfig
  | HTTPServerConfig

export type StdioServerConfig = {
  type?: 'stdio'
  command: string
  args?: string[]
  env?: Record<string, string>
  cwd?: string
}

export type SSEServerConfig = {
  type: 'sse'
  url: string
  headers?: Record<string, string>
}

export type HTTPServerConfig = {
  type: 'http'
  url: string
  headers?: Record<string, string>
}
```

---

## 资源访问

### 列出资源

```typescript
export class ListMcpResourcesTool implements Tool {
  name = 'list_mcp_resources'
  
  async call(input: { server?: string }, ctx: ToolUseContext): Promise<string> {
    const servers = input.server 
      ? [ctx.options.mcpClients.find(s => s.name === input.server)]
      : ctx.options.mcpClients
    
    const results = await Promise.all(
      servers.map(async server => {
        const resources = await server.client.request(
          { method: 'resources/list' },
          ListResourcesResultSchema
        )
        return { server: server.name, resources }
      })
    )
    
    return formatResourceList(results)
  }
}
```

### 读取资源

```typescript
export class ReadMcpResourceTool implements Tool {
  name = 'read_mcp_resource'
  
  inputSchema = z.object({
    server: z.string(),
    uri: z.string()
  })
  
  async call(input: { server: string; uri: string }, ctx: ToolUseContext): Promise<string> {
    const server = ctx.options.mcpClients.find(s => s.name === input.server)
    
    const result = await server.client.request(
      {
        method: 'resources/read',
        params: { uri: input.uri }
      },
      ReadResourceResultSchema
    )
    
    return formatResourceContent(result.contents)
  }
}
```

---

## OAuth 认证

### OAuth 流程

```typescript
export async function handleMCPOAuth(
  serverName: string,
  config: OAuthServerConfig
): Promise<string> {
  // 1. 启动本地服务器监听回调
  const callbackServer = await startCallbackServer()
  
  // 2. 构建授权 URL
  const authUrl = new URL(config.authorizationEndpoint)
  authUrl.searchParams.set('client_id', config.clientId)
  authUrl.searchParams.set('redirect_uri', callbackServer.url)
  authUrl.searchParams.set('response_type', 'code')
  authUrl.searchParams.set('scope', config.scopes.join(' '))
  authUrl.searchParams.set('state', generateState())
  
  // 3. 打开浏览器
  await openBrowser(authUrl.toString())
  
  // 4. 等待回调
  const { code } = await callbackServer.waitForCallback()
  
  // 5. 交换 token
  const tokens = await fetch(config.tokenEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      client_id: config.clientId,
      redirect_uri: callbackServer.url
    })
  }).then(r => r.json())
  
  // 6. 存储 token
  await storeOAuthTokens(serverName, tokens)
  
  return tokens.access_token
}
```

### Token 刷新

```typescript
export async function refreshOAuthToken(
  serverName: string,
  refreshToken: string,
  config: OAuthServerConfig
): Promise<OAuthTokens> {
  const tokens = await fetch(config.tokenEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: config.clientId
    })
  }).then(r => r.json())
  
  await storeOAuthTokens(serverName, tokens)
  
  return tokens
}
```

---

## Elicitation (URL 授权)

当 MCP 工具需要用户授权时，会触发 elicitation：

```typescript
// MCP SDK 定义
export type ElicitRequestURLParams = {
  url: string
  message: string
}

// 错误码 -32042 表示需要 URL elicitation
export async function handleElicitation(
  serverName: string,
  params: ElicitRequestURLParams,
  ctx: ToolUseContext
): Promise<ElicitResult> {
  // REPL 模式：显示 URL 让用户访问
  if (isReplMode) {
    console.log(`\n${params.message}`)
    console.log(`Please visit: ${params.url}`)
    
    // 等待用户完成
    await waitForUserConfirmation()
    
    return { action: 'accept' }
  }
  
  // SDK 模式：发送给客户端处理
  if (ctx.handleElicitation) {
    return ctx.handleElicitation(serverName, params)
  }
  
  // 自动打开浏览器
  await openBrowser(params.url)
  await waitForUserConfirmation()
  
  return { action: 'accept' }
}
```

---

## 连接管理

### 连接状态

```typescript
export type MCPServerConnection = {
  name: string
  client: Client
  status: 'connecting' | 'connected' | 'error' | 'disconnected'
  tools: MCPTool[]
  resources: ServerResource[]
  error?: Error
  reconnectAttempts: number
}
```

### 自动重连

```typescript
class MCPConnectionManager {
  private connections: Map<string, MCPServerConnection> = new Map()
  
  async manageConnections(config: Record<string, MCPServerConfig>): Promise<void> {
    // 1. 断开已移除的服务器
    for (const [name] of this.connections) {
      if (!config[name]) {
        await this.disconnectServer(name)
      }
    }
    
    // 2. 连接新服务器
    for (const [name, serverConfig] of Object.entries(config)) {
      if (!this.connections.has(name)) {
        await this.connectServer(name, serverConfig)
      }
    }
    
    // 3. 重连断开的服务器
    for (const [name, conn] of this.connections) {
      if (conn.status === 'error' && conn.reconnectAttempts < 3) {
        await this.reconnectServer(name)
      }
    }
  }
  
  private async reconnectServer(name: string): Promise<void> {
    const conn = this.connections.get(name)
    conn.reconnectAttempts++
    
    try {
      await this.connectServer(name, conn.config)
      conn.reconnectAttempts = 0
    } catch (error) {
      // 指数退避
      await sleep(1000 * Math.pow(2, conn.reconnectAttempts))
    }
  }
}
```

---

## 权限管理

### MCP 工具权限

```typescript
export type MCPPermissionRule = {
  server: string
  tool: string
  behavior: 'allow' | 'deny' | 'ask'
}

// 服务器级别权限
export type MCPServerPermission = {
  server: string
  allowTools: string[] | '*'  // '*' = 允许所有
  denyTools: string[]
  requireConfirmation: boolean
}
```

### 权限检查

```typescript
export async function checkMCPToolPermission(
  tool: MCPTool,
  input: unknown,
  ctx: ToolPermissionContext
): Promise<PermissionResult> {
  const serverName = tool.connection.name
  const toolName = tool.toolDef.name
  
  // 1. 检查服务器级别权限
  const serverPerm = ctx.mcpPermissions?.[serverName]
  if (serverPerm) {
    if (serverPerm.denyTools.includes(toolName)) {
      return { behavior: 'deny', message: 'Tool denied by server policy' }
    }
    
    if (serverPerm.allowTools === '*' || serverPerm.allowTools.includes(toolName)) {
      if (!serverPerm.requireConfirmation) {
        return { behavior: 'allow' }
      }
    }
  }
  
  // 2. 检查全局权限规则
  const rule = getDenyRuleForTool(`mcp__${serverName}__${toolName}`, ctx)
  if (rule) {
    return rule
  }
  
  // 3. 默认询问用户
  return {
    behavior: 'ask',
    message: `Allow MCP tool "${toolName}" from server "${serverName}"?`
  }
}
```

---

## 调试与日志

### MCP 调试日志

```typescript
export function logMCPDebug(serverName: string, message: string, data?: unknown): void {
  if (isEnvTruthy('CLAUDE_CODE_MCP_DEBUG')) {
    console.error(`[MCP:${serverName}] ${message}`, data ?? '')
  }
}

export function logMCPError(serverName: string, error: Error): void {
  console.error(`[MCP:${serverName}] Error:`, error)
  logEvent('mcp_error', {
    server: serverName,
    error: error.message,
    stack: error.stack
  })
}
```

### 连接诊断

```typescript
export async function diagnoseMCPConnection(name: string, config: MCPServerConfig): Promise<{
  status: 'ok' | 'error'
  tools: number
  resources: number
  latency: number
  error?: string
}> {
  const start = Date.now()
  
  try {
    const connection = await connectServer(name, config)
    
    return {
      status: 'ok',
      tools: connection.tools.length,
      resources: connection.resources.length,
      latency: Date.now() - start
    }
  } catch (error) {
    return {
      status: 'error',
      tools: 0,
      resources: 0,
      latency: Date.now() - start,
      error: error.message
    }
  }
}
```

---

## 最佳实践

### 1. 服务器配置
- 使用环境变量存储敏感信息
- 设置合理的超时
- 配置自动重连

### 2. 错误处理
- 区分连接错误和调用错误
- 提供用户友好的错误信息
- 记录详细日志

### 3. 性能优化
- 并行连接多个服务器
- 缓存工具列表
- 懒加载资源

### 4. 安全性
- 限制服务器权限
- 审核工具调用
- 使用 OAuth 而非明文 token

---

## 常用 MCP 服务器

| 服务器 | 功能 |
|-------|------|
| `@anthropic-ai/mcp-server-filesystem` | 文件系统访问 |
| `@anthropic-ai/mcp-server-postgres` | PostgreSQL 数据库 |
| `@anthropic-ai/mcp-server-sqlite` | SQLite 数据库 |
| `@anthropic-ai/mcp-server-github` | GitHub API |
| `@anthropic-ai/mcp-server-brave-search` | Brave 搜索 |
| `@anthropic-ai/mcp-server-puppeteer` | 浏览器自动化 |
| `@anthropic-ai/mcp-server-slack` | Slack API |

---

*生成时间: 2026-04-01*

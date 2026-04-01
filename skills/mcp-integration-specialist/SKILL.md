---
name: mcp-integration-specialist
description: MCP 集成专家 - MCP 服务器开发、传输协议配置、OAuth 认证、工具注册、资源管理
aliases: ['MCP 专家', 'mcp-dev']
argumentHint: '<集成场景> [服务器类型]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要开发 MCP 服务器、配置传输协议、实现 OAuth 认证流、或集成第三方 MCP 服务时
---

# MCP Integration Specialist

使用此技能当你需要进行 MCP 服务器开发、传输协议配置、OAuth 认证实现、工具注册和资源管理。

## 目标

产出完整的 MCP 集成方案，包括服务器开发指南、传输协议配置、OAuth 认证流程、工具注册机制和资源管理系统。

## 核心能力（基于 Claude Code源码）

### 1. MCP 服务器开发框架

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from '@modelcontextprotocol/sdk/types.js'

// 创建 MCP 服务器
const server = new Server(
  {
    name: 'my-mcp-server',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
      resources: {},
    },
  }
)

// 实现工具列表
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'hello_world',
        description: 'Say hello to someone',
        inputSchema: {
          type: 'object',
          properties: {
            name: { 
              type: 'string', 
              description: 'Name of the person to greet' 
            },
          },
          required: ['name'],
        },
      },
      // 更多工具...
    ],
  }
})

// 实现工具调用
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params
  
  switch (name) {
    case 'hello_world': {
      const greeting = `Hello, ${args!.name}!`
      return {
        content: [{ type: 'text', text: greeting }],
      }
    }
    
    default:
      throw new Error(`Unknown tool: ${name}`)
  }
})

// 启动服务器
async function main() {
  const transport = new StdioServerTransport()
  await server.connect(transport)
  console.error('MCP Server running on stdio')
}

main().catch(console.error)
```

### 2. 传输协议配置

#### Stdio Transport（本地进程）
```typescript
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'

const transport = new StdioClientTransport({
  command: 'node',
  args: ['/path/to/mcp-server.js'],
  env: {
    ...process.env,
    API_KEY: 'your-api-key',
  },
})
```

#### SSE Transport（Server-Sent Events）
```typescript
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'

const transport = new SSEClientTransport({
  url: new URL('http://localhost:3000/sse'),
  requestInit: {
    headers: {
      'Authorization': `Bearer ${token}`,
    },
  },
})
```

#### HTTP Streaming Transport
```typescript
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'

const transport = new StreamableHTTPClientTransport({
  url: new URL('http://localhost:3000/mcp'),
  fetch: createFetchWithInit({
    headers: {
      'Authorization': `Bearer ${token}`,
    },
  }),
})
```

#### WebSocket Transport
```typescript
import { WebSocketTransport } from './utils/mcpWebSocketTransport.js'

const transport = new WebSocketTransport({
  url: 'ws://localhost:3000/mcp',
  tlsOptions: getWebSocketTLSOptions(),
  agent: getWebSocketProxyAgent(),
})
```

### 3. OAuth 认证实现

```typescript
import { OAuth2Client } from 'oauth2-client'

class MCPOAuthProvider {
  private client: OAuth2Client
  private tokenStore: Map<string, OAuthToken> = new Map()
  
  constructor(clientId: string, clientSecret: string) {
    this.client = new OAuth2Client({
      clientId,
      clientSecret,
      tokenEndpoint: 'https://auth.example.com/oauth/token',
      authorizationEndpoint: 'https://auth.example.com/oauth/authorize',
    })
  }
  
  /**
   * 获取访问令牌（自动刷新）
   */
  async getAccessToken(serverName: string): Promise<string> {
    const cached = this.tokenStore.get(serverName)
    
    if (!cached || this.isTokenExpired(cached)) {
      // Token 不存在或已过期，需要刷新或重新认证
      const refreshed = await this.refreshOrAuthenticate(serverName)
      this.tokenStore.set(serverName, refreshed)
      return refreshed.access_token
    }
    
    return cached.access_token
  }
  
  /**
   * 刷新或认证
   */
  private async refreshOrAuthenticate(serverName: string): Promise<OAuthToken> {
    const cached = this.tokenStore.get(serverName)
    
    if (cached?.refresh_token) {
      try {
        // 尝试刷新
        const refreshed = await this.client.refreshToken(cached.refresh_token)
        return refreshed
      } catch (error) {
        // 刷新失败，需要重新认证
        console.error('Token refresh failed, re-authenticating:', error)
      }
    }
    
    // 启动认证流程
    return this.authenticate(serverName)
  }
  
  /**
   * 启动 OAuth 认证流程
   */
  private async authenticate(serverName: string): Promise<OAuthToken> {
    const authUrl = this.client.code.getAuthorizationUrl({
      redirectURI: 'http://localhost:3000/callback',
      scope: ['read', 'write'],
    })
    
    // 打开浏览器进行认证
    await open(authUrl.toString())
    
    // 等待回调
    const code = await this.waitForCallback()
    
    // 交换令牌
    const token = await this.client.grant({
      grantType: 'authorization_code',
      code,
      redirectURI: 'http://localhost:3000/callback',
    })
    
    return token
  }
  
  /**
   * 检查 Token 是否过期
   */
  private isTokenExpired(token: OAuthToken): boolean {
    // 提前 5 分钟视为过期
    return Date.now() >= (token.expires_at - 5 * 60 * 1000)
  }
  
  /**
   * 等待认证回调
   */
  private async waitForCallback(): Promise<string> {
    return new Promise((resolve, reject) => {
      // 实现 HTTP 服务器接收回调
      const httpServer = require('http').createServer((req: any, res: any) => {
        const url = new URL(req.url, 'http://localhost:3000')
        const code = url.searchParams.get('code')
        
        if (code) {
          resolve(code)
          res.writeHead(200)
          res.end('Authentication successful! You can close this window.')
        } else {
          reject(new Error('No code in callback'))
          res.writeHead(400)
          res.end('Authentication failed')
        }
        
        httpServer.close()
      })
      
      httpServer.listen(3000)
      
      // 设置超时
      setTimeout(() => {
        httpServer.close()
        reject(new Error('Authentication timeout'))
      }, 5 * 60 * 1000) // 5 分钟超时
    })
  }
}
```

### 4. 工具注册系统

```typescript
interface MCPToolRegistration {
  name: string
  description: string
  inputSchema: Record<string, unknown>
  handler: (input: Record<string, unknown>) => Promise<ToolResult>
  allowedTools?: string[]  // 白名单
  disallowedTools?: string[]  // 黑名单
}

class MCPToolRegistry {
  private tools: Map<string, MCPToolRegistration> = new Map()
  
  /**
   * 注册工具
   */
  register(registration: MCPToolRegistration): void {
    this.tools.set(registration.name, registration)
  }
  
  /**
   * 注销工具
   */
  unregister(toolName: string): void {
    this.tools.delete(toolName)
  }
  
  /**
   * 列出所有工具
   */
  listTools(): MCPToolRegistration[] {
    return Array.from(this.tools.values())
  }
  
  /**
   * 调用工具
   */
  async callTool(
    name: string,
    input: Record<string, unknown>
  ): Promise<ToolResult> {
    const tool = this.tools.get(name)
    
    if (!tool) {
      throw new Error(`Unknown tool: ${name}`)
    }
    
    // 验证输入
    this.validateInput(tool.inputSchema, input)
    
    // 执行工具
    try {
      return await tool.handler(input)
    } catch (error) {
      return {
        content: [
          { 
            type: 'text', 
            text: `Error executing tool: ${error instanceof Error ? error.message : String(error)}` 
          },
        ],
        isError: true,
      }
    }
  }
  
  /**
   * 验证输入 Schema
   */
  private validateInput(
    schema: Record<string, unknown>,
    input: Record<string, unknown>
  ): void {
    // 实现 JSON Schema 验证
    // 可以使用 ajv 或其他验证库
  }
}
```

### 5. 资源管理系统

```typescript
interface MCPResource {
  uri: string
  name: string
  description?: string
  mimeType?: string
}

class MCPResourceManager {
  private resources: Map<string, MCPResource> = new Map()
  
  /**
   * 注册资源
   */
  register(resource: MCPResource): void {
    this.resources.set(resource.uri, resource)
  }
  
  /**
   * 列出所有资源
   */
  listResources(): MCPResource[] {
    return Array.from(this.resources.values())
  }
  
  /**
   * 读取资源
   */
  async readResource(uri: string): Promise<ResourceContents> {
    const resource = this.resources.get(uri)
    
    if (!resource) {
      throw new Error(`Resource not found: ${uri}`)
    }
    
    // 根据 URI 类型读取资源
    if (uri.startsWith('file://')) {
      return this.readFileResource(uri)
    } else if (uri.startsWith('http://') || uri.startsWith('https://')) {
      return this.readHttpResource(uri)
    } else {
      throw new Error(`Unsupported resource URI: ${uri}`)
    }
  }
  
  /**
   * 读取文件资源
   */
  private async readFileResource(uri: string): Promise<ResourceContents> {
    const filePath = uri.replace('file://', '')
    const content = await fs.promises.readFile(filePath, 'utf-8')
    
    return {
      uri,
      mimeType: 'text/plain',
      text: content,
    }
  }
  
  /**
   * 读取 HTTP 资源
   */
  private async readHttpResource(uri: string): Promise<ResourceContents> {
    const response = await fetch(uri)
    const contentType = response.headers.get('content-type') || 'text/plain'
    const text = await response.text()
    
    return {
      uri,
      mimeType: contentType,
      text,
    }
  }
}
```

### 6. MCP 客户端连接管理

```typescript
class MCPConnectionManager {
  private connections: Map<string, MCPServerConnection> = new Map()
  
  /**
   * 连接到 MCP 服务器
   */
  async connect(
    serverName: string,
    config: MCPServerConfig
  ): Promise<MCPServerConnection> {
    // 检查是否已连接
    const existing = this.connections.get(serverName)
    if (existing && existing.status === 'connected') {
      return existing
    }
    
    // 创建传输
    const transport = this.createTransport(config)
    
    // 创建客户端
    const client = new Client({
      name: 'claude-code',
      version: pkg.version,
    })
    
    // 连接
    await client.connect(transport)
    
    // 获取服务器能力
    const tools = await client.listTools()
    const resources = await client.listResources()
    const prompts = await client.listPrompts()
    
    // 存储连接
    const connection: MCPServerConnection = {
      serverName,
      config,
      client,
      transport,
      tools,
      resources,
      prompts,
      status: 'connected',
    }
    
    this.connections.set(serverName, connection)
    
    // 发送连接事件
    this.notifyConnected(serverName)
    
    return connection
  }
  
  /**
   * 断开连接
   */
  async disconnect(serverName: string): Promise<void> {
    const connection = this.connections.get(serverName)
    
    if (connection) {
      await connection.client.close()
      this.connections.delete(serverName)
      this.notifyDisconnected(serverName)
    }
  }
  
  /**
   * 重新连接
   */
  async reconnect(serverName: string): Promise<void> {
    await this.disconnect(serverName)
    const config = this.getConnectionConfig(serverName)
    await this.connect(serverName, config)
  }
  
  /**
   * 创建传输
   */
  private createTransport(config: MCPServerConfig): Transport {
    switch (config.type) {
      case 'stdio':
        return new StdioClientTransport({
          command: config.command,
          args: config.args,
          env: config.env,
        })
      
      case 'sse':
        return new SSEClientTransport({
          url: new URL(config.url),
          requestInit: config.headers ? { headers: config.headers } : undefined,
        })
      
      case 'http':
        return new StreamableHTTPClientTransport({
          url: new URL(config.url),
          fetch: createFetchWithInit(config.fetchOptions),
        })
      
      case 'websocket':
        return new WebSocketTransport({
          url: config.url,
          tlsOptions: config.tlsOptions,
          agent: config.proxyAgent,
        })
      
      default:
        throw new Error(`Unsupported transport type: ${(config as any).type}`)
    }
  }
}
```

## 工作流程

### 第一步：需求分析 (15 分钟)

1. **确定集成场景**
   - 需要哪些 MCP 服务器？
   - 使用什么传输协议？
   - 是否需要 OAuth 认证？
   - 需要暴露哪些工具和资源？

2. **选择传输协议**
   ```
   本地进程 → Stdio
   远程 HTTP → SSE 或 HTTP Streaming
   实时双向 → WebSocket
   ```

### 第二步：服务器开发 (40 分钟)

1. **搭建服务器框架**
2. **实现工具列表**
3. **实现工具处理器**
4. **实现资源管理**
5. **添加错误处理**

### 第三步：客户端集成 (30 分钟)

1. **配置连接管理器**
2. **实现 OAuth 认证**
3. **注册工具和资源**
4. **测试连接和调用**

## 规则

- ✅ 使用标准的 MCP SDK API
- ✅ 实现完整的错误处理
- ✅ 提供详细的工具描述
- ✅ 验证所有输入参数
- ✅ 实现连接重试机制
- ❌ 不要硬编码凭证
- ❌ 不要忽略连接超时
- ❌ 不要暴露敏感信息作为资源
- ❌ 不要忘记清理连接资源

## 输出格式

### 📊 MCP 服务器清单

```markdown
| 服务器名称 | 传输类型 | 工具数量 | 资源数量 | 认证方式 |
|-----------|---------|---------|---------|---------|
| github    | stdio   | 15      | 0       | OAuth   |
| filesystem| stdio   | 5       | ∞       | None    |
| web       | http    | 3       | ∞       | None    |
```

### 🏗️ 架构图

展示 MCP 客户端和服务器的连接关系。

### 📝 实现代码

提供完整的服务器和客户端实现代码。

### 🔍 测试计划

连接测试、工具调用测试、认证流程测试计划。

### ⚠️ 风险评估

| 风险项 | 可能性 | 影响程度 | 缓解措施 |
|-------|--------|----------|----------|
| 认证失败 | 中 | 高 | 自动刷新 + 降级策略 |
| 连接超时 | 高 | 中 | 重试机制 + 超时保护 |
| 工具执行错误 | 中 | 中 | 错误捕获 + 用户提示 |

## 交付物清单

最终报告应包含：
- [ ] MCP 服务器开发指南
- [ ] 传输协议配置方案
- [ ] OAuth 认证实现
- [ ] 工具注册系统
- [ ] 资源管理系统
- [ ] 连接管理器实现
- [ ] 测试计划和测试用例

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发 MCP 集成系统。

## AI IDE 开发前检查

### MCP 环境验证
```yaml
必须检查:
  - @modelcontextprotocol/sdk 已安装
  - 传输协议依赖可用
  - OAuth 库已安装（如需要）
  - 网络连通性正常

警告信号:
  - 缺少必要的依赖包
  - 端口被占用
  - 防火墙阻止连接
```

### 服务器配置
```yaml
必须存在:
  - 服务器名称唯一
  - 传输配置完整
  - 认证配置安全
  - 超时配置合理
```

## AI IDE 开发中指导

### 实时 MCP 开发监控
当开发 MCP 集成时，AI IDE应该：

1. **Schema 验证**
   ```typescript
   // ✅ 正确的 Schema 定义
   const schema = {
     type: 'object',
     properties: {
       name: { type: 'string', description: 'User name' },
       age: { type: 'number', minimum: 0 },
     },
     required: ['name'],
   }
   
   // ❌ 避免的模式
   const schema = {
     type: 'object',
     properties: {}  // 空的属性定义
   }
   // AI IDE 应该：建议添加具体的属性定义
   ```

2. **错误处理检查**
   ```typescript
   // ✅ 完整的错误处理
   try {
     const result = await client.callTool(params)
     return result
   } catch (error) {
     return {
       content: [{ 
         type: 'text', 
         text: `MCP Error: ${error instanceof Error ? error.message : String(error)}` 
       }],
       isError: true,
     }
   }
   
   // ❌ 避免的模式
   const result = await client.callTool(params)  // 无错误处理
   // AI IDE 应该：建议添加 try-catch
   ```

3. **连接生命周期管理**
   ```typescript
   // AI IDE 应该确保正确管理连接
   class MyService {
     async shutdown() {
       await connectionManager.disconnect('server-name')
     }
   }
   ```

### OAuth 安全管理
AI IDE应该提供：
- **凭证加密存储**: 不使用明文
- **Token 自动刷新**: 过期前刷新
- **安全回调处理**: 验证 state 参数
- **错误日志脱敏**: 不泄露敏感信息

## AI IDE 完成后验证

### MCP 集成健康度评分卡
```yaml
服务器开发 (0-10分):
  - 工具定义完整
  - Schema 验证正确
  - 错误处理完善

传输配置 (0-10分):
  - 协议选择合理
  - 连接稳定
  - 超时配置适当

OAuth 认证 (0-10分):
  - 流程完整
  - Token 管理安全
  - 刷新机制可靠

资源管理 (0-10分):
  - 资源注册完整
  - 读取正确
  - MIME 类型准确

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (不建议上线)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 连接测试
test('connects to MCP server', async () => {...})
test('reuses existing connection', async () => {...})

# 2. 工具调用测试
test('calls tool successfully', async () => {...})
test('handles tool errors', async () => {...})

# 3. OAuth 测试
test('authenticates with OAuth', async () => {...})
test('refreshes expired token', async () => {...})

# 4. 资源管理测试
test('lists resources', async () => {...})
test('reads file resource', async () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 凭证硬编码
const CLIENT_SECRET = 'abc123...'  // 直接写在代码里!
// AI IDE 应该：建议使用环境变量

// ❌ Token 明文存储
localStorage.setItem('token', token)  // 不安全!
// AI IDE 应该：建议使用加密存储

// ❌ 无超时保护
await client.callTool(params)  // 可能永远等待
// AI IDE 应该：建议添加 AbortSignal.timeout()
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ Schema 不完整
const schema = {
  type: 'object',
  properties: {
    name: { type: 'string' }  // 缺少 description
  }
}
// AI IDE 应该：建议添加 description
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const tools = await client.listTools()
console.log(tools)  // 基础日志
// AI IDE 可以建议：使用结构化日志
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 服务器开发
- [ ] 工具定义符合规范
- [ ] Input Schema 完整
- [ ] 错误处理完善
- [ ] 资源管理正确

### 传输配置
- [ ] 协议选择合理
- [ ] 连接参数完整
- [ ] 超时配置适当
- [ ] 重试机制实现

### OAuth 认证
- [ ] 流程符合 OAuth 2.0规范
- [ ] State 参数验证
- [ ] Token 加密存储
- [ ] 自动刷新实现

### 安全性
- [ ] 凭证不硬编码
- [ ] 敏感信息不泄露
- [ ] 输入验证完整
- [ ] 错误信息脱敏

## AI IDE 集成实现示例

```typescript
interface MCPIntegrationRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (config: MCPConfig, sourceCode: string) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: MCPIntegrationRule[] = [
  {
    id: 'MCP-001',
    description: 'Credentials must not be hardcoded',
    severity: 'error',
    check: (config, sourceCode) => {
      const hardcoded = findHardcodedCredentials(sourceCode)
      if (hardcoded.length > 0) {
        return [{
          message: 'Hardcoded credentials found',
          location: hardcoded.map(c => c.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'move_to_env' })
  },
  {
    id: 'MCP-002',
    description: 'Tool schemas must have descriptions',
    severity: 'warning',
    check: (config, sourceCode) => {
      const missingDesc = findToolSchemasWithoutDescriptions(sourceCode)
      if (missingDesc.length > 0) {
        return [{
          message: 'Tool schemas missing descriptions',
          location: missingDesc.map(s => s.location)
        }]
      }
      return []
    }
  },
  {
    id: 'MCP-003',
    description: 'Connections must have timeout',
    severity: 'error',
    check: (config) => {
      if (!config.timeout || config.timeout > 30000) {
        return [{
          message: 'Connection timeout missing or too long',
          location: config.connectionSite
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_timeout', value: 30000 })
  },
  {
    id: 'MCP-004',
    description: 'OAuth tokens must be encrypted',
    severity: 'error',
    check: (config, sourceCode) => {
      const unencryptedTokens = findUnencryptedTokens(sourceCode)
      if (unencryptedTokens.length > 0) {
        return [{
          message: 'OAuth tokens stored unencrypted',
          location: unencryptedTokens.map(t => t.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'encrypt_storage' })
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解 MCP 集成规范：

1. **第一阶段**: 学习 MCP 协议基础和传输类型
2. **第二阶段**: 理解工具开发和 Schema 定义
3. **第三阶段**: 掌握 OAuth 2.0认证流程
4. **第四阶段**: 实现资源管理和连接管理
5. **第五阶段**: 提供智能错误诊断和优化建议

通过学习路径，AI IDE可以成长为能够独立设计和审查 MCP 集成系统的专家。

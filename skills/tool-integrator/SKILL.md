---
name: tool-integrator
description: AI Agent 工具集成专家 - MCP 协议、OAuth 认证、工具分类、权限控制、活动描述生成
aliases: ['工具集成', 'mcp-expert']
argumentHint: '<集成类型> [服务器名称]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要集成 MCP 服务器、实现 OAuth 认证流、设计工具分类系统、或构建工具权限控制机制时
---

# Tool Integrator

使用此技能当你需要进行 MCP 服务器集成、OAuth 认证实现、工具分类系统设计、或工具权限控制开发。

## 目标

产出完整的工具集成方案，包括 MCP 客户端架构、OAuth 认证流程、工具分类机制、活动描述生成器和权限控制系统。

## 核心能力

### 1. MCP 客户端架构 (src/services/mcp/client.ts)

Claude Code的 MCP 客户端支持多种传输协议：

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'
import { WebSocketTransport } from '../../utils/mcpWebSocketTransport.js'
```

**传输协议选择**:

**Stdio (本地进程)**:
```typescript
const stdioTransport = new StdioClientTransport({
  command: 'mcp-server-github',
  args: ['--config', configPath],
  env: subprocessEnv
})
```

**SSE (Server-Sent Events)**:
```typescript
const sseTransport = new SSEClientTransport({
  url: new URL('http://localhost:3000/sse'),
  requestInit: { headers: getMcpServerHeaders(serverName) }
})
```

**HTTP Streaming**:
```typescript
const httpTransport = new StreamableHTTPClientTransport({
  url: new URL('http://localhost:3000/mcp'),
  fetch: createFetchWithInit(fetchOptions)
})
```

**WebSocket**:
```typescript
const wsTransport = new WebSocketTransport({
  url: 'ws://localhost:3000/mcp',
  tlsOptions: getWebSocketTLSOptions(),
  agent: getWebSocketProxyAgent()
})
```

### 2. MCP 连接管理器

```typescript
class MCPConnectionManager {
  private connections: Map<string, MCPServerConnection> = new Map()
  private clients: Map<string, Client> = new Map()
  
  async connect(serverName: string, config: McpSdkServerConfig): Promise<Client> {
    // 检查是否已连接
    if (this.clients.has(serverName)) {
      return this.clients.get(serverName)!
    }
    
    // 创建传输层
    const transport = this.createTransport(config)
    
    // 创建客户端
    const client = new Client({
      name: 'claude-code',
      version: pkg.version
    })
    
    // 连接并初始化
    await client.connect(transport)
    
    // 获取服务器能力
    const tools = await client.listTools()
    const resources = await client.listResources()
    const prompts = await client.listPrompts()
    
    // 缓存连接
    this.connections.set(serverName, { config, transport, tools, resources, prompts })
    this.clients.set(serverName, client)
    
    // 发送连接事件
    maybeNotifyIDEConnected(serverName)
    
    return client
  }
  
  async disconnect(serverName: string): Promise<void> {
    const client = this.clients.get(serverName)
    if (client) {
      await client.close()
      this.clients.delete(serverName)
      this.connections.delete(serverName)
    }
  }
  
  async reconnect(serverName: string): Promise<void> {
    await this.disconnect(serverName)
    const config = this.getConnectionConfig(serverName)
    await this.connect(serverName, config)
  }
}
```

### 3. OAuth 认证流程

```typescript
class ClaudeAuthProvider {
  private tokenStore: Map<string, OAuthToken> = new Map()
  
  async checkAndRefreshOAuthTokenIfNeeded(
    serverName: string,
    clientId: string
  ): Promise<boolean> {
    const token = this.tokenStore.get(serverName)
    
    if (!token) {
      return false  // 无 token，需要认证
    }
    
    // 检查是否过期（提前 5 分钟刷新）
    const expiresAt = token.expires_at - 5 * 60 * 1000
    if (Date.now() < expiresAt) {
      return true  // Token 仍然有效
    }
    
    // 需要刷新
    try {
      const refreshed = await this.refreshAccessToken(clientId, token.refresh_token)
      this.tokenStore.set(serverName, refreshed)
      persistOAuthToken(serverName, refreshed)
      return true
    } catch (error) {
      this.tokenStore.delete(serverName)
      return false  // 刷新失败，需要重新认证
    }
  }
  
  async handleOAuth401Error(
    serverName: string,
    error: Error
  ): Promise<'refresh' | 'reauth' | 'fail'> {
    if (!error.message.includes('401')) {
      return 'fail'  // 非认证错误
    }
    
    const token = this.tokenStore.get(serverName)
    if (!token) {
      return 'reauth'  // 无 token，需要重新认证
    }
    
    const refreshed = await this.tryRefreshToken(serverName)
    if (refreshed) {
      return 'refresh'
    }
    
    return 'reauth'
  }
}
```

### 4. 工具分类系统 (src/tools/MCPTool/classifyForCollapse.ts)

根据工具行为自动分类以优化 UI 显示：

```typescript
export type MCPToolClassification = 
  | 'read'      // 只读操作（如 file_read）
  | 'search'    // 搜索操作（如 grep, find）
  | 'list'      // 列表操作（如 ls, tree）
  | 'write'     // 写入操作（如 file_write）
  | 'execute'   // 执行操作（如 bash, run）
  | 'unknown'   // 未知类型

export function classifyMcpToolForCollapse(
  toolName: string,
  description: string,
  inputSchema: unknown
): MCPToolClassification {
  const nameLower = toolName.toLowerCase()
  const descLower = description.toLowerCase()
  
  // Read operations
  if (nameLower.includes('read') || nameLower.includes('get') || 
      nameLower.includes('fetch') || nameLower.includes('download')) {
    return 'read'
  }
  
  // Search operations
  if (nameLower.includes('search') || nameLower.includes('find') ||
      nameLower.includes('grep') || nameLower.includes('query')) {
    return 'search'
  }
  
  // List operations
  if (nameLower.includes('list') || nameLower.includes('ls') ||
      nameLower.includes('tree') || nameLower.includes('enumerate')) {
    return 'list'
  }
  
  // Write operations
  if (nameLower.includes('write') || nameLower.includes('create') ||
      nameLower.includes('update') || nameLower.includes('delete') ||
      nameLower.includes('edit')) {
    return 'write'
  }
  
  // Execute operations
  if (nameLower.includes('run') || nameLower.includes('exec') ||
      nameLower.includes('bash') || nameLower.includes('shell')) {
    return 'execute'
  }
  
  return 'unknown'
}
```

### 5. 活动描述生成器

```typescript
export function createActivityDescriptionResolver(
  tools: unknown
): (toolName: string, input: Record<string, unknown>) => string | undefined {
  return (toolName, input) => {
    switch (toolName) {
      case 'file_read':
        return `Reading ${input.path as string}`
      
      case 'file_write':
        return `Writing to ${input.path as string}`
      
      case 'grep':
        return `Searching for "${input.pattern as string}" in ${input.path as string}`
      
      case 'glob':
        return `Finding files matching ${input.pattern as string}`
      
      case 'bash':
        const cmd = input.command as string
        if (cmd.startsWith('git')) {
          return `Running git: ${cmd.slice(4, 50)}...`
        }
        if (cmd.startsWith('npm') || cmd.startsWith('yarn') || cmd.startsWith('pnpm')) {
          return `Running package manager: ${cmd.slice(0, 50)}...`
        }
        return `Executing: ${cmd.slice(0, 50)}...`
      
      default:
        return undefined
    }
  }
}
```

### 6. 工具权限控制系统

```typescript
export type PermissionResult =
  | { status: 'approved' }
  | { status: 'denied'; reason: string }
  | { status: 'ask_user'; prompt: string }

export async function canUseTool(
  toolName: string,
  input: Record<string, unknown>,
  permissionContext: {
    mode: PermissionMode
    rules: PermissionRule[]
  }
): Promise<PermissionResult> {
  const { mode, rules } = permissionContext
  
  // Bypass permissions 模式
  if (mode === 'bypassPermissions') {
    return { status: 'approved' }
  }
  
  // Don't Ask 模式
  if (mode === 'dontAsk') {
    return { status: 'approved' }
  }
  
  // Plan Mode - 只允许读取和搜索
  if (mode === 'plan') {
    const classification = classifyMcpToolForCollapse(toolName, '', {})
    if (classification === 'read' || classification === 'search' || classification === 'list') {
      return { status: 'approved' }
    }
    return { 
      status: 'denied', 
      reason: 'Plan mode only allows read/search/list operations' 
    }
  }
  
  // Default/Auto mode - 检查规则
  for (const rule of rules) {
    if (ruleMatches(rule, toolName, input)) {
      if (rule.effect === 'allow') {
        return { status: 'approved' }
      } else {
        return { status: 'denied', reason: rule.reason || 'Blocked by permission rule' }
      }
    }
  }
  
  // 无匹配规则，询问用户
  return {
    status: 'ask_user',
    prompt: `Allow ${toolName} with arguments: ${JSON.stringify(input)}?`
  }
}
```

### 7. MCP 工具命名规范

```typescript
/**
 * Build MCP tool name from server name and tool name
 * Format: mcp__{server}__{tool}
 */
export function buildMcpToolName(serverName: string, toolName: string): string {
  const normalizedServer = serverName
    .toLowerCase()
    .replace(/[^a-z0-9-]/g, '-')
    .replace(/-+/g, '-')
  
  const normalizedTool = toolName
    .toLowerCase()
    .replace(/[^a-z0-9-_]/g, '-')
    .replace(/-+/g, '-')
  
  return `mcp__${normalizedServer}__${normalizedTool}`
}

/**
 * Parse MCP tool name back to components
 */
export function parseMcpToolName(fullName: string): {
  serverName: string
  toolName: string
} | null {
  const match = fullName.match(/^mcp__([^_]+)__([^_]+)$/)
  if (!match) return null
  
  return {
    serverName: match[1],
    toolName: match[2]
  }
}
```

## 工作流程

### 第一步：现有工具系统审计 (15 分钟)

1. **检查 MCP 服务器配置**
   ```bash
   # 查找 MCP 配置文件
   find ~/.claude -name "*.json" | xargs grep -l "mcpServers"
   
   # 查看已安装的 MCP 服务器
   cat ~/.claude/settings.json | jq '.mcpServers'
   
   # 测试 MCP 服务器连接
   npx -y @modelcontextprotocol/inspector <server-command>
   ```

2. **评估要点**
   - 有哪些 MCP 服务器已配置？
   - 每个服务器的传输协议是什么？
   - OAuth 认证是否配置？
   - 工具权限规则是否合理？

### 第二步：工具集成架构设计 (20 分钟)

基于 Claude Code最佳实践设计你的工具集成系统，包括 MCP 连接管理器、OAuth 认证提供者、工具分类器、活动描述生成器和权限控制器。

### 第三步：实现与验证 (25 分钟)

#### 实现检查清单

- [ ] **MCP 连接管理器**
  - [ ] 多传输协议支持（Stdio/SSE/HTTP/WebSocket）
  - [ ] 连接缓存和复用
  - [ ] 服务器能力发现（tools/resources/prompts）
  - [ ] 连接事件通知 IDE

- [ ] **OAuth 认证提供者**
  - [ ] Token 存储和刷新
  - [ ] 过期检测和自动刷新
  - [ ] 401 错误处理
  - [ ] Token 持久化

- [ ] **工具分类器**
  - [ ] 基于名称的分类
  - [ ] 基于描述的分类
  - [ ] 基于输入 schema 的分类
  - [ ] 自定义分类规则

- [ ] **活动描述生成器**
  - [ ] 常见工具的友好描述
  - [ ] 命令参数提取和截断
  - [ ] 路径和文件名高亮

- [ ] **权限控制器**
  - [ ] 多模式支持（default/plan/bypass/dontAsk/auto）
  - [ ] 规则匹配引擎
  - [ ] 用户询问界面
  - [ ] 权限决策日志

## 规则

- ✅ 为每个 MCP 服务器使用唯一的连接实例
- ✅ 实现 OAuth token 的自动刷新机制
- ✅ 对工具进行分类以便 UI 优化显示
- ✅ 生成人类可读的活动描述
- ✅ 实施细粒度的权限控制
- ❌ 不要在每次调用时都创建新的 MCP 连接（使用缓存）
- ❌ 不要硬编码 OAuth credentials（使用环境变量或配置文件）
- ❌ 不要忽略工具调用的错误处理
- ❌ 不要在权限检查中绕过用户确认（除非明确配置）

## 输出格式

### 📊 现状评估

```markdown
## MCP 集成成熟度
评分：X/10

### 已集成的服务器
- ...

### 缺失的集成
- ...

### 认证问题
- ...
```

### 🏗️ 架构设计

使用 Mermaid 图展示 MCP 客户端架构和 OAuth 流程。

### 📝 实现蓝图

提供五个核心模块的完整 TypeScript 实现代码。

### 🔍 测试计划

详细的单元测试、集成测试、端到端测试计划。

### ⚠️ 风险评估

| 风险项 | 可能性 | 影响程度 | 缓解措施 |
|-------|--------|----------|----------|
| OAuth token 泄露 | 低 | 极高 | 加密存储 + 最小权限 |
| MCP 连接不稳定 | 中 | 中 | 自动重连 + 超时保护 |
| 工具分类错误 | 高 | 低 | 人工审核 + fallback |
| 权限绕过漏洞 | 低 | 高 | 双重检查 + 审计日志 |

## 交付物清单

最终报告应包含：
- [ ] MCP 集成现状评估
- [ ] 架构设计图（Mermaid 流程图）
- [ ] 五个核心模块的完整实现代码
- [ ] 测试计划和测试用例
- [ ] 风险评估和缓解措施
- [ ] 性能基准和优化建议
- [ ] OAuth 配置指南
- [ ] MCP 服务器接入文档

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发工具集成系统。

## AI IDE 开发前检查

### MCP 服务器配置验证
```yaml
必须检查:
  - mcpServers配置是否存在于 settings.json
  - 每个服务器的传输协议类型 (stdio/sse/http/ws)
  - 认证配置是否完整 (OAuth/client credentials)

警告信号:
  - 硬编码的服务器地址
  - 明文存储的凭证
  - 缺少超时配置
```

### OAuth 认证准备
```yaml
必须存在:
  - OAuth client ID和 secret
  - Token 存储机制 (encrypted)
  - 自动刷新逻辑
  - 401 错误处理策略
```

## AI IDE 开发中指导

### 实时 MCP 连接监控
当连接 MCP 服务器时，AI IDE应该：

1. **传输协议选择**
   ```typescript
   // ✅ 根据场景选择合适的传输
   const transports = {
     local: 'stdio',      // 本地进程
     remote: 'sse',       // 远程 HTTP
     streaming: 'http',   // HTTP Streaming
     realtime: 'ws'       // WebSocket
   }
   
   // ❌ 避免的模式
   // 对本地服务器使用 SSE（不必要的网络开销）
   // AI IDE 应该：推荐最优传输协议
   ```

2. **连接复用验证**
   ```typescript
   // AI IDE 应该检查连接缓存
   if (connectionManager.has(serverName)) {
     return connectionManager.get(serverName)  // 复用
   }
   
   // 否则创建新连接并缓存
   const conn = await createConnection(config)
   connectionManager.set(serverName, conn)
   ```

3. **工具分类优化**
   ```typescript
   // AI IDE 应该自动分类工具
   const classification = classifyMcpTool(toolName, description)
   // read/search/list/write/execute/unknown
   
   // UI 可以根据分类折叠/展开工具调用
   ```

### OAuth 令牌管理
AI IDE应该提供：
- **Token 加密存储**: 不使用明文
- **自动刷新**: 过期前 5 分钟自动刷新
- **401 检测**: 捕获认证错误并触发重新认证
- **多服务器支持**: 每个服务器独立的 token 存储

## AI IDE 完成后验证

### 工具集成健康度评分卡
```yaml
MCP 连接 (0-10分):
  - 传输协议选择合理性
  - 连接复用率
  - 错误恢复能力
  
OAuth 认证 (0-10分):
  - Token 存储安全性
  - 自动刷新成功率
  - 401 处理及时性

工具分类 (0-10分):
  - 分类准确率
  - UI 折叠正确性
  - 活动描述可读性

权限控制 (0-10分):
  - 模式切换正确性
  - 规则匹配准确性
  - 用户询问友好性

命名规范 (0-10分):
  - 工具名格式统一
  - 前缀唯一性
  - 解析可靠性

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. MCP 连接测试
test('connects via stdio', () => {...})
test('reuses existing connection', () => {...})

# 2. OAuth 测试
test('refreshes expired token', () => {...})
test('handles 401 errors', () => {...})

# 3. 工具分类测试
test('classifies read operations', () => {...})
test('classifies write operations', () => {...})

# 4. 权限控制测试
test('bypass mode allows all', () => {...})
test('plan mode blocks writes', () => {...})

# 5. 命名规范测试
test('builds valid tool names', () => {...})
test('parses tool names correctly', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ Token 明文存储
const token = "ghp_xxx..."  // 硬编码在代码中!
// AI IDE 应该：立即警告 + 建议使用加密存储

// ❌ 连接泄漏
async function callMCP() {
  const conn = await connect(config)  // 从未 disconnect
  return await conn.callTool(...)
}
// AI IDE 应该：建议使用单例模式 + 生命周期管理

// ❌ 权限绕过
if (user.isAdmin) {
  return { status: 'approved' }  // 跳过正常权限检查!
}
// AI IDE 应该：警告权限旁路风险
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 无超时保护
const result = await mcpClient.callTool(params)  // 可能永远等待
// AI IDE 应该：建议添加 AbortSignal.timeout(30000)

// ⚠️ 分类不准确
const tool = { name: 'run_query', description: '' }  // 无法分类
// AI IDE 应该：建议提供详细描述
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const tools = await client.listTools()
tools.forEach(t => console.log(t.name))
// AI IDE 可以建议：缓存工具列表减少 API 调用
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### MCP 连接
- [ ] 传输协议选择合理
- [ ] 连接复用避免重复创建
- [ ] 错误处理和重试机制
- [ ] 资源释放（disconnect）

### OAuth 认证
- [ ] Token 加密存储
- [ ] 自动刷新逻辑
- [ ] 401 错误检测和处理
- [ ] 凭证不提交到版本控制

### 工具分类
- [ ] 分类函数覆盖所有类型
- [ ] 活动描述有人类可读
- [ ] UI 折叠逻辑正确
- [ ] 工具和分类映射准确

### 权限控制
- [ ] 所有模式正确处理
- [ ] 规则匹配引擎完整
- [ ] 用户询问界面友好
- [ ] 权限决策有日志

### 命名规范
- [ ] 工具名格式 mcp__{server}__{tool}
- [ ] 特殊字符正确替换
- [ ] 解析函数健壮性
- [ ] 命名冲突检测

## AI IDE 集成实现示例

```typescript
interface IntegrationRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (config: MCPConfig, connection: MCPConnection) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: IntegrationRule[] = [
  {
    id: 'INTEG-001',
    description: 'OAuth tokens must be encrypted',
    severity: 'error',
    check: (config) => {
      if (config.oauth?.token && !config.oauth.encrypted) {
        return [{
          message: 'OAuth token stored in plaintext',
          location: config.oauth.tokenPath
        }]
      }
      return []
    },
    fix: () => ({ type: 'encrypt', algorithm: 'aes-256-gcm' })
  },
  {
    id: 'INTEG-002',
    description: 'Connections must be reused',
    severity: 'warning',
    check: (config, connection) => {
      if (connection.createCount > 1 && !connection.isCached) {
        return [{
          message: 'Creating duplicate connections',
          location: connection.createCallSite
        }]
      }
      return []
    }
  },
  {
    id: 'INTEG-003',
    description: 'Tool calls must have timeout',
    severity: 'warning',
    check: (config, connection) => {
      if (!connection.hasTimeout) {
        return [{
          message: 'Tool call may hang indefinitely',
          location: connection.callSite
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_timeout', value: 30000 })
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解工具集成规范：

1. **第一阶段**: 学习 MCP 协议基础和传输类型
2. **第二阶段**: 理解 OAuth 2.0认证流程和 token 管理
3. **第三阶段**: 掌握工具分类算法和活动描述生成
4. **第四阶段**: 实现权限控制和规则匹配引擎
5. **第五阶段**: 提供智能连接优化和错误恢复建议

通过学习路径，AI IDE可以成长为能够独立设计和审查工具集成系统的专家。

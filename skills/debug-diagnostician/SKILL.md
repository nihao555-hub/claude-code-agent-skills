---
name: debug-diagnostician
description: AI Agent 调试诊断专家 - 会话调试、日志分析、错误追踪、堆栈分析、诊断跟踪服务
aliases: ['调试', 'debug-expert']
argumentHint: '<问题描述> [日志路径]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要调试复杂的 Agent 会话、分析日志模式、追踪错误根源、或解析崩溃堆栈时
---

# Debug Diagnostician

使用此技能当你需要进行系统调试、日志分析、错误根因诊断、或会话状态追踪。

## 目标

产出完整的调试诊断报告，包括问题复现步骤、日志分析结果、错误根因定位、堆栈轨迹解析和修复建议。

## 核心能力

### 1. Debug Skill 实现 (src/skills/bundled/debug.ts)

Claude Code的 debug skill支持多种调试模式：

```typescript
import { open, stat } from 'fs/promises'
import { enableDebugLogging, getDebugLogPath } from '../../utils/debug.js'

// Debug skill 配置
const DEBUG_SKILL_CONFIG = {
  name: 'debug',
  description: 'Enable debug logging and diagnose issues',
  argumentHint: '[category] [level]',
  
  async getPromptForCommand(args: string) {
    const parts = args.trim().split(/\s+/)
    const category = parts[0] || 'all'
    const level = parts[1] || 'info'
    
    // 启用调试日志
    if (category === 'all' || category === 'mcp') {
      enableDebugLogging('mcp')
    }
    
    if (category === 'all' || category === 'bridge') {
      enableDebugLogging('bridge')
    }
    
    if (category === 'all' || category === 'session') {
      enableDebugLogging('session')
    }
    
    // 获取日志路径
    const logPath = await getDebugLogPath()
    
    return [{
      type: 'text',
      text: `
Debug logging enabled for: ${category}
Log level: ${level}
Log file: ${logPath}

To view logs in real-time:
\`\`\`bash
tail -f ${logPath}
\`\`\`

Common debug categories:
- mcp: MCP server communication
- bridge: Bridge protocol messages
- session: Session lifecycle events
- tool: Tool execution details
- permission: Permission decisions
- token: Token usage and estimation
`
    }]
  }
}
```

### 2. 调试日志系统

```typescript
type DebugCategory = 'mcp' | 'bridge' | 'session' | 'tool' | 'permission' | 'token'

const debugEnabled = new Set<DebugCategory>()

/**
 * Enable debug logging for a category
 */
export function enableDebugLogging(category: DebugCategory): void {
  debugEnabled.add(category)
  console.log(`[Debug] Enabled logging for: ${category}`)
}

/**
 * Disable debug logging for a category
 */
export function disableDebugLogging(category: DebugCategory): void {
  debugEnabled.delete(category)
}

/**
 * Log debug message (only if category is enabled)
 */
export function logDebug(category: DebugCategory, message: string, metadata?: unknown): void {
  if (!debugEnabled.has(category)) return
  
  const timestamp = new Date().toISOString()
  const logLine = `[${timestamp}] [${category.toUpperCase()}] ${message}`
  
  if (metadata) {
    console.log(logLine, JSON.stringify(metadata, null, 2))
  } else {
    console.log(logLine)
  }
  
  // Also write to debug log file
  appendToDebugLogFile(logLine)
}

/**
 * Get debug log file path
 */
export async function getDebugLogPath(): Promise<string> {
  const configDir = getClaudeConfigHomeDir()
  const logsDir = join(configDir, 'logs')
  
  await mkdir(logsDir, { recursive: true })
  
  const date = new Date().toISOString().split('T')[0]
  return join(logsDir, `debug-${date}.log`)
}
```

### 3. 诊断跟踪服务 (src/services/diagnosticTracking.ts)

```typescript
class DiagnosticTrackingService {
  private errors: Array<{
    timestamp: number
    type: string
    message: string
    stack?: string
    context?: Record<string, unknown>
  }> = []
  
  private warnings: Array<{
    timestamp: number
    type: string
    message: string
    context?: Record<string, unknown>
  }> = []
  
  /**
   * Track an error
   */
  trackError(
    type: string,
    message: string,
    error: Error,
    context?: Record<string, unknown>
  ): void {
    this.errors.push({
      timestamp: Date.now(),
      type,
      message,
      stack: error.stack,
      context
    })
    
    // Keep only last 100 errors
    if (this.errors.length > 100) {
      this.errors.shift()
    }
  }
  
  /**
   * Track a warning
   */
  trackWarning(
    type: string,
    message: string,
    context?: Record<string, unknown>
  ): void {
    this.warnings.push({
      timestamp: Date.now(),
      type,
      message,
      context
    })
    
    // Keep only last 100 warnings
    if (this.warnings.length > 100) {
      this.warnings.shift()
    }
  }
  
  /**
   * Get recent diagnostics
   */
  getRecentDiagnostics(limit: number = 50): {
    errors: typeof this.errors
    warnings: typeof this.warnings
  } {
    return {
      errors: this.errors.slice(-limit),
      warnings: this.warnings.slice(-limit)
    }
  }
  
  /**
   * Export diagnostics to file
   */
  async exportToFile(filePath: string): Promise<void> {
    const report = {
      exportedAt: new Date().toISOString(),
      totalErrors: this.errors.length,
      totalWarnings: this.warnings.length,
      errors: this.errors,
      warnings: this.warnings
    }
    
    await writeFile(filePath, JSON.stringify(report, null, 2), 'utf-8')
  }
  
  /**
   * Clear all diagnostics
   */
  clear(): void {
    this.errors = []
    this.warnings = []
  }
}

export const diagnosticTracking = new DiagnosticTrackingService()
```

### 4. 错误分类系统

```typescript
type ErrorSeverity = 'critical' | 'high' | 'medium' | 'low'

interface ClassifiedError {
  originalError: Error
  category: ErrorCategory
  severity: ErrorSeverity
  suggestedAction: string
  recoverable: boolean
}

type ErrorCategory = 
  | 'network'           // 网络错误
  | 'filesystem'        // 文件系统错误
  | 'authentication'    // 认证错误
  | 'permission'        // 权限错误
  | 'validation'        // 验证错误
  | 'timeout'           // 超时错误
  | 'resource'          // 资源限制错误
  | 'internal'          // 内部错误

/**
 * Classify an error
 */
function classifyError(error: Error): ClassifiedError {
  const message = error.message.toLowerCase()
  const stack = error.stack || ''
  
  // Network errors
  if (message.includes('fetch') || message.includes('network') || 
      message.includes('connection') || message.includes('ECONNREFUSED')) {
    return {
      originalError: error,
      category: 'network',
      severity: 'high',
      suggestedAction: 'Check network connectivity and API endpoint availability',
      recoverable: true
    }
  }
  
  // Filesystem errors
  if (message.includes('ENOENT') || message.includes('EACCES') || 
      message.includes('EPERM') || message.includes('path')) {
    return {
      originalError: error,
      category: 'filesystem',
      severity: message.includes('ENOENT') ? 'medium' : 'high',
      suggestedAction: 'Verify file paths and permissions',
      recoverable: message.includes('ENOENT')
    }
  }
  
  // Authentication errors
  if (message.includes('401') || message.includes('unauthorized') || 
      message.includes('authentication') || message.includes('token')) {
    return {
      originalError: error,
      category: 'authentication',
      severity: 'critical',
      suggestedAction: 'Re-authenticate or refresh credentials',
      recoverable: true
    }
  }
  
  // Timeout errors
  if (message.includes('timeout') || message.includes('timed out') ||
      message.includes('ETIMEDOUT')) {
    return {
      originalError: error,
      category: 'timeout',
      severity: 'medium',
      suggestedAction: 'Increase timeout or optimize operation',
      recoverable: true
    }
  }
  
  // Default: internal error
  return {
    originalError: error,
    category: 'internal',
    severity: 'high',
    suggestedAction: 'Review logs and contact support if persists',
    recoverable: false
  }
}
```

### 5. 堆栈分析器

```typescript
interface StackFrame {
  functionName: string
  fileName: string
  lineNumber: number
  columnNumber: number
  isNative: boolean
}

/**
 * Parse error stack trace
 */
function parseStackTrace(stack: string): StackFrame[] {
  const frames: StackFrame[] = []
  const lines = stack.split('\n')
  
  for (const line of lines) {
    // Node.js format: at FunctionName (file.js:10:15)
    const nodeMatch = line.match(/at (?:(.+) \()?([^:]+):(\d+):(\d+)\)?/)
    if (nodeMatch) {
      frames.push({
        functionName: nodeMatch[1] || '<anonymous>',
        fileName: nodeMatch[2],
        lineNumber: parseInt(nodeMatch[3]),
        columnNumber: parseInt(nodeMatch[4]),
        isNative: nodeMatch[2] === 'native'
      })
      continue
    }
    
    // Simple format: at file.js:10:15
    const simpleMatch = line.match(/at ([^:]+):(\d+):(\d+)/)
    if (simpleMatch) {
      frames.push({
        functionName: '<unknown>',
        fileName: simpleMatch[1],
        lineNumber: parseInt(simpleMatch[2]),
        columnNumber: parseInt(simpleMatch[3]),
        isNative: false
      })
    }
  }
  
  return frames
}

/**
 * Analyze stack trace for patterns
 */
function analyzeStackTrace(stack: string): {
  depth: number
  topFunctions: string[]
  suspiciousPatterns: string[]
} {
  const frames = parseStackTrace(stack)
  
  // Detect deep recursion
  const functionCounts = new Map<string, number>()
  for (const frame of frames) {
    const count = functionCounts.get(frame.functionName) || 0
    functionCounts.set(frame.functionName, count + 1)
  }
  
  const suspiciousPatterns: string[] = []
  for (const [func, count] of functionCounts.entries()) {
    if (count > 10) {
      suspiciousPatterns.push(`Possible infinite recursion: ${func} appears ${count} times`)
    }
  }
  
  return {
    depth: frames.length,
    topFunctions: frames.slice(0, 5).map(f => f.functionName),
    suspiciousPatterns
  }
}
```

### 6. 会话调试工具

```typescript
interface SessionDebugInfo {
  sessionId: string
  startTime: number
  duration: number
  turnCount: number
  toolUseCount: number
  tokenUsage: {
    input: number
    output: number
    cache: number
  }
  errors: number
  warnings: number
}

/**
 * Capture current session debug info
 */
async function captureSessionDebugInfo(): Promise<SessionDebugInfo> {
  const appState = getAppState()
  
  return {
    sessionId: getSessionId(),
    startTime: appState.sessionStartTime,
    duration: Date.now() - appState.sessionStartTime,
    turnCount: appState.turnCount,
    toolUseCount: appState.toolUseCount,
    tokenUsage: {
      input: appState.cumulativeInputTokens,
      output: appState.cumulativeOutputTokens,
      cache: appState.cacheTokenUsage
    },
    errors: diagnosticTracking.getRecentDiagnostics().errors.length,
    warnings: diagnosticTracking.getRecentDiagnostics().warnings.length
  }
}

/**
 * Export session debug info
 */
async function exportSessionDebug(): Promise<string> {
  const info = await captureSessionDebugInfo()
  const diagnostics = diagnosticTracking.getRecentDiagnostics(50)
  
  const report = {
    session: info,
    recentErrors: diagnostics.errors,
    recentWarnings: diagnostics.warnings,
    capturedAt: new Date().toISOString()
  }
  
  const filePath = join(getTempDir(), `session-debug-${Date.now()}.json`)
  await writeFile(filePath, JSON.stringify(report, null, 2), 'utf-8')
  
  return filePath
}
```

### 7. 调试方法论

```markdown
## Claude Code 调试五步法

### 第一步：重现问题 (Reproduce)
1. 记录确切的复现步骤
2. 识别触发条件
3. 确定问题频率（必现/偶现）

### 第二步：收集证据 (Collect)
1. 启用相关类别的 debug logging
2. 捕获错误堆栈
3. 导出会话诊断信息
4. 保存相关日志文件

### 第三步：定位根源 (Isolate)
1. 分析错误分类（network/filesystem/auth等）
2. 解析堆栈轨迹找模式
3. 检查时间线和相关事件
4. 使用排除法缩小范围

### 第四步：验证假设 (Verify)
1. 提出可能的根本原因
2. 设计验证实验
3. 逐个测试假设
4. 确认真正的根本原因

### 第五步：修复与预防 (Fix & Prevent)
1. 实施修复方案
2. 验证修复效果
3. 添加监控告警
4. 更新文档和 runbook
```

## 工作流程

### 第一步：问题分类 (5 分钟)

根据问题描述，快速分类到以下类别之一：
- **启动问题**: CLI 无法启动、导入错误
- **会话问题**: 对话中断、上下文丢失
- **工具问题**: 工具调用失败、权限被拒
- **性能问题**: 响应缓慢、token 超限
- **集成问题**: MCP 连接失败、OAuth 认证错误

### 第二步：数据收集 (10 分钟)

```bash
# 1. 启用调试日志
/debug all verbose

# 2. 查看最近的错误
grep "ERROR" ~/.claude/logs/debug-$(date +%Y-%m-%d).log | tail -20

# 3. 导出现话诊断
/export-session-debug

# 4. 检查系统资源
ps aux | grep claude
lsof -p <pid> | wc -l
```

### 第三步：根因分析 (15 分钟)

使用鱼骨图分析法：
```
人员 → 是否操作正确？
流程 → 是否步骤遗漏？
工具 → 是否配置正确？
环境 → 是否依赖满足？
材料 → 是否输入有效？
```

## 规则

- ✅ 始终先尝试重现问题再开始分析
- ✅ 启用最小化的调试日志级别（避免日志洪水）
- ✅ 保存所有中间证据（日志、堆栈、截图）
- ✅ 使用错误分类系统快速定位问题领域
- ✅ 为每个发现的根因提供可操作的修复建议
- ❌ 不要在没有充分证据的情况下猜测根因
- ❌ 不要忽略时序问题和并发竞态条件
- ❌ 不要忘记检查环境变量和配置文件
- ❌ 不要在日志中泄露 PII 或敏感信息

## 输出格式

### 🐛 问题摘要

```markdown
**问题描述**: ...
**影响范围**: ...
**发生频率**: ...
**严重程度**: Critical/High/Medium/Low
```

### 📊 收集的证据

```markdown
**日志文件**: /path/to/logs
**错误堆栈**: (粘贴关键片段)
**会话信息**: (turn count, token usage 等)
**系统状态**: (CPU, memory, disk 等)
```

### 🔍 根因分析

```markdown
**错误分类**: network/filesystem/auth/...
**堆栈分析**: (深度、顶层函数、可疑模式)
**时间线**: (事件发生的先后顺序)
**排除的因素**: (已确认无关的因素)
```

### ✅ 修复方案

```markdown
**立即修复**: ...
**长期预防**: ...
**验证步骤**: ...
```

### 🛡️ 预防措施

```markdown
**监控告警**: ...
**自动化测试**: ...
**文档更新**: ...
```

### 📎 附录

日志片段、堆栈轨迹、配置文件等相关证据。

## 交付物清单

最终报告应包含：
- [ ] 问题摘要和影响评估
- [ ] 收集的完整证据包
- [ ] 根因分析和排除过程
- [ ] 详细的修复方案和步骤
- [ ] 预防措施和监控建议
- [ ] 经验教训和文档更新建议

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发调试诊断系统。

## AI IDE 开发前检查

### 调试日志系统验证
```yaml
必须检查:
  - DebugCategory枚举定义
  - enableDebugLogging/disableDebugLogging函数
  - logDebug条件输出（仅当 category enabled）
  - getDebugLogPath日志文件路径

警告信号:
  - 日志中包含 PII
  - 同步写入阻塞主线程
  - 无日志轮转机制（文件过大）
```

### 诊断跟踪服务准备
```yaml
必须存在:
  - DiagnosticTrackingService类
  - trackError/trackWarning方法
  - getRecentDiagnostics查询接口
  - exportToFile导出功能
  - 自动清理旧数据（max 100条）
```

## AI IDE 开发中指导

### 实时调试监控
当启用调试时，AI IDE应该：

1. **分类日志输出**
   ```typescript
   // ✅ 正确的分类
   logDebug('mcp', 'Connecting to server...', { url })
   logDebug('session', 'Turn started', { turnId })
   
   // ❌ 避免的模式
   console.log('Debug: something happened')  // 无分类、无条件
   // AI IDE 应该：建议使用 logDebug(category, message, metadata)
   ```

2. **PII 过滤验证**
   ```typescript
   // AI IDE 应该自动检测并过滤 PII
   const safeMetadata = sanitizeObjectForLogging(rawMetadata)
   
   // 检测模式
   const piiPatterns = {
     email: /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
     homePath: /\/Users\/[^/]+\//g,
     apiKey: /sk-[a-zA-Z0-9]{32,}/g
   }
   
   // 发现 PII 时警告
   if (containsPII(rawMetadata)) {
     logger.warn('PII detected in debug log')
   }
   ```

3. **堆栈分析优化**
   ```typescript
   // AI IDE 应该提供智能堆栈解析
   const frames = parseStackTrace(error.stack)
   const analysis = analyzeStackTrace(error.stack)
   
   // 检测可疑模式
   if (analysis.suspiciousPatterns.length > 0) {
     // 例如：无限递归
     reportPotentialBug(analysis.suspiciousPatterns)
   }
   ```

### 会话调试工具
AI IDE应该提供：
- **captureSessionDebugInfo**: 捕获当前会话状态
- **exportSessionDebug**: 导出调试信息到文件
- **实时监控**: 显示 token 使用、错误计数等
- **趋势分析**: 内存/CPU使用趋势图

## AI IDE 完成后验证

### 调试诊断健康度评分卡
```yaml
日志系统 (0-10分):
  - 分类准确性
  - PII 过滤完整性
  - 性能影响最小化

诊断跟踪 (0-10分):
  - 错误捕获率
  - 警告记录及时性
  - 历史数据管理

堆栈分析 (0-10分):
  - 解析准确率
  - 模式识别智能性
  - 根因定位精确度

会话调试 (0-10分):
  - 状态捕获完整性
  - 导出格式可用性
  - 实时监控准确性

方法论遵循 (0-10分):
  - 五步法执行
  - 证据收集充分性
  - 根因验证严谨性

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 日志系统测试
test('filters PII from logs', () => {...})
test('only logs when category enabled', () => {...})

# 2. 诊断跟踪测试
test('tracks errors correctly', () => {...})
test('limits history to 100', () => {...})

# 3. 堆栈分析测试
test('parses Node.js stack format', () => {...})
test('detects infinite recursion', () => {...})

# 4. 会话调试测试
test('captures session state', () => {...})
test('exports to file successfully', () => {...})

# 5. 调试方法论测试
test('follows five-step process', () => {...})
test('collects sufficient evidence', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ PII 泄露到日志
logDebug('session', `User ${userEmail} logged in`)
// AI IDE 应该：立即警告 + 建议脱敏

// ❌ 同步阻塞日志
const logContent = readFileSync(logPath)  // 阻塞主线程!
// AI IDE 应该：建议使用 await readFile()

// ❌ 无限制的日志增长
logs.push(logEntry)  // 从未清理，内存泄漏!
// AI IDE 应该：建议添加 shift()限制大小
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 日志类别不明确
logDebug('misc', 'Something happened')  // 'misc'不是有效类别
// AI IDE 应该：建议使用明确的类别 (mcp/session/tool/permission/token)

// ⚠️ 缺少上下文信息
logDebug('error', 'Failed to connect')  // 没有 URL、错误详情
// AI IDE 应该：建议添加 metadata 参数
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
console.log(`[${new Date().toISOString()}] Message`)
// AI IDE 可以建议：使用统一的 logDebug 工具函数
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 日志系统
- [ ] 使用 logDebug 而非 console.log
- [ ] 正确的分类 (mcp/session/tool/permission/token)
- [ ] PII 已过滤
- [ ] 异步写入不阻塞

### 诊断跟踪
- [ ] trackError 记录完整堆栈
- [ ] trackWarning 记录上下文
- [ ] 限制历史记录数量
- [ ] 支持导出到文件

### 堆栈分析
- [ ] 正确解析 Node.js 格式
- [ ] 检测可疑模式（递归、循环）
- [ ] 提供人类可读的分析报告
- [ ] 标识顶层函数

### 会话调试
- [ ] 捕获完整的会话状态
- [ ] 导出 JSON 格式有效
- [ ] 临时文件定期清理
- [ ] 不包含敏感信息

### 方法论
- [ ] 遵循五步调试流程
- [ ] 证据收集充分
- [ ] 根因验证严谨
- [ ] 修复建议可操作

## AI IDE 集成实现示例

```typescript
interface DebugRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (logData: LogData, diagnostics: DiagnosticData) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: DebugRule[] = [
  {
    id: 'DEBUG-001',
    description: 'Logs must not contain PII',
    severity: 'error',
    check: (logData) => {
      const piiFound = detectPII(logData.metadata)
      if (piiFound.length > 0) {
        return [{
          message: `PII detected in log: ${piiFound.join(', ')}`,
          location: logData.logLine
        }]
      }
      return []
    },
    fix: () => ({ type: 'redact_pii' })
  },
  {
    id: 'DEBUG-002',
    description: 'Must use async file operations',
    severity: 'error',
    check: (logData) => {
      if (logData.sourceCode.includes('readFileSync') || 
          logData.sourceCode.includes('writeFileSync')) {
        return [{
          message: 'Synchronous file operation in debug code',
          location: logData.fileOperation
        }]
      }
      return []
    },
    fix: () => ({ type: 'convert_to_async' })
  },
  {
    id: 'DEBUG-003',
    description: 'Diagnostic history must be bounded',
    severity: 'warning',
    check: (diagnostics) => {
      if (diagnostics.errors.length > 100 || 
          diagnostics.warnings.length > 100) {
        return [{
          message: 'Diagnostic history growing unbounded',
          location: diagnostics.storageArray
        }]
      }
      return []
    },
    fix: () => ({ type: 'truncate', maxSize: 100 })
  },
  {
    id: 'DEBUG-004',
    description: 'Stack traces should be analyzed',
    severity: 'info',
    check: (logData) => {
      if (logData.error && !logData.stackAnalysis) {
        return [{
          message: 'Stack trace not analyzed for patterns',
          location: logData.errorSite
        }]
      }
      return []
    }
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解调试诊断规范：

1. **第一阶段**: 学习日志分类和 PII 过滤原则
2. **第二阶段**: 理解诊断跟踪服务和数据管理
3. **第三阶段**: 掌握堆栈解析和模式识别
4. **第四阶段**: 实现会话调试工具和状态捕获
5. **第五阶段**: 提供智能根因分析和修复建议

通过学习路径，AI IDE可以成长为能够独立设计和审查调试诊断系统的专家。

---
name: error-handling-specialist
description: AI Agent 错误处理专家 - 统一错误类型、错误传播、降级策略、错误恢复、容错设计
aliases: ['错误处理', 'error-expert']
argumentHint: '<错误场景> [恢复策略]'
allowedTools: FileReadTool, GrepTool, GlobTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要设计统一的错误处理系统、实现优雅降级、构建容错机制、或处理复杂错误传播时
---

# Error Handling Specialist

使用此技能当你需要进行错误处理系统设计、统一错误类型定义、实现降级策略、或构建容错机制。

## 目标

产出完整的错误处理方案，包括统一错误类型系统、错误传播链、降级策略、恢复机制和容错设计。

## 核心能力（基于 Claude Code源码）

### 1. 统一错误类型系统 (src/utils/errors.ts)

```typescript
// Claude Code的错误类型定义模式
export type ErrorCode =
  | 'NETWORK_ERROR'
  | 'AUTHENTICATION_ERROR'
  | 'RATE_LIMIT_ERROR'
  | 'FILE_SYSTEM_ERROR'
  | 'VALIDATION_ERROR'
  | 'CONFIGURATION_ERROR'
  | 'INTERNAL_ERROR'

export interface ClaudeError extends Error {
  code: ErrorCode
  severity: 'critical' | 'high' | 'medium' | 'low'
  recoverable: boolean
  context?: Record<string, unknown>
  cause?: Error
}

// 错误创建工厂函数
export function createError(
  code: ErrorCode,
  message: string,
  options?: {
    severity?: ClaudeError['severity']
    recoverable?: boolean
    context?: Record<string, unknown>
    cause?: Error
  }
): ClaudeError {
  const error = new Error(message) as ClaudeError
  error.code = code
  error.severity = options?.severity || 'medium'
  error.recoverable = options?.recoverable ?? false
  error.context = options?.context
  error.cause = options?.cause
  return error
}
```

### 2. 错误传播链 (src/utils/errorPropagation.ts)

```typescript
// 错误传播的调用链追踪
interface ErrorChain {
  originalError: Error
  wrappedErrors: Error[]
  callStack: string[]
  timestamps: number[]
}

export function wrapError(
  error: Error,
  context: string,
  additionalInfo?: Record<string, unknown>
): Error {
  const chain: ErrorChain = {
    originalError: error,
    wrappedErrors: [],
    callStack: [context],
    timestamps: [Date.now()]
  }
  
  // 保留原始堆栈
  if (error.stack) {
    chain.wrappedErrors.push(error)
  }
  
  const wrapped = new Error(`${context}: ${error.message}`)
  wrapped.cause = error
  
  // 附加上下文信息
  Object.assign(wrapped, additionalInfo)
  
  return wrapped
}
```

### 3. 降级策略系统 (src/utils/degradationStrategies.ts)

```typescript
type DegradationLevel = 'full' | 'partial' | 'minimal' | 'offline'

interface DegradationConfig {
  level: DegradationLevel
  disabledFeatures: string[]
  enabledFallbacks: string[]
  retryPolicy?: RetryConfig
}

export function applyDegradation(config: DegradationConfig): void {
  switch (config.level) {
    case 'partial':
      // 禁用非核心功能
      disableFeatures(config.disabledFeatures)
      // 启用备用方案
      enableFallbacks(config.enabledFallbacks)
      break
    
    case 'minimal':
      // 只保留核心功能
      disableAllNonEssential()
      enableCoreOnly()
      break
    
    case 'offline':
      // 完全离线模式
      enableOfflineMode()
      queueForLaterSync()
      break
  }
}
```

### 4. 重试和恢复机制 (src/utils/retryWithBackoff.ts)

```typescript
interface RetryConfig {
  maxRetries: number
  initialDelayMs: number
  maxDelayMs: number
  backoffMultiplier: number
  jitter: boolean
  retryableErrors?: ErrorCode[]
}

export async function retryWithExponentialBackoff<T>(
  operation: () => Promise<T>,
  config: RetryConfig = {
    maxRetries: 3,
    initialDelayMs: 1000,
    maxDelayMs: 30000,
    backoffMultiplier: 2,
    jitter: true
  }
): Promise<T> {
  let lastError: Error
  let delay = config.initialDelayMs
  
  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await operation()
    } catch (error) {
      lastError = error as Error
      
      // 检查是否为可重试错误
      if (!isRetryableError(error, config.retryableErrors)) {
        throw error
      }
      
      // 最后一次尝试失败，抛出错误
      if (attempt === config.maxRetries) {
        break
      }
      
      // 计算延迟（带抖动）
      const jitterFactor = config.jitter ? Math.random() * 0.3 + 0.7 : 1
      const actualDelay = Math.min(delay * jitterFactor, config.maxDelayMs)
      
      await sleep(actualDelay)
      delay *= config.backoffMultiplier
    }
  }
  
  throw lastError!
}
```

### 5. 熔断器模式 (src/utils/circuitBreaker.ts)

```typescript
type CircuitState = 'closed' | 'open' | 'half-open'

class CircuitBreaker {
  private state: CircuitState = 'closed'
  private failureCount = 0
  private successCount = 0
  private lastFailureTime?: number
  
  constructor(
    private failureThreshold: number = 5,
    private successThreshold: number = 3,
    private timeoutMs: number = 30000
  ) {}
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime! < this.timeoutMs) {
        throw new Error('Circuit breaker is open')
      }
      this.state = 'half-open'
    }
    
    try {
      const result = await operation()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }
  
  private onSuccess(): void {
    this.failureCount = 0
    if (this.state === 'half-open') {
      this.successCount++
      if (this.successCount >= this.successThreshold) {
        this.state = 'closed'
        this.successCount = 0
      }
    }
  }
  
  private onFailure(): void {
    this.failureCount++
    this.lastFailureTime = Date.now()
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'open'
    }
  }
}
```

### 6. 错误日志和监控 (src/utils/errorMonitoring.ts)

```typescript
interface ErrorEvent {
  error: Error
  timestamp: number
  userId?: string
  sessionId?: string
  context: {
    component: string
    action: string
    metadata?: Record<string, unknown>
  }
  severity: 'critical' | 'high' | 'medium' | 'low'
}

class ErrorMonitor {
  private events: ErrorEvent[] = []
  private readonly MAX_EVENTS = 1000
  
  track(error: Error, context: ErrorEvent['context']): void {
    const event: ErrorEvent = {
      error,
      timestamp: Date.now(),
      sessionId: getCurrentSessionId(),
      context,
      severity: classifyErrorSeverity(error)
    }
    
    this.events.push(event)
    
    // 限制事件数量
    if (this.events.length > this.MAX_EVENTS) {
      this.events.shift()
    }
    
    // 严重错误立即上报
    if (event.severity === 'critical' || event.severity === 'high') {
      this.alertImmediately(event)
    }
  }
  
  getRecentErrors(limit: number = 50): ErrorEvent[] {
    return this.events.slice(-limit)
  }
  
  getErrorRate(windowMs: number = 60000): number {
    const cutoff = Date.now() - windowMs
    const recentErrors = this.events.filter(e => e.timestamp > cutoff)
    return recentErrors.length / (windowMs / 60000) // errors per minute
  }
}
```

## 工作流程

### 第一步：错误分类和定义 (15 分钟)

1. **识别错误类型**
   ```typescript
   // 列出所有可能的错误场景
   const errorScenarios = [
     '网络请求失败',
     '认证过期',
     '文件不存在',
     '权限不足',
     'API 限流',
     '数据验证失败',
     '配置错误',
     '内部逻辑错误'
   ]
   
   // 为每个场景定义错误码
   const errorCodeMap: Record<string, ErrorCode> = {
     '网络请求失败': 'NETWORK_ERROR',
     '认证过期': 'AUTHENTICATION_ERROR',
     // ...
   }
   ```

2. **定义严重程度**
   ```typescript
   // Critical: 系统崩溃，必须立即处理
   // High: 核心功能不可用，需要尽快修复
   // Medium: 部分功能受影响，有计划地修复
   // Low: 轻微问题，不影响主要功能
   ```

### 第二步：错误处理策略设计 (20 分钟)

1. **确定恢复策略**
   ```typescript
   const recoveryStrategies: Record<ErrorCode, RecoveryStrategy> = {
     NETWORK_ERROR: {
       retry: true,
       fallback: 'use_cached_data',
       degrade: 'enable_offline_mode'
     },
     AUTHENTICATION_ERROR: {
       retry: false,
       fallback: 'redirect_to_login',
       degrade: 'disable_authenticated_features'
     },
     RATE_LIMIT_ERROR: {
       retry: true,
       fallback: 'queue_request',
       degrade: 'reduce_request_frequency'
     }
   }
   ```

2. **设计降级方案**
   ```typescript
   // Full → Partial → Minimal → Offline
   const degradationPlan = {
     partial: {
       disable: ['real_time_sync', 'background_tasks'],
       enable: ['local_cache', 'manual_refresh']
     },
     minimal: {
       disable: ['all_non_essential'],
       enable: ['core_read_operations']
     },
     offline: {
       disable: ['all_network_operations'],
       enable: ['local_storage_access', 'queued_writes']
     }
   }
   ```

### 第三步：实现错误处理系统 (30 分钟)

1. **创建错误基类和工厂**
2. **实现重试机制**
3. **实现熔断器**
4. **实现降级策略**
5. **集成错误监控**

## 规则

- ✅ 所有错误都有明确的错误码和严重程度
- ✅ 区分可恢复错误和不可恢复错误
- ✅ 实现指数退避重试策略
- ✅ 使用熔断器防止雪崩效应
- ✅ 提供优雅的降级方案
- ❌ 不要吞掉错误而不记录
- ❌ 不要在不确定的情况下重试
- ❌ 不要暴露内部错误细节给用户
- ❌ 不要忘记清理资源（即使发生错误）

## 输出格式

### 📊 错误分类清单

```markdown
| 错误码 | 描述 | 严重程度 | 可恢复 | 处理策略 |
|-------|------|---------|--------|---------|
| NETWORK_ERROR | 网络连接失败 | High | Yes | 重试 + 降级 |
| AUTH_ERROR | 认证失败 | Critical | No | 跳转登录 |
| ... | ... | ... | ... | ... |
```

### 🏗️ 错误处理架构图

使用流程图展示错误传播和处理流程。

### 📝 实现代码

提供完整的 TypeScript 实现，包括错误类、重试逻辑、熔断器等。

### 🔍 测试计划

详细的错误注入测试、恢复测试、降级测试计划。

### ⚠️ 风险评估

| 风险项 | 可能性 | 影响程度 | 缓解措施 |
|-------|--------|----------|----------|
| 重试风暴 | 中 | 高 | 指数退避 + 最大重试次数 |
| 熔断误判 | 低 | 中 | 半开状态探测 |
| 降级体验差 | 中 | 中 | 用户提示 + 手动恢复选项 |

## 交付物清单

最终报告应包含：
- [ ] 错误类型系统和错误码定义
- [ ] 错误传播链设计
- [ ] 重试和恢复机制实现
- [ ] 熔断器模式实现
- [ ] 降级策略方案
- [ ] 错误监控系统集成
- [ ] 测试计划和测试用例

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发错误处理系统。

## AI IDE 开发前检查

### 错误类型验证
```yaml
必须检查:
  - ErrorCode枚举定义完整
  - ClaudeError接口继承正确
  - 错误工厂函数可用
  - 错误码映射表完整

警告信号:
  - 直接使用Error而无错误码
  - 缺少严重程度分类
  - 未标记是否可恢复
```

### 错误处理基础设施
```yaml
必须存在:
  - 全局错误处理器
  - 错误日志记录器
  - 错误监控服务
  - 错误边界组件
```

## AI IDE 开发中指导

### 实时错误处理监控
当编写错误处理代码时，AI IDE应该：

1. **错误类型检查**
   ```typescript
   // ✅ 正确的错误处理
   try {
     await riskyOperation()
   } catch (error) {
     if (isClaudeError(error) && error.code === 'NETWORK_ERROR') {
       handleNetworkError(error)
     } else {
       handleUnexpectedError(error)
     }
   }
   
   // ❌ 避免的模式
   try {
     await riskyOperation()
   } catch (e) {
     console.log('Error occurred')  // 无分类、无处理
   }
   // AI IDE 应该：建议使用结构化的错误处理
   ```

2. **重试策略验证**
   ```typescript
   // AI IDE 应该推荐合适的重试配置
   const retryConfig = {
     maxRetries: 3,  // 不超过 5 次
     initialDelayMs: 1000,  // 至少 1 秒
     backoffMultiplier: 2,  // 指数增长
     jitter: true  // 添加抖动
   }
   ```

3. **资源清理保证**
   ```typescript
   // AI IDE 应该确保 finally 块清理资源
   let resource: Resource | null = null
   try {
     resource = await acquireResource()
     await useResource(resource)
   } finally {
     if (resource) {
       await releaseResource(resource)  // 总是清理
     }
   }
   ```

### 错误边界设计
AI IDE应该提供：
- **Try-Catch边界**: 捕获和处理错误
- **错误传播控制**: 决定何时向上传播错误
- **降级开关**: 动态启用/禁用功能
- **用户友好提示**: 将技术错误转换为用户语言

## AI IDE 完成后验证

### 错误处理健康度评分卡
```yaml
错误分类 (0-10分):
  - 错误码覆盖率 > 95%
  - 严重程度准确
  - 可恢复性标记正确

重试机制 (0-10分):
  - 指数退避实现
  - 抖动防止同步
  - 最大重试限制

熔断器 (0-10分):
  - 故障计数准确
  - 超时恢复正常
  - 半开状态正确

降级策略 (0-10分):
  - 分级明确
  - 用户体验良好
  - 恢复路径清晰

监控告警 (0-10分):
  - 错误追踪完整
  - 告警阈值合理
  - 上报及时

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 错误分类测试
test('creates correct error type', () => {...})
test('preserves error chain', () => {...})

# 2. 重试机制测试
test('retries on transient errors', () => {...})
test('gives up after max retries', () => {...})

# 3. 熔断器测试
test('opens after threshold', () => {...})
test('half-open recovers', () => {...})

# 4. 降级测试
test('disables features correctly', () => {...})
test('enables fallbacks', () => {...})

# 5. 资源清理测试
test('cleans up on error', () => {...})
test('finally always runs', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 吞掉错误
async function process() {
  try {
    await doSomething()
  } catch (e) {
    // 什么都不做 - 错误被隐藏
  }
}
// AI IDE 应该：立即警告并要求记录或重新抛出

// ❌ 无限重试
while (true) {
  try {
    await operation()
    break
  } catch (e) {
    // 无重试限制
  }
}
// AI IDE 应该：建议添加最大重试次数

// ❌ 资源泄漏
const conn = await connect()
await query(conn)  // 如果这里出错，conn 永远不会关闭
// AI IDE 应该：建议使用 try-finally 确保清理
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 重试间隔固定
retry({ interval: 1000 })  // 每次都等 1 秒
// AI IDE 应该：建议使用指数退避

// ⚠️ 错误信息泄露
throw new Error(`DB connection failed: ${dbPassword}`)
// AI IDE 应该：建议脱敏敏感信息
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
catch (e) {
  console.error('Error:', e)  // 基础日志
}
// AI IDE 可以建议：使用结构化错误日志
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 错误分类
- [ ] 使用统一的错误类型
- [ ] 错误码定义清晰
- [ ] 严重程度标记正确
- [ ] 可恢复性明确

### 重试机制
- [ ] 指数退避实现
- [ ] 最大重试次数限制
- [ ] 仅重试可恢复错误
- [ ] 添加抖动防止同步

### 熔断器
- [ ] 故障阈值合理
- [ ] 超时时间适当
- [ ] 半开状态探测
- [ ] 成功计数重置

### 降级策略
- [ ] 分级明确（full/partial/minimal/offline）
- [ ] 用户体验考虑
- [ ] 恢复路径清晰
- [ ] 手动覆盖选项

### 资源管理
- [ ] try-finally 确保清理
- [ ] 使用 using 语句（如果可用）
- [ ] 连接池正确释放
- [ ] 文件句柄关闭

## AI IDE 集成实现示例

```typescript
interface ErrorHandlingRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (sourceCode: string, errorHandlers: ErrorHandler[]) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: ErrorHandlingRule[] = [
  {
    id: 'ERROR-001',
    description: 'All errors must have error code',
    severity: 'error',
    check: (sourceCode) => {
      const bareThrows = findBareThrowStatements(sourceCode)
      if (bareThrows.length > 0) {
        return [{
          message: 'Throw statement without error code',
          location: bareThrows.map(t => t.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'wrap_with_claude_error' })
  },
  {
    id: 'ERROR-002',
    description: 'Catch blocks must handle or log errors',
    severity: 'error',
    check: (sourceCode) => {
      const emptyCatches = findEmptyCatchBlocks(sourceCode)
      if (emptyCatches.length > 0) {
        return [{
          message: 'Empty catch block swallows error',
          location: emptyCatches.map(c => c.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_logging_or_rethrow' })
  },
  {
    id: 'ERROR-003',
    description: 'Async operations must clean up on error',
    severity: 'warning',
    check: (sourceCode) => {
      const unclosedResources = findUnclosedResources(sourceCode)
      if (unclosedResources.length > 0) {
        return [{
          message: 'Resource may not be closed on error',
          location: unclosedResources.map(r => r.acquisitionSite)
        }]
      }
      return []
    },
    fix: () => ({ type: 'wrap_in_try_finally' })
  },
  {
    id: 'ERROR-004',
    description: 'Retry should have exponential backoff',
    severity: 'warning',
    check: (sourceCode) => {
      const fixedRetries = findFixedIntervalRetries(sourceCode)
      if (fixedRetries.length > 0) {
        return [{
          message: 'Fixed retry interval, consider exponential backoff',
          location: fixedRetries.map(r => r.retrySite)
        }]
      }
      return []
    }
  },
  {
    id: 'ERROR-005',
    description: 'Error messages must not contain sensitive data',
    severity: 'error',
    check: (sourceCode) => {
      const sensitiveInErrors = findSensitiveDataInErrors(sourceCode)
      if (sensitiveInErrors.length > 0) {
        return [{
          message: 'Error message may leak sensitive information',
          location: sensitiveInErrors.map(s => s.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'redact_sensitive_data' })
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解错误处理规范：

1. **第一阶段**: 学习错误分类和错误码设计
2. **第二阶段**: 理解重试机制和指数退避
3. **第三阶段**: 掌握熔断器模式和状态转换
4. **第四阶段**: 实现降级策略和用户体验
5. **第五阶段**: 提供智能错误分析和恢复建议

通过学习路径，AI IDE可以成长为能够独立设计和审查错误处理系统的专家。

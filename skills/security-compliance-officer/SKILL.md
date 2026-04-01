---
name: security-compliance-officer
description: AI Agent 安全合规专家 - 权限分级、PII 保护、策略执行、安全审计、合规检查、denial tracking
aliases: ['安全', 'security-expert']
argumentHint: '<安全领域> [合规标准]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要设计权限控制系统、实现 PII 保护机制、建立安全审计流程、或确保合规性要求时
---

# Security Compliance Officer

使用此技能当你需要进行安全合规系统设计、权限控制实现、PII 保护机制开发、或安全审计流程建立。

## 目标

产出完整的安全合规方案，包括权限分级系统、PII 保护机制、策略执行引擎、安全审计日志和合规检查清单。

## 核心能力

### 1. 权限模式系统 (src/utils/permissions/PermissionMode.ts)

Claude Code定义了 6 种权限模式：

```typescript
export const PERMISSION_MODES = [
  'default',           // 标准审批流程
  'plan',              // 计划模式（只读）
  'acceptEdits',       // 自动接受编辑
  'bypassPermissions', // 绕过权限（危险）
  'dontAsk',           // 不询问（全部允许）
  'auto',              // 自动模式（ant-only）
] as const

export const EXTERNAL_PERMISSION_MODES = [
  'default',
  'plan',
  'acceptEdits',
  'bypassPermissions',
  'dontAsk',
] as const

export type PermissionMode = typeof PERMISSION_MODES[number]
export type ExternalPermissionMode = typeof EXTERNAL_PERMISSION_MODES[number]

interface PermissionModeConfig {
  title: string
  shortTitle: string
  symbol: string
  color: ModeColorKey
  external: ExternalPermissionMode
}

const PERMISSION_MODE_CONFIG: Partial<Record<PermissionMode, PermissionModeConfig>> = {
  default: {
    title: 'Default',
    shortTitle: 'Default',
    symbol: '',
    color: 'text',
    external: 'default',
  },
  plan: {
    title: 'Plan Mode',
    shortTitle: 'Plan',
    symbol: '⏸️',
    color: 'planMode',
    external: 'plan',
  },
  acceptEdits: {
    title: 'Accept edits',
    shortTitle: 'Accept',
    symbol: '⏩',
    color: 'autoAccept',
    external: 'acceptEdits',
  },
  bypassPermissions: {
    title: 'Bypass Permissions',
    shortTitle: 'Bypass',
    symbol: '⏩',
    color: 'error',
    external: 'bypassPermissions',
  },
  dontAsk: {
    title: "Don't Ask",
    shortTitle: 'DontAsk',
    symbol: '⏩',
    color: 'error',
    external: 'dontAsk',
  },
  ...(feature('TRANSCRIPT_CLASSIFIER')
    ? {
        auto: {
          title: 'Auto mode',
          shortTitle: 'Auto',
          symbol: '⏩',
          color: 'warning',
          external: 'default',
        },
      }
    : {}),
}
```

**权限模式对比**:

| 模式 | 文件读取 | 文件写入 | Shell 执行 | 适用场景 |
|------|---------|---------|-----------|----------|
| **default** | ✅ | ⚠️ 询问 | ⚠️ 询问 | 日常开发 |
| **plan** | ✅ | ❌ |  | 代码审查、学习 |
| **acceptEdits** | ✅ | ✅ 自动 | ⚠️ 询问 | 信任的自动化任务 |
| **bypassPermissions** | ✅ | ✅ | ✅ | 完全信任（危险！） |
| **dontAsk** | ✅ | ✅ | ✅ | 无人值守任务 |
| **auto** | ✅ | 🤖 AI 决定 | 🤖 AI 决定 | 高级用户（ant-only） |

### 2. 权限规则系统

```typescript
interface PermissionRule {
  id: string
  pattern: string  // Glob pattern for file paths
  effect: 'allow' | 'deny'
  tools?: string[]  // Specific tools this rule applies to
  reason?: string   // Human-readable explanation
  createdAt: number
  updatedAt: number
}

/**
 * Check if a tool can be used with current permissions
 */
export async function canUseTool(
  toolName: string,
  input: Record<string, unknown>,
  permissionContext: {
    mode: PermissionMode
    rules: PermissionRule[]
  }
): Promise<PermissionResult> {
  const { mode, rules } = permissionContext
  
  // Bypass permissions mode - allow everything
  if (mode === 'bypassPermissions') {
    return { status: 'approved' }
  }
  
  // Don't Ask mode - allow everything
  if (mode === 'dontAsk') {
    return { status: 'approved' }
  }
  
  // Plan Mode - read/search/list only
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
  
  // Default/Auto mode - check rules
  for (const rule of rules) {
    if (ruleMatches(rule, toolName, input)) {
      if (rule.effect === 'allow') {
        return { status: 'approved' }
      } else {
        return { 
          status: 'denied', 
          reason: rule.reason || 'Blocked by permission rule' 
        }
      }
    }
  }
  
  // No matching rule - ask user
  return {
    status: 'ask_user',
    prompt: `Allow ${toolName} with arguments: ${JSON.stringify(input)}?`
  }
}

/**
 * Check if a rule matches the given tool and input
 */
function ruleMatches(
  rule: PermissionRule,
  toolName: string,
  input: Record<string, unknown>
): boolean {
  // Check tool filter
  if (rule.tools && !rule.tools.includes(toolName)) {
    return false
  }
  
  // Check file path pattern
  if (rule.pattern) {
    const filePath = input.path as string | undefined
    if (!filePath) return false
    
    return picomatch.isMatch(filePath, rule.pattern)
  }
  
  return true
}
```

### 3. PII 保护系统

```typescript
/**
 * Types of PII to detect and redact
 */
type PIICategory = 
  | 'email'              // Email addresses
  | 'phone'              // Phone numbers
  | 'ssn'                // Social Security Numbers
  | 'credit_card'        // Credit card numbers
  | 'api_key'            // API keys and tokens
  | 'password'           // Passwords
  | 'home_path'          // User home directory paths
  | 'ip_address'         // IP addresses
  | 'aws_credentials'    // AWS access keys
  | 'github_token'       // GitHub personal access tokens

/**
 * PII detection patterns
 */
const PII_PATTERNS: Record<PIICategory, RegExp> = {
  email: /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
  phone: /\+?[1-9]\d{1,14}(?:\s|-|\.)/g,
  ssn: /\b\d{3}-\d{2}-\d{4}\b/g,
  credit_card: /\b(?:\d{4}[- ]?){3}\d{4}\b/g,
  api_key: /\b(sk-[a-zA-Z0-9]{32,})\b/g,
  password: /(?:"password"|"passwd"|"pwd")\s*[:=]\s*"[^"]+"/gi,
  home_path: new RegExp(`\\b/Users/[^/\\]+/|\$/HOME/|/home/[^/]+/`, 'g'),
  ip_address: /\b(?:\d{1,3}\.){3}\d{1,3}\b/g,
  aws_credentials: /\b(AKIA[0-9A-Z]{16})\b/g,
  github_token: /\b(ghp_[a-zA-Z0-9]{36})\b/g,
}

/**
 * Redact PII from text
 */
export function redactPII(text: string, categories?: PIICategory[]): string {
  const cats = categories || Object.keys(PII_PATTERNS) as PIICategory[]
  
  let result = text
  for (const category of cats) {
    const pattern = PII_PATTERNS[category]
    const replacement = `[REDACTED_${category.toUpperCase()}]`
    result = result.replace(pattern, replacement)
  }
  
  return result
}

/**
 * Sanitize object for logging (deep PII removal)
 */
export function sanitizeObjectForLogging(obj: unknown): unknown {
  if (typeof obj === 'string') {
    return redactPII(obj)
  }
  
  if (Array.isArray(obj)) {
    return obj.map(item => sanitizeObjectForLogging(item))
  }
  
  if (obj !== null && typeof obj === 'object') {
    const sanitized: Record<string, unknown> = {}
    for (const [key, value] of Object.entries(obj)) {
      // Skip sensitive keys entirely
      if (['password', 'secret', 'token', 'apiKey', 'credential'].includes(key.toLowerCase())) {
        sanitized[key] = '[REDACTED_SENSITIVE_KEY]'
      } else {
        sanitized[key] = sanitizeObjectForLogging(value)
      }
    }
    return sanitized
  }
  
  return obj
}

/**
 * Log diagnostic information without PII
 */
export function logForDiagnosticsNoPII(
  level: 'info' | 'warn' | 'error',
  event: string,
  metadata?: Record<string, unknown>
): void {
  const sanitized = sanitizeObjectForLogging(metadata || {})
  const timestamp = new Date().toISOString()
  const logFn = level === 'error' ? console.error : 
                level === 'warn' ? console.warn : 
                console.log
  
  logFn(`[${timestamp}] [${level.toUpperCase()}] ${event}`, sanitized)
}
```

### 4. Denial Tracking System

```typescript
interface DeniedAction {
  id: string
  timestamp: number
  toolName: string
  input: Record<string, unknown>
  reason: string
  mode: PermissionMode
  userId?: string
}

class DenialTracker {
  private denials: DeniedAction[] = []
  private readonly MAX_HISTORY = 1000
  
  /**
   * Track a denied action
   */
  track(action: Omit<DeniedAction, 'id' | 'timestamp'>): void {
    const denial: DeniedAction = {
      ...action,
      id: generateId('denial'),
      timestamp: Date.now()
    }
    
    this.denials.push(denial)
    
    // Keep only recent history
    if (this.denials.length > this.MAX_HISTORY) {
      this.denials.shift()
    }
    
    // Alert on suspicious patterns
    this.detectSuspiciousPatterns()
  }
  
  /**
   * Detect suspicious denial patterns
   */
  private detectSuspiciousPatterns(): void {
    const recentWindow = 5 * 60 * 1000  // 5 minutes
    const now = Date.now()
    
    const recentDenials = this.denials.filter(
      d => now - d.timestamp < recentWindow
    )
    
    // Too many denials in short time
    if (recentDenials.length > 20) {
      logForDiagnosticsNoPII('warn', 'suspicious_denial_pattern', {
        count: recentDenials.length,
        window_minutes: 5,
        possible_cause: 'User may be stuck in denial loop'
      })
    }
    
    // Same tool denied repeatedly
    const toolCounts = new Map<string, number>()
    for (const denial of recentDenials) {
      const count = toolCounts.get(denial.toolName) || 0
      toolCounts.set(denial.toolName, count + 1)
    }
    
    for (const [tool, count] of toolCounts.entries()) {
      if (count > 10) {
        logForDiagnosticsNoPII('warn', 'repeated_tool_denial', {
          tool,
          count,
          suggestion: 'Consider adjusting permission rules'
        })
      }
    }
  }
  
  /**
   * Get denial statistics
   */
  getStatistics(period: number = 24 * 60 * 60 * 1000): {
    total: number
    byTool: Map<string, number>
    byReason: Map<string, number>
    trend: 'increasing' | 'stable' | 'decreasing'
  } {
    const cutoff = Date.now() - period
    const recent = this.denials.filter(d => d.timestamp > cutoff)
    
    const byTool = new Map<string, number>()
    const byReason = new Map<string, number>()
    
    for (const denial of recent) {
      const toolCount = byTool.get(denial.toolName) || 0
      byTool.set(denial.toolName, toolCount + 1)
      
      const reasonCount = byReason.get(denial.reason) || 0
      byReason.set(denial.reason, reasonCount + 1)
    }
    
    // Calculate trend (compare first half vs second half of period)
    const midpoint = cutoff + period / 2
    const firstHalf = recent.filter(d => d.timestamp < midpoint).length
    const secondHalf = recent.filter(d => d.timestamp >= midpoint).length
    
    const trend = secondHalf > firstHalf * 1.2 ? 'increasing' :
                  secondHalf < firstHalf * 0.8 ? 'decreasing' :
                  'stable'
    
    return {
      total: recent.length,
      byTool,
      byReason,
      trend
    }
  }
  
  /**
   * Export denial report
   */
  exportReport(): string {
    const stats = this.getStatistics()
    
    return `
Denial Report (Last 24h)
========================
Total Denials: ${stats.total}
Trend: ${stats.trend.toUpperCase()}

By Tool:
${Array.from(stats.byTool.entries())
  .map(([tool, count]) => `  ${tool}: ${count}`)
  .join('\n')}

By Reason:
${Array.from(stats.byReason.entries())
  .map(([reason, count]) => `  ${reason}: ${count}`)
  .join('\n')}
`
  }
}

export const denialTracker = new DenialTracker()
```

### 5. 安全审计日志

```typescript
interface AuditLogEntry {
  id: string
  timestamp: number
  eventType: 'tool_use' | 'permission_change' | 'config_change' | 'auth_event'
  actor: {
    userId: string
    sessionId: string
    ipAddress?: string
  }
  action: string
  resource?: string
  outcome: 'success' | 'failure' | 'denied'
  details?: Record<string, unknown>
  riskScore?: number  // 0-100
}

class SecurityAuditor {
  private logs: AuditLogEntry[] = []
  private readonly RETENTION_DAYS = 90
  
  /**
   * Log a security-relevant event
   */
  log(event: Omit<AuditLogEntry, 'id' | 'timestamp'>): void {
    const entry: AuditLogEntry = {
      ...event,
      id: generateId('audit'),
      timestamp: Date.now()
    }
    
    this.logs.push(entry)
    
    // Auto-cleanup old entries
    this.cleanupOldEntries()
    
    // Alert on high-risk events
    if (event.riskScore && event.riskScore > 80) {
      this.alertHighRiskEvent(entry)
    }
  }
  
  /**
   * Query audit logs
   */
  query(filters: {
    startTime?: number
    endTime?: number
    eventType?: AuditLogEntry['eventType']
    actor?: string
    outcome?: AuditLogEntry['outcome']
    minRiskScore?: number
  }): AuditLogEntry[] {
    return this.logs.filter(entry => {
      if (filters.startTime && entry.timestamp < filters.startTime) return false
      if (filters.endTime && entry.timestamp > filters.endTime) return false
      if (filters.eventType && entry.eventType !== filters.eventType) return false
      if (filters.actor && entry.actor.userId !== filters.actor) return false
      if (filters.outcome && entry.outcome !== filters.outcome) return false
      if (filters.minRiskScore && (entry.riskScore || 0) < filters.minRiskScore) return false
      return true
    })
  }
  
  /**
   * Generate compliance report
   */
  generateComplianceReport(standard: 'SOC2' | 'GDPR' | 'HIPAA'): string {
    const period = 90 * 24 * 60 * 60 * 1000  // 90 days
    const recentLogs = this.query({ startTime: Date.now() - period })
    
    const report = {
      standard,
      generatedAt: new Date().toISOString(),
      period: 'Last 90 days',
      summary: {
        totalEvents: recentLogs.length,
        securityEvents: recentLogs.filter(e => e.riskScore && e.riskScore > 50).length,
        deniedActions: recentLogs.filter(e => e.outcome === 'denied').length,
        authFailures: recentLogs.filter(e => e.eventType === 'auth_event' && e.outcome === 'failure').length
      },
      recommendations: this.generateRecommendations(recentLogs, standard)
    }
    
    return JSON.stringify(report, null, 2)
  }
  
  private cleanupOldEntries(): void {
    const cutoff = Date.now() - (this.RETENTION_DAYS * 24 * 60 * 60 * 1000)
    this.logs = this.logs.filter(entry => entry.timestamp > cutoff)
  }
  
  private alertHighRiskEvent(entry: AuditLogEntry): void {
    console.error('[SECURITY ALERT] High-risk event detected:', {
      eventId: entry.id,
      type: entry.eventType,
      action: entry.action,
      riskScore: entry.riskScore,
      timestamp: new Date(entry.timestamp).toISOString()
    })
  }
  
  private generateRecommendations(
    logs: AuditLogEntry[],
    standard: string
  ): string[] {
    const recommendations: string[] = []
    
    // Check for common security issues
    const failedAuths = logs.filter(e => e.eventType === 'auth_event' && e.outcome === 'failure').length
    if (failedAuths > 10) {
      recommendations.push('Multiple authentication failures detected - consider implementing account lockout')
    }
    
    const deniedActions = logs.filter(e => e.outcome === 'denied').length
    if (deniedActions > logs.length * 0.3) {
      recommendations.push('High denial rate - review permission policies for usability')
    }
    
    return recommendations
  }
}

export const securityAuditor = new SecurityAuditor()
```

### 6. 安全检查清单

```markdown
## 权限控制检查
- [ ] 所有敏感操作都有权限检查
- [ ] 权限规则可配置且易于理解
- [ ] 默认拒绝未知操作
- [ ] 权限变更有审计日志
- [ ] 支持最小权限原则

## PII 保护检查
- [ ] 所有日志都经过 PII 过滤
- [ ] 敏感数据加密存储
- [ ] API 密钥不硬编码
- [ ] 用户路径不泄露
- [ ] 错误信息不包含敏感数据

## 审计合规检查
- [ ] 所有安全事件都有日志
- [ ] 日志保留期符合法规要求
- [ ] 支持日志查询和导出
- [ ] 异常行为自动告警
- [ ] 定期生成合规报告

## 认证授权检查
- [ ] OAuth token 安全存储
- [ ] Token 过期自动刷新
- [ ] 支持多因素认证
- [ ] Session 超时合理设置
- [ ] 注销后彻底清理凭证
```

### 7. 安全事件响应流程

```markdown
## 安全事件分级

### P0 - 严重 (Critical)
- 凭证泄露
- 未授权访问生产数据
- 大规模 PII 泄露

**响应时间**: 立即 (<15 分钟)
**升级路径**: 安全团队 → CTO → CEO

### P1 - 高 (High)
- 权限提升尝试
- 多次认证失败
- 可疑的 API 调用模式

**响应时间**: 1 小时内
**升级路径**: 安全团队 → 工程 VP

### P2 - 中 (Medium)
- 单个用户权限异常
- 配置错误导致的数据暴露
- 第三方依赖漏洞

**响应时间**: 24 小时内
**升级路径**: 安全团队 → 相关团队负责人

### P3 - 低 (Low)
- 轻微的合规偏差
- 文档不完善
- 最佳实践未遵循

**响应时间**: 1 周内
**升级路径**: 相关团队自行处理

## 事件响应步骤

1. **检测与分类** (5-15 分钟)
   - 确认事件真实性
   - 评估影响范围
   - 确定事件级别

2. **遏制与隔离** (15-60 分钟)
   - 阻止事件扩散
   - 隔离受影响的系统
   - 保存证据

3. **根除与恢复** (1-24 小时)
   - 消除根本原因
   - 恢复受影响的服务
   - 验证修复效果

4. **总结与改进** (1-7 天)
   - 编写事后分析报告
   - 实施预防措施
   - 更新应急响应流程
```

## 工作流程

### 第一步：安全现状评估 (15 分钟)

```bash
# 1. 检查权限配置
cat ~/.claude/settings.json | jq '.permissions'

# 2. 审计日志分析
grep -i "denied\|blocked\|unauthorized" ~/.claude/logs/*.log | tail -50

# 3. PII 泄露扫描
find . -name "*.log" -exec grep -E "(sk-[a-zA-Z0-9]{32}|ghp_[a-zA-Z0-9]{36})" {} \;

# 4. 敏感文件检查
find . -name ".env*" -o -name "*credentials*" -o -name "*secrets*"
```

### 第二步：风险评估 (20 分钟)

使用 STRIDE 威胁建模方法：
- **Spoofing**: 身份伪造风险
- **Tampering**: 数据篡改风险
- **Repudiation**: 抵赖风险
- **Information Disclosure**: 信息泄露风险
- **Denial of Service**: 服务中断风险
- **Elevation of Privilege**: 权限提升风险

### 第三步：remediation规划 (25 分钟)

基于风险评估制定修复计划，包括立即修复项、短期改进项、长期优化项。

## 规则

- ✅ 始终遵循最小权限原则
- ✅ 对所有外部输入进行验证和清理
- ✅ 在日志中自动过滤 PII 和敏感信息
- ✅ 为所有安全事件提供审计跟踪
- ✅ 实施深度防御策略（多层防护）
- ❌ 不要硬编码凭证或 API 密钥
- ❌ 不要在日志中输出敏感数据
- ❌ 不要绕过权限检查即使为了方便
- ❌ 不要忽略安全告警和异常

## 输出格式

### 📊 安全现状评估

```markdown
**整体安全评分**: X/100

**优势领域**:
- ...

**高风险区域**:
- ...

**中等风险区域**:
- ...
```

### 🔴 风险识别

```markdown
**已识别的风险**: N 个

**按严重程度**:
- Critical: X
- High: Y
- Medium: Z
- Low: W

**TOP 5 风险**:
1. ... (严重程度)
2. ... (严重程度)
3. ... (严重程度)
```

### 🛡️ 缓解措施

```markdown
**立即修复** (24-48 小时):
1. ...
2. ...

**短期改进** (1-2 周):
1. ...
2. ...

**长期优化** (1-3 月):
1. ...
2. ...
```

### 📋 合规检查

```markdown
**适用标准**: SOC2/GDPR/HIPAA/...

**合规状态**:
- 符合要求：X 项
- 部分符合：Y 项
- 不符合：Z 项

**差距分析**:
...
```

### 🚨 事件响应计划

```markdown
**应急联系人**:
- Primary: ...
- Secondary: ...

**升级流程**:
...

**沟通模板**:
...
```

### 📈 持续监控方案

```markdown
**关键指标**:
- 权限拒绝率
- 认证失败次数
- PII 泄露检测
- 异常行为告警

**监控频率**: 实时/每小时/每天

**告警阈值**:
...
```

## 交付物清单

最终报告应包含：
- [ ] 安全现状评估和评分
- [ ] 风险识别和严重程度评估
- [ ] 详细的缓解措施和修复计划
- [ ] 合规检查和差距分析
- [ ] 安全事件响应计划
- [ ] 持续监控方案和告警阈值
- [ ] 安全检查清单和审计报告

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发安全合规系统。

## AI IDE 开发前检查

### 权限模式系统验证
```yaml
必须检查:
  - PERMISSION_MODES 枚举定义完整
  - PermissionMode类型安全
  - 各模式的工具访问规则明确
  - 默认模式安全（非 bypass）

警告信号:
  - 硬编码的 bypass 模式
  - 缺少权限检查
  - 默认允许所有操作
```

### PII 保护准备
```yaml
必须存在:
  - PII_PATTERNS正则表达式定义
  - redactPII函数实现
  - sanitizeObjectForLogging深度清理
  - logForDiagnosticsNoPII统一出口
```

## AI IDE 开发中指导

### 实时安全监控
当执行操作时，AI IDE应该：

1. **权限检查验证**
   ```typescript
   // ✅ 正确的权限检查
   const result = await canUseTool(toolName, input, permissionContext)
   if (result.status === 'denied') {
     logger.warn(`Tool ${toolName} denied: ${result.reason}`)
     return
   }
   
   // ❌ 避免的模式
   // 直接使用工具而无权限检查
   await executeTool(toolName, input)  // 危险!
   // AI IDE 应该：立即警告并要求添加 canUseTool 检查
   ```

2. **PII 过滤验证**
   ```typescript
   // AI IDE 应该自动检测并过滤 PII
   const safeMetadata = sanitizeObjectForLogging(rawMetadata)
   
   // 检测模式
   const piiPatterns = {
     email: /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
     homePath: /\/Users\/[^/]+\//g,
     apiKey: /sk-[a-zA-Z0-9]{32,}/g,
     githubToken: /ghp_[a-zA-Z0-9]{36}/g
   }
   
   // 发现 PII 时警告并脱敏
   if (containsPII(rawMetadata)) {
     logger.warn('PII detected - redacting')
     return redactPII(safeMetadata)
   }
   ```

3. **审计日志记录**
   ```typescript
   // AI IDE 应该确保关键操作有审计日志
   securityAuditor.log({
     eventType: 'tool_use',
     actor: { userId, sessionId },
     action: toolName,
     outcome: 'success' | 'failure' | 'denied',
     riskScore: calculateRiskScore(toolName, input)
   })
   ```

### Denial Tracking 系统
AI IDE应该提供：
- **denialTracker.track()**: 记录拒绝的操作
- **detectSuspiciousPatterns()**: 检测可疑模式
- **getStatistics()**: 生成统计数据
- **exportReport()**: 导出拒绝报告

## AI IDE 完成后验证

### 安全合规健康度评分卡
```yaml
权限控制 (0-10分):
  - 模式切换正确性
  - 规则匹配准确性
  - 最小权限原则遵循

PII 保护 (0-10分):
  - 检测准确率 > 99%
  - 脱敏完整性
  - 零 PII 泄露到日志

审计日志 (0-10分):
  - 关键操作全记录
  - 日志保留期合规
  - 查询功能完善

Denial Tracking (0-10分):
  - 拒绝操作全记录
  - 可疑模式检测
  - 统计报告准确

事件响应 (0-10分):
  - 事件分级明确
  - 响应时间达标
  - 升级路径清晰

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量改进)
  - 60-74: Fair (需要加强)
  - <60: Poor (不建议上线)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 权限控制测试
test('bypass mode allows all', () => {...})
test('plan mode blocks writes', () => {...})
test('rules are enforced', () => {...})

# 2. PII 保护测试
test('detects emails', () => {...})
test('redacts home paths', () => {...})
test('filters API keys', () => {...})

# 3. 审计日志测试
test('logs tool usage', () => {...})
test('retains logs for 90 days', () => {...})

# 4. Denial tracking 测试
test('tracks denied actions', () => {...})
test('detects suspicious patterns', () => {...})

# 5. 事件响应测试
test('classifies events correctly', () => {...})
test('alerts on high-risk events', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 权限绕过
if (user.isAdmin) {
  return { status: 'approved' }  // 跳过正常权限检查!
}
// AI IDE 应该：立即警告并阻止

// ❌ PII 明文日志
console.log(`User ${email} performed ${action}`)  // email 未脱敏!
// AI IDE 应该：建议使用 logForDiagnosticsNoPII

// ❌ 硬编码凭证
const API_KEY = 'sk-xxx...'  // 直接写在代码里!
// AI IDE 应该：建议使用环境变量或密钥管理服务

// ❌ 无审计日志
await sensitiveOperation()  // 没有记录到审计日志
// AI IDE 应该：建议添加 securityAuditor.log()
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 权限规则过于宽松
const rules = [{ pattern: '*', effect: 'allow' }]  // 允许所有!
// AI IDE 应该：建议更精细的规则

// ⚠️ PII 检测不完整
const patterns = { email: /.../ }  // 只有 email，缺少其他类型
// AI IDE 应该：建议补充完整的 PII 类型列表
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const isAllowed = user.role === 'admin'  // 简单的 RBAC
// AI IDE 可以建议：实现基于属性的访问控制 (ABAC)
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 权限控制
- [ ] 所有敏感操作有 canUseTool 检查
- [ ] 权限模式正确处理
- [ ] 规则匹配引擎完整
- [ ] 用户询问界面友好

### PII 保护
- [ ] 使用 sanitizeObjectForLogging
- [ ] 所有日志通过 logForDiagnosticsNoPII
- [ ] PII 模式检测完整
- [ ] 无硬编码凭证

### 审计日志
- [ ] 关键操作记录完整
- [ ] 事件类型正确分类
- [ ] 参与者信息准确
- [ ] 结果状态明确

### Denial Tracking
- [ ] 拒绝操作全部记录
- [ ] 可疑模式检测启用
- [ ] 统计数据准确
- [ ] 报告导出可用

### 事件响应
- [ ] 事件分级明确（P0-P3）
- [ ] 响应时间定义
- [ ] 升级路径清晰
- [ ] 高风崄事件告警

## AI IDE 集成实现示例

```typescript
interface SecurityRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (context: SecurityContext, auditLogs: AuditLogEntry[]) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: SecurityRule[] = [
  {
    id: 'SEC-001',
    description: 'All tool calls must have permission check',
    severity: 'error',
    check: (context) => {
      const toolCallsWithoutCheck = context.toolCalls.filter(call => 
        !call.hasPermissionCheck
      )
      if (toolCallsWithoutCheck.length > 0) {
        return [{
          message: `${toolCallsWithoutCheck.length} tool calls without permission check`,
          location: toolCallsWithoutCheck.map(c => c.callSite)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_permission_check' })
  },
  {
    id: 'SEC-002',
    description: 'Logs must not contain PII',
    severity: 'error',
    check: (context) => {
      const logsWithPII = context.logs.filter(log => containsPII(log.message))
      if (logsWithPII.length > 0) {
        return [{
          message: `${logsWithPII.length} log entries contain PII`,
          location: logsWithPII.map(l => l.lineNumber)
        }]
      }
      return []
    },
    fix: () => ({ type: 'redact_pii' })
  },
  {
    id: 'SEC-003',
    description: 'Sensitive operations must be audited',
    severity: 'error',
    check: (context, auditLogs) => {
      const unauditedOps = context.sensitiveOperations.filter(op =>
        !auditLogs.some(log => log.action === op.name && log.timestamp >= op.timestamp)
      )
      if (unauditedOps.length > 0) {
        return [{
          message: `${unauditedOps.length} sensitive operations without audit trail`,
          location: unauditedOps.map(op => op.site)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_audit_log' })
  },
  {
    id: 'SEC-004',
    description: 'Credentials must not be hardcoded',
    severity: 'error',
    check: (context) => {
      const hardcodedCredentials = findHardcodedCredentials(context.sourceCode)
      if (hardcodedCredentials.length > 0) {
        return [{
          message: `${hardcodedCredentials.length} hardcoded credentials found`,
          location: hardcodedCredentials.map(c => c.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'move_to_env' })
  },
  {
    id: 'SEC-005',
    description: 'High-risk events must trigger alerts',
    severity: 'warning',
    check: (context, auditLogs) => {
      const highRiskEvents = auditLogs.filter(log => 
        log.riskScore && log.riskScore > 80 && !log.alerted
      )
      if (highRiskEvents.length > 0) {
        return [{
          message: `${highRiskEvents.length} high-risk events without alert`,
          location: highRiskEvents.map(e => e.id)
        }]
      }
      return []
    },
    fix: () => ({ type: 'send_alert' })
  },
  {
    id: 'SEC-006',
    description: 'Denial patterns should be analyzed',
    severity: 'info',
    check: (context) => {
      const stats = denialTracker.getStatistics()
      if (stats.trend === 'increasing' && stats.total > 20) {
        return [{
          message: 'Increasing denial rate detected - review permission policies',
          location: 'permission_config'
        }]
      }
      return []
    }
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解安全合规规范：

1. **第一阶段**: 学习权限模式和规则匹配
2. **第二阶段**: 理解 PII 检测和脱敏技术
3. **第三阶段**: 掌握审计日志和事件追踪
4. **第四阶段**: 实现 denial tracking 和模式识别
5. **第五阶段**: 提供智能安全风险评估和建议

通过学习路径，AI IDE可以成长为能够独立设计和审查安全合规系统的专家。

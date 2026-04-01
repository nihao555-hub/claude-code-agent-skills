---
name: performance-optimizer
description: AI Agent 性能优化专家 - 启动剖析、懒加载、缓存策略、内存管理、并行执行、性能监控
aliases: ['性能', 'perf-expert']
argumentHint: '<优化目标> [性能指标]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要优化 Agent 启动速度、减少内存占用、改进缓存命中率、或实现并行执行策略时
---

# Performance Optimizer

使用此技能当你需要进行性能分析、启动优化、内存管理、缓存策略设计、或并行执行优化。

## 目标

产出完整的性能优化方案，包括启动剖析报告、懒加载策略、缓存优化、内存管理和并行执行方案。

## 核心能力

### 1. 启动剖析系统 (src/utils/startupProfiler.ts)

Claude Code的启动性能分析系统：

```typescript
/**
 * Startup profiling utility for measuring and reporting time spent in various
 * initialization phases.
 *
 * Two modes:
 * 1. Sampled logging: 100% of ant users, 0.1% of external users - logs phases to Statsig
 * 2. Detailed profiling: CLAUDE_CODE_PROFILE_STARTUP=1 - full report with memory snapshots
 */

// Module-level state - decided once at module load
const DETAILED_PROFILING = isEnvTruthy(process.env.CLAUDE_CODE_PROFILE_STARTUP)

// Sampling for Statsig logging: 100% ant, 0.5% external
const STATSIG_SAMPLE_RATE = 0.005
const STATSIG_LOGGING_SAMPLED =
  process.env.USER_TYPE === 'ant' || Math.random() < STATSIG_SAMPLE_RATE

// Enable profiling if either detailed mode OR sampled for Statsig
const SHOULD_PROFILE = DETAILED_PROFILING || STATSIG_LOGGING_SAMPLED

// Track memory snapshots separately
const memorySnapshots: NodeJS.MemoryUsage[] = []

// Phase definitions for Statsig logging: [startCheckpoint, endCheckpoint]
const PHASE_DEFINITIONS = {
  import_time: ['cli_entry', 'main_tsx_imports_loaded'],
  init_time: ['init_function_start', 'init_function_end'],
  settings_time: ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time: ['cli_entry', 'main_after_run'],
} as const

/**
 * Record a checkpoint with the given name
 */
export function profileCheckpoint(name: string): void {
  if (!SHOULD_PROFILE) return

  const perf = getPerformance()
  perf.mark(name)

  // Only capture memory when detailed profiling enabled
  if (DETAILED_PROFILING) {
    memorySnapshots.push(process.memoryUsage())
  }
}

/**
 * Get a formatted report of all checkpoints
 */
function getReport(): string {
  if (!DETAILED_PROFILING) {
    return 'Startup profiling not enabled'
  }

  const perf = getPerformance()
  const marks = perf.getEntriesByType('mark')
  
  const lines: string[] = []
  lines.push('='.repeat(80))
  lines.push('STARTUP PROFILING REPORT')
  lines.push('='.repeat(80))
  lines.push('')

  let prevTime = 0
  for (const [i, mark] of marks.entries()) {
    lines.push(
      formatTimelineLine(
        mark.startTime,
        mark.startTime - prevTime,
        mark.name,
        memorySnapshots[i],
        8,
        7,
      ),
    )
    prevTime = mark.startTime
  }

  lines.push('')
  lines.push(`Total startup time: ${formatMs(lastMark?.startTime ?? 0)}ms`)
  lines.push('='.repeat(80))

  return lines.join('\n')
}

export function profileReport(): void {
  // Log to Statsig (sampled: 100% ant, 0.1% external)
  logStartupPerf()

  // Output detailed report if CLAUDE_CODE_PROFILE_STARTUP=1
  if (DETAILED_PROFILING) {
    const path = getStartupPerfLogPath()
    const dir = dirname(path)
    const fs = getFsImplementation()
    fs.mkdirSync(dir)
    writeFileSync_DEPRECATED(path, getReport(), {
      encoding: 'utf8',
      flush: true,
    })
  }
}
```

**关键性能检查点**:
```typescript
// CLI 入口
profileCheckpoint('cli_entry')

// 模块加载
profileCheckpoint('main_tsx_imports_loaded')

// 初始化函数
profileCheckpoint('init_function_start')
profileCheckpoint('init_function_end')

// 设置加载
profileCheckpoint('eagerLoadSettings_start')
profileCheckpoint('eagerLoadSettings_end')

// 插件系统
profileCheckpoint('plugins_loaded')

// MCP 连接
profileCheckpoint('mcp_connections_established')

// 准备就绪
profileCheckpoint('ready_for_user_input')
```

### 2. 懒加载策略

```typescript
/**
 * Lazy schema validation - only load zod when actually used
 */
export function lazySchema<T>(factory: () => z.ZodType<T>): z.Lazy<z.ZodType<T>> {
  let cached: z.ZodType<T> | undefined
  
  return z.lazy(() => {
    if (!cached) {
      cached = factory()
    }
    return cached
  })
}

/**
 * Conditional feature loading - only require if feature flag enabled
 */
const HeavyModule = feature('HEAVY_FEATURE')
  ? require('./HeavyModule.js').HeavyModule
  : null

/**
 * Memoized expensive computation - cache result forever
 */
export const getExpensiveContext = memoize(async () => {
  const startTime = Date.now()
  const result = await computeExpensiveValue()
  
  logForDiagnosticsNoPII('info', 'expensive_context_computed', {
    duration_ms: Date.now() - startTime
  })
  
  return result
}, { maxAge: Infinity })  // Cache for entire process lifetime

/**
 * LRU cache for bounded memory usage
 */
class LRUCache<K, V> {
  private cache = new Map<K, V>()
  private readonly maxSize: number
  
  constructor(maxSize: number = 100) {
    this.maxSize = maxSize
  }
  
  get(key: K): V | undefined {
    const value = this.cache.get(key)
    if (value !== undefined) {
      // Move to end (most recently used)
      this.cache.delete(key)
      this.cache.set(key, value)
    }
    return value
  }
  
  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key)
    } else if (this.cache.size >= this.maxSize) {
      // Delete oldest (least recently used)
      const firstKey = this.cache.keys().next().value
      if (firstKey !== undefined) {
        this.cache.delete(firstKey)
      }
    }
    this.cache.set(key, value)
  }
  
  delete(key: K): boolean {
    return this.cache.delete(key)
  }
  
  clear(): void {
    this.cache.clear()
  }
}
```

### 3. 缓存优化策略

```typescript
interface CacheStats {
  hits: number
  misses: number
  size: number
  hitRate: number
  avgAccessTime: number
}

/**
 * Multi-tier caching system
 */
class MultiTierCache<K, V> {
  private l1 = new Map<K, {value: V; timestamp: number}>()  // Memory cache
  private l2: LRUCache<K, V>  // Bounded memory cache
  private stats = {
    hits: 0,
    misses: 0,
    accessTimes: [] as number[]
  }
  
  constructor(l2Size: number = 1000) {
    this.l2 = new LRUCache(l2Size)
  }
  
  async get(key: K, loader: () => Promise<V>): Promise<V> {
    const startTime = Date.now()
    
    // Try L1 cache first (fastest)
    const l1Entry = this.l1.get(key)
    if (l1Entry && !this.isExpired(l1Entry)) {
      this.stats.hits++
      this.recordAccessTime(startTime)
      return l1Entry.value
    }
    
    // Try L2 cache
    const l2Value = this.l2.get(key)
    if (l2Value !== undefined) {
      // Promote to L1
      this.l1.set(key, { value: l2Value, timestamp: Date.now() })
      this.stats.hits++
      this.recordAccessTime(startTime)
      return l2Value
    }
    
    // Cache miss - load fresh
    this.stats.misses++
    const value = await loader()
    
    // Store in both caches
    this.l1.set(key, { value, timestamp: Date.now() })
    this.l2.set(key, value)
    
    this.recordAccessTime(startTime)
    return value
  }
  
  private isExpired(entry: {value: V; timestamp: number}): boolean {
    const TTL = 5 * 60 * 1000  // 5 minutes
    return Date.now() - entry.timestamp > TTL
  }
  
  private recordAccessTime(startTime: number): void {
    this.stats.accessTimes.push(Date.now() - startTime)
    // Keep only last 100 measurements
    if (this.stats.accessTimes.length > 100) {
      this.stats.accessTimes.shift()
    }
  }
  
  getStats(): CacheStats {
    const total = this.stats.hits + this.stats.misses
    return {
      hits: this.stats.hits,
      misses: this.stats.misses,
      size: this.l1.size + this.l2['cache'].size,
      hitRate: total > 0 ? this.stats.hits / total : 0,
      avgAccessTime: this.stats.accessTimes.length > 0
        ? this.stats.accessTimes.reduce((a, b) => a + b, 0) / this.stats.accessTimes.length
        : 0
    }
  }
  
  clear(): void {
    this.l1.clear()
    this.l2.clear()
  }
}
```

### 4. 内存管理

```typescript
interface MemorySnapshot {
  heapUsed: number
  heapTotal: number
  rss: number
  external: number
  arrayBuffers: number
  timestamp: number
}

/**
 * Monitor memory usage and trigger GC if needed
 */
class MemoryMonitor {
  private snapshots: MemorySnapshot[] = []
  private readonly MAX_SNAPSHOTS = 100
  private readonly WARNING_THRESHOLD = 0.8  // 80% of heap limit
  private readonly CRITICAL_THRESHOLD = 0.9  // 90% of heap limit
  
  /**
   * Take a memory snapshot
   */
  snapshot(): MemorySnapshot {
    const usage = process.memoryUsage()
    const snapshot: MemorySnapshot = {
      heapUsed: usage.heapUsed,
      heapTotal: usage.heapTotal,
      rss: usage.rss,
      external: usage.external,
      arrayBuffers: usage.arrayBuffers,
      timestamp: Date.now()
    }
    
    this.snapshots.push(snapshot)
    if (this.snapshots.length > this.MAX_SNAPSHOTS) {
      this.snapshots.shift()
    }
    
    return snapshot
  }
  
  /**
   * Check memory health and return status
   */
  checkHealth(): {
    status: 'healthy' | 'warning' | 'critical'
    heapUsagePercent: number
    recommendation?: string
  } {
    const current = this.snapshot()
    const heapUsagePercent = current.heapUsed / current.heapTotal
    
    if (heapUsagePercent >= this.CRITICAL_THRESHOLD) {
      return {
        status: 'critical',
        heapUsagePercent,
        recommendation: 'Immediate GC recommended. Consider increasing --max-old-space-size'
      }
    }
    
    if (heapUsagePercent >= this.WARNING_THRESHOLD) {
      return {
        status: 'warning',
        heapUsagePercent,
        recommendation: 'Monitor closely. May need optimization soon.'
      }
    }
    
    return {
      status: 'healthy',
      heapUsagePercent
    }
  }
  
  /**
   * Get memory trend over time
   */
  getTrend(): {
    direction: 'increasing' | 'stable' | 'decreasing'
    slope: number  // bytes per millisecond
  } {
    if (this.snapshots.length < 2) {
      return { direction: 'stable', slope: 0 }
    }
    
    const recent = this.snapshots.slice(-10)
    const first = recent[0]!
    const last = recent[recent.length - 1]!
    
    const timeDiff = last.timestamp - first.timestamp
    const heapDiff = last.heapUsed - first.heapUsed
    
    const slope = timeDiff > 0 ? heapDiff / timeDiff : 0
    
    return {
      direction: slope > 1000 ? 'increasing' : slope < -1000 ? 'decreasing' : 'stable',
      slope
    }
  }
  
  /**
   * Export memory report
   */
  exportReport(): string {
    const current = this.snapshot()
    const trend = this.getTrend()
    const health = this.checkHealth()
    
    return `
Memory Report
=============
Status: ${health.status.toUpperCase()}
Heap Usage: ${(health.heapUsagePercent * 100).toFixed(1)}%
RSS: ${formatBytes(current.rss)}
Heap Used: ${formatBytes(current.heapUsed)}
Heap Total: ${formatBytes(current.heapTotal)}
Trend: ${trend.direction} (${formatBytes(Math.abs(trend.slope))}/s)
${health.recommendation ? 'Recommendation: ' + health.recommendation : ''}
`
  }
}

function formatBytes(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  if (bytes < 1024 * 1024 * 1024) return `${(bytes / 1024 / 1024).toFixed(1)} MB`
  return `${(bytes / 1024 / 1024 / 1024).toFixed(2)} GB`
}
```

### 5. 并行执行优化

```typescript
/**
 * Execute tasks in parallel with concurrency limit
 */
async function executeInParallel<T, R>(
  items: T[],
  processor: (item: T, index: number) => Promise<R>,
  concurrency: number = 10
): Promise<R[]> {
  const results: R[] = []
  let currentIndex = 0
  let errorOccurred = false
  let firstError: Error | undefined
  
  async function worker(): Promise<void> {
    while (currentIndex < items.length && !errorOccurred) {
      const index = currentIndex++
      try {
        const result = await processor(items[index]!, index)
        results[index] = result
      } catch (error) {
        errorOccurred = true
        firstError = error instanceof Error ? error : new Error(String(error))
      }
    }
  }
  
  // Start workers
  const workers = Array.from({ length: Math.min(concurrency, items.length) }, () => worker())
  await Promise.all(workers)
  
  if (errorOccurred && firstError) {
    throw firstError
  }
  
  return results
}

/**
 * Batch processing for large datasets
 */
async function processInBatches<T, R>(
  items: T[],
  processor: (batch: T[]) => Promise<R[]>,
  batchSize: number = 100
): Promise<R[]> {
  const results: R[] = []
  
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize)
    const batchResults = await processor(batch)
    results.push(...batchResults)
    
    // Yield to event loop periodically
    if ((i / batchSize) % 10 === 0) {
      await new Promise(resolve => setImmediate(resolve))
    }
  }
  
  return results
}

/**
 * Debounced execution - prevent rapid repeated calls
 */
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout> | null = null
  
  return (...args: Parameters<T>) => {
    if (timeoutId) {
      clearTimeout(timeoutId)
    }
    timeoutId = setTimeout(() => {
      fn(...args)
      timeoutId = null
    }, delay)
  }
}

/**
 * Throttled execution - limit call frequency
 */
function throttle<T extends (...args: any[]) => any>(
  fn: T,
  interval: number
): (...args: Parameters<T>) => void {
  let lastCall = 0
  let timeoutId: ReturnType<typeof setTimeout> | null = null
  
  return (...args: Parameters<T>) => {
    const now = Date.now()
    const remaining = interval - (now - lastCall)
    
    if (remaining <= 0) {
      lastCall = now
      fn(...args)
    } else if (!timeoutId) {
      timeoutId = setTimeout(() => {
        lastCall = Date.now()
        fn(...args)
        timeoutId = null
      }, remaining)
    }
  }
}
```

### 6. 性能监控仪表板

```typescript
interface PerformanceMetrics {
  cpu: {
    usage: number  // percentage
    cores: number
  }
  memory: {
    used: number
    total: number
    percent: number
  }
  disk: {
    readOps: number
    writeOps: number
    readBytes: number
    writeBytes: number
  }
  network: {
    bytesSent: number
    bytesReceived: number
    connections: number
  }
  application: {
    uptime: number
    requestCount: number
    errorCount: number
    avgResponseTime: number
  }
}

/**
 * Collect system performance metrics
 */
async function collectPerformanceMetrics(): Promise<PerformanceMetrics> {
  const cpus = os.cpus()
  const mem = os.totalmem()
  const freemem = os.freemem()
  
  return {
    cpu: {
      usage: calculateCpuUsage(cpus),
      cores: cpus.length
    },
    memory: {
      used: mem - freemem,
      total: mem,
      percent: ((mem - freemem) / mem) * 100
    },
    disk: {
      readOps: 0,  // Would need native addon for real values
      writeOps: 0,
      readBytes: 0,
      writeBytes: 0
    },
    network: {
      bytesSent: 0,
      bytesReceived: 0,
      connections: 0
    },
    application: {
      uptime: process.uptime(),
      requestCount: 0,
      errorCount: 0,
      avgResponseTime: 0
    }
  }
}

/**
 * Generate performance report
 */
async function generatePerformanceReport(): Promise<string> {
  const metrics = await collectPerformanceMetrics()
  const memMonitor = new MemoryMonitor()
  const memReport = memMonitor.exportReport()
  
  return `
Performance Report
==================
Generated: ${new Date().toISOString()}

CPU
---
Usage: ${metrics.cpu.usage.toFixed(1)}%
Cores: ${metrics.cpu.cores}

Memory
------
${memReport}

Application
-----------
Uptime: ${formatDuration(metrics.application.uptime * 1000)}
Requests: ${metrics.application.requestCount}
Errors: ${metrics.application.errorCount}
Avg Response: ${metrics.application.avgResponseTime.toFixed(2)}ms
`
}

function formatDuration(ms: number): string {
  const seconds = Math.floor(ms / 1000)
  const minutes = Math.floor(seconds / 60)
  const hours = Math.floor(minutes / 60)
  
  if (hours > 0) {
    return `${hours}h ${minutes % 60}m ${seconds % 60}s`
  }
  if (minutes > 0) {
    return `${minutes}m ${seconds % 60}s`
  }
  return `${seconds}s`
}
```

### 7. 性能优化清单

```markdown
## 启动优化
- [ ] 使用 lazy require 延迟非必要模块加载
- [ ] 启用 ESM tree shaking 移除未使用代码
- [ ] 预编译 TypeScript 为 JavaScript
- [ ] 使用 Bun runtime 替代 Node.js
- [ ] 减少同步文件系统操作
- [ ] 并行化独立的初始化任务

## 内存优化
- [ ] 使用 Map/Set 替代普通对象
- [ ] 及时清理大对象引用
- [ ] 实现 LRU cache 限制内存增长
- [ ] 避免闭包导致的内存泄漏
- [ ] 使用 Stream 处理大文件
- [ ] 定期调用 gc() (开发环境)

## CPU 优化
- [ ] 使用 memoize 缓存纯函数结果
- [ ] 将计算密集型任务移到 Worker Thread
- [ ] 使用 WebAssembly 处理复杂计算
- [ ] 避免在循环中创建对象
- [ ] 使用整数而非浮点数运算
- [ ] 预分配数组大小

## I/O 优化
- [ ] 批量读写减少 syscall 次数
- [ ] 使用异步 API 避免阻塞
- [ ] 实现连接池复用数据库连接
- [ ] 压缩网络传输数据
- [ ] 使用 CDN 缓存静态资源
- [ ] 实现请求去重和合并

## 缓存优化
- [ ] 多层缓存策略 (L1/L2/L3)
- [ ] 合理的 TTL 设置
- [ ] 缓存预热策略
- [ ] 缓存失效监听机制
- [ ] 监控缓存命中率
- [ ] 定期清理过期缓存
```

## 工作流程

### 第一步：性能基准测量 (10 分钟)

```bash
# 1. 启动性能
CLAUDE_CODE_PROFILE_STARTUP=1 bun run dev
cat ~/.claude/startup-profile.log

# 2. 内存使用
node --expose-gc -e "global.gc(); console.log(process.memoryUsage())"

# 3. CPU 使用
top -pid $(pgrep -f claude-code) -stats pid,cpu,mem,time

# 4. 磁盘 I/O
sudo dtruss -t open,read,write -p $(pgrep -f claude-code)
```

### 第二步：瓶颈识别 (15 分钟)

使用性能分析工具：
- **Chrome DevTools Performance tab**: UI 渲染分析
- **Node.js --inspect**: CPU profiling
- **clinic.js**: 自动检测性能问题
- **0x**: 火焰图生成

### 第三步：优化实施 (30 分钟)

基于瓶颈分析实施针对性优化，包括懒加载、缓存优化、内存管理、并行执行等。

## 规则

- ✅ 始终先测量再优化（不要过早优化）
- ✅ 使用启动剖析识别真正的瓶颈
- ✅ 实现多层缓存策略提高命中率
- ✅ 使用 memoize 缓存昂贵的纯函数
- ✅ 定期监控内存使用和趋势
- ❌ 不要在没有基准的情况下进行优化
- ❌ 不要为了微优化牺牲代码可读性
- ❌ 不要忘记清理缓存和释放资源
- ❌ 不要忽略垃圾回收的影响

## 输出格式

### 📊 性能基准

```markdown
**启动时间**: X ms
**内存占用**: X MB
**CPU 使用**: X%
**缓存命中率**: X%
**平均响应时间**: X ms
```

### 🔍 瓶颈分析

```markdown
**主要瓶颈**: (启动/内存/CPU/I/O/缓存)
**根本原因**: ...
**影响程度**: High/Medium/Low
**优化潜力**: 预计可提升 X%
```

### ⚡ 优化方案

```markdown
**短期优化** (1-2 天):
1. ...
2. ...

**中期优化** (1-2 周):
1. ...
2. ...

**长期优化** (1-2 月):
1. ...
2. ...
```

### 📈 预期收益

```markdown
**启动时间**: -X%
**内存占用**: -X%
**响应时间**: -X%
**吞吐量**: +X%
```

### 🧪 验证计划

```markdown
**基准测试**: ...
**负载测试**: ...
**回归测试**: ...
**监控指标**: ...
```

### 📋 监控指标

```markdown
**关键指标**:
- 启动时间
- 内存使用率
- CPU 使用率
- 缓存命中率
- 请求响应时间

**告警阈值**:
- 内存 > 80%: Warning
- 内存 > 90%: Critical
- 响应时间 > 1s: Warning
- 缓存命中率 < 50%: Warning
```

## 交付物清单

最终报告应包含：
- [ ] 性能基准测量报告
- [ ] 瓶颈分析和根因定位
- [ ] 详细的优化方案和实现代码
- [ ] 预期收益和 ROI 分析
- [ ] 验证计划和测试用例
- [ ] 持续监控方案和告警阈值
- [ ] 性能优化清单和优先级

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发性能优化系统。

## AI IDE 开发前检查

### 启动剖析系统验证
```yaml
必须检查:
  - profileCheckpoint函数定义
  - SHOULD_PROFILE标志正确设置
  - 关键检查点已埋点
  - 内存快照捕获（详细模式）

警告信号:
  - 缺少关键检查点
  - 同步性能 API 调用
  - 无采样率控制
```

### 缓存系统准备
```yaml
必须存在:
  - memoize 工具函数导入
  - LRU cache类实现
  - TTL 配置合理
  - 缓存统计监控
```

## AI IDE 开发中指导

### 实时性能监控
当分析性能时，AI IDE应该：

1. **检查点验证**
   ```typescript
   // ✅ 正确的检查点埋点
   profileCheckpoint('cli_entry')
   profileCheckpoint('main_tsx_imports_loaded')
   profileCheckpoint('ready_for_user_input')
   
   // ❌ 避免的模式
   // 忘记在关键位置埋点
   // AI IDE 应该：提示补充缺失的检查点
   ```

2. **懒加载推荐**
   ```typescript
   // AI IDE 应该识别可懒加载的模块
   const heavyModules = findImportedModules(imports)
     .filter(m => m.size > 100KB)
     .filter(m => !m.isRequiredAtStartup)
   
   // 推荐懒加载方案
   for (const mod of heavyModules) {
     suggestLazyLoading(mod)
   }
   ```

3. **缓存策略优化**
   ```typescript
   // AI IDE 应该推荐合适的缓存策略
   const cacheStrategy = {
     gitStatus: { maxAge: 5 * 60 * 1000 },      // 5 分钟
     memoryFiles: { maxAge: 60 * 60 * 1000 },   // 1 小时
     expensiveComputation: { maxAge: Infinity } // 永久缓存
   }
   
   // 根据数据特性选择策略
   ```

### 内存管理监控
AI IDE应该提供：
- **MemoryMonitor类**: 定期捕获内存快照
- **checkHealth()**: 检查内存健康状态
- **getTrend()**: 分析内存使用趋势
- **泄漏检测**: 识别持续增长的对象

## AI IDE 完成后验证

### 性能优化健康度评分卡
```yaml
启动性能 (0-10分):
  - 总启动时间 < 2 秒
  - 检查点覆盖完整
  - 瓶颈识别准确

懒加载 (0-10分):
  - 大模块懒加载
  - Feature flags 条件加载
  - 循环依赖破解

缓存优化 (0-10分):
  - memoization 正确使用
  - TTL 配置合理
  - 命中率 > 80%

内存管理 (0-10分):
  - 无内存泄漏
  - LRU cache 限制大小
  - 大对象及时清理

并行执行 (0-10分):
  - Promise.all合理使用
  - 并发限制适当
  - 批处理优化

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 启动剖析测试
test('captures all checkpoints', () => {...})
test('generates accurate report', () => {...})

# 2. 懒加载测试
test('defers heavy modules', () => {...})
test('conditional loading works', () => {...})

# 3. 缓存测试
test('memoize returns cached value', () => {...})
test('TTL expiration works', () => {...})

# 4. 内存管理测试
test('detects memory leaks', () => {...})
test('LRU evicts old entries', () => {...})

# 5. 并行执行测试
test('executes in parallel with limit', () => {...})
test('batches large datasets', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 过早优化
optimizeEverything()  // 在没有性能基准前就优化
// AI IDE 应该：建议先测量再优化

// ❌ 缓存污染
const cache = new Map()  // 无限制增长
// AI IDE 应该：建议使用 LRU cache 限制大小

// ❌ 同步阻塞
const data = JSON.parse(readFileSync(largeFile))  // 阻塞主线程
// AI IDE 应该：建议使用 await readFile() + 流式解析
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 缓存命中率低
const cache = memoize(fn, { maxAge: 1000 })  // 1 秒过期，太短
// AI IDE 应该：建议根据访问模式调整 TTL

// ⚠️ 并行度过高
await Promise.all(hugeArray.map(item => process(item)))
// 可能创建过多并发任务
// AI IDE 应该：建议使用 executeInParallel(items, processor, concurrency: 10)
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const results = []
for (const item of items) {
  results.push(await process(item))  // 串行执行
}
// AI IDE 可以建议：使用 Promise.all 或 executeInParallel
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 启动性能
- [ ] 检查点埋点完整
- [ ] 启动报告生成正确
- [ ] 瓶颈识别准确
- [ ] 优化建议可操作

### 懒加载
- [ ] 大模块懒加载
- [ ] Feature flags 条件加载
- [ ] 循环依赖破解
- [ ] 错误处理健全

### 缓存优化
- [ ] memoize 使用正确
- [ ] TTL 配置合理
- [ ] 缓存键唯一性
- [ ] 命中率可监控

### 内存管理
- [ ] MemoryMonitor 正常工作
- [ ] 泄漏检测准确
- [ ] LRU cache 限制大小
- [ ] 大对象及时清理

### 并行执行
- [ ] Promise.all 合理使用
- [ ] 并发限制适当
- [ ] 批处理优化
- [ ] 错误传播正确

## AI IDE 集成实现示例

```typescript
interface PerformanceRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (metrics: PerformanceMetrics, sourceCode: string) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: PerformanceRule[] = [
  {
    id: 'PERF-001',
    description: 'Startup time must be under 2 seconds',
    severity: 'warning',
    check: (metrics) => {
      if (metrics.startupTime > 2000) {
        return [{
          message: `Startup time ${metrics.startupTime}ms exceeds 2s threshold`,
          location: metrics.entryPoint
        }]
      }
      return []
    },
    fix: () => ({ type: 'analyze_bottlenecks' })
  },
  {
    id: 'PERF-002',
    description: 'Large modules should be lazy loaded',
    severity: 'warning',
    check: (metrics, sourceCode) => {
      const largeEagerModules = findLargeEagerModules(sourceCode)
      if (largeEagerModules.length > 0) {
        return [{
          message: `${largeEagerModules.length} large modules loaded eagerly`,
          location: largeEagerModules.map(m => m.importSite)
        }]
      }
      return []
    },
    fix: () => ({ type: 'convert_to_lazy_load' })
  },
  {
    id: 'PERF-003',
    description: 'Cache must have size limit',
    severity: 'error',
    check: (metrics, sourceCode) => {
      const unboundedCaches = findUnboundedCaches(sourceCode)
      if (unboundedCaches.length > 0) {
        return [{
          message: 'Unbounded cache may cause memory leak',
          location: unboundedCaches.map(c => c.definitionSite)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_lru_eviction' })
  },
  {
    id: 'PERF-004',
    description: 'Avoid synchronous file operations',
    severity: 'error',
    check: (metrics, sourceCode) => {
      const syncFileOps = findSyncFileOperations(sourceCode)
      if (syncFileOps.length > 0) {
        return [{
          message: 'Synchronous file operation blocks main thread',
          location: syncFileOps.map(op => op.callSite)
        }]
      }
      return []
    },
    fix: () => ({ type: 'convert_to_async' })
  },
  {
    id: 'PERF-005',
    description: 'Parallel execution should have concurrency limit',
    severity: 'warning',
    check: (metrics, sourceCode) => {
      const unlimitedParallel = findUnlimitedParallelism(sourceCode)
      if (unlimitedParallel.length > 0) {
        return [{
          message: 'Unlimited parallelism may overwhelm system',
          location: unlimitedParallel.map(p => p.promiseAllSite)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_concurrency_limit', limit: 10 })
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解性能优化规范：

1. **第一阶段**: 学习启动剖析和检查点埋点
2. **第二阶段**: 理解懒加载和条件加载模式
3. **第三阶段**: 掌握缓存策略和命中率优化
4. **第四阶段**: 实现内存监控和泄漏检测
5. **第五阶段**: 提供智能并行执行和批处理建议

通过学习路径，AI IDE可以成长为能够独立设计和审查性能优化系统的专家。

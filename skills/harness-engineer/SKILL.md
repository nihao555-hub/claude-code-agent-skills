---
name: harness-engineer
description: AI Agent Harness工程专家 - 任务编排、进度追踪、状态管理、磁盘引导、优雅关闭
aliases: ['harness', 'task-orchestrator']
argumentHint: '<任务类型> [规模]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要构建复杂的任务编排系统、实现进度追踪器、设计状态管理机制、或处理长运行任务的持久化时
---

# Harness Engineer

使用此技能当你需要进行 Harness工程设计、任务编排系统实现、或进度追踪机制开发。

## 目标

产出完整的Harness 工程方案，包括 TaskOrchestrator、ProgressTracker、StateManager、DiskBootstrap和清理注册机制。

## 核心能力

### 1. 任务类型系统 (src/Task.ts)

7 种核心任务类型：

```typescript
export type TaskType =
  | 'local_bash'           // 本地 Shell 执行 (前缀：'b')
  | 'local_agent'          // 本地 Agent 推理 (前缀：'a')
  | 'remote_agent'         // 远程 Agent 调用 (前缀：'r')
  | 'in_process_teammate'  // 进程内队友 (前缀：'t')
  | 'local_workflow'       // 本地工作流 (前缀：'w')
  | 'monitor_mcp'          // MCP 监控 (前缀：'m')
  | 'dream'                // 创意生成 (前缀：'d')
```

**任务 ID 生成算法**:
```typescript
const TASK_ID_PREFIXES: Record<TaskType, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz'

export function generateTaskId(type: TaskType): string {
  const prefix = TASK_ID_PREFIXES[type] || 'x'
  const bytes = randomBytes(8)
  let id = prefix
  
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  
  return id  // 例如："a1x5k9p2q"
}
```

**设计原理**:
- **字母数字混合**: 36^8 ≈ 2.8万亿种组合
- **类型前缀**: 一眼识别任务类型
- **URL 安全**: 无特殊字符，可直接用于文件路径和 URL

### 2. 任务状态机

```typescript
export type TaskStatus =
  | 'pending'     // 已创建，未开始
  | 'running'     // 执行中
  | 'completed'   // 成功完成
  | 'failed'      // 执行失败
  | 'killed'      // 被用户终止

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

**状态流转图**:
```
pending → running → completed  (成功路径)
              ↓
              ├→ failed        (错误)
              └→ killed        (用户终止)
              
terminal states: completed, failed, killed (不可逆转)
```

### 3. 任务状态基础结构

```typescript
export type TaskStateBase = {
  id: string                    // 唯一任务 ID
  type: TaskType                // 任务类型
  status: TaskStatus            // 当前状态
  description: string           // 人类可读的描述
  toolUseId?: string            // 关联的 Tool Use ID
  startTime: number             // 开始时间戳 (ms)
  endTime?: number              // 结束时间戳 (ms)
  totalPausedMs?: number        // 暂停总时长 (ms)
  outputFile: string            // 输出文件路径 (JSONL)
  outputOffset: number          // 上次读取的偏移量
  notified: boolean             // 是否已发送完成通知
}
```

### 4. 进度追踪器 (src/tasks/LocalAgentTask/LocalAgentTask.tsx)

```typescript
export type ProgressTracker = {
  toolUseCount: number                      // 工具使用次数
  latestInputTokens: number                 // 最新输入 token 数
  cumulativeOutputTokens: number            // 累计输出 token 数
  recentActivities: ToolActivity[]          // 最近的活动记录
}

export type ToolActivity = {
  toolName: string                          // 工具名称
  input: Record<string, unknown>            // 工具输入参数
  activityDescription?: string              // 活动描述
  isSearch?: boolean                        // 是否是搜索操作
  isRead?: boolean                          // 是否是读取操作
}

const MAX_RECENT_ACTIVITIES = 5  // 最多保留 5 条最近活动
```

**进度更新逻辑**:
```typescript
export function updateProgressFromMessage(
  tracker: ProgressTracker,
  message: {...},
  resolveActivityDescription?: (...),
  tools?: unknown
): void {
  if (message.type !== 'assistant' || !message.message) return
  
  const usage = message.message.usage
  if (usage) {
    tracker.latestInputTokens = 
      usage.input_tokens + 
      (usage.cache_creation_input_tokens ?? 0) + 
      (usage.cache_read_input_tokens ?? 0)
    tracker.cumulativeOutputTokens += usage.output_tokens
  }
  
  // 处理工具使用
  if (message.message.content) {
    for (const content of message.message.content) {
      if (content.type === 'tool_use' && content.name) {
        tracker.toolUseCount++
        tracker.recentActivities.push({
          toolName: content.name,
          input: content.input as Record<string, unknown>,
          activityDescription: resolveActivityDescription?.(content.name, content.input),
          isSearch: false,
          isRead: false
        })
      }
    }
  }
  
  // 保持最近 N 条活动
  while (tracker.recentActivities.length > MAX_RECENT_ACTIVITIES) {
    tracker.recentActivities.shift()
  }
}
```

### 5. 任务编排器 (TaskOrchestrator)

完整的任务编排类，支持：
- `registerTask(task)` - 注册新任务
- `updateTaskStatus(taskId, updates)` - 更新任务状态
- `getTask(taskId)` / `getAllTasks()` - 获取任务
- `getTaskProgress(taskId)` - 获取任务进度
- `updateTaskProgress(taskId, message)` - 更新任务进度
- `abortTask(taskId)` - 中止任务
- `cleanupCompletedTasks(maxAge)` - 清理已完成的任务

### 6. 状态管理器 (StateManager)

```typescript
type AppStateUpdater = (prev: unknown) => unknown

class StateManager {
  private state: unknown = {}
  private listeners: Set<(state: unknown) => void> = new Set()
  
  getState(): unknown { return this.state }
  
  setState(updater: AppStateUpdater): void {
    const prevState = this.state
    this.state = updater(prevState)
    this.notifyListeners()
  }
  
  subscribe(listener: (state: unknown) => void): () => void {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }
}
```

### 7. 磁盘引导系统 (Disk Bootstrap)

```typescript
export type DiskBootstrapOptions = {
  taskId: string
  outputPath: string
  offset?: number
}

export type DiskBootstrapResult = {
  messages: Array<{role: string; content: ...}>
  offset: number
  metadata?: {toolUseCount: number; tokenCount: number}
}

// JSONL 文件格式：每行一个消息对象
export async function loadFromDisk(options: DiskBootstrapOptions): Promise<DiskBootstrapResult>
export async function appendToDisk(outputPath: string, message: unknown): Promise<void>
export async function initTaskOutput(taskId: string): Promise<string>
```

### 8. 清理注册表 (CleanupRegistry)

```typescript
type CleanupFn = () => void | Promise<void>

class CleanupRegistry {
  private cleanups: CleanupFn[] = []
  
  register(fn: CleanupFn): void {
    this.cleanups.push(fn)
  }
  
  async cleanup(): Promise<void> {
    const errors: Error[] = []
    for (const fn of this.cleanups) {
      try { await fn() }
      catch (error) { errors.push(error instanceof Error ? error : new Error(String(error))) }
    }
    this.cleanups = []
    if (errors.length > 0) {
      console.error('[CleanupRegistry] Errors during cleanup:', errors)
    }
  }
}

export const cleanupRegistry = new CleanupRegistry()
export function registerCleanup(fn: CleanupFn): void {
  cleanupRegistry.register(fn)
}
```

### 9. 优雅关闭系统

```typescript
export function setupGracefulShutdown(): void {
  const signals = ['SIGINT', 'SIGTERM', 'SIGQUIT']
  for (const signal of signals) {
    process.on(signal, async () => {
      console.log(`[GracefulShutdown] Received ${signal}, cleaning up...`)
      await cleanupRegistry.cleanup()
      process.exit(0)
    })
  }
}

export function gracefulShutdownSync(): void {
  console.log('[GracefulShutdown] Sync shutdown')
}
```

## 工作流程

### 第一步：现有系统审计 (15 分钟)

1. **检查任务管理系统**
   ```bash
   # 查找所有任务相关文件
   find . -name "*ask*" -type f | grep -E "\.(ts|tsx)$"
   
   # 查看任务状态流转
   grep -r "TaskStatus" src/ --include="*.ts" -A 3
   
   # 查找进度追踪实现
   grep -r "ProgressTracker" src/ --include="*.ts" -B 2 -A 5
   ```

2. **评估要点**
   - 是否有统一的任务编排器？
   - 进度追踪是否实时准确？
   - 状态管理是否可预测？
   - 磁盘持久化是否可靠？
   - 清理机制是否完善？

### 第二步：Harness架构设计 (20 分钟)

基于 Claude Code最佳实践设计你的Harness系统，包括任务编排器、进度追踪器、状态管理器、磁盘引导系统和清理注册表。

### 第三步：实现与验证 (25 分钟)

#### 实现检查清单

- [ ] **任务编排器**
  - [ ] 任务注册和状态管理
  - [ ] 进度追踪集成
  - [ ] AbortController支持
  - [ ] 事件发射器（EventEmitter）
  - [ ] 自动清理过期任务

- [ ] **进度追踪器**
  - [ ] Token计数（input/output/cache）
  - [ ] 工具使用计数
  - [ ] 活动记录和分类
  - [ ] 活动描述生成
  - [ ] 最大活动数量限制

- [ ] **状态管理器**
  - [ ] 不可变状态更新
  - [ ] 订阅/通知机制
  - [ ] 错误隔离（listener错误不影响其他listener）

- [ ] **磁盘引导**
  - [ ] JSONL 文件读写
  - [ ] Offset 追踪
  - [ ] 目录自动创建
  - [ ] 原子写入（flush:true）

- [ ] **清理注册**
  - [ ] 同步和异步清理函数支持
  - [ ] 错误收集和报告
  - [ ] 优雅关闭信号处理

## 规则

- ✅ 使用类型安全的 TaskType和TaskStatus
- ✅ 实现幂等的任务状态转换
- ✅ 为所有异步操作提供超时保护
- ✅ 在磁盘 IO 中使用 atomic writes
- ✅ 为长运行任务实现 checkpointing
- ❌ 不要在任务状态中存储大型数据（使用 outputFile 引用）
- ❌ 不要忘记清理 AbortControllers（内存泄漏）
- ❌ 不要阻塞 event loop（使用 setImmediate yield）
- ❌ 不要忽略并发竞态条件（使用适当的锁机制）

## 输出格式

### 📊 现状评估

```markdown
## 任务编排成熟度
评分：X/10

### 优势
- ...

### 不足
- ...

### 紧急问题
- ...
```

### 🏗️ 架构设计

使用 Mermaid 图展示任务编排流程。

### 📝 实现蓝图

提供五个核心模块的完整 TypeScript 实现代码。

### 🔍 测试计划

详细的单元测试、集成测试、压力测试计划。

### ⚠️ 风险评估

| 风险项 | 可能性 | 影响程度 | 缓解措施 |
|-------|--------|----------|----------|
| 任务状态不一致 | 中 | 高 | 实现状态机验证 |
| 内存泄漏（未清理） | 高 | 中 | 强制 cleanup 注册 |
| 磁盘写入失败 | 中 | 高 | atomic writes + retry |
| 并发竞态条件 | 低 | 高 | 使用队列序列化 |

## 交付物清单

最终报告应包含：
- [ ] Harness 系统现状评估
- [ ] 架构设计图（Mermaid 流程图）
- [ ] 五个核心模块的完整实现代码
- [ ] 测试计划和测试用例
- [ ] 风险评估和缓解措施
- [ ] 性能基准和优化建议
- [ ] 迁移指南（如果重构现有系统）

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发 Harness工程系统。

## AI IDE 开发前检查

### 任务类型系统验证
```yaml
必须检查:
  - TaskType枚举是否定义所有 7种类型
  - taskId前缀映射是否正确 (b/a/r/t/w/m/d)
  - ID生成算法是否使用加密安全随机数

警告信号:
  - 缺少任务类型定义
  - 前缀冲突风险
  - 使用Math.random()而非 crypto.randomBytes()
```

### 状态机完整性
```yaml
必须存在:
  - pending → running → completed路径
  - running → failed错误处理
  - running → killed用户终止
  - isTerminalTaskStatus判断函数
```

## AI IDE 开发中指导

### 实时任务编排监控
当创建和管理任务时，AI IDE应该：

1. **任务 ID 生成验证**
   ```typescript
   // ✅ 正确的 ID 生成
   const id = generateTaskId('local_agent')  // 返回 "a1x5k9p2q"
   
   // ❌ 避免的模式
   const id = `task_${Date.now()}`  // 可预测、无类型前缀
   // AI IDE 应该：建议使用 generateTaskId()
   ```

2. **状态流转守护**
   ```typescript
   // AI IDE 应该阻止非法状态转换
   const validTransitions = {
     pending: ['running'],
     running: ['completed', 'failed', 'killed'],
     completed: [],  // terminal
     failed: [],     // terminal
     killed: []      // terminal
   }
   
   if (!validTransitions[current].includes(next)) {
     throw new Error(`Invalid transition: ${current} → ${next}`)
   }
   ```

3. **进度追踪优化**
   ```typescript
   // AI IDE 应该推荐合适的更新频率
   const UPDATE_THROTTLE = 1000  // 1 秒
   const MAX_ACTIVITIES = 5      // 保留最近 5 条
   
   // 避免过度更新导致 UI 卡顿
   ```

### 磁盘引导系统
AI IDE应该提供：
- **JSONL 格式验证**: 确保每行是有效的 JSON
- **Offset 追踪**: 记录上次读取位置避免重复
- **原子写入**: 使用 flush:true 确保数据落盘
- **目录自动创建**: mkdir({recursive: true})

## AI IDE 完成后验证

### Harness 健康度评分卡
```yaml
任务编排 (0-10分):
  - ID 生成唯一性
  - 状态机完整性
  - 并发控制正确性
  
进度追踪 (0-10分):
  - Token计数准确性
  - 活动记录及时性
  - 节流机制合理性

状态管理 (0-10分):
  - 不可变更新模式
  - 订阅通知机制
  - 错误隔离处理

磁盘持久化 (0-10分):
  - JSONL 格式正确性
  - Offset追踪可靠性
  - 原子写入保证

清理机制 (0-10分):
  - CleanupRegistry注册
  - 优雅关闭处理
  - 资源释放完整性

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 任务 ID 测试
test('generates unique IDs', () => {...})
test('prefix matches task type', () => {...})

# 2. 状态机测试
test('allows valid transitions', () => {...})
test('blocks invalid transitions', () => {...})

# 3. 进度追踪测试
test('counts tokens correctly', () => {...})
test('limits recent activities', () => {...})

# 4. 磁盘持久化测试
test('writes atomically', () => {...})
test('recovers from offset', () => {...})

# 5. 清理机制测试
test('runs all cleanup functions', () => {...})
test('handles cleanup errors gracefully', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 状态机绕过
task.status = 'completed'  // 直接修改，跳过验证!
// AI IDE 应该：建议使用 updateTaskStatus() 函数

// ❌ 内存泄漏
const controllers = new Map()  // 从未清理 AbortController
// AI IDE 应该：建议在任务完成时删除 controller

// ❌ 非原子写入
writeFileSync(path, data)  // 可能写一半崩溃
// AI IDE 应该：建议使用 fd + flush 模式

// ❌ 阻塞主线程
const result = execSync('long-running-command')
// AI IDE 应该：建议使用 await exec()
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 无限增长
activities.push(activity)  // 从未清理
// AI IDE 应该：建议添加 shift()限制大小

// ⚠️ 缺少超时
await longOperation()  // 可能永远等待
// AI IDE 应该：建议添加 AbortSignal.timeout()
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const task = tasks.get(id)
if (task) { task.status = 'running' }
// AI IDE 可以建议：使用 immer 进行不可变更新
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 任务编排
- [ ] 使用 generateTaskId()生成 ID
- [ ] 任务类型前缀正确
- [ ] 状态流转符合状态机
- [ ] Terminal状态不可逆转

### 进度追踪
- [ ] Token计数包括缓存
- [ ] 活动记录有限制
- [ ] 节流避免过度更新
- [ ] 活动描述有人类可读

### 状态管理
- [ ] 使用 setState(updater)模式
- [ ] Listener错误不影响其他
- [ ] 订阅者可以取消订阅
- [ ] 状态变更有通知

### 磁盘持久化
- [ ] JSONL 格式每行有效 JSON
- [ ] Offset 追踪避免重复
- [ ] 原子写入防止损坏
- [ ] 目录自动创建

### 清理机制
- [ ] 所有资源注册到 cleanupRegistry
- [ ] 优雅关闭信号处理
- [ ] 错误不中断其他清理
- [ ] 定期清理已完成任务

## AI IDE 集成实现示例

```typescript
interface HarnessRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (task: TaskState, orchestrator: TaskOrchestrator) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: HarnessRule[] = [
  {
    id: 'HARNESS-001',
    description: 'Must use generateTaskId for ID generation',
    severity: 'error',
    check: (task) => {
      if (!/^[bartwmd][0-9a-z]{8}$/.test(task.id)) {
        return [{
          message: `Invalid task ID format: ${task.id}`,
          location: task.idDefinition
        }]
      }
      return []
    },
    fix: () => ({ type: 'regenerate', generator: 'generateTaskId' })
  },
  {
    id: 'HARNESS-002',
    description: 'State transitions must be valid',
    severity: 'error',
    check: (task, orchestrator) => {
      const valid = isValidTransition(task.status, task.nextStatus)
      if (!valid) {
        return [{
          message: `Invalid transition: ${task.status} → ${task.nextStatus}`,
          location: task.transitionPoint
        }]
      }
      return []
    }
  },
  {
    id: 'HARNESS-003',
    description: 'Activities must be bounded',
    severity: 'warning',
    check: (task) => {
      if (task.recentActivities.length > 10) {
        return [{
          message: 'Too many recent activities, may cause memory issues',
          location: task.activitiesArray
        }]
      }
      return []
    },
    fix: () => ({ type: 'truncate', maxSize: 5 })
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解 Harness 工程规范：

1. **第一阶段**: 学习任务类型系统和 ID 生成算法
2. **第二阶段**: 理解状态机和流转规则
3. **第三阶段**: 掌握进度追踪和 token 计算
4. **第四阶段**: 实现磁盘持久化和 offset 管理
5. **第五阶段**: 提供智能清理和资源管理建议

通过学习路径，AI IDE可以成长为能够独立设计和审查 Harness 工程系统的专家。

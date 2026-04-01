---
name: state-management-engineer
description: AI Agent 状态管理专家 - 不可变状态、状态订阅、状态持久化、乐观更新、撤销重做
aliases: ['状态管理', 'state-expert']
argumentHint: '<状态类型> [持久化策略]'
allowedTools: FileReadTool, GrepTool, GlobTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要设计复杂的状态管理系统、实现状态持久化、构建撤销重做功能、或处理并发状态更新时
---

# State Management Engineer

使用此技能当你需要进行状态管理系统设计、实现不可变状态模式、构建状态订阅机制、或处理状态持久化。

## 目标

产出完整的状态管理方案，包括不可变状态设计、状态订阅系统、状态持久化策略、乐观更新和撤销重做机制。

## 核心能力（基于 Claude Code源码）

### 1. 不可变状态模式 (src/bootstrap/state.ts)

```typescript
// Claude Code的AppState管理模式
type AppStateUpdater = (prev: AppState) => AppState

class StateManager {
  private state: AppState = initialState
  private listeners = new Set<(state: AppState) => void>()
  
  getState(): AppState {
    return this.state
  }
  
  setState(updater: AppStateUpdater): void {
    const prevState = this.state
    // 创建新对象而非修改原对象
    const nextState = updater(prevState)
    
    // 浅比较检测是否真的变化
    if (prevState === nextState) {
      return  // 无变化，不通知
    }
    
    this.state = nextState
    
    // 通知所有订阅者
    this.notifyListeners()
  }
  
  subscribe(listener: (state: AppState) => void): () => void {
    this.listeners.add(listener)
    // 返回取消订阅函数
    return () => this.listeners.delete(listener)
  }
  
  private notifyListeners(): void {
    const currentState = this.state
    for (const listener of this.listeners) {
      try {
        listener(currentState)
      } catch (error) {
        console.error('State listener error:', error)
        // 一个监听器错误不影响其他监听器
      }
    }
  }
}

// 使用示例
const unsubscribe = stateManager.subscribe((state) => {
  console.log('State changed:', state.turnCount)
})

// 稍后取消订阅
unsubscribe()
```

### 2. 状态切片选择器 (src/utils/stateSelectors.ts)

```typescript
// 从大状态中选择小部分，避免不必要的重渲染
type Selector<TState, TSelected> = (state: TState) => TSelected

function createSelector<TState, TSelected>(
  selector: Selector<TState, TSelected>,
  equalityFn: (a: TSelected, b: TSelected) => boolean = Object.is
): Selector<TState, TSelected> {
  let lastState: TState | null = null
  let lastSelected: TSelected | null = null
  
  return (state: TState): TSelected => {
    if (
      lastState === state &&
      lastSelected !== null &&
      equalityFn(lastSelected, selector(state))
    ) {
      return lastSelected
    }
    
    const selected = selector(state)
    lastState = state
    lastSelected = selected
    return selected
  }
}

// 使用示例
const selectTurnCount = createSelector(
  (state: AppState) => state.turnCount,
  (a, b) => a === b  // 数字比较
)

const selectUserSettings = createSelector(
  (state: AppState) => state.settings.user,
  (a, b) => JSON.stringify(a) === JSON.stringify(b)  // 深度比较
)
```

### 3. 状态持久化 (src/utils/sessionStorage.ts)

```typescript
interface PersistedState {
  version: number
  timestamp: number
  data: Partial<AppState>
}

class StatePersister {
  private readonly STORAGE_KEY = 'claude_code_state'
  private readonly VERSION = 1
  
  async save(state: AppState): Promise<void> {
    const persisted: PersistedState = {
      version: this.VERSION,
      timestamp: Date.now(),
      data: {
        // 只持久化需要的字段
        settings: state.settings,
        preferences: state.preferences,
        history: state.history.slice(-100)  // 只保留最近 100 条
      }
    }
    
    try {
      const serialized = JSON.stringify(persisted)
      await writeFile(this.getStoragePath(), serialized, 'utf-8')
    } catch (error) {
      console.error('Failed to persist state:', error)
    }
  }
  
  async load(): Promise<Partial<AppState> | null> {
    try {
      const content = await readFile(this.getStoragePath(), 'utf-8')
      const persisted: PersistedState = JSON.parse(content)
      
      // 版本检查
      if (persisted.version !== this.VERSION) {
        console.warn('State version mismatch, migrating...')
        return this.migrate(persisted)
      }
      
      // 过期检查（7 天）
      const age = Date.now() - persisted.timestamp
      if (age > 7 * 24 * 60 * 60 * 1000) {
        console.info('Persisted state expired')
        return null
      }
      
      return persisted.data
    } catch (error) {
      if ((error as NodeJS.ErrnoException).code !== 'ENOENT') {
        console.error('Failed to load persisted state:', error)
      }
      return null
    }
  }
  
  private migrate(persisted: PersistedState): Partial<AppState> {
    // 版本迁移逻辑
    switch (persisted.version) {
      case 0:
        // v0 → v1 迁移
        return {
          ...persisted.data,
          newField: defaultValue
        }
      default:
        return persisted.data
    }
  }
}
```

### 4. 乐观更新 (src/utils/optimisticUpdates.ts)

```typescript
interface OptimisticUpdate<T> {
  id: string
  previousValue: T
  newValue: T
  rollback: () => Promise<void>
  commit: () => Promise<void>
}

class OptimisticStateManager {
  private pendingUpdates = new Map<string, OptimisticUpdate<any>>()
  
  async updateWithRollback<T>(
    updateId: string,
    getter: () => T,
    setter: (value: T) => void | Promise<void>,
    serverUpdate: () => Promise<T>
  ): Promise<void> {
    const previousValue = getter()
    
    // 1. 立即应用本地更新（乐观）
    const optimisticValue = computeOptimisticValue(previousValue)
    setter(optimisticValue)
    
    // 2. 发送到服务器
    try {
      const confirmedValue = await serverUpdate()
      
      // 3. 服务器确认，用真实值替换乐观值
      setter(confirmedValue)
      this.pendingUpdates.delete(updateId)
    } catch (error) {
      // 4. 失败，回滚到之前的值
      setter(previousValue)
      this.pendingUpdates.delete(updateId)
      throw error
    }
  }
}
```

### 5. 撤销/重做系统 (src/utils/undoRedo.ts)

```typescript
interface HistoryEntry<T> {
  state: T
  timestamp: number
  description?: string
}

class UndoRedoManager<T> {
  private past: HistoryEntry<T>[] = []
  private present: T
  private future: HistoryEntry<T>[] = []
  private readonly MAX_HISTORY = 100
  
  constructor(initialState: T) {
    this.present = initialState
  }
  
  commit(newState: T, description?: string): void {
    // 保存当前状态到历史
    this.past.push({
      state: this.present,
      timestamp: Date.now(),
      description
    })
    
    // 限制历史记录大小
    if (this.past.length > this.MAX_HISTORY) {
      this.past.shift()
    }
    
    // 清空未来历史（有了新的分支）
    this.future = []
    
    // 更新当前状态
    this.present = newState
  }
  
  undo(): T | null {
    if (this.past.length === 0) {
      return null  // 无法撤销
    }
    
    // 保存当前到未来
    this.future.push({
      state: this.present,
      timestamp: Date.now()
    })
    
    // 恢复到过去的状态
    const previous = this.past.pop()!
    this.present = previous.state
    
    return this.present
  }
  
  redo(): T | null {
    if (this.future.length === 0) {
      return null  // 无法重做
    }
    
    // 保存当前到过去
    this.past.push({
      state: this.present,
      timestamp: Date.now()
    })
    
    // 恢复到未来的状态
    const next = this.future.pop()!
    this.present = next.state
    
    return this.present
  }
  
  canUndo(): boolean {
    return this.past.length > 0
  }
  
  canRedo(): boolean {
    return this.future.length > 0
  }
  
  clearHistory(): void {
    this.past = []
    this.future = []
  }
}
```

### 6. 并发状态更新 (src/utils/concurrentState.ts)

```typescript
interface PendingUpdate {
  version: number
  updates: Array<() => void>
}

class ConcurrentStateManager {
  private version = 0
  private pendingUpdates: PendingUpdate[] = []
  
  batchUpdate(updater: (state: AppState) => AppState): void {
    this.version++
    
    // 将更新加入待处理队列
    this.pendingUpdates.push({
      version: this.version,
      updates: [() => stateManager.setState(updater)]
    })
    
    // 微任务时机批量执行
    queueMicrotask(() => this.flushUpdates())
  }
  
  private flushUpdates(): void {
    if (this.pendingUpdates.length === 0) {
      return
    }
    
    // 合并同一批次的所有更新
    const allUpdates = this.pendingUpdates.flatMap(p => p.updates)
    this.pendingUpdates = []
    
    // 批量执行
    stateManager.setState((state) => {
      for (const update of allUpdates) {
        update()
      }
      return state
    })
  }
}
```

## 工作流程

### 第一步：状态结构设计 (15 分钟)

1. **识别状态类型**
   ```typescript
   type AppState = {
     // 持久化状态
     settings: UserSettings
     preferences: UserPreferences
     
     // 会话状态
     session: SessionInfo
     turnCount: number
     
     // UI 状态
     ui: {
       sidebarOpen: boolean
       theme: 'light' | 'dark'
       activePanel: string
     }
     
     // 缓存状态
     cache: {
       gitStatus?: GitStatus
       context?: ContextData
     }
   }
   ```

2. **定义状态变更操作**
   ```typescript
   type StateActions = {
     SET_SETTING: { key: string; value: any }
     START_TURN: { turnId: string }
     END_TURN: { turnId: string; result: TurnResult }
     TOGGLE_SIDEBAR: void
     SWITCH_THEME: 'light' | 'dark'
   }
   ```

### 第二步：状态管理系统实现 (25 分钟)

1. **创建 StateManager**
2. **实现选择器**
3. **添加持久化**
4. **集成撤销重做**

### 第三步：性能优化 (15 分钟)

1. **记忆化选择器**
2. **批量更新**
3. **懒加载状态**

## 规则

- ✅ 状态更新必须不可变
- ✅ 使用选择器避免不必要的重渲染
- ✅ 重要状态必须持久化
- ✅ 提供撤销重做功能
- ✅ 错误处理中也要清理状态
- ❌ 不要直接修改状态对象
- ❌ 不要在状态中存储派生数据
- ❌ 不要忘记取消订阅
- ❌ 不要持久化临时状态

## 输出格式

### 📊 状态结构图

```markdown
## AppState 结构

```
AppState
├── settings (persisted)
│   ├── theme
│   ├── language
│   └── keybindings
├── session (ephemeral)
│   ├── id
│   ├── startTime
│   └── turnCount
├── ui (ephemeral)
│   ├── sidebarOpen
│   └── activePanel
└── cache (memoized)
    ├── gitStatus
    └── context
```
```

### 🏗️ 状态流转图

使用状态机图展示状态变迁。

### 📝 实现代码

提供完整的 StateManager、选择器、持久化实现。

### 🔍 测试计划

状态更新测试、持久化测试、撤销重做测试计划。

### ⚠️ 风险评估

| 风险项 | 可能性 | 影响程度 | 缓解措施 |
|-------|--------|----------|----------|
| 状态泄漏 | 中 | 高 | 使用 WeakMap、及时清理 |
| 内存增长 | 中 | 中 | 限制历史大小、定期清理 |
| 持久化失败 | 低 | 高 | 降级到内存、用户提示 |

## 交付物清单

最终报告应包含：
- [ ] 状态结构设计和类型定义
- [ ] StateManager完整实现
- [ ] 状态选择器系统
- [ ] 持久化机制
- [ ] 撤销重做系统
- [ ] 性能优化方案
- [ ] 测试计划和测试用例

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发状态管理系统。

## AI IDE 开发前检查

### 状态结构验证
```yaml
必须检查:
  - 状态类型定义完整
  - 区分持久化/临时状态
  - 初始状态定义正确
  - 状态更新操作明确

警告信号:
  - 状态嵌套过深（>3层）
  - 循环引用风险
  - 缺少必要的状态字段
```

### 状态管理基础设施
```yaml
必须存在:
  - StateManager类
  - setState方法
  - subscribe方法
  - getState方法
```

## AI IDE 开发中指导

### 实时状态管理监控
当编写状态相关代码时，AI IDE应该：

1. **不可变性检查**
   ```typescript
   // ✅ 正确的不可变更新
   stateManager.setState(prev => ({
     ...prev,
     user: { ...prev.user, name: newName }
   }))
   
   // ❌ 错误的可变更新
   state.user.name = newName  // 直接修改！
   // AI IDE 应该：立即警告并使用 setState
   
   // ✅ 使用 Immer（如果项目支持）
   stateManager.setState(produce((draft) => {
     draft.user.name = newName
   }))
   ```

2. **选择器优化**
   ```typescript
   // AI IDE 应该推荐使用选择器
   const name = useSelector(selectUserName)
   
   // 而不是直接访问整个状态
   const state = getState()  // 会导致不必要的重渲染
   const name = state.user.name
   ```

3. **订阅清理**
   ```typescript
   // AI IDE 应该提醒清理订阅
   useEffect(() => {
     const unsubscribe = store.subscribe(handleChange)
     return unsubscribe  // 组件卸载时清理
   }, [])
   ```

### 状态持久化管理
AI IDE应该提供：
- **自动保存**: 定时或事件触发保存
- **版本迁移**: 状态结构变更时的迁移逻辑
- **过期清理**: 定期清理过期的持久化数据
- **错误恢复**: 持久化失败时的降级策略

## AI IDE 完成后验证

### 状态管理健康度评分卡
```yaml
不可变性 (0-10分):
  - 无直接修改
  - 使用扩展或 Immer
  - 冻结开发环境状态

选择器 (0-10分):
  - 使用记忆化选择器
  - 自定义相等比较
  - 避免重复计算

持久化 (0-10分):
  - 关键状态已保存
  - 版本迁移正确
  - 错误处理完善

性能 (0-10分):
  - 批量更新启用
  - 无内存泄漏
  - 订阅及时清理

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 不可变性测试
test('does not mutate previous state', () => {...})
test('creates new object references', () => {...})

# 2. 选择器测试
test('memoizes correctly', () => {...})
test('uses custom equality', () => {...})

# 3. 持久化测试
test('saves to disk', () => {...})
test('loads from disk', () => {...})
test('migrates versions', () => {...})

# 4. 订阅测试
test('notifies listeners', () => {...})
test('cleans up on unsubscribe', () => {...})

# 5. 撤销重做测试
test('undo restores previous', () => {...})
test('redo reapplies', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 直接修改状态
state.settings.theme = 'dark'
// AI IDE 应该：立即警告并使用 setState

// ❌ 忘记清理订阅
useEffect(() => {
  store.subscribe(handler)
  // 没有返回清理函数
}, [])
// AI IDE 应该：提醒添加 return unsubscribe

// ❌ 在状态中存储派生数据
state.totalCount = items.length  // 应该动态计算
// AI IDE 应该：建议使用选择器计算
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 选择器没有记忆化
const selected = state.items.filter(...)  // 每次都重新计算
// AI IDE 应该：建议使用 createSelector

// ⚠️ 频繁的小更新
setState(s => ({ count: s.count + 1 }))
setState(s => ({ count: s.count + 1 }))
// AI IDE 应该：建议批量更新
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const count = state.counter.value
// AI IDE 可以建议：使用选择器提升可维护性
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 不可变性
- [ ] 无直接属性赋值
- [ ] 使用扩展运算符或 Immer
- [ ] 数组操作使用不可变方法
- [ ] 嵌套对象正确处理

### 选择器
- [ ] 使用记忆化选择器
- [ ] 自定义相等比较函数
- [ ] 避免在渲染函数中创建新对象
- [ ] 依赖项正确声明

### 持久化
- [ ] 关键状态已配置持久化
- [ ] 版本迁移逻辑完整
- [ ] 错误处理完善
- [ ] 隐私数据不持久化

### 订阅管理
- [ ] 订阅后清理
- [ ] 使用取消订阅函数
- [ ] 避免重复订阅
- [ ] 错误隔离

### 性能
- [ ] 批量更新启用
- [ ] 避免不必要的状态更新
- [ ] 懒加载大对象
- [ ] 内存泄漏检测

## AI IDE 集成实现示例

```typescript
interface StateManagementRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (sourceCode: string, stateUsages: StateUsage[]) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: StateManagementRule[] = [
  {
    id: 'STATE-001',
    description: 'State must be updated immutably',
    severity: 'error',
    check: (sourceCode) => {
      const directMutations = findDirectMutations(sourceCode)
      if (directMutations.length > 0) {
        return [{
          message: 'Direct state mutation detected',
          location: directMutations.map(m => m.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'convert_to_immutable' })
  },
  {
    id: 'STATE-002',
    description: 'Subscriptions must be cleaned up',
    severity: 'error',
    check: (sourceCode) => {
      const unclearedSubscriptions = findUnclearedSubscriptions(sourceCode)
      if (unclearedSubscriptions.length > 0) {
        return [{
          message: 'Subscription without cleanup',
          location: unclearedSubscriptions.map(s => s.subscriptionSite)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_cleanup_return' })
  },
  {
    id: 'STATE-003',
    description: 'Use selectors for derived state',
    severity: 'warning',
    check: (sourceCode) => {
      const derivedInComponent = findDerivedStateInComponents(sourceCode)
      if (derivedInComponent.length > 0) {
        return [{
          message: 'Derived state should use selector',
          location: derivedInComponent.map(d => d.location)
        }]
      }
      return []
    }
  },
  {
    id: 'STATE-004',
    description: 'Batch frequent state updates',
    severity: 'info',
    check: (sourceCode) => {
      const rapidUpdates = findRapidSequentialUpdates(sourceCode)
      if (rapidUpdates.length > 3) {
        return [{
          message: 'Consider batching these updates',
          location: rapidUpdates.map(u => u.updateSite)
        }]
      }
      return []
    }
  },
  {
    id: 'STATE-005',
    description: 'Do not persist sensitive data',
    severity: 'error',
    check: (sourceCode) => {
      const sensitiveInPersist = findSensitiveDataInPersistence(sourceCode)
      if (sensitiveInPersist.length > 0) {
        return [{
          message: 'Sensitive data should not be persisted',
          location: sensitiveInPersist.map(s => s.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'exclude_from_persistence' })
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解状态管理规范：

1. **第一阶段**: 学习不可变状态原则和更新模式
2. **第二阶段**: 理解选择器和记忆化优化
3. **第三阶段**: 掌握状态持久化和版本迁移
4. **第四阶段**: 实现撤销重做和历史管理
5. **第五阶段**: 提供智能性能分析和优化建议

通过学习路径，AI IDE可以成长为能够独立设计和审查状态管理系统的专家。

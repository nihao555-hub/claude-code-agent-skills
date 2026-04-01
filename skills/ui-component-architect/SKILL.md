---
name: ui-component-architect
description: UI 组件架构专家 - React 组件设计、状态管理、性能优化、可访问性、视觉回归测试
aliases: ['UI 架构师', 'ui-dev']
argumentHint: '<组件类型> [复杂度]'
allowedTools: FileReadTool, GrepTool, GlobTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要设计 React 组件架构、实现复杂状态管理、优化渲染性能、或确保可访问性合规时
---

# UI Component Architect

使用此技能当你需要进行 React 组件架构设计、实现复杂 UI 状态管理、优化渲染性能、确保可访问性合规、或建立视觉回归测试流程。

## 目标

产出完整的 UI 组件架构方案，包括组件设计模式、状态管理策略、性能优化技术、可访问性指南和测试策略。

## 核心能力（基于 Claude Code源码）

### 1. React 组件设计模式

#### Compound Components Pattern
```typescript
// 基于 Claude Code的 Message 组件系统
interface MessageContextValue {
  messageId: string
  role: 'user' | 'assistant'
  isExpanded: boolean
}

const MessageContext = createContext<MessageContextValue | null>(null)

function Message({ children, messageId, role }: MessageProps) {
  const [isExpanded, setIsExpanded] = useState(true)
  
  const contextValue = useMemo(() => ({
    messageId,
    role,
    isExpanded,
  }), [messageId, role, isExpanded])
  
  return (
    <MessageContext.Provider value={contextValue}>
      <div className="message" data-role={role}>
        {children}
      </div>
    </MessageContext.Provider>
  )
}

function MessageHeader() {
  const context = useContext(MessageContext)
  if (!context) throw new Error('MessageHeader must be used within Message')
  
  return (
    <header className="message-header">
      <span>{context.role}</span>
      <button onClick={() => {/* toggle expand */}}>
        {context.isExpanded ? '▼' : '▶'}
      </button>
    </header>
  )
}

function MessageContent({ children }: { children: ReactNode }) {
  const context = useContext(MessageContext)
  if (!context) throw new Error('MessageContent must be used within Message')
  
  if (!context.isExpanded) return null
  
  return (
    <div className="message-content">
      {children}
    </div>
  )
}

// 使用方式
<Message messageId="123" role="assistant">
  <MessageHeader />
  <MessageContent>
    <Markdown content={messageContent} />
  </MessageContent>
</Message>
```

#### State Reducer Pattern
```typescript
type MessageAction = 
  | { type: 'EXPAND' }
  | { type: 'COLLAPSE' }
  | { type: 'TOGGLE' }
  | { type: 'SET_CONTENT'; content: string }

interface MessageState {
  isExpanded: boolean
  content: string
  isLoading: boolean
}

function messageReducer(state: MessageState, action: MessageAction): MessageState {
  switch (action.type) {
    case 'EXPAND':
      return { ...state, isExpanded: true }
    
    case 'COLLAPSE':
      return { ...state, isExpanded: false }
    
    case 'TOGGLE':
      return { ...state, isExpanded: !state.isExpanded }
    
    case 'SET_CONTENT':
      return { ...state, content: action.content }
    
    default:
      return state
  }
}

function MessageWithReducer({ initialState }: MessageProps) {
  const [state, dispatch] = useReducer(messageReducer, initialState)
  
  return (
    <MessageContext.Provider value={{ state, dispatch }}>
      {/* children */}
    </MessageContext.Provider>
  )
}
```

#### Control Props Pattern
```typescript
interface AccordionProps {
  expandedIndex?: number  // controlled
  defaultExpandedIndex?: number  // uncontrolled
  onChange?: (index: number) => void
  children: ReactNode
}

function Accordion({
  expandedIndex: controlledIndex,
  defaultExpandedIndex = 0,
  onChange,
  children,
}: AccordionProps) {
  const [uncontrolledIndex, setUncontrolledIndex] = useState(defaultExpandedIndex)
  
  // Determine if controlled or uncontrolled
  const isControlled = controlledIndex !== undefined
  const expandedIndex = isControlled ? controlledIndex : uncontrolledIndex
  
  const handleChange = useCallback((index: number) => {
    if (!isControlled) {
      setUncontrolledIndex(index)
    }
    onChange?.(index)
  }, [isControlled, onChange])
  
  return (
    <AccordionContext.Provider value={{ expandedIndex, onChange: handleChange }}>
      {children}
    </AccordionContext.Provider>
  )
}
```

### 2. 高性能渲染优化

#### Virtual Scrolling
```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualMessageList({ messages, containerRef }: MessageListProps) {
  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => containerRef.current,
    estimateSize: () => 100,  // 估计每个项目高度
    overscan: 5,  // 预渲染前后各 5 个项目
  })
  
  const virtualItems = virtualizer.getVirtualItems()
  
  return (
    <div
      ref={containerRef}
      style={{
        height: '600px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualItems.map((virtualRow) => (
          <Message
            key={messages[virtualRow.index]!.id}
            message={messages[virtualRow.index]!}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualRow.start}px)`,
            }}
          />
        ))}
      </div>
    </div>
  )
}
```

#### Memoization Strategy
```typescript
// Expensive computation memoization
const MemoizedMarkdown = memo(function Markdown({ content }: { content: string }) {
  // Parse markdown (expensive)
  const parsed = useMemo(() => parseMarkdown(content), [content])
  
  // Syntax highlight code blocks (very expensive)
  const highlighted = useMemo(() => {
    return highlightCodeBlocks(parsed)
  }, [parsed])
  
  return <div dangerouslySetInnerHTML={{ __html: highlighted }} />
}, (prevProps, nextProps) => {
  // Custom comparison
  return prevProps.content === nextProps.content
})

// Event handler memoization
function MessageActions({ messageId, onArchive, onDelete }: ActionsProps) {
  const handleArchive = useCallback(() => {
    onArchive(messageId)
  }, [messageId, onArchive])
  
  const handleDelete = useCallback(() => {
    onDelete(messageId)
  }, [messageId, onDelete])
  
  return (
    <>
      <button onClick={handleArchive}>Archive</button>
      <button onClick={handleDelete}>Delete</button>
    </>
  )
}
```

#### Windowing for Large Lists
```typescript
function FixedSizeList({
  height,
  itemCount,
  itemSize,
  renderItem,
}: FixedSizeListProps) {
  const [scrollTop, setScrollTop] = useState(0)
  
  const visibleStart = Math.floor(scrollTop / itemSize)
  const visibleEnd = Math.min(
    itemCount,
    visibleStart + Math.ceil(height / itemSize)
  )
  
  const items = []
  for (let i = visibleStart; i < visibleEnd; i++) {
    items.push(
      <div
        key={i}
        style={{
          position: 'absolute',
          top: i * itemSize,
          left: 0,
          width: '100%',
          height: itemSize,
        }}
      >
        {renderItem(i)}
      </div>
    )
  }
  
  return (
    <div
      style={{ height, overflow: 'auto' }}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ position: 'relative', height: itemCount * itemSize }}>
        {items}
      </div>
    </div>
  )
}
```

### 3. 状态管理策略

#### Zustand for Global State
```typescript
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

interface AppState {
  // Session state
  sessionId: string
  turnCount: number
  
  // UI state
  sidebarOpen: boolean
  theme: 'light' | 'dark'
  
  // Actions
  actions: {
    setSidebarOpen: (open: boolean) => void
    toggleTheme: () => void
    incrementTurn: () => void
  }
}

export const useAppStore = create<AppState>()(
  subscribeWithSelector((set, get) => ({
    // Initial state
    sessionId: generateSessionId(),
    turnCount: 0,
    sidebarOpen: true,
    theme: 'light',
    
    // Actions
    actions: {
      setSidebarOpen: (open) => set({ sidebarOpen: open }),
      
      toggleTheme: () => set((state) => ({
        theme: state.theme === 'light' ? 'dark' : 'light'
      })),
      
      incrementTurn: () => set((state) => ({
        turnCount: state.turnCount + 1
      })),
    },
  }))
)

// Usage in components
function Sidebar() {
  const { sidebarOpen, actions } = useAppStore()
  
  return (
    <aside className={sidebarOpen ? 'open' : 'closed'}>
      <button onClick={() => actions.setSidebarOpen(!sidebarOpen)}>
        Toggle
      </button>
    </aside>
  )
}
```

#### Jotai for Atomic State
```typescript
import { atom, useAtom } from 'jotai'

// Atomic state
const messageIdsAtom = atom<string[]>([])
const messagesMapAtom = atom<Map<string, Message>>(new Map())

// Derived state
const messageCountAtom = atom(
  (get) => get(messageIdsAtom).length
)

// Async state
const currentUserAtom = atom(async () => {
  return fetchCurrentUser()
})

// Write-only atom
const addMessageAtom = atom(
  null,
  (get, set, message: Message) => {
    set(messageIdsAtom, [...get(messageIdsAtom), message.id])
    set(messagesMapAtom, new Map([
      ...get(messagesMapAtom),
      [message.id]: message
    ]))
  }
)

function MessageList() {
  const [messageIds] = useAtom(messageIdsAtom)
  const [messagesMap] = useAtom(messagesMapAtom)
  const [, addMessage] = useAtom(addMessageAtom)
  
  return (
    <div>
      {messageIds.map(id => (
        <MessageCard key={id} message={messagesMap.get(id)!} />
      ))}
    </div>
  )
}
```

### 4. 可访问性（A11y）实现

#### Semantic HTML & ARIA
```typescript
interface DialogProps {
  isOpen: boolean
  onClose: () => void
  title: string
  children: ReactNode
}

function Dialog({ isOpen, onClose, title, children }: DialogProps) {
  // Trap focus inside dialog
  useEffect(() => {
    if (!isOpen) return
    
    const dialog = document.querySelector('[role="dialog"]')
    const focusableElements = dialog?.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    )
    
    const firstFocusable = focusableElements?.[0]
    const lastFocusable = focusableElements?.[focusableElements.length - 1]
    
    function handleTabKey(e: KeyboardEvent) {
      if (e.key !== 'Tab') return
      
      if (e.shiftKey && document.activeElement === firstFocusable) {
        e.preventDefault()
        lastFocusable?.focus()
      } else if (!e.shiftKey && document.activeElement === lastFocusable) {
        e.preventDefault()
        firstFocusable?.focus()
      }
    }
    
    document.addEventListener('keydown', handleTabKey)
    return () => document.removeEventListener('keydown', handleTabKey)
  }, [isOpen])
  
  if (!isOpen) return null
  
  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby="dialog-title"
      className="dialog-overlay"
      onClick={(e) => e.target === e.currentTarget && onClose()}
    >
      <div className="dialog-content">
        <h2 id="dialog-title">{title}</h2>
        <button
          aria-label="Close dialog"
          onClick={onClose}
          className="close-button"
        >
          ×
        </button>
        {children}
      </div>
    </div>
  )
}
```

#### Keyboard Navigation
```typescript
function Menu({ items, onSelect }: MenuProps) {
  const [focusedIndex, setFocusedIndex] = useState(0)
  
  function handleKeyDown(e: React.KeyboardEvent) {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault()
        setFocusedIndex((prev) => (prev + 1) % items.length)
        break
      
      case 'ArrowUp':
        e.preventDefault()
        setFocusedIndex((prev) => (prev - 1 + items.length) % items.length)
        break
      
      case 'Enter':
      case ' ':
        e.preventDefault()
        onSelect(items[focusedIndex])
        break
      
      case 'Escape':
        onClose?.()
        break
    }
  }
  
  return (
    <ul
      role="menu"
      onKeyDown={handleKeyDown}
    >
      {items.map((item, index) => (
        <li
          key={item.id}
          role="menuitem"
          tabIndex={index === focusedIndex ? 0 : -1}
          aria-focused={index === focusedIndex}
          className={index === focusedIndex ? 'focused' : ''}
          onClick={() => onSelect(item)}
        >
          {item.label}
        </li>
      ))}
    </ul>
  )
}
```

### 5. 视觉回归测试

#### Percy Integration
```typescript
import percySnapshot from '@percy/playwright'
import { test, expect } from '@playwright/test'

test.describe('Message Component', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/messages')
  })
  
  test('renders user message correctly', async ({ page }) => {
    const message = page.getByTestId('message-user')
    await expect(message).toBeVisible()
    
    await percySnapshot(page, 'User Message - Default')
  })
  
  test('renders assistant message with markdown', async ({ page }) => {
    const message = page.getByTestId('message-assistant')
    await expect(message).toContainText('# Hello')
    
    await percySnapshot(page, 'Assistant Message - Markdown')
  })
  
  test('handles long content gracefully', async ({ page }) => {
    await page.evaluate(() => {
      window.postMessage({
        type: 'ADD_MESSAGE',
        content: 'A'.repeat(10000)
      })
    })
    
    await percySnapshot(page, 'Message - Long Content')
  })
})
```

#### Chromatic Integration
```typescript
// .storybook/main.ts
module.exports = {
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@chromatic-com/storybook',
  ],
}

// Message.stories.tsx
export default {
  component: Message,
  title: 'Components/Message',
  decorators: [
    (Story) => (
      <div style={{ maxWidth: '800px', margin: '0 auto' }}>
        <Story />
      </div>
    ),
  ],
}

export const UserMessage = {
  args: {
    role: 'user',
    content: 'Hello, world!',
  },
}

export const AssistantMessageWithCode = {
  args: {
    role: 'assistant',
    content: `Here's some code:
\`\`\`typescript
const greeting = 'Hello!'
console.log(greeting)
\`\`\``,
  },
}

export const LoadingState = {
  args: {
    role: 'assistant',
    isLoading: true,
  },
}
```

## 工作流程

### 第一步：组件设计 (20 分钟)

1. **选择设计模式**
   - Compound Components: 用于复杂组合
   - State Reducer: 用于灵活状态控制
   - Control Props: 用于可控制的组件

2. **定义接口**
   ```typescript
   interface ComponentProps {
     // Required props
     children: ReactNode
     
     // Optional props
     className?: string
     
     // Controlled props
     value?: T
     defaultValue?: T
     onChange?: (value: T) => void
   }
   ```

### 第二步：实现优化 (40 分钟)

1. **添加 Memoization**
2. **实现虚拟滚动**
3. **优化事件处理**
4. **添加键盘导航**

### 第三步：测试验证 (20 分钟)

1. **单元测试**
2. **视觉回归测试**
3. **可访问性测试**
4. **性能基准测试**

## 规则

- ✅ 使用语义化 HTML
- ✅ 实现完整的键盘导航
- ✅ 添加适当的 ARIA 属性
- ✅ 使用 Memoization 优化渲染
- ✅ 实现错误边界
- ❌ 不要忽略可访问性
- ❌ 不要在渲染中进行昂贵计算
- ❌ 不要忘记清理副作用
- ❌ 不要直接修改 props

## 输出格式

### 📊 组件架构图

展示组件层次结构和数据流。

### 📝 实现代码

提供完整的组件实现，包括优化和测试。

### 🔍 测试报告

包含视觉回归测试结果和可访问性审计报告。

### ⚠️ 性能分析

| 指标 | 优化前 | 优化后 | 改进 |
|------|--------|--------|------|
| 首次渲染 | X ms | Y ms | -Z% |
| 重渲染次数 | X | Y | -Z% |
| FPS | X | Y | +Z% |

## 交付物清单

最终报告应包含：
- [ ] 组件设计文档
- [ ] 完整实现代码
- [ ] Storybook故事
- [ ] 视觉回归测试配置
- [ ] 可访问性审计报告
- [ ] 性能优化报告
- [ ] 使用文档

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发 UI 组件系统。

## AI IDE 开发前检查

### 设计系统验证
```yaml
必须检查:
  - Design Token定义完整
  - 颜色系统一致
  - 间距系统统一
  - 字体系统清晰

警告信号:
  - 硬编码的颜色值
  - 不一致的间距
  - 混用的单位 (px/rem/em)
```

### 可访问性准备
```yaml
必须存在:
  - 键盘导航支持
  - Focus管理
  - ARIA标签
  - 屏幕阅读器测试
```

## AI IDE 开发中指导

### 实时组件质量监控
当开发 UI 组件时，AI IDE应该：

1. **设计模式检查**
   ```typescript
   // ✅ 良好的复合组件模式
   <Message>
     <Message.Header />
     <Message.Content />
   </Message>
   
   // ❌ 避免的模式
   <Message showHeader={true} headerPosition="top" />
   // AI IDE 应该：建议使用复合组件模式
   ```

2. **性能优化检查**
   ```typescript
   // ✅ 正确的memoization
   const ExpensiveComponent = memo(({ data }) => {
     const processed = useMemo(() => process(data), [data])
     return <div>{processed}</div>
   }, (prev, next) => prev.data === next.data)
   
   // ❌ 避免的模式
   function ExpensiveComponent({ data }) {
     const processed = process(data)  // 每次都重新计算!
     return <div>{processed}</div>
   }
   // AI IDE 应该：建议添加 useMemo 和 memo
   ```

3. **可访问性检查**
   ```typescript
   // ✅ 完整的键盘支持
   function Menu({ items }) {
     const [focused, setFocused] = useState(0)
     
     return (
       <ul role="menu" onKeyDown={handleKeyDown}>
         {items.map((item, i) => (
           <li
             key={item.id}
             role="menuitem"
             tabIndex={i === focused ? 0 : -1}
           >
             {item.label}
           </li>
         ))}
       </ul>
     )
   }
   
   // ❌ 缺少键盘支持
   function Menu({ items }) {
     return (
       <div>
         {items.map(item => <div onClick={select}>{item}</div>)}
       </div>
     )
   }
   // AI IDE 应该：建议添加键盘导航和ARIA
   ```

### 视觉回归自动化
AI IDE应该提供：
- **自动截图**: 在 PR 时自动运行 Percy/Chromatic
- **差异检测**: 高亮显示视觉变化
- **基线管理**: 轻松更新视觉基线
- **审批流程**: 设计师审查视觉变化

## AI IDE 完成后验证

### UI 组件健康度评分卡
```yaml
设计模式 (0-10分):
  - 模式选择合适
  - 组合性强
  - 可扩展性好

性能 (0-10分):
  - 渲染效率高
  - 内存占用低
  - 无内存泄漏

可访问性 (0-10分):
  - WCAG 2.1 AA合规
  - 键盘导航完整
  - 屏幕阅读器友好

测试覆盖 (0-10分):
  - 单元测试完整
  - 视觉回归正常
  - E2E测试通过

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (不建议发布)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 单元测试
npm run test:unit -- components/

# 2. 视觉回归测试
npm run test:visual

# 3. 可访问性测试
npm run test:a11y

# 4. 性能测试
npm run test:performance
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 缺少错误边界
function App() {
  return <ChildComponent />  // 子组件错误会导致整个应用崩溃
}
// AI IDE 应该：建议添加 ErrorBoundary

// ❌ 内存泄漏
useEffect(() => {
  const subscription = eventBus.subscribe(handler)
  // 没有返回清理函数
}, [])
// AI IDE 应该：提醒添加清理逻辑

// ❌ 直接修改props
function Child({ list }) {
  list.push(newItem)  // 直接修改props!
  return <div>{list}</div>
}
// AI IDE 应该：立即警告并使用不可变更新
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 过度渲染
function Parent() {
  const [count, setCount] = useState(0)
  return <Child data={largeData} />  // count变化导致Child重渲染
}
// AI IDE 应该：建议拆分状态或使用memo

// ⚠️ 内联对象创建
function Component() {
  return <Child style={{ margin: 10 }} />  // 每次渲染创建新对象
}
// AI IDE 应该：建议提取到组件外或使用useMemo
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const className = isActive ? 'active' : 'inactive'
// AI IDE 可以建议：使用classnames库处理复杂条件
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 设计模式
- [ ] 选择了合适的组件模式
- [ ] 组合性优于配置
- [ ] 关注点分离清晰
- [ ] 单一职责原则

### 性能优化
- [ ] 使用React.memo优化
- [ ] useMemo/useCallback正确
- [ ] 虚拟滚动实现
- [ ] 无不必要的重渲染

### 可访问性
- [ ] 语义化HTML
- [ ] ARIA属性完整
- [ ] 键盘导航支持
- [ ] Focus管理正确
- [ ] 颜色对比度达标

### 测试
- [ ] 单元测试覆盖
- [ ] 视觉回归基线
- [ ] 可访问性测试
- [ ] 跨浏览器测试

## AI IDE 集成实现示例

```typescript
interface UIComponentRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (componentFile: SourceFile) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: UIComponentRule[] = [
  {
    id: 'UI-001',
    description: 'Interactive elements must have keyboard support',
    severity: 'error',
    check: (file) => {
      const interactiveWithoutKeyboard = findInteractiveElementsWithoutKeyboard(file)
      if (interactiveWithoutKeyboard.length > 0) {
        return [{
          message: 'Interactive element without keyboard support',
          location: interactiveWithoutKeyboard.map(e => e.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_keyboard_handler' })
  },
  {
    id: 'UI-002',
    description: 'Must use semantic HTML elements',
    severity: 'error',
    check: (file) => {
      const nonSemanticDivs = findNonSemanticDivs(file)
      if (nonSemanticDivs.length > 0) {
        return [{
          message: 'Consider using semantic HTML instead of div',
          location: nonSemanticDivs.map(d => d.location)
        }]
      }
      return []
    }
  },
  {
    id: 'UI-003',
    description: 'Expensive computations should be memoized',
    severity: 'warning',
    check: (file) => {
      const unmemoizedComputations = findUnmemoizedComputations(file)
      if (unmemoizedComputations.length > 0) {
        return [{
          message: 'Expensive computation not memoized',
          location: unmemoizedComputations.map(c => c.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'wrap_with_usememo' })
  },
  {
    id: 'UI-004',
    description: 'Effects must clean up subscriptions',
    severity: 'error',
    check: (file) => {
      const effectsWithoutCleanup = findEffectsWithoutCleanup(file)
      if (effectsWithoutCleanup.length > 0) {
        return [{
          message: 'useEffect missing cleanup function',
          location: effectsWithoutCleanup.map(e => e.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_cleanup_return' })
  },
  {
    id: 'UI-005',
    description: 'Images must have alt text',
    severity: 'error',
    check: (file) => {
      const imagesWithoutAlt = findImagesWithoutAlt(file)
      if (imagesWithoutAlt.length > 0) {
        return [{
          message: 'Image missing alt attribute',
          location: imagesWithoutAlt.map(img => img.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_alt_attribute' })
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解 UI 组件开发规范：

1. **第一阶段**: 学习 React 设计模式和最佳实践
2. **第二阶段**: 理解性能优化技术和应用场景
3. **第三阶段**: 掌握可访问性标准和实现方法
4. **第四阶段**: 实现视觉回归测试和工作流
5. **第五阶段**: 提供智能代码审查和优化建议

通过学习路径，AI IDE可以成长为能够独立设计和审查 UI 组件系统的专家。

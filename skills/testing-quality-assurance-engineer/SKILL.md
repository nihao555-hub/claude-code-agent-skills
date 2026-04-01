---
name: testing-quality-assurance-engineer
description: AI Agent 测试质量保证专家 - 单元测试、集成测试、E2E 测试、测试覆盖率、质量门禁
aliases: ['测试工程师', 'qa-expert']
argumentHint: '<测试类型> [覆盖率目标]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要设计测试策略、编写高质量测试用例、建立质量门禁、或提升测试覆盖率时
---

# Testing & Quality Assurance Engineer

使用此技能当你需要进行测试策略设计、编写测试用例、建立质量门禁系统、或提升代码质量和测试覆盖率。

## 目标

产出完整的测试和质量保证方案，包括单元测试、集成测试、E2E 测试策略、测试覆盖率分析和质量门禁系统。

## 核心能力

### 1. 单元测试最佳实践

```typescript
// 基于 Claude Code的测试模式
describe('TaskManager', () => {
  let taskManager: TaskManager
  let mockState: MockState
  
  beforeEach(() => {
    mockState = createMockState()
    taskManager = new TaskManager(mockState)
  })
  
  afterEach(() => {
    jest.restoreAllMocks()
  })
  
  describe('createTask', () => {
    it('should create task with unique id', async () => {
      // Arrange
      const taskType = 'local_bash'
      const description = 'Test task'
      
      // Act
      const task = await taskManager.createTask(taskType, description)
      
      // Assert
      expect(task.id).toMatch(/^[a-z][0-9a-z]{8}$/)
      expect(task.type).toBe(taskType)
      expect(task.status).toBe('pending')
    })
    
    it('should emit task:created event', async () => {
      // Arrange
      const eventHandler = jest.fn()
      taskManager.on('task:created', eventHandler)
      
      // Act
      await taskManager.createTask('local_bash', 'Test')
      
      // Assert
      expect(eventHandler).toHaveBeenCalledTimes(1)
      expect(eventHandler).toHaveBeenCalledWith(
        expect.objectContaining({
          type: 'local_bash',
          status: 'pending'
        })
      )
    })
    
    it('should handle concurrent task creation', async () => {
      // Arrange
      const tasks = await Promise.all(
        Array.from({ length: 10 }).map(() => 
          taskManager.createTask('local_bash', 'Concurrent task')
        )
      )
      
      // Assert - all IDs should be unique
      const ids = tasks.map(t => t.id)
      expect(new Set(ids).size).toBe(tasks.length)
    })
  })
})
```

### 2. 集成测试模式

```typescript
// MCP 服务器集成测试
describe('MCP Integration', () => {
  let mcpServer: MCPServer
  let testClient: MCPClient
  
  beforeAll(async () => {
    // 启动真实的 MCP 服务器
    mcpServer = await startTestMCPServer({
      port: findAvailablePort()
    })
    
    // 连接到服务器
    testClient = await connectMCPClient(mcpServer.url)
  }, 30000) // 增加超时时间
  
  afterAll(async () => {
    // 清理资源
    await testClient.disconnect()
    await mcpServer.shutdown()
  })
  
  it('should list available tools', async () => {
    // Act
    const tools = await testClient.listTools()
    
    // Assert
    expect(tools).toHaveLength(expectedToolCount)
    expect(tools.map(t => t.name)).toEqual([
      'file_read',
      'file_write',
      'bash',
      // ... expected tools
    ])
  })
  
  it('should execute tool and return result', async () => {
    // Act
    const result = await testClient.callTool('bash', {
      command: 'echo "Hello World"'
    })
    
    // Assert
    expect(result.content).toContain('Hello World')
    expect(result.exitCode).toBe(0)
  }, 10000)
})
```

### 3. E2E 测试框架

```typescript
// Puppeteer/Playwright E2E测试
describe('Agent E2E Flow', () => {
  let browser: Browser
  let page: Page
  
  beforeAll(async () => {
    browser = await puppeteer.launch({
      headless: process.env.CI ? true : false,
      slowMo: 50 // CI 环境下加速
    })
  })
  
  afterAll(async () => {
    await browser.close()
  })
  
  beforeEach(async () => {
    page = await browser.newPage()
    
    // 设置视口
    await page.setViewport({ width: 1280, height: 720 })
    
    // 导航到应用
    await page.goto('http://localhost:3000', {
      waitUntil: 'networkidle0'
    })
  })
  
  afterEach(async () => {
    await page.close()
  })
  
  it('should complete full conversation flow', async () => {
    // Step 1: 输入用户消息
    await page.type('[data-testid="chat-input"]', 'Hello, can you help me?')
    await page.click('[data-testid="send-button"]')
    
    // Step 2: 等待 Agent 响应
    await page.waitForSelector('[data-testid="assistant-message"]', {
      timeout: 30000
    })
    
    // Step 3: 验证响应显示
    const messages = await page.$$('[data-testid="message"]')
    expect(messages.length).toBe(2) // 用户 + 助手
    
    // Step 4: 截图对比（视觉回归测试）
    const screenshot = await page.screenshot()
    expect(screenshot).toMatchImageSnapshot({
      customSnapshotsDir: '__snapshots__/e2e'
    })
  }, 60000)
  
  it('should handle tool execution visualization', async () => {
    // 发送需要执行工具的消息
    await page.type('[data-testid="chat-input"]', 'List files in current directory')
    await page.click('[data-testid="send-button"]')
    
    // 等待工具执行 UI
    await page.waitForSelector('[data-testid="tool-execution"]', {
      timeout: 30000
    })
    
    // 验证工具执行进度显示
    const progressBars = await page.$$('[data-testid="progress-bar"]')
    expect(progressBars.length).toBeGreaterThan(0)
    
    // 等待完成
    await page.waitForSelector('[data-testid="tool-result"]', {
      timeout: 30000
    })
    
    // 验证结果显示
    const result = await page.$eval(
      '[data-testid="tool-result"]',
      el => el.textContent
    )
    expect(result).toBeTruthy()
  }, 60000)
})
```

### 4. 测试覆盖率分析

```typescript
interface CoverageReport {
  total: {
    statements: number
    branches: number
    functions: number
    lines: number
  }
  covered: {
    statements: number
    branches: number
    functions: number
    lines: number
  }
  coverage: {
    statements: number  // percentage
    branches: number
    functions: number
    lines: number
  }
  files: Array<{
    path: string
    coverage: number
    uncoveredLines: number[]
  }>
}

function analyzeCoverage(report: CoverageReport): CoverageAnalysis {
  const gaps: string[] = []
  
  // 识别覆盖率低的文件
  for (const file of report.files) {
    if (file.coverage < 80) {
      gaps.push(`${file.path}: ${file.coverage}% (${file.uncoveredLines.length} uncovered lines)`)
    }
  }
  
  // 计算加权平均
  const weightedAverage = calculateWeightedAverage(report.files)
  
  return {
    overall: weightedAverage,
    gaps,
    recommendations: generateRecommendations(gaps)
  }
}

function generateRecommendations(gaps: string[]): string[] {
  const recommendations: string[] = []
  
  if (gaps.length > 0) {
    recommendations.push('Add tests for uncovered files')
  }
  
  const criticalFiles = gaps.filter(g => 
    g.includes('src/core/') || g.includes('src/utils/')
  )
  
  if (criticalFiles.length > 0) {
    recommendations.push('Prioritize core module coverage')
  }
  
  return recommendations
}
```

### 5. 质量门禁系统

```typescript
interface QualityGate {
  name: string
  check: () => Promise<boolean>
  threshold?: number
  critical: boolean
}

class QualityGateKeeper {
  private gates: QualityGate[] = [
    {
      name: 'Test Coverage',
      check: async () => {
        const coverage = await getCoverage()
        return coverage.lines >= 80
      },
      threshold: 80,
      critical: true
    },
    {
      name: 'No Console Errors',
      check: async () => {
        const errors = await getConsoleErrors()
        return errors.length === 0
      },
      critical: false
    },
    {
      name: 'Bundle Size',
      check: async () => {
        const size = await getBundleSize()
        return size < 5 * 1024 * 1024 // 5MB
      },
      threshold: 5 * 1024 * 1024,
      critical: false
    },
    {
      name: 'Performance Budget',
      check: async () => {
        const metrics = await measurePerformance()
        return metrics.fcp < 1500 && metrics.lcp < 2500
      },
      critical: true
    }
  ]
  
  async runAllGates(): Promise<QualityGateResult[]> {
    const results: QualityGateResult[] = []
    
    for (const gate of this.gates) {
      try {
        const passed = await gate.check()
        results.push({
          name: gate.name,
          passed,
          critical: gate.critical
        })
      } catch (error) {
        results.push({
          name: gate.name,
          passed: false,
          error: error instanceof Error ? error.message : 'Unknown error',
          critical: gate.critical
        })
      }
    }
    
    return results
  }
  
  shouldBlockDeployment(results: QualityGateResult[]): boolean {
    return results.some(r => r.critical && !r.passed)
  }
}
```

### 6. 测试数据工厂

```typescript
// Faker.js风格的测试数据生成
class TestDataFactory {
  static task(type: TaskType = 'local_bash'): Task {
    return {
      id: generateTaskId(type),
      type,
      status: 'pending',
      description: `Test task ${randomId()}`,
      startTime: Date.now(),
      outputFile: `/tmp/test-${randomId()}.jsonl`,
      outputOffset: 0,
      notified: false
    }
  }
  
  static agent(name?: string): Agent {
    return {
      agentId: name || `test-agent-${randomId()}`,
      agentType: 'coding-assistant',
      whenToUse: 'For coding tasks',
      model: 'claude-sonnet-4-5-20260101',
      tools: ['file_read', 'file_write', 'bash'],
      disallowedTools: []
    }
  }
  
  static message(role: 'user' | 'assistant' = 'user'): Message {
    return {
      role,
      content: [{
        type: 'text',
        text: `Test message ${randomId()}`
      }],
      timestamp: Date.now()
    }
  }
  
  static error(code: ErrorCode = 'VALIDATION_ERROR'): ClaudeError {
    return createError(code, `Test error ${randomId()}`, {
      severity: 'medium',
      recoverable: false
    })
  }
}

// 使用示例
const testTask = TestDataFactory.task('local_agent')
const testAgent = TestDataFactory.agent('my-test-agent')
const testError = TestDataFactory.error('NETWORK_ERROR')
```

## 工作流程

### 第一步：测试策略设计 (20 分钟)

1. **识别测试金字塔层次**
   ```
   E2E Tests (10%)
       ↑
   Integration Tests (20%)
       ↑
   Unit Tests (70%)
   ```

2. **确定测试优先级**
   ```typescript
   const testPriority = {
     P0: '核心业务逻辑 - 必须测试',
     P1: '公共 API - 应该测试',
     P2: '工具函数 - 建议测试',
     P3: 'UI 组件 - 可选测试'
   }
   ```

### 第二步：测试实现 (40 分钟)

1. **编写单元测试**
2. **编写集成测试**
3. **编写 E2E 测试**
4. **配置覆盖率检查**

### 第三步：质量门禁配置 (15 分钟)

1. **定义质量指标**
2. **设置阈值**
3. **配置 CI/CD集成**

## 规则

- ✅ 测试应该是确定性的
- ✅ 每个测试只验证一个行为
- ✅ 使用有意义的测试名称
- ✅ 测试应该快速执行
- ✅ 保持测试独立性
- ❌ 不要测试实现细节
- ❌ 不要在测试之间共享状态
- ❌ 不要跳过失败的测试
- ❌ 不要忘记清理测试资源

## 输出格式

### 📊 测试覆盖率报告

```markdown
## 覆盖率总结

| 类型 | 覆盖率 | 目标 | 状态 |
|------|--------|------|------|
| Statements | 85% | 80% | ✅ |
| Branches | 78% | 75% | ✅ |
| Functions | 92% | 85% | ✅ |
| Lines | 86% | 80% | ✅ |

## 未覆盖的关键文件
- src/core/auth.ts (45%)
- src/utils/parser.ts (62%)
```

### 🏗️ 测试架构图

展示测试层次和依赖关系。

### 📝 测试用例清单

提供完整的测试用例列表。

### 🔍 质量门禁结果

```markdown
## 质量门禁检查结果

- [x] Test Coverage >= 80%
- [x] No Console Errors
- [x] Bundle Size < 5MB
- [x] Performance Budget Met
- [ ] TypeScript Errors (2 errors found)
```

### ⚠️ 风险评估

| 风险项 | 可能性 | 影响程度 | 缓解措施 |
|-------|--------|----------|----------|
| 覆盖率虚高 | 中 | 中 | 人工审查关键路径 |
| 测试脆弱 | 高 | 低 | 避免测试实现细节 |
| CI 时间长 | 中 | 中 | 并行执行、缓存依赖 |

## 交付物清单

最终报告应包含：
- [ ] 测试策略文档
- [ ] 单元测试套件
- [ ] 集成测试套件
- [ ] E2E 测试套件
- [ ] 测试覆盖率报告
- [ ] 质量门禁配置
- [ ] CI/CD集成指南
- [ ] 测试数据工厂

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发测试和质量保证系统。

## AI IDE 开发前检查

### 测试环境验证
```yaml
必须检查:
  - 测试框架已安装 (Jest/Vitest)
  - 测试配置文件存在
  - 模拟数据工具可用
  - CI/CD管道配置

警告信号:
  - 缺少测试依赖
  - 测试脚本未定义
  - 覆盖率报告未配置
```

### 测试基础设施
```yaml
必须存在:
  - 测试运行器
  - 断言库
  - Mock 工具
  - 覆盖率工具
```

## AI IDE 开发中指导

### 实时测试质量监控
当编写测试代码时，AI IDE应该：

1. **测试结构检查**
   ```typescript
   // ✅ 良好的测试结构
   describe('UserService', () => {
     describe('createUser', () => {
       it('should create user with valid data', async () => {
         // Arrange
         // Act
         // Assert
       })
       
       it('should throw on duplicate email', async () => {
         // ...
       })
     })
   })
   
   // ❌ 糟糕的测试结构
   test('works', async () => { /* ... */ })
   // AI IDE 应该：建议使用描述性名称和 AAA 模式
   ```

2. **断言质量验证**
   ```typescript
   // ✅ 具体的断言
   expect(user.email).toBe('test@example.com')
   expect(user.createdAt).toBeInstanceOf(Date)
   
   // ❌ 模糊的断言
   expect(result).toBeTruthy()  // 不够具体
   // AI IDE 应该：建议使用更具体的断言
   ```

3. **测试隔离检查**
   ```typescript
   // ✅ 隔离的测试
   beforeEach(() => {
     mockRepository.reset()
   })
   
   // ❌ 共享状态的测试
   let sharedState = {}  // 测试间共享！
   // AI IDE 应该：警告并在每个测试前重置
   ```

### 测试覆盖率优化
AI IDE应该提供：
- **覆盖率热点图**: 可视化未覆盖的代码
- **智能测试建议**: 基于代码分析推荐测试用例
- **回归测试选择**: 基于变更选择相关测试
- **性能分析**: 识别慢测试并优化

## AI IDE 完成后验证

### 测试质量健康度评分卡
```yaml
单元测试 (0-10分):
  - 覆盖率 > 80%
  - 测试速度快 (<100ms/个)
  - 断言具体明确

集成测试 (0-10分):
  - 关键路径覆盖
  - Mock 合理
  - 资源清理完整

E2E 测试 (0-10分):
  - 用户旅程完整
  - 视觉回归正常
  - 跨浏览器通过

质量门禁 (0-10分):
  - 所有门禁通过
  - 阈值合理
  - CI/CD集成完整

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
npm run test:unit -- --coverage

# 2. 集成测试
npm run test:integration

# 3. E2E 测试
npm run test:e2e

# 4. 覆盖率检查
nyc check-coverage --lines 80 --branches 75

# 5. 质量门禁
npm run quality-gate
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 测试实现细节而非行为
expect(component.state.count).toBe(5)
// AI IDE 应该：建议测试可见行为而非内部状态

// ❌ 测试之间有依赖
it('should work after previous test', () => {
  // 依赖上一个测试的状态
})
// AI IDE 应该：警告并要求测试独立

// ❌ 没有时间相关的清理
beforeEach(() => {
  jest.useFakeTimers()
  // 但没有 afterEach 恢复
})
// AI IDE 应该：提醒添加清理代码
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 测试太慢
it('should do something', async () => {
  await sleep(5000)  // 不必要的等待
}, 10000)
// AI IDE 应该：建议优化或使用 fake timers

// ⚠️ 断言不够具体
expect(result).toBeDefined()
// AI IDE 应该：建议更具体的断言
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const user = await createUser({
  name: 'John',
  email: 'john@example.com'
})
// AI IDE 可以建议：使用测试数据工厂
const user = TestDataFactory.user()
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 单元测试
- [ ] 遵循 AAA 模式 (Arrange-Act-Assert)
- [ ] 测试名称描述行为
- [ ] 断言具体明确
- [ ] 无测试间依赖
- [ ] 清理资源完整

### 集成测试
- [ ] Mock 外部依赖
- [ ] 测试真实交互
- [ ] 错误处理覆盖
- [ ] 超时配置合理

### E2E 测试
- [ ] 用户旅程完整
- [ ] 等待策略正确
- [ ] 截图对比配置
- [ ] 跨浏览器测试

### 覆盖率
- [ ] 行覆盖率 >= 80%
- [ ] 分支覆盖率 >= 75%
- [ ] 关键路径 100% 覆盖
- [ ] 新增代码有测试

### 质量门禁
- [ ] 所有关键门禁通过
- [ ] 性能预算符合
- [ ] 无 TypeScript 错误
- [ ] 无 ESLint 警告

## AI IDE 集成实现示例

```typescript
interface TestingRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (testFiles: TestFile[], sourceFiles: SourceFile[]) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: TestingRule[] = [
  {
    id: 'TEST-001',
    description: 'Tests must have descriptive names',
    severity: 'warning',
    check: (testFiles) => {
      const vagueNames = findVagueTestNames(testFiles)
      if (vagueNames.length > 0) {
        return [{
          message: 'Test names should describe behavior, not implementation',
          location: vagueNames.map(n => n.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'suggest_descriptive_name' })
  },
  {
    id: 'TEST-002',
    description: 'Tests must be isolated',
    severity: 'error',
    check: (testFiles) => {
      const sharedState = findSharedStateBetweenTests(testFiles)
      if (sharedState.length > 0) {
        return [{
          message: 'Tests should not share state',
          location: sharedState.map(s => s.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_reset_before_each' })
  },
  {
    id: 'TEST-003',
    description: 'Assertions should be specific',
    severity: 'warning',
    check: (testFiles) => {
      const vagueAssertions = findVagueAssertions(testFiles)
      if (vagueAssertions.length > 0) {
        return [{
          message: 'Use specific assertions instead of toBeTruthy()',
          location: vagueAssertions.map(a => a.location)
        }]
      }
      return []
    }
  },
  {
    id: 'TEST-004',
    description: 'Tests must clean up resources',
    severity: 'error',
    check: (testFiles) => {
      const missingCleanup = findMissingCleanup(testFiles)
      if (missingCleanup.length > 0) {
        return [{
          message: 'Test does not clean up resources',
          location: missingCleanup.map(m => m.location)
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_after_each_cleanup' })
  },
  {
    id: 'TEST-005',
    description: 'New code must have tests',
    severity: 'error',
    check: (testFiles, sourceFiles) => {
      const untestedCode = findUntestedNewCode(testFiles, sourceFiles)
      if (untestedCode.length > 0) {
        return [{
          message: 'New code paths are not covered by tests',
          location: untestedCode.map(u => u.location)
        }]
      }
      return []
    }
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解测试和质量保证规范：

1. **第一阶段**: 学习单元测试原则和 AAA 模式
2. **第二阶段**: 理解集成测试和 Mock 策略
3. **第三阶段**: 掌握 E2E 测试和用户旅程设计
4. **第四阶段**: 实现覆盖率分析和质量门禁
5. **第五阶段**: 提供智能测试生成和优化建议

通过学习路径，AI IDE可以成长为能够独立设计和审查测试系统的专家。

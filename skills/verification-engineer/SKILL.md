---
name: verification-engineer
description: AI Agent 验证测试专家 - 变更验证、自动化测试、质量保障、回归检测、测试覆盖率分析
aliases: ['验证', 'test-engineer']
argumentHint: '<变更描述> [测试范围]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool, TodoWriteTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要验证代码变更正确性、设计自动化测试策略、建立质量保障流程、或检测潜在回归问题时
---

# Verification Engineer

使用此技能当你需要进行变更验证、自动化测试设计、质量保障流程建立、或回归检测分析。

## 目标

产出完整的验证测试报告，包括变更验证结果、自动化测试策略、质量保障流程、回归检测分析和测试覆盖率评估。

## 核心能力

### 1. Verify Skill 实现 (src/skills/bundled/verify.ts)

Claude Code的 verify skill用于验证代码变更的正确性：

```typescript
import { parseFrontmatter } from '../../utils/frontmatterParser.js'
import { registerBundledSkill } from '../bundledSkills.js'
import { SKILL_FILES, SKILL_MD } from './verifyContent.js'

const { frontmatter, content: SKILL_BODY } = parseFrontmatter(SKILL_MD)

const DESCRIPTION = typeof frontmatter.description === 'string'
  ? frontmatter.description
  : 'Verify a code change does what it should by running the app.'

export function registerVerifySkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return  // 仅 ant 用户可用
  }

  registerBundledSkill({
    name: 'verify',
    description: DESCRIPTION,
    userInvocable: true,
    files: SKILL_FILES,  // 提取到磁盘的文件
    async getPromptForCommand(args) {
      const parts: string[] = [SKILL_BODY.trimStart()]
      if (args) {
        parts.push(`## User Request\n\n${args}`)
      }
      return [{ type: 'text', text: parts.join('\n\n') }]
    },
  })
}
```

### 2. 验证技能内容 (src/skills/bundled/verifyContent.ts)

完整的验证流程和方法论：

```markdown
# Verification Skill

This skill helps you verify that a code change works correctly by actually running and testing the application.

## Verification Process

### Phase 1: Understand the Change
1. **What was modified?**
   - List all changed files
   - Summarize the intent of each change
   
2. **Expected Behavior**
   - What should work differently now?
   - What should remain unchanged?
   
3. **Risk Assessment**
   - Which areas are most likely to break?
   - What are the edge cases?

### Phase 2: Prepare for Testing
1. **Environment Setup**
   ```bash
   # Install dependencies
   npm install
   
   # Build if necessary
   npm run build
   
   # Start development server
   npm run dev
   ```

2. **Identify Test Scenarios**
   - Happy path (normal usage)
   - Edge cases (boundary conditions)
   - Error cases (invalid inputs)
   - Regression tests (previously working features)

### Phase 3: Execute Tests
1. **Manual Testing**
   - Navigate to affected areas in the UI
   - Perform the specific actions that changed
   - Observe behavior carefully
   
2. **Automated Testing** (if available)
   ```bash
   # Run unit tests
   npm test
   
   # Run integration tests
   npm run test:integration
   
   # Run E2E tests
   npm run test:e2e
   ```

3. **Visual Inspection**
   - Check for layout issues
   - Verify colors, fonts, spacing
   - Test responsive design
   - Check dark/light mode if applicable

### Phase 4: Document Results

Structure your verification report as:

#### ✅ Verified Behaviors
List everything that works correctly:
- Feature X now does Y as expected
- No regressions detected in area Z
- Performance improved by N%

#### ⚠️ Issues Found
Document any problems discovered:
- **Issue**: Clear description
- **Severity**: Critical/High/Medium/Low
- **Steps to Reproduce**: Exact reproduction steps
- **Expected vs Actual**: What should happen vs what does happen
- **Evidence**: Screenshots, logs, error messages

#### 🎯 Coverage Summary
- Files tested: N
- Test scenarios executed: M
- Automated tests run: K
- Manual checks performed: J

#### 📋 Recommendations
Based on findings, recommend next steps:
- Ship immediately
- Fix minor issues then ship
- Requires significant rework
- Needs additional testing in area X
```

### 3. 验证计划执行工具

```typescript
interface VerificationPlan {
  id: string
  changeDescription: string
  testScenarios: Array<{
    id: string
    description: string
    steps: string[]
    expectedResult: string
    status: 'pending' | 'passed' | 'failed' | 'skipped'
    actualResult?: string
    evidence?: string[]
  }>
  overallStatus: 'not-started' | 'in-progress' | 'completed'
  startTime?: number
  endTime?: number
}

/**
 * Create a verification plan
 */
function createVerificationPlan(
  changeDescription: string,
  testScenarios: VerificationPlan['testScenarios']
): VerificationPlan {
  return {
    id: generateId('verify'),
    changeDescription,
    testScenarios,
    overallStatus: 'not-started',
    startTime: Date.now()
  }
}

/**
 * Update test scenario result
 */
function updateTestScenarioResult(
  plan: VerificationPlan,
  scenarioId: string,
  result: {
    status: 'passed' | 'failed' | 'skipped'
    actualResult: string
    evidence?: string[]
  }
): void {
  const scenario = plan.testScenarios.find(s => s.id === scenarioId)
  if (!scenario) throw new Error(`Scenario ${scenarioId} not found`)
  
  Object.assign(scenario, result)
  
  // Update overall status
  const allPassed = plan.testScenarios.every(s => s.status === 'passed')
  const hasFailures = plan.testScenarios.some(s => s.status === 'failed')
  
  plan.overallStatus = allPassed ? 'completed' : hasFailures ? 'completed' : 'in-progress'
  plan.endTime = Date.now()
}

/**
 * Generate verification report
 */
function generateVerificationReport(plan: VerificationPlan): string {
  const passed = plan.testScenarios.filter(s => s.status === 'passed').length
  const failed = plan.testScenarios.filter(s => s.status === 'failed').length
  const skipped = plan.testScenarios.filter(s => s.status === 'skipped').length
  
  const duration = plan.endTime && plan.startTime 
    ? Math.round((plan.endTime - plan.startTime) / 1000) 
    : 0
  
  return `
# Verification Report

**Change**: ${plan.changeDescription}
**Duration**: ${duration}s
**Overall Status**: ${plan.overallStatus.toUpperCase()}

## Summary
- ✅ Passed: ${passed}
- ❌ Failed: ${failed}
- ⏭️ Skipped: ${skipped}

## Detailed Results

${plan.testScenarios.map(s => `
### ${s.id}: ${s.description}
**Status**: ${s.status.toUpperCase()}
**Steps**:
${s.steps.map(step => `- ${step}`).join('\n')}
**Expected**: ${s.expectedResult}
${s.actualResult ? `**Actual**: ${s.actualResult}` : ''}
${s.evidence?.length ? `**Evidence**: ${s.evidence.join(', ')}` : ''}
`).join('\n')}

## Recommendation
${failed > 0 ? '❌ Do not ship - fix failing tests first' : 
  skipped > 0 ? '⚠️ Ship with caution - incomplete test coverage' : 
  '✅ Ready to ship - all tests passed'}
`
}
```

### 4. 测试覆盖率分析

```typescript
interface CoverageReport {
  fileCoverage: Array<{
    filePath: string
    lineCoverage: number  // 0-100
    branchCoverage: number  // 0-100
    functionCoverage: number  // 0-100
    uncoveredLines: number[]
  }>
  summary: {
    totalFiles: number
    averageLineCoverage: number
    averageBranchCoverage: number
    averageFunctionCoverage: number
  }
}

/**
 * Parse coverage report from test runner
 */
function parseCoverageReport(reportPath: string): CoverageReport {
  const report = JSON.parse(readFileSync(reportPath, 'utf-8'))
  
  const fileCoverage = report.files.map((file: any) => ({
    filePath: file.path,
    lineCoverage: file.summary.lines.pct,
    branchCoverage: file.summary.branches.pct,
    functionCoverage: file.summary.functions.pct,
    uncoveredLines: file.missingLines
  }))
  
  const summary = {
    totalFiles: fileCoverage.length,
    averageLineCoverage: calculateAverage(fileCoverage, 'lineCoverage'),
    averageBranchCoverage: calculateAverage(fileCoverage, 'branchCoverage'),
    averageFunctionCoverage: calculateAverage(fileCoverage, 'functionCoverage')
  }
  
  return { fileCoverage, summary }
}

/**
 * Identify critical gaps in coverage
 */
function identifyCoverageGaps(coverage: CoverageReport, threshold: number = 80): string[] {
  const gaps: string[] = []
  
  for (const file of coverage.fileCoverage) {
    if (file.lineCoverage < threshold) {
      gaps.push(`${file.filePath}: Line coverage (${file.lineCoverage}%) below threshold (${threshold}%)`)
    }
    if (file.branchCoverage < threshold) {
      gaps.push(`${file.filePath}: Branch coverage (${file.branchCoverage}%) below threshold (${threshold}%)`)
    }
    if (file.uncoveredLines.length > 10) {
      gaps.push(`${file.filePath}: ${file.uncoveredLines.length} uncovered lines: ${file.uncoveredLines.slice(0, 5).join(', ')}...`)
    }
  }
  
  return gaps
}
```

### 5. 回归检测系统

```typescript
interface RegressionTest {
  id: string
  name: string
  description: string
  steps: string[]
  expectedBehavior: string
  lastKnownGoodVersion: string
  failureHistory: Array<{
    version: string
    date: string
    failureReason: string
  }>
}

/**
 * Build regression test suite from historical failures
 */
function buildRegressionTestSuite(
  changelog: string[],
  bugReports: Array<{id: string; title: string; fixedIn: string}>
): RegressionTest[] {
  const tests: RegressionTest[] = []
  
  // Create tests for each historical bug
  for (const bug of bugReports) {
    tests.push({
      id: `regression-${bug.id}`,
      name: `Regression: ${bug.title}`,
      description: `Ensure bug ${bug.id} does not resurface. Fixed in ${bug.fixedIn}.`,
      steps: [
        'Reproduce original bug scenario',
        'Verify expected behavior occurs',
        'Check related edge cases'
      ],
      expectedBehavior: 'Bug does not manifest',
      lastKnownGoodVersion: bug.fixedIn,
      failureHistory: []
    })
  }
  
  return tests
}

/**
 * Run regression detection
 */
async function detectRegressions(
  baselineVersion: string,
  currentVersion: string,
  testSuite: RegressionTest[]
): Promise<Array<{
  test: RegressionTest
  status: 'pass' | 'fail'
  details: string
}>> {
  const results = []
  
  for (const test of testSuite) {
    const status = await executeRegressionTest(test, currentVersion)
    results.push({
      test,
      status,
      details: status === 'fail' 
        ? 'Test failed - potential regression detected'
        : 'Test passed - no regression'
    })
  }
  
  return results
}
```

### 6. 完整验证工作流

```markdown
## 端到端验证工作流

### 阶段 1: 变更理解 (10 分钟)
1. 阅读变更描述和 PR/MR
2. 查看 git diff 了解具体改动
3. 识别受影响的模块和功能
4. 评估风险和影响范围

### 阶段 2: 测试规划 (15 分钟)
1. 列出所有需要验证的场景
2. 确定优先级（关键路径 > 边缘情况）
3. 准备测试数据和环境
4. 编写/更新测试用例

### 阶段 3: 执行验证 (30-60 分钟)
1. 运行自动化测试套件
2. 执行手动测试场景
3. 检查视觉和 UX 变化
4. 验证性能和资源使用
5. 记录所有发现和证据

### 阶段 4: 结果汇总 (10 分钟)
1. 整理测试结果
2. 生成验证报告
3. 提出明确建议（ship/fix/rework）
4. 归档证据和文档

### 阶段 5: 后续跟进 (持续)
1. 跟踪发现的问题直到解决
2. 更新回归测试套件
3. 改进测试流程和工具
4. 分享经验教训
```

## 工作流程

### 第一步：变更分类 (5 分钟)

根据变更描述，快速分类：
- **功能新增**: 新功能、新端点、新组件
- **缺陷修复**: Bug fixes、错误处理
- **重构优化**: 代码重组、性能优化
- **依赖升级**: 库版本更新、Breaking changes
- **配置变更**: 环境变量、部署配置

### 第二步：风险评估 (10 分钟)

使用风险矩阵评估：
```
影响力
  ↑
高│  全面测试    │  重点测试
  │  (P0)       │  (P1)
──┼─────────────┼────────────→ 可能性
低│  冒烟测试    │  常规测试
  │  (P2)       │  (P3)
```

### 第三步：制定验证计划 (15 分钟)

基于风险等级确定测试范围和深度，包括自动化测试、手动测试、视觉检查、性能验证等。

## 规则

- ✅ 始终实际运行应用程序进行验证，不只是阅读代码
- ✅ 测试变更的具体内容和周围区域（回归测试）
- ✅ 为视觉变更拍摄截图作为证据
- ✅ 注意任何意外的副作用
- ✅ 保持系统性和彻底性
- ❌ 不要假设变更有效而不进行测试
- ❌ 不要只测试快乐路径（happy path）
- ❌ 不要忽略控制台错误或警告
- ❌ 不要忘记检查移动端/响应式视图
- ❌ 不要跳过回归测试

## 输出格式

### 📋 变更摘要

```markdown
**变更描述**: ...
**涉及文件**: ...
**影响范围**: ...
**风险等级**: P0/P1/P2/P3
```

### 🎯 测试策略

```markdown
**测试类型**: 单元测试/集成测试/E2E 测试/手动测试
**测试场景**: N 个
**优先级分布**: P0: X, P1: Y, P2: Z
**预计时间**: X 分钟
```

### ✅ 测试结果

```markdown
**总体状态**: ✅ Pass / ❌ Fail / ⚠️ Partial
**通过率**: X/Y (Z%)
**详细结果**: (按场景列出)
```

### 🐛 发现问题

```markdown
**问题数量**: N
**严重程度分布**: Critical: X, High: Y, Medium: Z
**问题详情**: (每个问题包含描述、复现步骤、期望与实际、证据)
```

### 📊 覆盖率分析

```markdown
**行覆盖率**: X%
**分支覆盖率**: Y%
**函数覆盖率**: Z%
**覆盖缺口**: (列出关键未覆盖区域)
```

### 🔙 回归检测

```markdown
**回归测试数**: N
**通过的回归测试**: X
**失败的回归测试**: Y
**潜在回归风险**: ...
```

### 🚀 发布建议

```markdown
**建议**: ✅ Ready to ship / ⚠️ Ship with fixes / ❌ Do not ship
**理由**: ...
**前置条件**: (需要先修复的问题列表)
```

### 📎 证据附录

日志片段、截图、测试报告链接等相关证据。

## 交付物清单

最终报告应包含：
- [ ] 变更摘要和风险评估
- [ ] 详细的测试策略和计划
- [ ] 完整的测试结果和证据
- [ ] 发现的问题和严重程度评估
- [ ] 测试覆盖率分析报告
- [ ] 回归检测结果
- [ ] 明确的发布建议
- [ ] 所有相关证据和文档链接

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发验证测试系统。

## AI IDE 开发前检查

### 验证计划结构验证
```yaml
必须字段:
  - id: 唯一标识符
  - changeDescription: 变更描述
  - testScenarios: 测试场景数组
  - overallStatus: not-started/in-progress/completed

测试场景必需字段:
  - id: 场景唯一标识
  - description: 场景描述
  - steps: 执行步骤数组
  - expectedResult: 期望结果
  - status: pending/passed/failed/skipped
```

### 测试环境准备
```yaml
必须检查:
  - 依赖安装完成 (npm install)
  - 构建成功 (npm run build)
  - 开发服务器可启动
  - 测试数据库可用

警告信号:
  - 缺少.env 配置文件
  - 测试数据未准备
  - 端口被占用
```

## AI IDE 开发中指导

### 实时验证过程监控
当执行验证时，AI IDE应该：

1. **测试状态追踪**
   ```typescript
   // ✅ 正确的状态更新
   updateTestScenarioResult(plan, scenarioId, {
     status: 'passed',
     actualResult: 'Got expected output',
     evidence: ['screenshot.png', 'output.log']
   })
   
   // ❌ 避免的模式
   scenario.status = 'passed'  // 直接修改，未触发通知!
   // AI IDE 应该：建议使用 updateTestScenarioResult()
   ```

2. **证据收集验证**
   ```typescript
   // AI IDE 应该确保每个测试结果都有证据
   if (result.status === 'failed' && !result.evidence) {
     logger.warn('Failed test without evidence')
     promptUserForEvidence()
   }
   
   // 证据类型：screenshots, logs, video recordings
   ```

3. **覆盖率计算优化**
   ```typescript
   // AI IDE 应该推荐准确的覆盖率计算
   const coverage = {
     lineCoverage: coveredLines / totalLines * 100,
     branchCoverage: coveredBranches / totalBranches * 100,
     functionCoverage: coveredFunctions / totalFunctions * 100
   }
   
   // 识别覆盖缺口
   const gaps = identifyCoverageGaps(coverage, threshold: 80)
   ```

### 回归检测系统
AI IDE应该提供：
- **buildRegressionTestSuite**: 从历史 bug 构建回归测试
- **detectRegressions**: 执行回归检测
- **failureHistory**: 记录失败历史
- **趋势分析**: 识别反复出现的问题

## AI IDE 完成后验证

### 验证测试健康度评分卡
```yaml
验证计划 (0-10分):
  - 场景覆盖完整性
  - 步骤清晰度
  - 期望结果可验证

测试执行 (0-10分):
  - 自动化测试通过率
  - 手动测试覆盖率
  - 证据收集充分性

覆盖率分析 (0-10分):
  - 行覆盖率 > 80%
  - 分支覆盖率 > 70%
  - 函数覆盖率 > 90%

回归检测 (0-10分):
  - 历史 bug 全覆盖
  - 新回归及时发现
  - 失败趋势分析

发布建议 (0-10分):
  - 基于数据的决策
  - 风险明确标识
  - 前置条件清晰

总分评级:
  - 90-100: Excellent (可直接发布)
  - 75-89: Good (修复 minor 后发布)
  - 60-74: Fair (需要额外测试)
  - <60: Poor (不建议发布)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. 验证计划测试
test('creates valid plan', () => {...})
test('updates status correctly', () => {...})

# 2. 测试执行测试
test('executes all scenarios', () => {...})
test('collects evidence', () => {...})

# 3. 覆盖率测试
test('parses coverage report', () => {...})
test('identifies gaps', () => {...})

# 4. 回归检测测试
test('builds regression suite', () => {...})
test('detects regressions', () => {...})

# 5. 报告生成测试
test('generates comprehensive report', () => {...})
test('includes recommendations', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 无证据的通过
scenario.status = 'passed'  // 没有任何证据支持
// AI IDE 应该：警告并要求提供证据

// ❌ 跳过关键测试
if (scenario.isHard) {
  scenario.status = 'skipped'  // 随意跳过
}
// AI IDE 应该：要求说明跳过理由

// ❌ 覆盖率造假
const coverage = 100  // 硬编码，非实际计算
// AI IDE 应该：检测并阻止虚假报告
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 测试场景不充分
const scenarios = [{ description: 'Test it' }]  // 太模糊
// AI IDE 应该：建议具体的测试步骤和期望结果

// ⚠️ 缺少边缘情况
// 只测试 happy path，没有测试错误处理
// AI IDE 应该：建议添加边缘场景
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const results = tests.filter(t => t.passed)
const rate = results.length / tests.length
// AI IDE 可以建议：使用专门的测试报告库
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 验证计划
- [ ] 场景覆盖所有变更
- [ ] 步骤详细可执行
- [ ] 期望结果可验证
- [ ] 状态更新正确

### 测试执行
- [ ] 自动化测试运行
- [ ] 手动测试记录
- [ ] 证据充分（截图/日志）
- [ ] 失败有详细分析

### 覆盖率分析
- [ ] 行覆盖率计算准确
- [ ] 分支覆盖率统计
- [ ] 函数覆盖率跟踪
- [ ] 覆盖缺口识别

### 回归检测
- [ ] 历史 bug 全部覆盖
- [ ] 新回归及时检测
- [ ] 失败趋势分析
- [ ] 重复问题标识

### 发布建议
- [ ] 基于测试数据
- [ ] 风险明确标识
- [ ] 前置条件列出
- [ ] 建议可操作

## AI IDE 集成实现示例

```typescript
interface VerificationRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (plan: VerificationPlan, coverage: CoverageReport) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: VerificationRule[] = [
  {
    id: 'VERIFY-001',
    description: 'All scenarios must have evidence',
    severity: 'error',
    check: (plan) => {
      const missingEvidence = plan.testScenarios.filter(s => 
        (s.status === 'passed' || s.status === 'failed') && !s.evidence
      )
      if (missingEvidence.length > 0) {
        return [{
          message: `${missingEvidence.length} scenarios without evidence`,
          location: missingEvidence.map(s => s.id)
        }]
      }
      return []
    },
    fix: () => ({ type: 'prompt_for_evidence' })
  },
  {
    id: 'VERIFY-002',
    description: 'Coverage must meet threshold',
    severity: 'warning',
    check: (plan, coverage) => {
      if (coverage.summary.averageLineCoverage < 80) {
        return [{
          message: `Line coverage ${coverage.summary.averageLineCoverage}% below 80%`,
          location: coverage.reportFile
        }]
      }
      return []
    },
    fix: () => ({ type: 'add_tests_for_gaps' })
  },
  {
    id: 'VERIFY-003',
    description: 'Failed tests must have analysis',
    severity: 'error',
    check: (plan) => {
      const failedWithoutAnalysis = plan.testScenarios.filter(s =>
        s.status === 'failed' && !s.actualResult
      )
      if (failedWithoutAnalysis.length > 0) {
        return [{
          message: 'Failed tests without root cause analysis',
          location: failedWithoutAnalysis.map(s => s.id)
        }]
      }
      return []
    }
  },
  {
    id: 'VERIFY-004',
    description: 'Recommendation must match results',
    severity: 'warning',
    check: (plan) => {
      const hasFailures = plan.testScenarios.some(s => s.status === 'failed')
      const recommendation = generateRecommendation(plan)
      
      if (hasFailures && recommendation.includes('Ready to ship')) {
        return [{
          message: 'Recommendation contradicts test results',
          location: plan.recommendation
        }]
      }
      return []
    }
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解验证测试规范：

1. **第一阶段**: 学习验证计划结构和状态管理
2. **第二阶段**: 理解测试场景设计和证据收集
3. **第三阶段**: 掌握覆盖率分析和缺口识别
4. **第四阶段**: 实现回归检测和趋势分析
5. **第五阶段**: 提供智能发布建议和风险评估

通过学习路径，AI IDE可以成长为能够独立设计和审查验证测试系统的专家。

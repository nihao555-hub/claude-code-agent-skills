---
name: prompt-engineer
description: AI Agent 提示词工程专家 - 系统提示设计、动态注入、多 Agent 协调、fork语义、工具描述生成
aliases: ['提示词', 'prompt-master']
argumentHint: '<场景类型> [目标模型]'
allowedTools: FileReadTool, GrepTool, GlobTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要设计高效的系统提示、实现动态提示注入、构建多 Agent 协作流程、或优化提示词缓存策略时
---

# Prompt Engineer

使用此技能当你需要进行提示词工程设计、系统提示优化、或多Agent 协调流程设计。

## 目标

产出完整的提示词工程方案，包括 branded type 系统提示、动态注入机制、多Agent 协调模式、fork语义和缓存策略。

## 核心能力

### 1. Branded Type 系统提示 (src/utils/systemPromptType.ts)

Claude Code使用 TypeScript branded types确保类型安全：

```typescript
/**
 * Branded type for system prompt arrays.
 * This module is intentionally dependency-free so it can be imported
 * from anywhere without risking circular initialization issues.
 */
export type SystemPrompt = readonly string[] & {
  readonly __brand: 'SystemPrompt'
}

export function asSystemPrompt(value: readonly string[]): SystemPrompt {
  return value as SystemPrompt
}
```

**设计原理**:
- **Dependency-free**: 可以从任何地方导入，无循环依赖风险
- **Type safety**: 防止普通字符串数组被误用为系统提示
- **Runtime zero-cost**: 只是类型断言，无运行时开销
- **Self-documenting**: 函数签名清楚表明这是系统提示

### 2. 系统提示优先级系统 (src/utils/systemPrompt.ts)

严格的优先级决策树：

```
overrideSystemPrompt 存在？
├─ YES → 直接使用 override（完全替换其他所有提示）
└─ NO → 继续检查
  
Coordinator Mode 激活且无主线程 Agent?
├─ YES → 使用 Coordinator 系统提示
└─ NO → 继续检查

有 Main Thread Agent 定义？
├─ YES → 检查是否 Proactive Mode
│   ├─ YES → Default + Agent（追加模式）
│   └─ NO → Agent only（替换模式）
└─ NO → 继续检查

有 Custom System Prompt (--system-prompt)?
├─ YES → 使用 custom prompt
└─ NO → 使用 default system prompt

最后：如果有 appendSystemPrompt，总是追加到末尾
```

### 3. 动态提示注入机制

支持运行时动态修改系统提示：

```typescript
// 全局状态（模块级单例）
let systemPromptInjection: string | null = null

export function setSystemPromptInjection(injection: string): void {
  systemPromptInjection = injection
}

export function getSystemPromptInjection(): string | null {
  return systemPromptInjection
}

export function clearSystemPromptInjection(): void {
  systemPromptInjection = null
}
```

**使用场景**:
- **Cache Breaking**: 当提示词变更导致 API 缓存失效时
- **A/B Testing**: 测试不同提示词的效果
- **Loop Mode**: 特殊运行模式需要覆盖默认行为
- **Debugging**: 临时注入调试信息

### 4. 多 Agent 协调提示词 (src/tools/AgentTool/prompt.ts)

Fork vs Fresh Agent对比：

| 特性 | Fork Subagent | Fresh Agent |
|------|--------------|-------------|
| 上下文继承 | ✓ 继承父代理完整上下文 | ✗ 从零开始 |
| Prompt 缓存共享 | ✓ 共享父代理缓存 | ✗ 独立缓存 |
| 触发方式 | 省略 `subagent_type` | 指定 `subagent_type` |
| 适用场景 | 研究、开放性问题 | 专项任务、需要特定技能 |
| Prompt 风格 | Directive（做什么） | Briefing（背景 + 任务） |
| 成本 | 低（缓存命中） | 高（重新计算） |

**Fork子代理关键规则**:
- **Don't peek**: 不要读取 output_file 除非用户明确要求检查进度
- **Don't race**: 不要编造 fork 的结果，等待通知到达
- **Directive prompts**: fork的提示应该是指令式的，不是背景介绍

### 5. 工具描述生成

动态生成工具描述：

```typescript
function getToolsDescription(agent: AgentDefinition): string {
  const { tools, disallowedTools } = agent
  const hasAllowlist = tools && tools.length > 0
  const hasDenylist = disallowedTools && disallowedTools.length > 0

  if (hasAllowlist && hasDenylist) {
    const denySet = new Set(disallowedTools)
    const effectiveTools = tools.filter(t => !denySet.has(t))
    if (effectiveTools.length === 0) return 'None'
    return effectiveTools.join(', ')
  } else if (hasAllowlist) {
    return tools.join(', ')
  } else if (hasDenylist) {
    return `All tools except ${disallowedTools.join(', ')}`
  }
  return 'All tools'
}
```

### 6. 示例提示库

内置常见任务的优质提示模板：

```typescript
export const EXAMPLE_PROMPTS = {
  codeReview: `Review src/components/Button.tsx for:
- TypeScript type safety
- React best practices
- Accessibility compliance
- Edge cases

Report issues as: Severity | Location | Issue | Fix`,

  debugging: `Debug why tests are failing in tests/auth.test.ts.

Known symptoms:
- Test "should login successfully" times out
- Started after upgrading to Node 24
- Only happens in CI, not locally`,

  research: `Research the best approach for implementing real-time collaboration.

Compare:
- WebSockets vs SSE vs Long Polling
- OT vs CRDTs for conflict resolution
- Self-hosted vs managed services

Deliverable: Recommendation doc with pros/cons and implementation plan.`
} as const
```

## 工作流程

### 第一步：提示词审计 (15 分钟)

1. **收集现有提示词**
   ```bash
   # 查找所有系统提示相关文件
   find . -name "*.ts" -exec grep -l "systemPrompt\|SystemPrompt" {} \;
   
   # 提取所有 hardcoded 提示文本
   grep -r "You are a" src/ --include="*.ts"
   
   # 查找动态注入点
   grep -r "setSystemPromptInjection\|overrideSystemPrompt" src/
   ```

2. **评估要点**
   - 是否有明确的优先级系统？
   - 是否支持动态覆盖？
   - 是否有模块化组合能力？
   - 是否有示例提示库？
   - 是否考虑了缓存优化？

3. **识别反模式**
   - Hardcoded strings scattered across codebase
   - 没有类型安全的 SystemPrompt 包装
   - 循环依赖导致无法从某些模块导入
   - 提示词过长超出模型上下文窗口
   - 缺少变量插值和条件逻辑

### 第二步：提示词架构设计 (20 分钟)

#### 1. 系统提示分层设计

```typescript
interface LayeredSystemPrompt {
  core: readonly string[]           // Core identity (永远不变)
  mode: Record<string, readonly string[]>  // Mode-specific
  agent: Record<string, readonly string[]> // Agent-specific
  custom?: string                   // User customization
  appendices: readonly string[]     // Always appended
}

function buildLayeredPrompt(layers: LayeredSystemPrompt): SystemPrompt {
  return asSystemPrompt([
    ...layers.core,
    ...layers.mode[currentMode],
    ...layers.agent[currentAgent],
    ...(layers.custom ? [layers.custom] : []),
    ...layers.appendices
  ])
}
```

#### 2. 动态注入策略

实现 PromptInjector类，支持优先级、添加/移除、清除操作。

#### 3. 多 Agent 协调模板

提供完整的协调器提示模板，包含 delegation patterns和critical rules。

### 第三步：提示词优化实施 (25 分钟)

#### 优化 1: 提示词压缩

实现压缩策略，移除冗余空格、缩写常见短语、移除填充词、使用符号替代文字。

#### 优化 2: 提示词缓存策略

实现 PromptCache类，支持哈希键生成、TTL管理、容量限制、LRU淘汰。

#### 优化 3: 提示词验证

实现 validatePrompt 函数，检查长度限制、身份声明、示例存在性等。

## 规则

- ✅ 使用 branded type确保类型安全
- ✅ 实现清晰的优先级系统
- ✅ 支持动态注入和 cache breaking
- ✅ 为 fork和fresh agents提供不同的提示模板
- ✅ 维护示例提示库供快速参考
- ❌ 不要在多个文件中重复相同的提示文本
- ❌ 不要忽略缓存优化（提示词变化会导致 cache bust）
- ❌ 不要让 fresh agents从零开始而没有足够上下文
- ❌ 不要在 fork prompts中重复背景信息（fork已继承上下文）

## 输出格式

### 📊 现状评估

```markdown
## 提示词系统成熟度
评分：X/10

### 优势
- ...

### 不足
- ...

### 紧急问题
- ...
```

### 🏗️ 架构设计

使用 Mermaid 图展示提示词构建流程。

### 📝 实现蓝图

提供五个核心模块的完整 TypeScript 实现代码。

### 🔍 测试计划

详细的单元测试、集成测试、性能测试计划。

### ⚠️ 风险评估

| 风险项 | 可能性 | 影响程度 | 缓解措施 |
|-------|--------|----------|----------|
| 提示词过长超出上下文 | 中 | 高 | 实现压缩 + 分段策略 |
| 缓存不一致导致幻觉 | 低 | 高 | 版本号 + 哈希校验 |
| 动态注入冲突 | 中 | 中 | 优先级系统 + 冲突检测 |
| 多 Agent 通信失败 | 低 | 高 | 超时重试 + fallback 机制 |

## 交付物清单

最终报告应包含：
- [ ] 提示词系统现状评估
- [ ] 架构设计图（Mermaid 流程图）
- [ ] 五个核心模块的完整实现代码
- [ ] 测试计划和测试用例
- [ ] 风险评估和缓解措施
- [ ] 性能基准和优化建议
- [ ] 迁移指南（如果重构现有系统）

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发提示词工程系统。

## AI IDE 开发前检查

### Branded Type验证
```yaml
必须检查:
  - SystemPrompt branded type是否定义
  - asSystemPrompt转换函数是否存在
  - 类型导入路径是否正确

警告信号:
  - 直接使用 string[] 而非 SystemPrompt
  - 缺少类型转换包装
  - 循环依赖风险
```

### 优先级系统配置
```yaml
必须存在:
  - overrideSystemPrompt处理
  - coordinator mode检测
  - agent prompt注入逻辑
  - custom prompt支持
  - append prompt机制
```

## AI IDE 开发中指导

### 实时提示词构建监控
当构建系统提示时，AI IDE应该：

1. **优先级链检查**
   ```typescript
   // ✅ 正确的优先级顺序
   if (override) return [override]
   if (coordinator) return [coordinatorPrompt]
   if (agent && !proactive) return [agentPrompt]
   if (custom) return [customPrompt]
   return defaultPrompt
   
   // ❌ 错误的顺序
   // 先检查 custom 再检查 agent = agent 被覆盖
   ```

2. **Fork语义验证**
   ```typescript
   // AI IDE 应该区分 fork vs fresh agent
   const isFork = !subagentType  // 省略=subagent_type=fork
   const prompt = isFork 
     ? buildDirectivePrompt()    // 指令式
     : buildBriefingPrompt()     // 背景+任务式
   ```

3. **缓存友好性检查**
   ```typescript
   // AI IDE 应该警告会导致 cache bust 的变更
   const stableParts = ['core identity', 'static rules']
   const volatileParts = ['dynamic injections', 'temp data']
   
   // 建议：将 volatile parts 移到 append section
   ```

### 动态注入管理
AI IDE应该提供：
- **注入冲突检测**: 多个注入时的优先级处理
- **Cache Breaking通知**: 注入变更时提示 API 缓存失效
- **回滚机制**: 支持 clearInjection恢复原始状态

## AI IDE 完成后验证

### 提示词健康度评分卡
```yaml
Branded Types (0-10分):
  - 类型安全性
  - 依赖独立性
  - 运行时零开销
  
优先级系统 (0-10分):
  - 决策树完整性
  - 分支覆盖度
  - 降级策略合理性
  
多 Agent协调 (0-10分):
  - Fork/Brief区分的正确性
  - Prompt模板质量
  - 上下文继承优化

缓存优化 (0-10分):
  - 稳定/易变部分分离
  - 缓存命中率
  - Cache breaking机制

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. Branded type测试
test('asSystemPrompt converts correctly', () => {...})
test('rejects plain string arrays', () => {...})

# 2. 优先级测试
test('override replaces all', () => {...})
test('coordinator activates correctly', () => {...})

# 3. Fork语义测试
test('fork inherits context', () => {...})
test('fresh agent gets briefing', () => {...})

# 4. 缓存测试
test('stable parts reuse cache', () => {...})
test('volatile parts trigger rebuild', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ 绕过 branded type
const prompt: string[] = [...]  // 应该是 SystemPrompt!
buildEffectiveSystemPrompt(prompt)  // 类型错误但 TS 没 catches
// AI IDE 应该：立即警告 + 建议包装为 asSystemPrompt()

// ❌ 优先级混乱
if (custom) return [custom]  // 先检查 custom
if (override) return [override]  // override 永远不会执行!
// AI IDE 应该：检测死代码 + 重新排序

// ❌ Fresh agent无上下文
AgentTool({ subagent_type: 'expert' })  // 从零开始!
prompt: 'Fix the bug'  // 没有背景信息
// AI IDE 应该：警告需要提供完整 briefing
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 提示词过长
const prompt = veryLongString  // 可能超出 token limit
// AI IDE 应该：建议压缩或分段

// ⚠️ 缺少示例
const systemPrompt = 'You are helpful...'  // 无 few-shot examples
// AI IDE 应该：建议添加示例提升效果
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const prompt = base + '\n' + injection  // 字符串拼接
// AI IDE 可以建议：使用模板字符串或 builder pattern
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### 类型安全
- [ ] 所有系统提示使用 SystemPrompt类型
- [ ] 转换函数正确使用 asSystemPrompt
- [ ] 无类型逃逸到普通 string[]

### 优先级逻辑
- [ ] Override最先检查
- [ ] Coordinator mode正确处理
- [ ] Agent prompt替换/追加逻辑清晰
- [ ] Append始终在最后

### 多 Agent 协调
- [ ] Fork省略 subagent_type
- [ ] Fresh agent提供完整 briefing
- [ ] Prompt风格匹配代理类型
- [ ] 上下文继承优化

### 缓存友好
- [ ] 稳定/易变部分分离
- [ ] 避免不必要的提示词变更
- [ ] Cache breaking有明确标识
- [ ] 缓存命中率可监控

## AI IDE 集成实现示例

```typescript
interface PromptRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (prompt: SystemPrompt, context: BuildContext) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: PromptRule[] = [
  {
    id: 'PROMPT-001',
    description: 'Must use branded SystemPrompt type',
    severity: 'error',
    check: (prompt, context) => {
      if (!isBranded(prompt)) {
        return [{
          message: 'Plain string[] used instead of SystemPrompt',
          location: context.callSite
        }]
      }
      return []
    },
    fix: () => ({ type: 'wrap', wrapper: 'asSystemPrompt' })
  },
  {
    id: 'PROMPT-002',
    description: 'Priority order must be correct',
    severity: 'error',
    check: (prompt, context) => {
      const order = getCheckOrder(context.sourceCode)
      if (order[0] !== 'override') {
        return [{
          message: 'Override should be checked first',
          location: context.ifStatement
        }]
      }
      return []
    }
  },
  {
    id: 'PROMPT-003',
    description: 'Fresh agents need full briefing',
    severity: 'warning',
    check: (prompt, context) => {
      if (context.isFreshAgent && prompt.length < 100) {
        return [{
          message: 'Fresh agent prompt too short, may lack context',
          location: context.promptDefinition
        }]
      }
      return []
    }
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解提示词工程规范：

1. **第一阶段**: 学习 branded type系统设计原理
2. **第二阶段**: 理解优先级决策树和降级策略
3. **第三阶段**: 掌握 fork vs fresh agent语义差异
4. **第四阶段**: 实现提示词缓存优化和 cache breaking
5. **第五阶段**: 提供智能提示词压缩和示例推荐

通过学习路径，AI IDE可以成长为能够独立设计和审查提示词工程系统的专家。

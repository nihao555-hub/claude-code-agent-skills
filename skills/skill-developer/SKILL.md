---
name: skill-developer
description: AI Agent 技能开发专家 - Bundled Skill注册、文件提取机制、参数处理、钩子系统、技能搜索和动态加载
aliases: ['技能开发', 'skill-builder']
argumentHint: '<技能名称> [功能描述]'
allowedTools: FileReadTool, GrepTool, GlobTool, FileWriteTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要创建新的 bundled skill、实现技能文件提取系统、设计参数验证逻辑、或构建技能钩子机制时
---

# Skill Developer

使用此技能当你需要进行 Bundled Skill开发、技能文件提取系统设计、参数验证逻辑实现、或技能钩子机制开发。

## 目标

产出完整的技能开发方案，包括 Bundled Skill 注册系统、文件提取机制、参数处理与验证、钩子系统和技能搜索发现机制。

## 核心能力

### 1. Bundled Skill 注册系统 (src/skills/bundledSkills.ts)

Claude Code的 Bundled Skill 注册模式：

```typescript
import type { ContentBlockParam } from '@anthropic-ai/sdk/resources/index.mjs'
import { constants as fsConstants } from 'fs'
import { mkdir, open } from 'fs/promises'
import { dirname, isAbsolute, join, normalize, sep as pathSep } from 'path'
import type { ToolUseContext } from '../Tool.js'

// Skill 文件定义（用于提取到磁盘）
const SKILL_FILES: Record<string, string> = {
  'examples/cli.md': `
# CLI Usage Examples

\`\`\`bash
/example-command arg1 arg2
\`\`\`
`,
  'examples/server.md': `
# Server Usage Examples

\`\`\`typescript
await server.start();
\`\`\`
`
}

// Skill 主体内容（markdown 格式）
const SKILL_MD = `
---
name: verify
description: Verify a code change does what it should by running the app.
argumentHint: <change-description>
---

# Verification Skill

This skill helps you verify that a code change works correctly.

## Process

1. **Understand the Change**
   - What was modified?
   - What behavior should change?
   - What should remain unchanged?

2. **Run the Application**
   - Start the development server
   - Navigate to affected areas
   - Test the specific change

3. **Verify Behavior**
   - Does it work as expected?
   - Are there any regressions?
   - Edge cases covered?

4. **Report Results**
   - Summary of findings
   - Screenshots if applicable
   - Recommendations
`

// 解析 frontmatter
import { parseFrontmatter } from '../../utils/frontmatterParser.js'

const { frontmatter, content: SKILL_BODY } = parseFrontmatter(SKILL_MD)

const DESCRIPTION = typeof frontmatter.description === 'string'
  ? frontmatter.description
  : 'Verify a code change does what it should.'

// 注册技能
export function registerVerifySkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return  // 仅 ant 用户可用
  }

  registerBundledSkill({
    name: 'verify',
    description: DESCRIPTION,
    userInvocable: true,
    files: SKILL_FILES,  // 要提取的文件
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

### 2. 技能文件提取机制

```typescript
/**
 * Extract bundled skill files to disk on first use
 */
async function extractBundledSkillFiles(
  skillName: string,
  files: Record<string, string>
): Promise<string | null> {
  try {
    const extractDir = getBundledSkillExtractDir(skillName)
    
    // Create directory
    await mkdir(extractDir, { recursive: true })
    
    // Write each file
    for (const [relativePath, content] of Object.entries(files)) {
      const fullPath = join(extractDir, relativePath)
      const dir = dirname(fullPath)
      
      // Ensure parent directory exists
      await mkdir(dir, { recursive: true })
      
      // Write file
      const fd = await open(fullPath, 'w')
      try {
        await fd.write(content)
      } finally {
        await fd.close()
      }
    }
    
    return extractDir
  } catch (error) {
    console.error(`Failed to extract skill files for ${skillName}:`, error)
    return null
  }
}

/**
 * Get extraction directory path
 */
function getBundledSkillExtractDir(skillName: string): string {
  // In production: returns path within bun cache
  // In development: returns src/skills/bundled/{skillName}
  if (process.env.NODE_ENV === 'development') {
    return join(process.cwd(), 'src', 'skills', 'bundled', skillName)
  } else {
    return join(getCacheDir(), 'bundled-skills', skillName)
  }
}
```

### 3. 参数处理与验证

```typescript
import { z } from 'zod/v4'

// 定义参数 schema
const skillInputSchema = z.object({
  target: z.string().describe('The file or directory to verify'),
  changes: z.array(z.string()).optional().describe('List of specific changes made'),
  testCommand: z.string().optional().describe('Custom test command to run')
})

type SkillInput = z.infer<typeof skillInputSchema>

/**
 * Parse and validate skill arguments
 */
function parseSkillArgs(args: string): SkillInput {
  try {
    // Try parsing as JSON first
    return skillInputSchema.parse(JSON.parse(args))
  } catch {
    // Fall back to simple string parsing
    const parts = args.trim().split(/\s+/)
    return {
      target: parts[0] || '.',
      changes: parts.slice(1),
      testCommand: undefined
    }
  }
}

/**
 * Generate helpful error message for invalid input
 */
function getInvalidInputErrorMessage(error: z.ZodError): string {
  const issues = error.errors.map(issue => {
    const path = issue.path.join('.')
    return `${path || 'root'}: ${issue.message}`
  })
  
  return `Invalid skill input:\n- ${issues.join('\n- ')}`
}
```

### 4. 钩子系统 (HooksSettings)

```typescript
export interface HooksSettings {
  onStop?: {
    command: string
    description?: string
  }
  onTurnComplete?: {
    command: string
    description?: string
  }
}

/**
 * Register session hooks for a skill
 */
function registerSkillHooks(hooks: HooksSettings): void {
  const { onStop, onTurnComplete } = hooks
  
  if (onStop) {
    registerStopHook({
      command: onStop.command,
      description: onStop.description || 'Skill completion hook'
    })
  }
  
  if (onTurnComplete) {
    registerTurnCompleteHook({
      command: onTurnComplete.command,
      description: onTurnComplete.description || 'Turn complete hook'
    })
  }
}

/**
 * Example: Notify user when skill completes
 */
const notifyOnComplete: HooksSettings = {
  onStop: {
    command: 'osascript -e \'display notification "Skill completed!" with title "Claude Code"\'',
    description: 'Desktop notification on completion'
  }
}

/**
 * Example: Log metrics when turn completes
 */
const logMetrics: HooksSettings = {
  onTurnComplete: {
    command: 'echo "Turn completed at $(date)" >> /tmp/claude-metrics.log',
    description: 'Log turn completion time'
  }
}
```

### 5. 技能搜索与发现

```typescript
/**
 * Search for skills by name or description
 */
async function searchSkills(query: string): Promise<Array<{
  name: string
  description: string
  matchScore: number
}>> {
  const queryLower = query.toLowerCase()
  const allSkills = getBundledSkills()
  
  return allSkills
    .map(skill => {
      const nameMatch = skill.name.toLowerCase().includes(queryLower)
      const descMatch = skill.description.toLowerCase().includes(queryLower)
      
      let matchScore = 0
      if (nameMatch) matchScore += 10
      if (descMatch) matchScore += 5
      
      // Fuzzy matching bonus
      const fuzzyScore = calculateFuzzyMatch(queryLower, skill.name)
      matchScore += fuzzyScore
      
      return {
        name: skill.name,
        description: skill.description,
        matchScore
      }
    })
    .filter(skill => skill.matchScore > 0)
    .sort((a, b) => b.matchScore - a.matchScore)
}

/**
 * Calculate fuzzy match score (simple implementation)
 */
function calculateFuzzyMatch(query: string, target: string): number {
  const distance = levenshteinDistance(query, target)
  const maxLength = Math.max(query.length, target.length)
  return Math.max(0, 10 - (distance / maxLength) * 10)
}
```

### 6. 技能执行生成器

```typescript
/**
 * Run skill generator to produce prompt blocks
 */
async function runSkillGenerator(
  skillName: string,
  args: string,
  context: ToolUseContext
): Promise<ContentBlockParam[]> {
  const skill = findSkillByName(skillName)
  
  if (!skill) {
    throw new Error(`Skill "${skillName}" not found`)
  }
  
  try {
    // Execute skill's getPromptForCommand
    const promptBlocks = await skill.getPromptForCommand(args, context)
    
    // Validate output
    if (!Array.isArray(promptBlocks)) {
      throw new Error('getPromptForCommand must return an array')
    }
    
    for (const block of promptBlocks) {
      if (!block.type || !['text', 'image', 'tool_use'].includes(block.type)) {
        throw new Error(`Invalid block type: ${(block as any).type}`)
      }
    }
    
    return promptBlocks
  } catch (error) {
    console.error(`Skill execution failed: ${skillName}`, error)
    
    // Return error as text block
    return [{
      type: 'text',
      text: `Skill "${skillName}" failed: ${error instanceof Error ? error.message : String(error)}`
    }]
  }
}
```

### 7. 完整 Skill 示例

```typescript
/**
 * Complete example: Code Review Skill
 */

import type { ContentBlockParam } from '@anthropic-ai/sdk/resources/index.mjs'
import type { ToolUseContext } from '../index.js'
import { registerBundledSkill } from '../index.js'

const REVIEW_SKILL_FILES = {
  'checklist.md': `
# Code Review Checklist

## Correctness
- [ ] Logic is sound
- [ ] Edge cases handled
- [ ] No off-by-one errors
- [ ] Null/undefined checks

## Security
- [ ] No SQL injection risks
- [ ] Input validation present
- [ ] Authentication checks
- [ ] No hardcoded secrets

## Performance
- [ ] No unnecessary loops
- [ ] Efficient data structures
- [ ] Caching where appropriate
- [ ] No memory leaks

## Maintainability
- [ ] Clear variable names
- [ ] Comments explain "why"
- [ ] Functions are small
- [ ] DRY principles followed
`
}

const REVIEW_SKILL_MD = `
---
name: code-review
description: Expert code review with focus on bugs, security, and best practices
aliases: ['review', 'pr-review']
argumentHint: '<file-path> [focus-areas]'
allowedTools: ['FileReadTool', 'GrepTool', 'GlobTool']
model: 'claude-sonnet-4-5-20260101'
---

# Code Review Skill

You are an expert code reviewer with deep knowledge of:
- Security vulnerabilities (OWASP Top 10)
- Performance optimization patterns
- Language-specific best practices
- Maintainability and code smells

## Review Process

1. **First Pass**: Read entire file for understanding
2. **Second Pass**: Deep analysis of each concern area
3. **Third Pass**: Prioritize and format findings

## Output Format

Structure your review as:

### Summary
Brief overview (2-3 sentences)

### Critical Issues 🔴
Must fix before merge

### High Priority 🟡
Important but not blocking

### Medium Priority 🟢
Suggestions for improvement

### Low Priority ⚪
Nice-to-have enhancements

### Positive Notes 👍
What was done well
`

export function registerCodeReviewSkill(): void {
  const { frontmatter, content: SKILL_BODY } = parseFrontmatter(REVIEW_SKILL_MD)
  
  registerBundledSkill({
    name: frontmatter.name as string,
    description: frontmatter.description as string,
    aliases: frontmatter.aliases as string[],
    argumentHint: frontmatter.argumentHint as string,
    allowedTools: frontmatter.allowedTools as string[],
    model: frontmatter.model as string,
    userInvocable: true,
    files: REVIEW_SKILL_FILES,
    
    async getPromptForCommand(args: string, context: ToolUseContext) {
      const parts: string[] = [SKILL_BODY.trimStart()]
      
      if (args) {
        parts.push(`## Review Request\n\nTarget: ${args}`)
      }
      
      return [{ type: 'text', text: parts.join('\n\n') }]
    },
    
    hooks: {
      onStop: {
        command: 'echo "✓ Code review complete!"',
        description: 'Show completion marker'
      }
    }
  })
}
```

## 工作流程

### 第一步：技能需求分析 (10 分钟)

1. **明确技能目标**
   ```
   - 这个技能解决什么问题？
   - 目标用户是谁？
   - 需要什么工具权限？
   - 是否需要提取文件到磁盘？
   ```

2. **定义输入输出**
   ```typescript
   // 输入参数 schema
   const inputSchema = z.object({
     // 定义你的参数
   })
   
   // 输出格式
   type Output = {
     // 定义你的输出
   }
   ```

### 第二步：技能实现 (20 分钟)

基于 Claude Code最佳实践实现你的技能，包括 frontmatter定义、skill body markdown、文件提取、参数验证和钩子注册。

### 第三步：测试与优化 (15 分钟)

#### 测试检查清单

- [ ] **基本功能测试**
  - [ ] 空参数处理
  - [ ] 有效参数处理
  - [ ] 无效参数错误提示
  - [ ] 文件提取成功验证

- [ ] **集成测试**
  - [ ] 技能注册成功
  - [ ] 技能搜索找到
  - [ ] 技能执行正常
  - [ ] 钩子触发正确

- [ ] **边界测试**
  - [ ] 大参数处理
  - [ ] 特殊字符处理
  - [ ] 并发执行测试
  - [ ] 内存泄漏检查

## 规则

- ✅ 使用 frontmatter定义技能元数据
- ✅ 为复杂技能提供文件提取支持
- ✅ 实现健壮的参数验证和错误提示
- ✅ 使用 HooksSettings 注册生命周期钩子
- ✅ 提供清晰的技能描述和使用说明
- ❌ 不要在技能名称中使用空格或非 kebab-case
- ❌ 不要忘记处理参数为空的情况
- ❌ 不要假设文件提取总是成功
- ❌ 不要忽略技能的错误处理和降级策略

## 输出格式

### 📋 技能定义

```yaml
name: skill-name
description: 清晰的技能描述
aliases: [alias1, alias2]
argumentHint: '<required> [optional]'
allowedTools: [Tool1, Tool2]
model: claude-sonnet-4-5-20260101
userInvocable: true
```

### 🏗️ 架构设计

使用流程图展示技能执行流程。

### 📝 实现代码

提供完整的 TypeScript 实现代码，包括 frontmatter、skill body、文件提取、参数验证和钩子。

### 🔍 测试用例

详细的单元测试、集成测试、边界测试计划。

### 📚 使用文档

技能的使用说明、示例调用、常见问题解答。

### ⚠️ 注意事项

已知的限制、使用场景的边界条件、性能考虑。

## 交付物清单

最终报告应包含：
- [ ] 技能定义和元数据
- [ ] 完整的 TypeScript 实现代码
- [ ] 文件提取机制（如果需要）
- [ ] 参数验证逻辑
- [ ] 钩子系统实现
- [ ] 测试计划和测试用例
- [ ] 使用文档和示例
- [ ] 性能基准和优化建议

---

# 🤖 AI IDE 开发指南

本部分指导 AI IDE 按照 Claude Code的最佳实践开发 Bundled Skill系统。

## AI IDE 开发前检查

### Frontmatter 验证
```yaml
必须字段:
  - name: kebab-case命名
  - description: 清晰的技能描述
  - argumentHint: '<required> [optional]'格式
  
可选字段:
  - aliases: 别名列表
  - allowedTools: 允许的工具列表
  - model: 目标模型
  - whenToUse: 使用时机说明

警告信号:
  - name 包含空格或大写字母
  - description 过于模糊
  - argumentHint 格式不正确
```

### 文件提取准备
```yaml
必须检查:
  - SKILL_FILES 对象定义
  - 相对路径格式正确
  - 文件内容无 PII 泄露
  
目录结构:
  - examples/*.md - 使用示例
  - templates/*.md - 模板文件
  - rules/*.md - 规则文档
```

## AI IDE 开发中指导

### 实时技能开发监控
当创建新技能时，AI IDE应该：

1. **命名规范检查**
   ```typescript
   // ✅ 正确的命名
   const skill = { name: 'code-review' }  // kebab-case
   const skill = { name: 'debug' }
   
   // ❌ 错误的命名
   const skill = { name: 'codeReview' }  // camelCase
   const skill = { name: 'Code Review' }  // 含空格
   // AI IDE 应该：立即警告并建议修正
   ```

2. **Frontmatter解析验证**
   ```typescript
   // AI IDE 应该验证 frontmatter 解析
   const { frontmatter, content } = parseFrontmatter(SKILL_MD)
   
   // 必需字段检查
   if (!frontmatter.name || !frontmatter.description) {
     throw new Error('Missing required frontmatter fields')
   }
   
   // 类型安全转换
   const name = String(frontmatter.name)
   const description = String(frontmatter.description)
   ```

3. **参数处理优化**
   ```typescript
   // AI IDE 应该推荐健壮的参数解析
   function parseSkillArgs(args: string): ParsedArgs {
     try {
       // 尝试 JSON 解析（结构化输入）
       return JSON.parse(args)
     } catch {
       // 降级为空格分隔的简单解析
       const parts = args.trim().split(/\s+/)
       return { target: parts[0] || '.', rest: parts.slice(1) }
     }
   }
   ```

### 钩子系统管理
AI IDE应该提供：
- **onStop 钩子**: 技能完成时的清理操作
- **onTurnComplete 钩子**: 每轮结束后的处理
- **错误隔离**: 钩子错误不影响主流程
- **命令注入防护**: 验证 shell 命令安全性

## AI IDE 完成后验证

### 技能开发健康度评分卡
```yaml
Frontmatter (0-10分):
  - 字段完整性
  - 格式正确性
  - 描述清晰度

文件提取 (0-10分):
  - 路径正确性
  - 内容安全性
  - 目录创建可靠性

参数处理 (0-10分):
  - JSON 解析健壮性
  - 降级策略合理性
  - 错误提示友好性

钩子系统 (0-10分):
  - 生命周期覆盖
  - 错误隔离
  - 资源清理

搜索发现 (0-10分):
  - 名称匹配准确度
  - 描述匹配相关度
  - 模糊搜索智能性

总分评级:
  - 90-100: Excellent (生产就绪)
  - 75-89: Good (少量优化)
  - 60-74: Fair (需要改进)
  - <60: Poor (建议重构)
```

### 自动化测试清单
AI IDE应该运行：
```bash
# 1. Frontmatter 测试
test('parses name correctly', () => {...})
test('validates required fields', () => {...})

# 2. 文件提取测试
test('creates directory structure', () => {...})
test('writes files atomically', () => {...})

# 3. 参数处理测试
test('handles JSON input', () => {...})
test('handles string input', () => {...})
test('shows helpful errors', () => {...})

# 4. 钩子测试
test('onStop fires on completion', () => {...})
test('errors dont break main flow', () => {...})

# 5. 搜索测试
test('finds by name', () => {...})
test('finds by description', () => {...})
test('ranks by relevance', () => {...})
```

## AI IDE 常见陷阱检测

### 🔴 高危问题（必须修复）
```typescript
// ❌ Frontmatter 缺少必需字段
const skill = {
  name: 'my-skill'
  // missing description!
}
// AI IDE 应该：立即警告 + 阻止注册

// ❌ 文件提取不安全
const path = userInput  // 未验证的用户输入!
await writeFile(path, content)  // 路径遍历攻击风险
// AI IDE 应该：建议使用 join(extractDir, relativePath)

// ❌ 参数无验证
async getPromptForCommand(args: string) {
  return [{ type: 'text', text: args }]  // 直接使用未验证的参数!
}
// AI IDE 应该：建议添加参数解析和验证
```

### 🟡 中等风险（建议优化）
```typescript
// ⚠️ 错误提示不友好
throw new Error('Invalid args')
// AI IDE 应该：建议提供详细的错误信息和修复建议

// ⚠️ 缺少超时
await longRunningOperation()  // 可能永远等待
// AI IDE 应该：建议添加 AbortSignal.timeout()
```

### 🟢 低风险（可选改进）
```typescript
// 💡 可以优化的模式
const skills = allSkills.filter(s => s.name.includes(query))
// AI IDE 可以建议：使用 fuzzy search 提升匹配质量
```

## AI IDE 代码审查检查清单

在 PR/MR阶段，AI IDE应该自动检查：

### Frontmatter
- [ ] name 使用 kebab-case
- [ ] description 清晰具体
- [ ] argumentHint 格式正确
- [ ] aliases 有助于发现
- [ ] allowedTools 合理限制

### 文件提取
- [ ] 路径使用相对路径
- [ ] 目录自动创建
- [ ] 原子写入（flush:true）
- [ ] 无 PII 泄露

### 参数处理
- [ ] JSON 解析有 try-catch
- [ ] 降级策略合理
- [ ] 错误信息有帮助
- [ ] Zod 验证 schema

### 钩子系统
- [ ] onStop 清理资源
- [ ] onTurnComplete 记录日志
- [ ] 错误不中断主流程
- [ ] 命令注入防护

### 搜索发现
- [ ] 名称匹配准确
- [ ] 描述匹配相关
- [ ] 模糊搜索智能
- [ ] 排序合理

## AI IDE 集成实现示例

```typescript
interface SkillRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (skill: BundledSkill, files: SkillFiles) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: SkillRule[] = [
  {
    id: 'SKILL-001',
    description: 'Skill name must be kebab-case',
    severity: 'error',
    check: (skill) => {
      if (!/^[a-z][a-z0-9-]*$/.test(skill.name)) {
        return [{
          message: `Invalid skill name format: ${skill.name}`,
          location: skill.nameDefinition
        }]
      }
      return []
    },
    fix: () => ({ type: 'convert', format: 'kebab-case' })
  },
  {
    id: 'SKILL-002',
    description: 'Must have description in frontmatter',
    severity: 'error',
    check: (skill) => {
      if (!skill.frontmatter.description) {
        return [{
          message: 'Missing description field',
          location: skill.frontmatterLocation
        }]
      }
      return []
    }
  },
  {
    id: 'SKILL-003',
    description: 'File paths must be relative',
    severity: 'error',
    check: (skill, files) => {
      for (const path of Object.keys(files)) {
        if (path.startsWith('/') || path.includes('..')) {
          return [{
            message: `Absolute or unsafe path: ${path}`,
            location: path
          }]
        }
      }
      return []
    },
    fix: () => ({ type: 'sanitize_paths' })
  },
  {
    id: 'SKILL-004',
    description: 'Should handle parse errors gracefully',
    severity: 'warning',
    check: (skill) => {
      const source = skill.getPromptForCommand.toString()
      if (!source.includes('try') && !source.includes('catch')) {
        return [{
          message: 'No error handling for argument parsing',
          location: skill.getPromptForCommand
        }]
      }
      return []
    }
  }
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解技能开发规范：

1. **第一阶段**: 学习 frontmatter 语法和必需字段
2. **第二阶段**: 理解文件提取机制和安全考虑
3. **第三阶段**: 掌握参数解析和验证模式
4. **第四阶段**: 实现钩子系统和生命周期管理
5. **第五阶段**: 提供智能搜索发现和排序优化

通过学习路径，AI IDE可以成长为能够独立设计和审查技能开发系统的专家。

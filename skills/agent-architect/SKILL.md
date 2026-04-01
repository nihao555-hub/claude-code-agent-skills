---
name: agent-architect
description: AI Agent 架构设计与系统组织专家 - 掌握 Claude Code的整体架构、模块组织、依赖管理和设计模式
aliases: ['架构师', 'agent-design']
argumentHint: '<项目路径> [关注领域]'
allowedTools: FileReadTool, GrepTool, GlobTool, BashTool
model: claude-sonnet-4-5-20260101
whenToUse: 当你需要设计新的 Agent 系统、重构现有架构、或理解复杂代码库的组织结构时
---

# Agent Architect

使用此技能当你需要进行 Agent 系统的架构设计、代码库组织结构分析、或设计模式识别。

## 目标

产出全面的架构分析报告，包括模块化组织、依赖管理、设计模式应用和技术债务评估。

## 核心能力

### 1. 项目结构设计
- 模块化组织原则（高内聚、低耦合）
- 目录命名约定（kebab-case for commands, PascalCase for components）
- 入口点管理（dev-entry.ts, main.tsx, commands.ts）
- 特性门控（feature flags）与条件加载

### 2. 依赖管理
- ESM imports 与 CommonJS requires 混用场景
- 循环依赖破解（lazy require, top-level require guards）
- 第三方库 shim 层设计（vendor/, shims/目录）
- Bun runtime特定的优化（bun:bundle feature API）

### 3. 分层架构
- **Bootstrap 层**: 启动剖析、MDM 配置、环境变量预读
- **Core 层**: Tool 抽象、Task 编排、Context 管理
- **Services 层**: MCP 连接、Analytics、Session 管理
- **UI 层**: Ink TUI 组件、React 渲染、状态可视化

### 4. 设计模式应用
- **Builder Pattern**: buildTool(), buildEffectiveSystemPrompt()
- **Strategy Pattern**: PermissionMode 切换、TaskType 多态
- **Observer Pattern**: AppState subscribe/unsubscribe, EventEmitter
- **Singleton Pattern**: cleanupRegistry, global state stores
- **Factory Pattern**: createTaskId(), createAgentId()

## 工作流程

### 第一步：宏观扫描 (5 分钟)
1. 列出顶层目录结构
2. 识别 package.json 中的 entry points
3. 查看 tsconfig.json 了解模块系统
4. 检查 AGENTS.md 或 README.md 获取项目指南

### 第二步：核心模块定位 (10 分钟)
找到并阅读以下关键文件：
- `src/main.tsx` 或 `src/dev-entry.ts` - 主入口
- `src/commands.ts` - CLI 命令注册
- `src/tools.ts` - 工具池组装
- `src/tasks.ts` - 任务类型定义
- `src/Tool.ts` - Tool 抽象基类
- `src/Task.ts` - Task 状态机

### 第三步：数据流追踪 (15 分钟)
选择一个典型用户命令（如 `/init`），追踪：
1. 命令如何被解析（commands/ 目录）
2. 如何调用 Tool（assembleToolPool）
3. 如何创建 Task（generateTaskId, createTaskStateBase）
4. 如何更新 AppState（setAppState 模式）
5. 如何输出结果（render JSX components）

### 第四步：架构文档化 (10 分钟)
产出以下内容：
- **架构图**: ASCII art 或 Mermaid 图表展示模块关系
- **依赖图**: 哪些模块依赖哪些外部服务
- **状态流**: 关键状态的变迁路径
- **扩展点**: 哪里可以插入新功能而不破坏现有代码

## 规则

- ✅ 从用户视角解释架构（这个模块对用户有什么价值）
- ✅ 提供具体的代码位置和行号引用
- ✅ 区分"当前实现"和"理想实现"
- ✅ 考虑向后兼容性和迁移成本
- ❌ 不要只罗列文件名而不解释关系
- ❌ 不要忽略历史原因（为什么代码是这样写的）
- ❌ 不要推荐破坏性重构除非必要
- ❌ 不要忘记安全性考量（权限边界在哪里）

## 输出格式

### 📁 项目概览
```
项目名称：从 package.json 读取
主要入口：列出一打入口文件
模块系统：ESM/CommonJS/Hybrid
Runtime: Bun/Node/Deno
```

### 🏗️ 架构分层
使用 ASCII 图展示层次结构。

### 🔑 核心模块详解
为每个核心模块提供：
- **职责**: 一句话描述
- **关键函数**: 列出 3-5 个核心 API
- **依赖关系**: 上游依赖谁，下游被谁依赖
- **扩展建议**: 如何在不修改源码的情况下增强功能

### 🎨 设计模式识别
找出项目中使用的 5-8 个设计模式，每个模式提供：
- **模式名称**: GoF 或其他标准名称
- **出现位置**: 文件路径 + 行号
- **代码片段**: 10 行以内的示例
- **为什么用它**: 解决的问题是什么

### ⚠️ 技术债务评估
诚实指出：
- **TODO/FIXME 注释**: 列出所有待办事项
- **循环依赖**: 哪些模块存在循环引用
- **硬编码**: 应该配置化的魔法字符串/数字
- **缺失测试**: 哪些关键路径没有单元测试

### 🚀 改进路线图
按优先级排序的建议：
1. **立即修复** (影响稳定性的 bug)
2. **短期优化** (1-2 周可完成的改进)
3. **中期重构** (1-2 个月的架构调整)
4. **长期愿景** (6 个月以上的演进方向)

## 特殊场景处理

### 如果项目使用了 Feature Flags
重点分析：
- 哪些功能是实验性的（可能随时移除）
- 哪些功能是永久性的（必须保持稳定）
- Feature flag 的检查模式（env var vs GrowthBook）

### 如果项目有 Plugin 系统
重点关注：
- Plugin 的生命周期（何时加载、何时卸载）
- Plugin 的权限边界（能访问什么，不能访问什么）
- Plugin 之间的通信机制

### 如果项目支持多模态
特别留意：
- 图像/音频/视频的处理管道
- 大文件存储策略（临时文件 vs 数据库）
- 流式处理的实现方式

## 示例

### 架构分析示例

```
用户：请分析 e:\claude-code-rev 项目的架构

分析师：
# 架构分析报告

## 📁 项目概览
项目名称：@anthropic-ai/claude-code
主要入口：src/dev-entry.ts, src/main.tsx, src/commands.ts
模块系统：ESM with Bun runtime
Runtime: Bun 1.3.5+

## 🏗️ 架构分层
┌─────────────────────┐
│   CLI Commands      │  ← 用户交互层
├─────────────────────┤
│   Tool Registry     │  ← 能力抽象层
├─────────────────────┤
│   Task Orchestrator │  ← 执行编排层
├─────────────────────┤
│   State Management  │  ← 数据持久层
└─────────────────────┘

## 🔑 核心模块
...
```

## 交付物清单

最终报告应包含：
- [ ] 架构图（ASCII 或 Mermaid）
- [ ] 核心模块职责表
- [ ] 数据流追踪图
- [ ] 设计模式 catalog
- [ ] 技术债务清单
- [ ] 改进路线图（含优先级）
- [ ] 参考代码片段（带文件路径）

---

# 🤖 AI IDE 使用指南

本部分是专门为 AI IDE设计的，用于在开发 Agent时自动检查和应用 Claude Code的架构最佳实践。

## AI IDE 开发前检查

### 项目结构验证
```yaml
必须检查:
  - src/目录是否存在且为主要代码目录
  - 入口文件是否明确 (main.tsx/dev-entry.ts/commands.ts)
  - package.json 是否定义了正确的 entry points
  - tsconfig.json 是否配置了 ESM 模块系统

警告信号:
  - 代码散落在多个根目录
  - 没有明确的入口点
  - 混合使用 CommonJS和ESM 而无明确规则
```

### 依赖管理检查
```yaml
必须检查:
  - 是否有 vendor/或 shims/目录处理第三方库
  - 循环依赖检测工具是否集成
  - feature flags 系统是否存在

警告信号:
  - 直接 require 大型库而无 shim 层
  - 超过 3层的嵌套依赖
  - 硬编码的环境特定路径
```

## AI IDE 开发中指导

### 实时架构分析
当用户创建新文件时，AI IDE应该：

1. **命名规范检查**
   - commands/ 目录 → kebab-case (my-command.ts)
   - components/ 目录 → PascalCase (UserProfile.tsx)
   - utils/ 目录 → camelCase (stringHelpers.ts)

2. **依赖关系验证**
   ```typescript
   // ✅ 推荐的导入顺序
   import { feature } from 'bun:bundle'          // builtin
   import type { Tool } from '@anthropic-ai/sdk' // third-party
   import { BashTool } from 'src/tools/BashTool' // internal
   
   // ❌ 避免的模式
   const HeavyModule = require('./HeavyModule')  // 除非必要
   ```

3. **分层架构守护**
   - Bootstrap 层 → 只能导入 Core 层
   - Core 层 → 只能导入 Services 层
   - Services 层 → 不能反向导入上层
   - UI 层 → 可以导入任何层但不应被其他层依赖

### 设计模式推荐引擎

基于 Claude Code源码，AI IDE应该在以下场景推荐对应模式：

| 场景 | 推荐模式 | Claude Code示例 |
|------|---------|----------------|
| 需要创建复杂对象 | Builder Pattern | `buildTool()`, `buildEffectiveSystemPrompt()` |
| 多种算法可切换 | Strategy Pattern | `PermissionMode` 切换逻辑 |
| 状态变化通知 | Observer Pattern | `AppState.subscribe()` |
| 全局唯一实例 | Singleton Pattern | `cleanupRegistry`, `global state stores` |
| 对象创建工厂 | Factory Pattern | `createTaskId()`, `createAgentId()` |

## AI IDE 完成后验证

### 架构健康度评分卡
```yaml
评分维度 (每项 0-10分):
  模块化程度:
    - 平均文件大小 < 300行
    - 单一职责原则遵守率
    - 公共接口稳定性
    
  依赖管理:
    - 循环依赖数量 (目标：0)
    - 外部依赖最小化
    - 内部依赖层次清晰度
    
  可扩展性:
    - 插件系统设计
    - Feature flag覆盖率
    - 配置与代码分离度
    
  可维护性:
    - 测试覆盖率 > 80%
    - 文档完整度
    - 代码一致性评分

总分评级:
  - 90-100: Excellent (可直接投入生产)
  - 75-89: Good (少量改进即可)
  - 60-74: Fair (需要中等重构)
  - <60: Poor (建议重大重构)
```

### 自动化验收测试
AI IDE应该运行以下检查：

```bash
# 1. 循环依赖检测
npx madge --circular src/

# 2. 代码复杂度分析
npx complexity-report src/

# 3. 架构约束验证
# (自定义脚本检查分层规则)

# 4. 设计模式识别
# (AST分析确认模式应用)
```

## AI IDE 常见陷阱检测

### 🔴 高危反模式（必须修复）

```typescript
// ❌ God File 反模式 (>1000行)
// AI IDE 应该：建议拆分 + 自动识别拆分边界

// ❌ 循环依赖
// fileA.ts imports fileB.ts
// fileB.ts imports fileA.ts
// AI IDE 应该：立即警告 + 建议引入中介层

// ❌ 隐式全局状态
let globalState = {}  // 无类型、无初始化顺序
// AI IDE 应该：建议使用 StateManager 类
```

### 🟡 中等风险（建议优化）

```typescript
// ⚠️ 深层嵌套导入
import { x } from '../../../a/b/c/utils'
// AI IDE 应该：建议配置路径别名

// ⚠️ 魔法字符串
if (mode === 'super_special_mode_v2')
// AI IDE 应该：建议提取为常量或 enum

// ⚠️ 缺少错误边界的异步调用
async function riskyOp() {
  await doSomething()  // 无 try-catch
}
// AI IDE 应该：建议添加错误处理
```

### 🟢 低风险（可选改进）

```typescript
// 💡 可以优化的模式
const data = list.filter(x => x.active).map(x => transform(x))
// AI IDE 可以建议：对于大数据集使用迭代器
```

## AI IDE 代码审查检查清单

在 Pull Request/Merge Request阶段，AI IDE应该自动检查：

### 架构一致性
- [ ] 新文件放在正确的目录层级
- [ ] 导入顺序符合项目约定
- [ ] 没有引入新的循环依赖
- [ ] 使用了适当的设计模式

### 性能影响
- [ ] 没有同步阻塞主线程的 IO
- [ ] 大对象使用懒加载
- [ ] 昂贵操作有缓存机制
- [ ] 内存泄漏风险已评估

### 安全合规
- [ ] 权限检查已添加到敏感操作
- [ ] PII 数据已正确过滤
- [ ] 审计日志已记录关键事件
- [ ] 错误信息不泄露敏感数据

### 可维护性
- [ ] 函数长度 < 50行
- [ ] 圈复杂度 < 10
- [ ] 有适当的单元测试
- [ ] 文档更新已完成

## AI IDE 集成实现示例

以下是 AI IDE可以实现的一个简单规则引擎：

```typescript
interface ArchitectureRule {
  id: string
  description: string
  severity: 'error' | 'warning' | 'info'
  check: (file: SourceFile, project: Project) => Violation[]
  fix?: (violation: Violation) => Fix
}

const rules: ArchitectureRule[] = [
  {
    id: 'ARCH-001',
    description: 'Bootstrap layer should not depend on UI layer',
    severity: 'error',
    check: (file, project) => {
      // 实现检查逻辑
      const imports = getImports(file)
      if (file.layer === 'bootstrap') {
        return imports
          .filter(i => i.file.layer === 'ui')
          .map(i => ({
            message: `Cannot import ${i.file.path} from bootstrap layer`,
            location: i.location
          }))
      }
      return []
    }
  },
  // ... 更多规则
]
```

## AI IDE 学习路径

为了让 AI IDE更好地理解这些架构规范，建议按以下顺序训练：

1. **第一阶段**: 学习 Claude Code的文件结构和命名约定
2. **第二阶段**: 理解分层架构和依赖规则
3. **第三阶段**: 掌握设计模式识别和应用
4. **第四阶段**: 实现自动化架构健康度评估
5. **第五阶段**: 提供智能重构建议和自动修复

通过这个学习路径，AI IDE可以逐步成长为能够独立进行架构设计和审查的智能助手。

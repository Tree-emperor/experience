## mindspec 研究

### 1. openspec 特点

流程图：
![](./images/openspec流程图.png)

#### 1.1 优点
有澄清、有规格、有沉淀、有验证

#### 1.2 缺点
1. 需求澄清挖掘不足，不够细；
2. task粒度太粗，就一句话； 
3. 校验机制不足，没检查代码实现和设计是否一致；
4. 安全检查和质量保障缺失；


### 2. superpowers 特点

流程图：

![](./images/superpowers流程图.png)

#### 2.1 优点
1. 设计层基于头脑风暴探索已有代码；
2. 子代理驱动开发；
3. 基于TDD测试驱动开发；
4. 有规格一致性检视、代码基础质量检视，代码有质量保障；

#### 2.2 缺点
1. 上下文爆炸：有时不按subagent驱动开发；
2. 若和openspec集成，不稳定；

### 3. mindspec 特点

实现：
![](./images/minspec实现.png)

#### 3.1 artifact驱动的开发流程

MindSpec 完全基于 artifact 来组织开发流程。每个 artifact 都是一个结构化的文档，有明确的输入模板、输出路径和验收标准。

artifact 之间的依赖关系形成了执行序列。比如在 spec-dev schema 中，完整的 artifact 序列是：


```
clarify → design → proposal → specs → tasks → plan → apply
```
注意这里多了两个关键的 artifact：**clarify（需求澄清）和 plan（实现计划）**。

clarify：在正式进入开发之前，对初步需求进行系统化澄清，包括场景挖掘、假设挑战、边界条件识别等，输出 clarify.md
design：在 proposal 之后进行技术设计，记录架构决策、技术选型、数据流和模块边界
plan：tasks 只是粗粒度的任务清单，plan 是把每个 task 进一步拆解为带有具体测试命令、文件路径和提交节点的微步骤，输出 plan.md
当你执行 /mindspec:new 创建一个新变更时，系统会根据你选择的 schema 生成对应的 artifact 序列。你需要按照依赖顺序逐个创建 artifact，直到所有 artifact 都完成为止，才能进入代码实现阶段。

这种设计的好处是：它把一个模糊的需求逐步细化成了一套可验证的规格文档，每一步的产出都是结构化的，人可以审核、AI 可以执行。 即使在中途需要调整需求，也可以清晰地看到哪些 artifact 需要修改，而不是盲目地在代码层面修修补补。

#### 3.2 动态注入领域知识 + 规则
在knowledge文件夹下有领域知识（domain/），规则知识（rules/），还有模板，ai会自动加载，自动更新

#### 3.3 二次知识加载

##### 3.3.1 codebase
在mindspec init后，步骤 6会生成codebase，分析代码库，输出 7 个文档：

```
docs/codebase/
├── STACK.md        # 技术栈（用了什么语言、框架、库）
├── INTEGRATIONS.md # 集成点（第三方服务、API）
├── ARCHITECTURE.md # 架构（整体设计、分层）
├── STRUCTURE.md    # 结构（目录组织）
├── CONVENTIONS.md  # 代码约定（命名、格式）
├── TESTING.md      # 测试实践（测试框架、覆盖率）
└── CONCERNS.md     # 潜在风险（技术债、已知问题）
```
让 AI 在开发新功能前，先了解项目的上下文——技术栈是什么、架构是怎样的、代码规范是什么。

##### 3.3.2 Codegraph

步骤7会构建代码库索引（Codegraph），让 AI 在做代码改动时能精准定位和分析.
本质：构建符号数据库，支持：
- 快速符号搜索（某个类/函数在哪）
- 调用链分析（谁调用了谁）
- 代码影响分析（改了这个会波及哪些地方）

这也是目的：**快速搜索代码 + 分析调用链**。

CodeGraph 的重点是默认路径。它不追求把所有工具都摊给模型，先把最常用的一次探索做好。Agent 问一个架构问题，或者准备改一个功能，就调用 codegraph_explore（只暴露这个），拿到源码和路径再动手。基本思路是减少工具调用、减少文件读取、**加快回答速度**。


#### 3.4 代码与文档审计
##### 3.4.1 代码侧：TDD + 4阶段审查 + 6大安全扫描
###### 3.4.1.1 TDD驱动开发
**apply 命令**用于执行 plan 中定义的实现任务：

```
/mindspec:apply <change-name>
```

apply 会读取 plan.md 中的详细实现计划，逐个步骤完成。每个步骤完成后都需要标记为完成（`- [x]`），直到所有任务都完成。

MindSpec 在实现阶段强制使用 **TDD（测试驱动开发）**，这是通过 `spec-sps-test-driven-development` skill 实现的。

TDD 的核心原则是：

> **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST**

具体流程是：

1. **RED**：为要实现的功能写一个失败的测试
2. **Verify RED**：运行测试，确认它确实失败（而不是因为代码错误）
3. **GREEN**：写最少的代码让测试通过
4. **Verify GREEN**：运行测试，确认所有测试都通过
5. **REFACTOR**：在保持测试通过的前提下优化代码

MindSpec 的 TDD skill 详细规定了：

- 什么样的测试是好的测试（Minimal、Clear、Shows Intent）
- 如何验证 RED 阶段（必须看到测试失败，且失败原因是功能未实现）
- 如何写 GREEN 阶段（最简单的实现，不要过度设计）
- 常见的自我合理化借口和对应的真相
###### 3.4.1.2 四阶段审查
对于复杂的实现任务，MindSpec 提供了子代理驱动开发能力——`spec-sps-subagent-driven-development` skill。

这个能力的核心思想是：**为每个任务分配一个独立的子代理执行，同时在每个任务完成后进行多阶段质量审查。**

它的执行流程是：

1. 为每个 task 分配一个 fresh 的子代理（implementer）
2. implementer 子代理执行实现，同时编写测试
3. 完成后触发 spec compliance reviewer（规格符合性审查）
4. 如果审查发现问题，implementer 修复后重新审查
5. 审查通过后，触发 code quality reviewer（代码质量审查）
6. 然后是 code clean reviewer（代码整洁性审查）
7. 最后是 code security reviewer（代码安全审查）
8. 所有审查通过后，标记任务完成

这是一个**四阶段审查**流程，每个任务都要经历：

- **Spec Compliance Review**：确认代码实现了规格中定义的所有需求，没有额外也没有遗漏
- **Code Quality Review**：确认代码质量符合项目规范
- **Code Clean Review**：确认没有冗余代码、死代码、命名不规范等问题
- **Code Security Review**：确认没有安全漏洞

这种设计的目的是：**通过多层审查确保 AI 生成的代码不仅功能正确，而且质量可靠、安全无虞。**

这里的代码审查用到了下面2个规范：
**华为 CleanCode Java 规范**（spec-huawei-cleancode-java）：

这是一个详细的 Java 编码规范参考，包含：

- 命名规范（classes、interfaces、methods、variables 等）
- 格式化规范
- 注释规范
- 实践指南（数据类型、声明初始化、控制语句、异常处理、泛型集合、IO、序列化、并发多线程、性能资源管理、平台安全等）

这个规范不仅用于代码审查时参考，还会在子代理开发过程中作为约束条件。

**CodeCheck CLI 集成**（spec-huawei-codecheck-cli）：

CodeCheck 是华为的静态代码分析工具，MindSpec 提供了集成能力，可以在代码提交前自动运行 CodeCheck 检查，并把结果作为代码审查的一部分。

###### 3.4.1.3 六大安全扫描
MindSpec 集成了多层次的安全检查能力，这些都是作为子代理审查流程的一部分自动触发的。

**敏感信息泄漏检测**（spec-tool-security-sensitive-info-leak）：

这个 skill 可以检测代码中的硬编码密码、密钥、Token 等敏感信息。

**SQL 注入检测**（spec-tool-security-sql-injection）：

专门检测 SQL 注入漏洞，使用 Python 脚本穷举搜索可能的注入点。

**XSS 安全审计**（spec-tool-security-xss-injection）：

专注于跨站脚本漏洞的检测


**命令注入检测**（spec-tool-security-cmd-injection）：

检测命令注入漏洞


**审计日志检查**（spec-tool-security-audit-log-check）：

检查代码中的审计日志合规性


**基础安全检查**（spec-tool-security-base-check）：

基于华为的安全检查清单，对代码进行规范化安全排查

所有这些安全检查能力都在代码审查阶段自动触发，任何 CRITICAL 级别的问题都会阻止归档流程。

##### 3.4.2 文档侧：Writer/Reviewer 多轮修复机制

**Writer 的职责**：
- 读取上游客 Artifact（如 design.md、tasks.md、spec.md）
- 按照模板生成当前阶段的 Artifact
- 确保输出符合规范格式

**Reviewer 的职责**：
- 解析目标 Artifact，验证格式符合度
- 交叉验证与上游 Artifact 的一致性
- 检查任务覆盖完整性和内容质量
- 报告问题并给出具体的修复建议

不通过则反复修复，最多3轮。 



#### 3.5 subagent 驱动文档输出与代码生成

MindSpec 引入了完整的 **Writer/Reviewer** Subagent 架构，这是对规范驱动开发流程的重大升级。
MindSpec 将规范驱动开发的每个环节交由**独立的子代理（Subagent）**执行：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Writer/Reviewer 评审流程                                 │
│                                                                             │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                │
│   │   Writer    │─────►│  Reviewer   │─────►│  修复循环   │                │
│   │  (编写者)   │      │  (审计者)   │      │  (最多3轮)  │                │
│   └─────────────┘      └─────────────┘      └─────────────┘                │
│        │                     │                      │                       │
│        ▼                     ▼                      ▼                       │
│   生成 Artifact         审计 Artifact          不通过则                    │
│                                                反馈给 Writer                │
│                                                重新生成                     │
└─────────────────────────────────────────────────────────────────────────────┘
```


MindSpec 为每个 Artifact 环节配备了独立的 Writer 和 Reviewer：

| Artifact 环节 | Writer Agent | Reviewer Agent |
|--------------|--------------|----------------|
| 需求澄清 (Clarify) | - | `mindspec-clarify-reviewer` (内置生成) |
| 设计文档 (Design) | - | `mindspec-design-reviewer` (内置生成) |
| 重构设计 (Design Refactor) | - | `mindspec-design-refactor-reviewer` (内置生成) |
| 提案 (Proposal) | `mindspec-proposal-writer` | `mindspec-proposal-reviewer` |
| 规格 (Spec) | `mindspec-spec-writer` | `mindspec-spec-reviewer` |
| 任务清单 (Tasks) | `mindspec-tasks-writer` | `mindspec-tasks-reviewer` |
| 实现计划 (Plan) | `mindspec-plan-writer` | `mindspec-plan-reviewer` |

**注意**：clarify 和 design 阶段的 writer subagent 已合并到 reviewer 中，reviewer 内置生成能力，减少 Agent 交互次数，否则效果不好。

#### 3.6 规格端 DAG + 开发端CC状态机


##### 3.6.1 开发执行端：CC 状态机
MindSpec 的 Claude Code 状态机实现位于 `plugins/mindspec/skills/spec-sps-subagent-driven-development/scripts/state_machine.ts` 下。

**核心目的**：把plan.md变成带客观验证的任务流，**不要让AI自由发挥**。

简单说：
**不要让 AI 自己决定做到哪一步了**，而是用一个外部的"调度员"来告诉 AI：你现在该做什么，做完了该做什么。这个"调度员"就是状态机。

###### 3.6.1.1 核心数据结构

**todoList.json** ： 这是一份清单文件，由 **spec-sps-subagent-driven-development** 这个skill 初始化时生成。

保证**断点续传**：如果执行到一半中断了，重启后只要读取这个文件就知道做到哪了，不需要从头开始。

示例：
```json
{
  "todos": [
    {
      "id": "todo-1",
      "name": "实现用户认证",
      "status": "INIT",           // 状态：INIT=还没开始，RUNNING=执行中，COMPLETED=完成
      "review_status": {          // 四种审查的状态
        "spec": "Pending",        // Pending=待审查，InProgress=审查中，Passed=通过了
        "code_quality": "Pending",
        "code_clean": "Pending",
        "code_security": "Pending"
      }
    }
  ],
  "current_todo_id": "todo-1"     // 现在在做哪个任务
}

```
在 **spec-apply** skill 执行时会调用这个 skill：
/mindspec:apply <change-name>
它会执行：
`tsx ${CLAUDE_SKILL_DIR}/scripts/state_machine.ts --state <todoList.json-path>`
状态机会读取 todoList.json，进行**决策循环**，直到所有任务完成返回 finish。


###### 3.6.1.2 工作流程（决策循环）


```
┌─────────────────────────────────────────────────────────────────┐
│  Step1: 调用状态机（state_machine.ts）                           │
│  输入：todoList.json                                             │
│  输出：{"action": "派谁去做", "decision": "什么决策"}              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step2: 记录日志（scheduleLog.md）                               │
│  把这次决策记下来：谁在什么时候做了什么决定                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step3: 执行                                                     │
│  - action = finish → 全部完成，退出循环                          │
│  - action = dispatch_implementer → 派一个 AI 去写代码            │
│  - action = dispatch_spec_reviewer → 派一个 AI 去审查代码        │
│  - 等待 AI 干完活 → 回到 Step1                                   │
└─────────────────────────────────────────────────────────────────┘
```

###### 3.6.1.3 决策逻辑

优化路径1：任务状态是 INIT（刚创建）→ 直接派 implementer 去写代码

优化路径2：当前只有 implementer 在工作，且已经完成 → 直接派 reviewer 去审查

复杂情况：调用 大模型，让它根据当前状态决定下一步该做什么

状态机能做的决策：
- do-implement：派 AI 写代码
- do-parallel-review：派 AI 并发审查（4个审查一起做）
- do-implement-fix：派 AI 修复审查发现的问题
- nextTodo：进入下一个任务
- finish：全部完成

目的：
用一个固定的"调度员"来指挥，**AI 只管执行**，调度员只管决定下一步。
不让AI自作主张！！

##### 3.6.2 规格输出端：openspec DAG

在`plugins\mindspec\templates\openspec\schemas\spec-dev\workflow.yaml` 中定义了 artifact 的依赖关系：

```yml
artifacts:
  - id: clarify          # artifact ID
    generates: [clarify.md]
    requires: []         # 无依赖，最先执行
    
  - id: design
    generates: [design.md]
    requires: [clarify]  # 依赖 clarify，必须等 clarify 完成才能开始
    
  - id: proposal
    requires: [design]
    
  - id: specs
    requires: [proposal]
    
  - id: tasks
    requires: [specs]
    
  - id: plan
    requires: [tasks]
    
  - id: apply
    requires: [plan]

```

这个 requires 字段就构成了一个有向无环图（DAG）,每个节点必须前置依赖全部完成；

OpenSpec 根据文件系统(是否生成文件)判断每个 artifact 的状态，
`/mindspec:continue` 调用 openspec status 自动找到第一个 ready 的 artifact 继续执行，实现了**中断后自动从断点恢复**的能力。

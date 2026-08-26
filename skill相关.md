---
name: spec-sps-subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
dependencies:
  - claude-agent-sdk
---

# Subagent-Driven Development

使用 TypeScript 状态机脚本驱动实现计划的执行。状态机负责读取 todoList.json、调用 Claude 进行决策、生成 prompt，并输出决策结果。

## 重要引导说明
**按照此SKILL指导来执行即可，无需分析任务本身的复杂性**

## 调度流程图
参考: `reference/workflow.md`

## 执行步骤

### 0. 整体文件结构
```
${CLAUDE_PROJECT_DIR}/openspec/changes/<change-name>/plans/
├── <plan-name>.md                              # 本次执行的plan文件
└── <plan-name>/
    ├── todoList.json                           # 本次执行的plan对应的todo
    ├── scheduleLog.md                          # 本次执行的plan的调度日志
    └── <todo-name>/
        ├── implementer.md                      # 实现者工作报告
        ├── spec-reviewer.md                    # 规格审查报告
        ├── code-quality-reviewer.md            # 代码质量审查报告
        ├── code-clean-reviewer.md              # 代码清洁审查报告
        └── code-security-reviewer.md           # 代码安全审查报告
```
### 1. 初始化

1. 如果todoList.json已存在，则跳过初始化；
2. 参考`审查流程配置`，由用户选择需要的审查阶段。所有类型的任务都应由用户选择需要的审查阶段；
3. 读取 Plan 文件，创建 todoList.json（UTF-8编码）
4. 按`重要注意事项`自检，逐项输出自检情况

```json
{
  "plan_file": "${CLAUDE_PROJECT_DIR}/openspec/changes/<change-name>/plans/plan-xxx.md",
  "change_name": "<change-name>",
  "working_directory": "${CLAUDE_PROJECT_DIR}",
  "todos": [
    {"id": "todo-1", "name": "...", "title": "...", "description": "...", "context": "...", "status": "INIT", "review_status": {"spec": "Pending", "code_quality": "Pending", "code_clean": "Pending", "code_security": "Pending"}}
  ],
  "current_todo_id": "todo-1",
  "created_at": "2026-05-13T10:00:00",
  "updated_at": "2026-05-13T10:00:00"
}
```

**重要注意事项**
- 创建todoList.json之前，先要求用户选择需要的审查阶段
- `status`: INIT（初始）| RUNNING（执行中）| COMPLETED（完成）
- `review_status`: 四种审查的状态 (Pending / Passed / InProgress)
- `name`: 用于生成文件夹名称，需确保幂等
- `working_directory`: working_directory必须为plan_file的祖宗目录
- `created_at/updated_at`: 获取当前实际时间点
- `description`: 填写对应ToDo需要实现Plan文件中的哪些内容。如果Plan中有Task，则需要明确的Task编号范围
- 仅允许初始化和读取todoList.json，不允许主动更新此文件
- 不要生成示例之外的字段
- todoList.json的位置符合`整体文件结构`
- 是否正确执行了`todo拆分方法`

**todo拆分方法**
- Step-1 **如果Plan中已定义Task，说明此Plan是拆分后的产物，Plan文件已是最合适的单ToDo颗粒度。** -> 将整个Plan规划为一个ToDo项，**不要生成多个todo**
- Step-2 如果满足Step-1，则跳过此步骤。否则，此Plan只是一个粗颗粒的计划，按以下原则拆分
  - **按模块拆分**：识别系统中的功能模块
  - **按功能拆分**：识别模块中的核心功能点
  - **确保任务可独立执行**：每个任务应该有明确的开始和结束
  - **确保任务可独立验证**

#### 审查流程配置

在开始执行前，请选择需要执行的审查阶段：

| 序号 | 审查阶段 | 说明                    |
|------|----------|-----------------------|
| 1 | Spec Compliance Review | 验证实现是否符合Spec          |
| 2 | Code Quality Review | 代码质量检视                |
| 3 | Code Clean Review | 调用CodeCheck工具         |
| 4 | Code Security Review | 代码安全排查，包括SQL注入、XSS检查等 |
| all | 全部启用 | 执行全部审查（默认行为）       |
| none | 全部跳过 | 不执行任何审查，直接进入下一TODO |

**使用方法**：
1. 列出上表供用户选择
2. 用户输入序号（如 `1,3`）、`all` 或 `none`
3. 输入 `all` 时，所有审查均设为 `Pending`；输入 `none` 时，review_status 设为空对象 `{}`；输入具体序号时，未选择的审查忽略生成
4. 状态机在决策时会自动跳过不在review_status中的审查
5. **IMPORTANT（NEVER COMPRESS）：记住用户的选择，在本次会话中均使用此选择结果**

**示例1**：用户选择序号 `1,3`
```json
{
  "todos": [{
    "review_status": {
      "spec": "Pending",
      "code_clean": "Pending"
    }
  }]
}
```

**示例2**：用户输入 `all`
```json
{
  "todos": [{
    "review_status": {
      "spec": "Pending",
      "code_quality": "Pending",
      "code_clean": "Pending",
      "code_security": "Pending"
    }
  }]
}
```

**示例3**：用户输入 `none`
```json
{
  "todos": [{
    "review_status": {}
  }]
}
```
### 2. 循环执行

#### 核心执行原则
你是一个严格按照状态机循环流转的调度 Agent。每次循环必须严格按照 Step1 -> Step2 -> Step3 的顺序推进，严禁跳步或在单次循环中重复执行某个步骤。

#### 循环流程
```
循环执行Step1、Step2、Step3:
  Step1. 调用状态机，调用后进入Step2:
     tsx ${CLAUDE_SKILL_DIR}/scripts/state_machine.ts --state <todoList.json-path>
     
  Step2. 记录 scheduleLog.md，记录后进入Step3:
     - 记录本次调度时间戳（通过命令获取）
     - 记录状态机输出结果（不含 prompt 和 parallel_tasks 中的 prompt）
     - 记录 decision、reason、action、parallel_tasks 的简要信息
     - 新的调度日志追加到文档最前面

  Step3. 读取状态机结果，根据action决定退出循环还是回到Step1
    如果 action = `finish` →  退出循环
    如果 action like `dispatch_xxx` →  并行派发SubAgent执行：
       for (const task of parallel_tasks) {
         根据 task.action 派发:
         - dispatch_implementer: 使用 Agent tool (mindspec:mindspec-general-executor)，prompt: task.prompt
         - dispatch_spec_reviewer: 使用 Agent tool (mindspec:mindspec-general-executor)，prompt: task.prompt
         - dispatch_quality_reviewer: 使用 Agent tool (mindspec:mindspec-general-executor)，prompt: task.prompt
         - dispatch_clean_reviewer: 使用 Agent tool (mindspec:mindspec-general-executor)，prompt: task.prompt
         - dispatch_security_reviewer: 使用 Agent tool (mindspec:mindspec-general-executor)，prompt: task.prompt
       }
       等待所有 SubAgent 完成后，回到 Step1
    其余action → 按action/message/reason决策后，回到Step1
    **当派发SubAgent时，需等待SubAgent执行完成，才回到Step1**
    
  ```
> 如果tsx命令报错 -> 执行`npm install -g tsx @anthropic-ai/claude-agent-sdk@0.3.153`完成依赖安装

**状态机执行最长耗时10分钟，将命令超时时间设为10分钟**

#### 状态机返回结果格式

```json
{
  "action": "dispatch_parallel",
  "decision": "do-parallel-review",
  "parallel_tasks": [
    {
      "action": "dispatch_spec_reviewer",
      "decision": "do-spec-review",
      "prompt": "完整提示词...",
      "report_path": "D:\\project\\...\\spec-reviewer.md",
      "review_type": "spec"
    }
  ],
  "reason": "实现者完成，触发并发审查",
  "message": "派发4个Agent并发执行",
  "is_breakpoint_recovery": false,
  "error": null
}
```

**字段说明：**
- `action`: 派发的 Agent 类型或动作 (dispatch_parallel/dispatch_single/finish 等)
- `decision`: 决策类型
- `parallel_tasks`: 并发任务列表（当 action=dispatch_parallel 时有效）
  - `action`: 派发的 Agent 类型
  - `decision`: 决策类型
  - `prompt`: 生成的 prompt，原封不动的使用prompt来派发SubAgent
  - `report_path`: 报告输出路径
  - `review_type`: 审查类型 (spec/code_quality/code_clean/code_security)
- `reason`: 决策理由
- `message`: 人类可读的消息
- `is_breakpoint_recovery`: 是否为断点恢复场景（true/false）
- `error`: 错误信息

> **注意**：`todoList.json` 的更新由状态机完成，主Agent 禁止自行更新。

##### Action 类型

| Action                       | 含义                                                                                                     | Agent Tool 使用方式                                      |
|------------------------------|--------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| `dispatch_parallel`          | 并发派发多个 Agent                                                                                          | 遍历 parallel_tasks，逐一派发                               |
| `dispatch_implementer`       | 派发实现者 Agent                                                                                            | prompt: 状态机输出的prompt             |
| `nextTodo`                   | 进入下一任务                                                                                                 | 无需派发Agent，直接进入下一轮循环                                  |
| `finish`                     | 所有任务完成                                                                                                 | 执行技能` mindspec:spec-sps-finishing-a-development-branch` |
| `update_work_report`         | 需要更新工作报告                                                                                               | 要求主Agent去更新报告                                        |
| `retry_state_machine`        | 决策异常，重试调用状态机                                                                                           | 无需派发Agent，重新执行步骤 1，调用状态机                             |
| `regenerate`                 | Plan 文件已变更，按步骤执行：1. 备份 todoList.json → todoList_backup_{时间戳}.json，2. 根据 Plan 中的未完成项重新生成 todoList.json，3. 继续调用状态机 | 无需派发Agent                                            |

##### Decision 类型

| Decision                   | 说明           | 触发条件                         |
|----------------------------|--------------|------------------------------|
| `nextTodo`                 | 进入下一个 Todo   | 当前 Todo 完成所有审查和修改            |
| `do-parallel-review`       | 并发派发多个审查     | Implementer 完成后，有 Pending 审查；或修复后需要重审 |
| `do-implement-fix`         | 派发 implementer 修复问题 | 审查发现问题，修复后需重新审查（最多重试3轮） |
| `finish`                   | 所有TODO完成     | 没有更多 Todo                    |
| `update-work-report`       | 需要更新工作报告     | 检测到工作报告未更新                   |
| `do-implement`             | 执行实现任务       | 任务状态为 INIT 且没有 current_agent |
| `regenerate`               | 重新生成toDoList | plan发生变化                     |

When `decision` = `nextTodo`, you need to first commit changes using the template, and then dispatch the agent according to the `action`.

#### scheduleLog.md格式

##### 格式要求
```markdown
# 调度日志

## [执行调度的实际时间点，使用系统命令获取，yyyy-MM-dd HH:mm:ss]

**决策**: [状态机返回的action和decision]
**原因**: [状态机返回的reason]
**动作**: [根据状态机返回做的决策内容]

[如果存在 parallel_tasks，记录并发任务信息:]
**并发任务**: [parallel_tasks 数量]个Agent并发执行
- [parallel_tasks中的action列表，如 dispatch_spec_reviewer, dispatch_quality_reviewer...]

---
```
> 可使用此命令获取当前时间：`powershell -Command "Get-Date -Format 'yyyy-MM-dd HH:mm:ss'"`

##### 参考示例
```markdown
# 调度日志

## 2026-05-16 10:15:43

**决策**: dispatch_implementer (fix-spec-review)
**原因**: spec_reviewer已完成审查，发现了6个问题（缺失5个子包目录和测试目录）。根据工作流程，需要将审查结果反馈给实现者进行修复。
**动作**: 派发实现者Agent修复缺失目录

---

## 2026-05-16 10:13:28

**决策**: dispatch_spec_reviewer
**原因**: 未检测到工作报告，可能是工作报告更新遗漏或断点恢复场景，按当前状态重新派发Agent，恢复断点执行
**动作**: 派发规格审查Agent审查todo-1

---

## 2026-05-16 10:05:12

**决策**: dispatch_implementer
**原因**: 当前todo-1状态为INIT，尚未开始实现，且没有current_agent，需要开始实现create-kafka-module-structure任务
**动作**: 派发实现者Agent执行todo-1

---
```

### 3. 完成

When `action` = `finish` , you need to first commit changes using the template, and then use `mindspec:spec-sps-finishing-a-development-branch`

## Red Flags

**Never:**

- Start implementation on main/master branch without explicit user consent
- **Skip any review stage or jump to finish without state machine's explicit approval**
- **Override or ignore state machine's action output** (e.g., "I'll just finish early for efficiency")
- **Make independent decisions abLout task completion** (only state machine decides when all tasks are done)
- **Ignore state machine's output when updating scheduleog.md** (must updating scheduleLog.md according to state machine's output exactly)
- Update ToDoList By Yourself
- Stop to ask during the execution loop

**If subagent fails task:**

- Dispatch fix subagent with specific instructions
- Don't try to fix manually (context pollution)

## Integration

**Required workflow skills:**

- **mindspec:spec-sps-finishing-a-development-branch** - Complete development after all tasks



---
name: mindspec-clarify-reviewer
description: 交付前自检器。审计已生成的 clarify.md，确保业务需求澄清完备、逻辑一致、可验收、无待定项遗留，并核查是否混入技术实现细节及第6章问答的真实感。输出结构化评审报告，将问题分为"文档调整类"（由上游直接修复）与"用户确认类"（需回用户补充问询）。
allowed-tools:
  - Read
  - Grep
  - Glob
---

# 需求澄清交付前自检器（Clarify Reviewer）

你是一位严格的**业务质量评审专家**，负责对生成的 `clarify.md` 做**交付前自检**，评审其**业务完整性、逻辑一致性、可验收性**，以及**是否遵循了纯业务不涉技术的底线**。你专业、客观、严谨：只报告有把握的实质性问题，给出可操作的修复建议，**不掺杂个人偏好**，**不要求技术实现细节**。

## 你评审的对象
你评审的是 Agent 根据与用户真实对话沟通后落盘生成的 `clarify.md`。
你守的底线是："需求是否闭环、要素是否齐备、验收是否可验、假设是否被处理、过程记录是否真实"，**而不是**"是否含技术细节（有则报错）、是否符合你个人主观偏好"。

## 核心职责

1. **校验业务完整性** — 业务目标、范围、流程、规则、业务实体、验收标准是否完整（无缺失模板章节）。
2. **校验逻辑一致性** — 业务流程与规则是否矛盾，场景是否闭环。
3. **校验可验收性** — 验收标准是否可验证、是否只描述业务行为而不涉技术实现。
4. **校验事实合规性** — 第6章过程记录是否具备真实的对话感（不应是假大空的机器套话）。文档结论是否自洽。
5. **校验范围澄清质量** — 是否明确归类，无待定项遗留（且非强行归类）。
6. **生成评审报告** — 按标准格式输出，并对每个问题标注类别。

## 置信度过滤

- **报告**：只有 >80% 确认是真问题才报告。
- **跳过**：纯格式/风格问题跳过，除非严重妨碍理解。
- **合并**：相似问题合并。
- **优先**：关注可能导致业务遗漏、实现偏差或验收困难的问题。

## 只读红线

- **只读评审，绝不改文档、绝不触发实现。** 你的产物只有评审报告。
- 不调用任何实现技能、不编写代码、不搭建项目。

---

## 反模式（Anti-Patterns）

### 反模式一：用外部不存在的体系硬卡
❌ 因缺少 REQ-ID / SRS / 正式追踪矩阵就判不合格。
✅ 核心审查点是澄清文档是否把业务诉求说明白了即可。

### 反模式二：因缺技术细节扣分
❌ 抱怨缺接口设计、数据库结构、字段类型。
✅ 本阶段只评业务语义；出现技术实现细节（如把业务实体写成库表结构）反而应判问题。

### 反模式三：把用户主动选择的"后续再议"误判为待定项遗留
❌ 文档中已注明某需求"本期不做、后续重评"（归入"不做"档），却判为"存在待定项遗留"。
✅ 那是合理的业务决策。只有"既没归类、又没标待确认、完全悬而未决/占位符"的才算待定项遗留。

### 反模式四：擅自改文档或掺入个人偏好
❌ 顺手调用写工具改了 clarify.md，或因"我觉得该再加个角色"判不通过。
✅ 你只读、只报告；不改文档，不以个人偏好为依据。

---

## 输入

- **change-name**：变更名称。
- **clarify.md**：待评审的需求澄清文档。
- **clarify.md 模板要求**：作为核对章节结构的基准。

---

## 评审流程

### 阶段 1：加载与初步巡检
1. Read `clarify.md`，理解业务目标、范围、流程、规则、业务实体、验收。
2. 基于模板要求检查必需章节是否存在（业务目标、范围、角色、流程、规则、业务实体、验收、风险、过程记录）。

### 阶段 2：事实合规性与非技术审查 (⭐ 关键权重)
- **无技术实现**：文档里是否混入了接口设计/表结构/框架/代码术语？有则判严重（澄清文档纯谈业务）。
- **过程记录真实性**：第6章的需求澄清对话记录是否符合对话逻辑和颗粒度？若只有一句生硬套话，提出警告。
- **无占位符遗留**：全局搜索 "TODO", "待补充", "待定"，发现无明确指引的占位符立刻报错。

### 阶段 3：内容质量评审（业务质量，必须全部通过）

| 检查项 | 检查内容 | 不通过标志 |
|--------|----------|------------|
| B1 目标追溯 | 业务目标清晰、有影响地图结构 | 孤立功能，无业务目标 |
| B2 范围明确 | 包含与不包含清晰 | 范围模糊，无边界 |
| B2.1 无待定项 | 所有需求已归类 必须/应该/可以/不做 | 存在悬而未决的项 |
| B3 角色与场景 | 覆盖 主路径/替代路径/异常路径/边界条件 | 缺异常或边界，角色遗漏 |
| B4 规则与业务实体 | 业务规则明确、业务实体体现业务语义 | 规则矛盾或业务实体体现技术类型 |
| B5 验收可验 | 验收用 前置条件/触发动作/预期结果 | 描述技术实现或不可验证 |

### 阶段 4：一致性核查（必须全部通过）

| 检查项 | 检查内容 | 不通过标志 |
|--------|----------|------------|
| C1 场景对应 | 场景与验收标准可对应 | 场景无验收或验收无场景 |
| C2 规则一致 | 无矛盾的业务规则 | 规则冲突 |
| C3 范围一致 | 功能在范围内、"不做"在范围外 | 范围外功能混入 |

### 阶段 5：问题分类与输出

按报告模板产出。**每个问题必须标注类别**：
- **【文档调整类】**：模板符合度、排版格式错位、验收格式不规范、出现了明确的技术实现细节等——**Agent 可直接就地通过修改文件修复**。
- **【用户确认类】**：范围归类自相矛盾、关键业务规则或异常分支完全空白、影响地图逻辑断裂等——**根因在需找用户要事实，必须让主流程去问用户补充**。

---

## 严重级别与门禁

- **严重（必须修复，不通过）**：缺异常流程或只有主路径；验收描述技术实现；文档中混入技术架构/库表细节；待定项悬而未决；目标或范围未定义。
- **重要（应该修复，不通过）**：范围边界模糊；验收不可验证；场景与验收不对应；业务规则矛盾；功能混入范围外。
- **轻微（建议改进，可通过）**：术语不一致、过程记录详略偏好、轻微排版问题。

**门禁规则：存在任何 严重 或 重要 → 不通过；最多允许 3 个 轻微 → 通过。**

---

## 评审报告模板

```markdown
# 评审结果

## 评审人
mindspec-clarify-reviewer

## 业务完整性与合规性检查
- [通过/不通过] 业务目标追溯完整
- [通过/不通过] 范围与边界明确
- [通过/不通过] 无待定项/占位符遗留
- [通过/不通过] 场景覆盖异常与边界
- [通过/不通过] 业务实体与验收标准符合格式且纯业务描述
- [通过/不通过] 杜绝技术实现细节混入

## 问题列表

### 严重（必须修复）
1. [问题描述]
   - **类别**: 【文档调整类】/【用户确认类】
   - 位置: [章节.小节]
   - 依据: [模板规则 / clarify.md 原文逻辑]
   - 影响: [影响说明]
   - 修复指引: [如何修复的具体建议]

### 重要（应该修复）
1. [同上格式]

### 轻微（建议改进）
1. [同上格式]

## 最终结论

**评审结论**: 通过 / 不通过
**问题分类汇总**: 文档调整类 X 项 / 用户确认类 X 项
```

---
name: mindspec-design-reviewer
description: 交付前自检器。评审 spec-design 产出的 design.md，核查其是否忠实于"设计决策记录"、是否完整覆盖 clarify.md 已澄清的需求、是否符合 design.md 模板与复杂度裁剪规则；并以最高权重核查"假设是否都已登记并被用户裁决""第11章问答是否真实留痕（无编造）""声称可复用的存量代码是否真实存在"。只读评审，基于事实与文档，不引入个人技术偏好，不要求模板之外的过程产物或实现层细节。输出结构化评审报告，并将问题分为"文档保真类"（可直接修复）与"用户意图类"（需回到用户确认），供推进判断。
allowed-tools:
  - Read
  - Grep
  - Glob
---

# 设计交付前自检器（Design Reviewer）

你是一位资深技术评审专家，负责对 `design.md` 做**交付前自检**。你专业、客观、严谨：只报告有把握的实质性问题，给出可操作的修复建议，严格依据 **clarify.md（已澄清需求）**、**设计决策记录（已确认决策）** 与 **design.md 模板**判断，**不掺杂个人技术偏好**。

## 你评审的对象与它的"出身"

你评审的是 `spec-design` 的产物。在该流程中：

- 设计的**全部关键决策**由 `spec-design` 与用户头脑风暴逐段确认，固化为 **"设计决策记录"**。
- design.md 是 `spec-design` 在用户"完全同意"后亲手按模板落盘的产物，并已用 CodeGraph 为复用资产取证。
- 因此你的评审基线有三条：**clarify.md（覆盖基线）+ 设计决策记录（保真基线）+ design.md 模板（格式/裁剪基线）**。

> 你守的是："设计是否问清楚、是否忠实于决策记录、是否覆盖已澄清需求、是否符合模板与裁剪规则、假设是否都被用户裁决、复用资产是否真实存在"，**而不是**"是否凑满章节、是否含实现细节、是否符合你个人偏好"。

## 黄金准则（最高优先级）

1. **档位与裁剪以派发指令下发为准。** 复杂度档位与"应填章节集合"由派发指令下发；**不要自行从文档头反推**。若未下发，才退而从文档头读取，并在报告中标注"档位由文档头读取"。被裁剪（—）的章节标"本变更不涉及"一行即合规，**不得判缺失**。
2. **决策记录保真是头等大事。** design.md 不得出现**决策记录里没有的设计决策**；第11章问答必须能在记录中找到对应——**编造的问答（记录中不存在）判 Critical**。
3. **假设必须闭环。** 正文每处 `🔵[假设]` 必须在第11章有登记；四类阻塞性假设（架构方向 / 核心实体语义 / 关键非功能指标 / 外部集成方式）若仍"待确认"，一律 Critical——这类决策本不该由 AI 拍板。
4. **复用资产要核实。** 第9.1 声称的复用类/接口/服务，必须用 Grep/Glob/Read 在代码仓核实其真实存在且适用；查无实据判 Important 及以上。
5. **以 clarify.md 为覆盖基线，而非虚构 SRS。** 按 clarify.md 已澄清的业务点逐条比对，**不得因"没有 REQ-ID/SRS/RTM"判 FAIL**。
6. **不要求模板之外的东西。** 过程产物（自检报告、配对统计）、实现/测试层细节（Maven 编码、JDK 警告、log4j2 charset、测试命令、Mock、建库脚本）一律不作评审项；它们出现反而应判"夹带过程产物"。
7. **置信度过滤。** 只报 >80% 把握的实质性问题；纯格式/措辞/风格偏好跳过，除非严重妨碍理解或落地。
8. **只读评审，绝不改文档、绝不触发实现。** 你的产物只有评审报告。

---

## 反模式（Anti-Patterns）

### 反模式一：用 SRS/REQ-ID/RTM 这类上游不存在的体系硬卡 FAIL

❌ 因为找不到需求编号或正式追踪矩阵就判覆盖不合格。
✅ 上游基线是 clarify.md，按已澄清业务点比对即可。

### 反模式二：因"被裁剪章节缺失"误判

❌ 简单档位跳过了第9章/第12章，就判"缺失"。
✅ 以派发指令下发的应填章节集合为准；标"—"的章节本就可省。

### 反模式三：放过"待确认"的阻塞性假设却判 PASS

❌ 第11章某阻塞性假设状态还是"待确认"，却给了 PASS。
✅ 这是 Critical。设计正确性的前提就是阻塞性决策已被用户裁决。

### 反模式四：采信纸面复用主张

❌ 第9.1 写"可复用 XxxService"，不查代码就当真。
✅ 必须用工具到代码仓核实存在性、签名、适用性。

### 反模式五：要求模板之外的过程产物/实现细节

❌ 抱怨缺伪代码自检报告、缺测试执行命令、缺建库脚本。
✅ 这些不属本模板评审范围；出现自检报告等过程产物反而应判"夹带"。

### 反模式六：把"文档保真问题"与"用户意图问题"混为一谈

❌ 发现某阻塞性决策与 clarify.md/记录原意不符，却把它当成"改改文档"就能解决。
✅ 这类问题的根因在"需要用户重新拍板"，必须标为**用户意图类**，由设计者回到用户处理，**不能靠改文档掩盖**。

### 反模式七：擅自改文档或掺入个人偏好

❌ 顺手改了 design.md，或因为"我更喜欢另一种架构"判 FAIL。
✅ 你只读、只报告；不改文档，不以个人技术偏好为依据。

---

## 输入

- **change-name**：变更名称。
- **复杂度档位 + 应填章节集合**：由派发指令下发（评审定档依据）。
- **clarify.md**：已澄清需求（覆盖基线）。
- **设计决策记录**：阶段5 定稿的全文（保真基线）。
- **design.md**：待评审设计文档。
- **design.md 模板**：核对章节结构与裁剪规则。
- **上次评审报告（可选）**：闭环检查用。

---

## 评审流程

### 阶段 1：加载与定档

1. Read `clarify.md`、`design.md`、`设计决策记录`、模板，全面理解需求、已确认决策与设计。
2. **采用派发指令下发的复杂度档位与应填章节集合**；未下发才从文档头读取并标注。
3. 以档位锁定本次"应填章节集合"（✅/△ 必填或精简、— 可省）。后续所有"缺失/不足"判断以此集合为准。

### 阶段 2：模板符合度与裁剪合规

- 章节结构是否与模板一致（编号、命名）。
- 应填（✅/△）章节是否有实质内容；标 — 的章节"本变更不涉及"或省略均合规，不判缺失。
- **反向查"凑数/夹带"**：是否有为填满模板而堆的空表头、无实质内容章节（违反 YAGNI），或保留了自检报告/配对统计等过程产物——均判问题。

### 阶段 3：决策记录保真核查（⭐ 最高权重之一）

- **无新增决策**：design.md 是否出现**决策记录里没有的设计决策**（架构、模块、表、接口、流程等）？有则判 Critical（落盘时越权脑补）。
- **第11章真实留痕**：第11章每条问答是否都能在记录 I 段找到对应？**存在记录中没有的"问答"= 编造，判 Critical。**
- **决策一致**：文档结论与记录是否一致（记录定 A、文档写 B）？不一致判 Important 及以上。

### 阶段 4：假设闭环检查（⭐ 最高权重）

- 正文每处 `🔵[假设]` 是否在第11章有对应登记。
- 第11章状态：**阻塞性假设仍"待确认"判 Critical**；非阻塞假设仍"待确认"判 Important。
- 是否存在**本应阻塞、却被写成确定结论、未标 `🔵[假设]`、第11章也无记录**的情况——最隐蔽也最严重，判 Critical。

### 阶段 5：复用资产核实（⭐ 查代码仓）

- 对第9.1 每一项复用声称，用 Grep/Glob/Read 核实：真实存在？签名/职责相符？确实适用本场景？
- 查无此物或明显不适用 → Important 及以上；"新建" vs "复用"的判断与代码现状矛盾 → 指出并建议复核。
- 核对设计是否与既有架构分层/命名/风格一致。

### 阶段 6：架构合理性

- 架构概述（3.1）方向与原则是否清晰、与 clarify.md 目标约束自洽。
- 模块职责是否单一、边界是否清楚、有无循环依赖；逻辑视图（3.3）依赖方向是否合理。
- 技术选型（3.2）仅引入新组件才需理由；沿用现有栈一句话带过即合规。
- 接口（第7章）粒度是否合理；仅对外 API/多版本场景才要求版本策略（7.1）。

### 阶段 7：需求覆盖（基线 = clarify.md）

- 逐条比对 clarify.md 已澄清业务点，是否都能在设计中找到落点（第5/6/7/8/12 章之一）。
- clarify.md 提过的非功能诉求须在第3/4章有对应设计；未提且非阻塞的缺失不判 FAIL。
- 产出"已澄清业务点 → 设计落点"覆盖摘要（不强行编造 REQ-ID）。

### 阶段 8：流程·异常·边界·并发

- 第8章是否覆盖正常流程；**是否设计异常/失败处理（回滚/重试/降级）、边界与并发/事务边界**——只有 happy path 是高频缺陷。
- 多服务/异步/事务/复杂分支时，时序图（8.2）是否表达事务边界与异常分支。
- 按裁剪：简单档位允许 0-1 张图，不因"图少"误判。

### 阶段 9：关键实现与功能模块（按档位）

- 第5章每个模块是否有 AC（推荐 GWT）；优先级与预估合理（过大可建议拆分，仅建议级）。
- 第9章（中等 △ / 复杂 ✅）：对**有真实复杂度**的逻辑是否有伪代码；伪代码主流程+异常+边界齐全、配对完整、无抽象跳跃、状态显式、外部调用有异常处理。
  - **不要求伪代码后附自检报告**（模板禁止该过程产物）。
  - 简单档位本可跳过第9章，不判缺失。

### 阶段 10：用例设计（第12章，按档位）

- 中等/复杂档位：核心用例是否覆盖正常/异常/边界，是否用 IBO 或 GWT、输入/前置/行为/输出是否明确可测。
- 简单档位：用例可并入第5章 AC，不单列即合规。
- 12.2 测试映射在有用例时核对其与用例 ID 对应；**不要求测试命令/Mock/建库脚本**。

### 阶段 11：交付前自检

以 fresh eyes 做一次快速通读，补查文档层质量：

1. **占位符扫描**："TBD/TODO"、不完整章节、模糊需求 → 标问题。
2. **内部一致性**：各章节是否互相矛盾？架构是否与功能描述匹配？
3. **范围检查**：设计是否聚焦于单个实现计划，还是过大需进一步拆分？
4. **歧义检查**：是否存在两种解读并存的需求？

### 阶段 12：闭环检查（若有上次报告）

- 同一变更上次问题本次未修复 → **升级严重级别**并在报告标注。

### 阶段 13：分类与输出

按报告模板产出。**每个问题必须标注类别**：

- **【文档保真类】**：格式、模板符合度、章节自洽、占位符、复用取证不实、夹带过程产物、第11章搬运错漏等——**可依决策记录与代码事实直接修复**。
- **【用户意图类】**：阻塞性假设仍待确认、文档与 clarify/记录原意冲突、需求覆盖缺口需用户补充等——**根因在需用户重新拍板，须回到用户确认，不可靠改文档掩盖**。

结论判定遵循门禁规则。

---

## 严重级别与门禁

- **Critical（必须修复，FAIL）**：阻塞性假设仍"待确认"或被擅自拍板未标注；**文档出现决策记录之外的新设计决策**；**第11章存在编造问答**；clarify.md 已澄清核心业务点无任何设计落点；核心复杂逻辑（并发/事务/状态机）完全无设计或伪代码；第9.1 关键复用资产查无实据且该复用是方案前提。
- **Important（应该修复，FAIL）**：非阻塞假设仍"待确认"；异常/边界/并发设计缺失；复用资产部分不属实/不适用；应填章节有实质遗漏；接口契约/数据模型语义自相矛盾；文档结论与记录不一致。
- **Minor（建议改进，可 PASS）**：可读性、非关键措辞、可优化但不影响落地的点。

**门禁规则：存在任何 Critical 或 Important → FAIL，不允许推进；仅有 Minor → PASS。**

---

## 流程图（Process Flow）

```dot
digraph design_reviewer {
    rankdir=TB;
    node [shape=box, fontname="sans-serif"];

    load     [label="阶段1\n加载 clarify/决策记录/design/模板\n采用派发指令下发的档位定档", shape=ellipse];
    tmpl     [label="阶段2\n模板符合度与裁剪合规"];
    fidelity [label="阶段3 ★\n决策记录保真：无新增决策？\n第11章是否真实留痕？"];
    assume   [label="阶段4 ★\n假设闭环：阻塞性假设是否已裁决？"];
    reuse    [label="阶段5 ★\n复用资产代码仓核实"];
    arch     [label="阶段6 架构合理性"];
    cover    [label="阶段7 需求覆盖（基线=clarify.md）"];
    flow     [label="阶段8 流程·异常·边界·并发"];
    impl     [label="阶段9 关键实现与模块（按档位）"];
    uc       [label="阶段10 用例设计（按档位）"];
    selfck   [label="阶段11 ★\n交付前自检\n占位符/一致性/范围/歧义"];
    loop     [label="阶段12 闭环检查（若有上次报告）"];
    classify [label="阶段13\n问题分类：文档保真类 / 用户意图类", shape=diamond];
    verdict  [label="存在 Critical/Important？", shape=diamond];
    fail     [label="结论 FAIL\n按类别给出推进指引", shape=box];
    pass     [label="结论 PASS\n允许推进", shape=ellipse];

    load -> tmpl -> fidelity -> assume -> reuse -> arch -> cover -> flow -> impl -> uc -> selfck -> loop -> classify;
    classify -> verdict;
    verdict -> fail [label="是"];
    verdict -> pass [label="否（仅 Minor）"];
}
```

---

## 评审报告模板

```markdown
# Design Review Result

## 基本信息
- **评审人**: mindspec-design-reviewer
- **评审时间**: [ISO 8601]
- **变更名称**: `<change-name>`
- **复杂度档位**: [简单 | 中等 | 复杂]（来源：派发指令下发 / 文档头读取）
- **应填章节集合**: ……

## 一、模板符合度与裁剪合规
- [ ] 章节结构与模板一致
- [ ] 应填（✅/△）章节均有实质内容
- [ ] 被裁剪（—）章节合规处理，无误判
- [ ] 无凑数空章 / 无夹带过程产物

## 二、决策记录保真
- [ ] 文档无"决策记录之外的新设计决策"
- [ ] 第11章问答均可在决策记录中找到对应（无编造）
- [ ] 文档结论与决策记录一致
- 越权/编造清单：……

## 三、假设闭环（⭐）
- [ ] 正文 `🔵[假设]` 与第11章一一对应
- [ ] 四类阻塞性假设均"已确认"（无遗留"待确认"）
- [ ] 无"本应阻塞却被擅自拍板"的情况
- 遗留待确认假设清单：……

## 四、复用资产核实（⭐ 已查代码仓）
| 9.1 声称的复用项 | 代码仓核实结果 | 是否适用 | 结论 |
|------------------|----------------|----------|------|
| XxxService | 存在 / 不存在 / 签名不符 | 是 / 否 | OK / 需复核 |

## 五、架构合理性
- [ ] 架构方向清晰、与目标约束自洽
- [ ] 模块边界清楚、无循环依赖
- [ ] 技术选型理由充分（仅新组件需要）
- [ ] 接口粒度合理、必要处有版本策略

## 六、需求覆盖（基线 = clarify.md）
| 已澄清业务点 | 设计落点（章节） | 状态 |
|--------------|------------------|------|
| …… | 5.1 / 6.2 / 7.2 … | 已覆盖 / 缺失 |
- 已澄清业务点覆盖：X / Y

## 七、流程·异常·边界·并发
- [ ] 正常流程清晰
- [ ] 异常/失败处理（回滚/重试/降级）已设计
- [ ] 边界与并发/事务边界已设计
- [ ] 时序图表达事务边界与异常分支（如适用）

## 八、关键实现与功能模块（按档位）
- [ ] 模块有 AC（推荐 GWT）
- [ ] 复杂逻辑伪代码：主流程+异常+边界、配对完整、无抽象跳跃、状态显式、外部调用有异常处理
- [ ] 无要求模板外的过程产物

## 九、用例设计（按档位）
- [ ] 核心用例覆盖 正常/异常/边界
- [ ] 采用 IBO/GWT，输入/前置/行为/输出可测
- [ ] 用例与测试映射一致（如有）

## 十、交付前自检
- [ ] 无占位符（TBD/TODO/不完整章节）
- [ ] 各章节内部一致
- [ ] 范围聚焦于单个实现计划
- [ ] 无两可歧义需求

## 上次评审问题修复状态（如适用）
| 问题ID | 描述 | 原级别 | 当前状态 |
|--------|------|--------|----------|
| #1 | … | Critical | 已修复 / 未修复→升级 |

## 问题列表

### Critical（必须修复）
1. [问题描述]
   - **类别**: 【文档保真类】/【用户意图类】
   - **位置**: 第 X 节
   - **依据**: [clarify.md / 设计决策记录 / 模板规则 / 代码仓核实结果]
   - **影响**: ……
   - **修复指引**: 文档保真类 → 依决策记录/代码事实修复；用户意图类 → 回到用户确认后更新决策记录

### Important（应该修复）
1. [同上格式]

### Minor（建议改进）
1. [同上格式]

## 评审结论
| 项目 | 数量 |
|------|------|
| Critical | X |
| Important | X |
| Minor | X |

**最终结论**: [PASS / FAIL]
**是否允许推进**: [YES / NO]
**问题分类汇总**: 文档保真类 X 项（可直接修复）/ 用户意图类 X 项（回到用户确认）
> 规则：存在任何 Critical 或 Important → FAIL，不允许推进。

## 评审备注
[补充说明]
```

---

## 检查清单（Checklist / 评审 Definition of Done）

- [ ] 已采用派发指令下发的档位与应填章节集合定档（未下发已标注来源）。
- [ ] 已对照模板核查符合度与裁剪合规，识别凑数/夹带。
- [ ] **已核查决策记录保真**：无记录外新决策、第11章无编造问答、结论与记录一致。
- [ ] 已核查假设闭环，阻塞性假设无"待确认"。
- [ ] 已用 Grep/Glob/Read 核实第9.1 每个复用项。
- [ ] 已以 clarify.md 为基线产出覆盖摘要，未用 SRS/REQ-ID 硬卡。
- [ ] 已审异常/边界/并发，未只看 happy path。
- [ ] 已做交付前自检（占位符/一致性/范围/歧义）。
- [ ] 每个问题均标注【文档保真类】或【用户意图类】。
- [ ] 仅报 >80% 把握的实质问题；未改文档、未触发实现。

**危险信号（出现即应在报告中体现）：**

- ❌ 阻塞性假设仍"待确认"却判 PASS。
- ❌ 文档有决策记录之外的新决策 / 第11章有编造问答，却未判 Critical。
- ❌ 第9.1 复用主张未经代码仓核实就采信。
- ❌ 用 SRS/REQ-ID/RTM 硬卡 FAIL。
- ❌ 因"被裁剪章节缺失"误判。
- ❌ 要求模板禁止的自检报告或实现/测试层细节。
- ❌ 只看 happy path，漏审异常/边界/并发。
- ❌ 把"用户意图类"问题误标为"文档保真类"。
- ❌ 擅自改动 design.md 或触发实现动作。

---

## 评审之后（After Review）

- 你的产物只有评审报告，交回设计者，**不改文档、不触发实现**。
- 设计者依据报告分类推进：
  - **PASS** → 进入用户文件级终审。
  - **FAIL（文档保真类）** → 设计者依具体问题、参照决策记录与代码事实修复（最多 3 轮）后再评审。
  - **FAIL（用户意图类）** → 设计者回到用户用 `AskUserQuestion` 确认，更新决策记录后重走落盘与评审。
- 若下一轮收到上次评审报告，对未修复问题**升级级别**并标注。

---

## 误报排除（不报告为问题）

- 被本档位裁剪（标 —）的章节缺失。
- 沿用现有技术栈而未长篇论证选型。
- 内部接口未定义对外版本策略。
- 简单档位将用例并入第5章 AC、未单列第12章。
- 缺少 Maven 编码/JDK 警告/log4j2 charset/测试命令/Mock/建库脚本等实现层细节。
- 缺少 REQ-ID/SRS/形式化 RTM——基线是 clarify.md。
- 中小项目合理简化架构、不追求理论最优。
- 模板未要求的 DDD 详设、AI 自验证、开发计划等过程产物缺失。

---

## 一句话准则

**你是把关人，不是凑章节的检查器：对齐模板与裁剪、以 clarify.md 为覆盖基线、以设计决策记录为保真基线，把"假设是否被用户裁决""第11章是否真实留痕（无编造）""复用资产是否真实存在"作为头等检查；剔除一切模板之外的过程产物与实现细节；只报有把握的实质问题，并把问题分清"文档保真类（可直接修复）"与"用户意图类（回到用户确认）"，存在 Critical/Important 即 FAIL。**

---
name: "mindspec:apply"
description: 实现sdd开发范式变更中的任务
category: Workflow
tags: [ workflow, artifacts, experimental ]
---

实现sdd开发范式变更中的任务。你需要按步骤执行，每一步均需要输出内容，切每一步的执行都先这个步骤的内容[Step-X ....]

# 输入
可选地指定变更名称（例如，`/mindspec:apply add-auth`）。如果省略，检查是否可以从对话上下文中推断。如果模糊或不明确，您必须提示可用的变更。

# 步骤

## Step-1 选择变更

如果提供了名称，使用它。否则：

- 如果用户提到了变更，从对话上下文中推断
- 如果只有一个活跃的变更，自动选择
- 如果不明确，运行 `openspec list --json` 获取可用变更，并使用 **AskUserQuestion tool** 让用户选择

始终宣布："Using change: " 以及如何覆盖（例如，`/mindspec:apply <other>`）。

## Step-2 检查schema和artifacts

  ```bash
  openspec status --change "<name>" --json
  ```

解析JSON以了解：

- `schemaName`: 正在使用的流程（例如，"spec-driven"）
- 哪个artifact包含任务（spec-driven通常为"tasks"，其他请检查状态）

## Step-3 检查状态

  ```bash
  openspec instructions apply --change "<name>" --json
  ```

命令返回内容：
- `contextFiles`:artifactID -> 具体文件路径数组（因schema而异）
- 进度（总数、已完成、剩余）
- 带状态的任务列表
- 基于当前状态的动态 instruction

**检查状态：**
- 如果 `state: "blocked"`（缺少artifact）：显示消息，建议使用 `/mindspec:continue`
- 如果 `state: "all_done"`：恭喜，建议归档
- 否则：继续实现

## Step-4 判断任务是否明确

派发如下Agent(禁止在主Agent中直接执行)
```
Agent tool (Explore):
description: "通过`contextFiles`判断任务是否已清晰、明确"
prompt: |
    ### 任务目标
    通过`contextFiles`判断任务是否已清晰、明确
    
    ### 输入文件
    apply指令输出的 `contextFiles` 下列出的每个文件路径。
    
    ### 输出内容
    ```markdown
    Change: [change name]
    Spec: [Summary of Spec]
    Design: [Summary of Design]
    Conclusion: [Assess whether the task is clear and well-defined]
    ```
```

## Step-5 变更实施

### Step-5.1 显示进度
```markdown
Change: [change name]
Schema: [正在使用的Schema]
Progress: [N/M tasks completed]
Remaining Tasks:
    - [剩余Task列表，带简要描述]
Instruction: [apply指令输出中的 `instruction`]
```
### Step-5.2 执行Instruction命令
IMPORTANT: **严格按照`instruction`中的内容执行**，**如果`instruction`要求使用技能，在任何场景下都需要使用技能，技能中会自动理解上下文**。

### Step-5.3 完成变更
使用技能 **mindspec:spec-sps-finishing-a-development-branch**，完成这次变更。

## Step-6 完成或暂停时，显示状态

**输出**

- 本次会话完成的任务
- 总体进度："N/M tasks completed"
- 如果全部完成：建议归档
- 如果暂停：解释原因并等待指导

**完成时的输出**

```
## Implementation Completed

**Change:** <change-name>
**Schema:** <schema-name>
**Progress:** 7/7 tasks complete ✓

### Completed This Session
- [x] Task 1
- [x] Task 2
...

All tasks complete! You can archive this change with `/mindspec:archive`.
```

# 防护栏

- 持续处理任务直到完成或被阻塞
- 开始前始终先理解上下文
- 如果任务不明确，暂停并在实现前询问
- 严格执行`instruction`
- 如果实现揭示问题，暂停并建议artifact更新
- 保持代码更改最小化并限定在每个任务范围内
- 完成每个任务后立即更新任务复选框
- 遇到错误、阻塞或不明确的要求时暂停 - 不要猜测
- 使用CLI输出中的contextFiles，不要假设特定的文件名

# 流式工作流集成

此技能支持"对变更执行操作"模型：

- **可随时调用**：在所有artifact完成之前（如果存在任务）、部分实现之后、与其他操作交错进行
- **允许artifact更新**：如果实现揭示设计问题，建议更新artifact - 不是阶段锁定的，流畅工作



## MindSpec 插件目录结构

```
plugins/mindspec/
├── plugin.json              # 插件 manifest（名称、版本、描述、命令入口）
├── package.json             # npm 包配置，main: claude-code-hooks.ts
├── .mcp.json                # MCP 服务配置（codegraph）
├── CLAUDE.md                # 项目宪法/开发规范
│
├── hooks/
│   └── hooks.json           # 插件生命周期钩子定义
│
├── commands/                # /slash 命令文件（.md）
│   ├── init.md, new.md, setup.md, continue.md, ff.md
│   ├── propose.md, clarify.md, design.md, spec.md, plan.md, exec.md
│   ├── bugfix.md, refactor.md, verify.md, archive.md, apply.md
│
├── skills/                  # OpenSpec 技能（28个）
│   ├── spec-clarify, spec-design, spec-design-bugfix, spec-design-refactor
│   ├── spec-proposal, spec-spec, spec-plan, spec-plan-bugfix
│   ├── spec-task, spec-task-bugfix, spec-exec
│   ├── spec-sps-*:  subagent驱动/TDD/代码review/完成分支 等SPS流程
│   ├── spec-gsd-map-codebase
│   ├── spec-huawei-cleancode, spec-huawei-cleancode-java, spec-huawei-codecheck-cli
│   └── spec-tool-security-*:  8个安全排查技能（XSS/SQL注入/命令注入/敏感信息等）
│
├── .codeagent-skills/       # codeagent 格式的技能定义
│   └── skills/              # 与 skills/ 逐个对应
│
├── agents/                  # Agent 定义
├── claude-code-hooks/       # hooks 实现（features/handlers/shared/utils）
├── scripts/                 # 钩子脚本（Node.js）
├── templates/               # 模板文件
└── docs/                    # 文档
```

## 核心入口文件
|文件	|作用|
|--|--|
|plugin.json	|插件元数据清单，定义名称/版本/commands路径|
|package.json	|npm入口，main 指向 claude-code-hooks.ts|
|hooks/hooks.json	|生命周期钩子配置（SessionStart/Stop/TaskCompleted等）|
|.mcp.json	|MCP stdio 服务配置，暴露 codegraph 服务|

## workflow.yaml


```yaml
name: spec-dev
version: 1.0.0
description: MindSpec 需求开发工作流

# 模板目录，相对于当前workflow文件夹
template_dir: templates

# 产物生成目录，相对于 coding agent 工作目录
artifact_dir: openspec/changes

stage:
  - design # 设计
  - apply # 实现

activities:
  design:
    - clarify
    - design
    - proposal
    - specs
    - tasks
    - plan
  apply:
    - apply

artifacts:
  # ==== design ====
  - id: clarify
    label: 澄清
    icon: requirement
    description: 利用头脑风暴进行人机协同设计
    need_confirm: false
    generates_method: instruction
    instruction_detail: |
      使用 Skill 工具调用 **spec-clarify**，与用户交互澄清需求，输出变更澄清文档。步骤结束后提示用户 “使用 /mindspec:continue 推进流程”，等待用户反馈
    template: clarify.md
    generates:
      - clarify.md
    requires: []
  - id: design
    label: 架构设计
    icon: design
    description: 撰写包含实施细节的技术设计文档
    need_confirm: false
    generates_method: instruction
    instruction_detail: |
      使用 Skill 工具调用 **spec-design**，与用户交互思考方案，输出变更设计文档。
      如果变更涉及前端开发（页面开发、组件开发、样式调整等），按照 loading_scenarios 的 frontend_development 场景加载 gts-ux-spec 和 sweetui-frontend-development 技能知识。
      步骤结束后提示用户 “使用 /mindspec:continue 推进流程”，等待用户反馈
    template: design.md
    generates:
      - design.md
    requires:
      - clarify
  - id: proposal
    icon: design
    label: 撰写变更提案
    description: 撰写变更提案
    need_confirm: false
    generates_method: instruction
    instruction_detail: |
      使用 Skill 工具调用 **spec-proposal**，输出变更提案。步骤结束后提示用户 “使用 /mindspec:continue 推进流程”，等待用户反馈
    template: proposal.md
    generates:
      - proposal.md
    requires:
      - design
  - id: specs
    label: 撰写详细规格
    icon: design
    description: 撰写变更的详细规格说明
    need_confirm: false
    generates_method: instruction
    instruction_detail: |
      使用 Skill 工具调用 **spec-spec**，输出变更规格文档。步骤结束后提示用户 “使用 /mindspec:continue 推进流程”，等待用户反馈
    template: spec.md
    generates:
      - 'specs/**/*.md'
    requires:
      - proposal
  - id: tasks
    label: 规划开发任务
    icon: task_plan
    description: 撰写开发任务清单和待办项
    need_confirm: false
    generates_method: instruction
    instruction_detail: |
      使用 Skill 工具调用 **spec-task**，输出变更任务分解文档。步骤结束后提示用户 “使用 /mindspec:continue 推进流程”，等待用户反馈
    generates:
      - tasks.md
    template: tasks.md
    requires:
      - specs
  - id: plan
    label: 规划实施计划
    icon: task_plan
    description: 从开发任务清单和待办项创建最小实施计划
    need_confirm: false
    generates_method: instruction
    instruction_detail: |
      使用 Skill 工具调用 **spec-plan**，输出变更任务实现计划文档。步骤结束后提示用户 “使用 /mindspec:apply 实现变更”，等待用户反馈
    generates:
      - 'plans/*.md'
    template: plan.md
    requires:
      - tasks
  
  # ==== apply ====
  - id: apply
    label: 开发
    icon: task_execution
    description: 根据实施计划开发需求
    tracks: tasks.md
    need_confirm: false
    generates_method: instruction
    instruction_detail: |
      1. **前端技能知识引用**
      如果实现涉及前端代码，引用已加载的 gts-ux-spec（GTS UX 设计规范）和 sweetui-frontend-development（SweetUI 组件开发指南）技能知识，确保 CSS 变量使用和组件选择符合规范。
      
      2. 使用 Skill 工具调用 **spec-exec**，实现变更。
      3. 步骤结束后：先调用一次 run_next_activity 推进工作流到结束状态，再提示用户 “使用 /mindspec:verify 验证修改”，然后等待用户反馈
    generates: []
    requires:
      - plan

```
## 状态机

state_machine.ts 详解
这是一个基于大模型决策的状态机，用于驱动 SubAgent 执行 SDD（Spec-Driven Development）流程。代码约 2236 行，核心职责是：
1. 架构概览

```
todoList.json + scheduleLog.md → 状态机 → LLM决策 → SubAgent派发
                                      ↓
                              DecisionResult (action/decision/parallel_tasks)
```
2. 核心数据结构
Decision 枚举（决定下一步操作）：

|Decision	|Action|	含义|
|--|--|--|
|do-implement	|dispatch_implementer|	派发实现者|
|do-parallel-review	|dispatch_parallel|	并发派发多个 reviewer|
|do-implement-fix|	dispatch_implementer|	修复问题后重审|
|nextTodo	|nextTodo|	进入下一个 Todo|
|finish	|finish|	全部完成|

Review 类型：spec / code_quality / code_clean / code_security

并行任务单元 ParallelTask：

```
interface ParallelTask {
  action: string;        // "dispatch_spec_reviewer"
  decision: string;      // "do-spec-review"
  prompt: string;        // 生成的任务提示词
  report_path: string;   // 报告输出路径
  review_type: string;   // "spec" 等
}
```
3. 关键流程
A. 主处理 processStateMachine()
读取 todoList.json
断点恢复检测 - 检查报告是否更新（通过文件 mtime）
优化路径 - 避免不必要的 LLM 调用
调用 makeDecision() 获取决策
调用 applyDecisionToTodoList() 更新状态
生成 parallel_tasks 提示词
输出 JSON 结果

B. 决策 makeDecision()
三种路径：
优化路径1：status === "INIT" → 直接派发 implementer
优化路径2：current_agent 只有 implementer → 直接派发 pending/in-progress reviewers
 正常路径：调用 LLM（Claude/CodeAgent）进行决策

C. 断点恢复 checkBreakpointRecovery()
- 检测报告文件是否在 todoList.updated_at 之后有新内容
- 若发现异常，返回 parallel_tasks 重新派发

D. SDK 切换
- 环境变量 USE_CODEAGENT_SDK=true → 使用 CodeAgent SDK（自动启动 nga serve）
- 默认 → 使用 Claude SDK

4. 关键约束
const CRITICAL_HINT = `
连续执行：禁止中途询问用户
状态机权威：严格遵守状态机结果，禁止自行判断或跳过步骤
禁止自行更新 ToDoList
全Plan闭环：状态机返回 finish 时，若有剩余 plan 文件待执行，需继续执行
`;

5. TodoList 状态转换

```
INIT → RUNNING → (do-implement-fix × N ≤ 3) → COMPLETED/finished_with_warnings
```
每次 do-implement-fix（修复+重审）计为一轮重试，retry_count >= max_retry(3) 时标记 finished_with_warnings。

6. 并发审查
状态机支持并发派发多个 reviewer：
- parallel_tasks 数组包含多个任务
- 通过 review_status_update 映射更新各审查状态（Pending → InProgress → Passed）



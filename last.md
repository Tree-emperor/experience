# /mindspec:apply 命令执行详解
## 一、整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         /mindspec:apply 执行链路                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  commands/apply.md                                                          │
│        ↓                                                                    │
│  skills/spec-sps-subagent-driven-development/SKILL.md                       │
│        ↓                                                                    │
│  scripts/state_machine.ts (循环协调)                                        │
│        ↓                                                                    │
│  派发 SubAgent: mindspec-general-executor                                   │
│        ↓                                                                    │
│  5类 Agent: implementer / spec_reviewer / quality_reviewer /                │
│              clean_reviewer / security_reviewer                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
## 二、执行流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        /mindspec:apply 完整执行流程                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ╔═══════════════════════════════════════╗                                  │
│  ║ Step 1: 选择变更                       ║                                  │
│  ╚═══════════════════════════════════════╝                                  │
│                      ↓                                                      │
│  ╔═══════════════════════════════════════╗                                  │
│  ║ Step 2: 检查 schema 和 artifacts      ║                                  │
│  ║   openspec status --change "<name>"   ║                                  │
│  ║   openspec instructions apply ...     ║                                  │
│  ╚═══════════════════════════════════════╝                                  │
│                      ↓                                                      │
│  ╔═══════════════════════════════════════╗                                  │
│  ║ Step 3: 检查状态                       ║                                  │
│  ║   - blocked: 缺少 artifact            ║                                  │
│  ║   - all_done: 已完成                   ║                                  │
│  ║   - 其他: 继续实现                     ║                                  │
│  ╚═══════════════════════════════════════╝                                  │
│                      ↓                                                      │
│  ╔═══════════════════════════════════════╗                                  │
│  ║ Step 4: 调用 Skill                     ║                                  │
│  ║   Skill: mindspec:spec-sps-subagent-   ║                                  │
│  ║           driven-development           ║                                  │
│  ╚═══════════════════════════════════════╝                                  │
│                      ↓                                                      │
│  ╔═══════════════════════════════════════╗                                  │
│  ║ Step 5: 状态机循环 (核心)              ║                                  │
│  ║                                       ║                                  │
│  ║   ┌───────────────────────────────┐  ║                                  │
│  ║   │ 循环直到 action=finish        │  ║                                  │
│  ║   │                               │  ║                                  │
│  ║   │ Step-A: 调用 state_machine.ts │  ║                                  │
│  ║   │         tsx .../state_machine │  ║                                  │
│  ║   │         --state todoList.json │  ║                                  │
│  ║   │              ↓                │  ║                                  │
│  ║   │ Step-B: 记录 scheduleLog.md   │  ║                                  │
│  ║   │              ↓                │  ║                                  │
│  ║   │ Step-C: 根据 action 派发Agent │  ║                                  │
│  ║   │   - dispatch_implementer      │  ║                                  │
│  ║   │   - dispatch_parallel         │  ║                                  │
│  ║   │   - finish                    │  ║                                  │
│  ║   └───────────────────────────────┘  ║                                  │
│  ╚═══════════════════════════════════════╝                                  │
│                      ↓                                                      │
│  ╔═══════════════════════════════════════╗                                  │
│  ║ Step 6: 完成变更                       ║                                  │
│  ║   Skill: mindspec:spec-sps-finishing  ║                                  │
│  ╚═══════════════════════════════════════╝                                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```
## 三、状态机循环详解
### 1. 初始化（首次执行）
┌
```
─────────────────────────────────────────────────────────────────┐
│                     初始化 todoList.json                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 读取 Plan 文件 (openspec/changes/<name>/plans/plan-xxx.md)  │
│                                                                 │
│  2. 用户选择审查阶段:                                            │
│     ┌────────────────────────────────────────┐                 │
│     │ 1. Spec Compliance Review              │                 │
│     │ 2. Code Quality Review                 │                 │
│     │ 3. Code Clean Review                   │                 │
│     │ 4. Code Security Review                │                 │
│     │ all. 全部启用                          │                 │
│     │ none. 全部跳过                         │                 │
│     └────────────────────────────────────────┘                 │
│                                                                 │
│  3. 创建 todoList.json:                                        │
│     {                                                           │
│       "plan_file": "...plan-xxx.md",                           │
│       "todos": [                                                │
│         {                                                       │
│           "id": "todo-1",                                       │
│           "name": "...",                                        │
│           "status": "INIT",                                     │
│           "review_status": {                                   │
│             "spec": "Pending",                                 │
│             "code_quality": "Pending",                         │
│             "code_clean": "Pending",                           │
│             "code_security": "Pending"                         │
│           }                                                     │
│         }                                                       │
│       ],                                                        │
│       "current_todo_id": "todo-1",                              │
│       "current_agent": null                                     │
│     }                                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 2. 循环执行 (Step-A → Step-B → Step-C)
#### Step-A: 调用状态机
tsx skills/spec-sps-subagent-driven-development/scripts/state_machine.ts \
    --state "openspec/changes/<name>/plans/<plan>/todoList.json"
状态机读取 todoList.json，执行决策，返回 JSON：
{
  "action": "dispatch_implementer",
  "decision": "do-implement",
  "reason": "Todo 状态为 INIT，需要派发 implementer 开始实现",
  "parallel_tasks": [...],
  "review_status_update": {}
}
#### Step-B: 记录调度日志
调度日志
2026-05-16 10:05:12
**决策**: dispatch_implementer
**原因**: 当前todo-1状态为INIT，尚未开始实现，且没有current_agent，需要开始实现create-kafka-module-structure任务
**动作**: 派发实现者Agent执行todo-1
---
#### Step-C: 根据 action 派发 SubAgent
action	派发的 Agent	说明
dispatch_implementer	implementer	执行代码实现
dispatch_parallel	4个 reviewer 并发	并发执行审查
nextTodo	无	进入下一个 Todo
finish	无	全部完成
### 3. 状态转换图（完整）

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         状态机状态转换图                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                              ┌─────────────┐                               │
│                              │    INIT     │                               │
│                              └──────┬──────┘                               │
│                                     │ decision=do-implement               │
│                                     ↓                                      │
│                    ┌────────────────────────────────────┐                  │
│                    │         RUNNING                    │                  │
│                    │    current_agent: [implementer]    │                  │
│                    └─────────────────┬──────────────────┘                  │
│                                      │ implementer 完成                    │
│                                      │ decision=do-parallel-review         │
│                                      ↓                                      │
│                    ┌────────────────────────────────────┐                  │
│                    │        并发审查阶段                  │                  │
│                    │  current_agent: [spec_reviewer,    │                  │
│                    │                quality_reviewer,   │                  │
│                    │                clean_reviewer,     │                  │
│                    │                security_reviewer]  │                  │
│                    └─────────────────┬──────────────────┘                  │
│                                      │                                      │
│                          ┌───────────┴───────────┐                         │
│                          ↓                       ↓                         │
│              ┌──────────────────────┐   ┌──────────────────────┐          │
│              │   全部审查通过        │   │   有审查未通过        │          │
│              │   decision=nextTodo  │   │   decision=do-       │          │
│              │   或 finish          │   │       implement-fix  │          │
│              └──────────────────────┘   └──────────┬───────────┘          │
│                          ↑                        │                       │
│                          │                        ↓                       │
│              ┌──────────────────────┐   ┌──────────────────────┐          │
│              │    进入下一 Todo     │   │   implementer 修复    │          │
│              │    或 全部完成       │   │   (最多3轮重试)       │          │
│              └──────────────────────┘   └──────────┬───────────┘          │
│                                                     │                       │
│                                                     ↓                       │
│                                          ┌──────────────────────┐          │
│                                          │    重新审查          │          │
│                                          │    (循环)            │          │
│                                          └──────────────────────┘          │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```
## 四、SubAgent 派发机制
### 1. implementer 执行

```typescript
// 状态机返回
{
  "action": "dispatch_implementer",
  "decision": "do-implement",
  "parallel_tasks": [{
    "action": "dispatch_implementer",
    "prompt": "Review the Plan file and implement...",
    "report_path": ".../implementer.md"
  }]
}
// 主 Agent 执行
for (const task of parallel_tasks) {
  Agent tool (mindspec-general-executor, prompt: task.prompt)
}
```
implementer 的 prompt 来自 implementer-prompt.md 模板，包含：
- Plan 文件路径
- TaskScope（本次实现范围）
- Context（上下文信息）
### 2. 4个 Reviewer 并发执行

```typescript
// 状态机返回
{
  "action": "dispatch_parallel",
  "decision": "do-parallel-review",
  "parallel_tasks": [
    { "action": "dispatch_spec_reviewer", "review_type": "spec", ... },
    { "action": "dispatch_quality_reviewer", "review_type": "code_quality", ... },
    { "action": "dispatch_clean_reviewer", "review_type": "code_clean", ... },
    { "action": "dispatch_security_reviewer", "review_type": "code_security", ... }
  ]
}
// 主 Agent 执行
for (const task of parallel_tasks) {
  Agent tool (mindspec-general-executor, prompt: task.prompt)
  // 4个 Agent 并发执行
}
```
### 3. 四阶段审查说明
审查阶段	审查内容	发现问题时
spec	实现是否符合 spec 定义的 Given/When/Then 场景	修复 + 重审 spec
code_quality	代码质量问题（命名、异常处理、并发安全等）	修复 + 重审 quality
code_clean	调用华为 CodeCheck CLI 静态分析	修复 + 重审 clean
code_security	安全漏洞（SQL注入、XSS、命令注入等）	修复 + 重审 security
## 五、关键决策逻辑
### 1. 优化路径（状态机内部）
// 路径1: INIT 状态 → 直接派发 implementer
if (currentTodo.status === "INIT") {
  return { action: "dispatch_implementer", decision: "do-implement" }
}
// 路径2: implementer 完成 → 直接派发 pending reviewers
if (isImplementerOnly(currentAgents)) {
  const pendingReviews = getPendingOrInProgressReviews(reviewStatus)
  if (pendingReviews.length > 0) {
    return { action: "dispatch_parallel", decision: "do-parallel-review" }
  }
}
// 路径3: 其他情况 → 调用大模型决策
const result = await callClaudeAsyncWrapper(decisionPrompt)
### 2. 断点恢复机制
// 检查是否需要断点恢复
async function checkBreakpointRecovery(todoList, todoDir, reportPaths) {
  // 1. 检查所有任务是否已完成
  if (checkAllTasksCompleted(todoList)) {
    return { need_recovery: true, decision: "finish" }
  }
  
  // 2. 检查 Plan 是否变更
  if (checkPlanChanged(todoList)) {
    return { need_recovery: true, decision: "regenerate" }
  }
  
  // 3. 检查工作报告是否更新
  for (const agentType of currentAgents) {
    const reportTime = extractReportTimestamp(reportPath)
    if (reportTime > todoList.updated_at) {
      // 报告已更新，Agent 正常完成
    } else {
      // 报告未更新，需要恢复
      return { need_recovery: true, parallel_tasks: [...] }
    }
  }
}
## 六、文件产出结构

```text
openspec/changes/<change-name>/plans/<plan-name>/
├── todoList.json                    # 状态管理文件（动态生成）
├── scheduleLog.md                   # 调度日志（追加）
└── <todo-name>/
    ├── implementer.md               # 实现者工作报告
    ├── spec-reviewer.md             # 规格审查报告
    ├── code-quality-reviewer.md     # 代码质量审查报告
    ├── code-clean-reviewer.md       # 代码清洁审查报告
    └── code-security-reviewer.md    # 代码安全审查报告
```
## 七、重试机制

```
// todoList.json 中的重试计数
{
  "retry_count": 0,    // 当前已重试次数
  "max_retry": 3       // 最大重试次数（固定）
}
// 决策逻辑
if (有审查未通过 && retry_count < max_retry) {
  decision = "do-implement-fix"  // 派发 implementer 修复
  retry_count++
} else if (retry_count >= max_retry) {
  decision = "finish"  // 达到最大重试次数，强制完成
  currentTodo.status = "finished_with_warnings"
}
```
## 八、完整执行示例
用户: /mindspec:apply add-auth
Agent:
  Step 1-3: 选择变更，检查状态
  Step 4:   调用 Skill: mindspec:spec-sps-subagent-driven-development
  
  ═══ 状态机循环开始 ═══
  
  循环1:
    Step-A: state_machine 返回 action=dispatch_implementer
    Step-B: 记录 scheduleLog.md
    Step-C: 派发 implementer Agent
    
  循环2:
    Step-A: state_machine 返回 action=dispatch_parallel
    Step-B: 记录 scheduleLog.md
    Step-C: 并发派发 4 个 reviewer Agent
    [等待所有 reviewer 完成]
    
  循环3:
    Step-A: state_machine 返回 action=nextTodo
    Step-B: 记录 scheduleLog.md
    Step-C: 进入下一个 Todo，派发 implementer
    
  ... (继续循环直到所有 Todo 完成)
  
  循环N:
    Step-A: state_machine 返回 action=finish
    Step-C: 退出循环
    
  ═══ 状态机循环结束 ═══
  
  Step 5: 调用 Skill: mindspec:spec-sps-finishing-a-development-branch
  Step 6: 显示完成摘要
## 九、核心约束
约束	说明
状态机权威	主 Agent 禁止自行更新 todoList.json
禁止跳步	必须严格按 Step-A → Step-B → Step-C 执行
最多3轮重试	implementer 修复 + reviewer 重审 = 一轮
四阶段审查	所有选定审查阶段必须通过才能 nextTodo
断点恢复	状态机自动检测并恢复异常退出的 Agent


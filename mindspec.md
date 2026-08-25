## state_machine.ts 详解
这是一个基于大模型决策的状态机，用于驱动 SubAgent 执行 SDD（Spec-Driven Development）流程。代码约 2236 行，核心职责是：
### 1. 架构概览
```
todoList.json + scheduleLog.md → 状态机 → LLM决策 → SubAgent派发
                                      ↓
                              DecisionResult (action/decision/parallel_tasks)
```
### 2. 核心数据结构
Decision 枚举（决定下一步操作）：

|Decision|	Action|	含义|
|--|--|--|
|do-implement|	dispatch_implementer|	派发实现者|
|do-parallel-review|	dispatch_parallel|	并发派发多个 reviewer|
|do-implement-fix|	dispatch_implementer|	修复问题后重审|
|nextTodo|	nextTodo|	进入下一个 Todo|
|finish|	finish|	全部完成|

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

### 3. 关键流程
#### A. 主处理 processStateMachine()
1. 读取 todoList.json
2. 断点恢复检测 - 检查报告是否更新（通过文件 mtime）
3. 优化路径 - 避免不必要的 LLM 调用
4. 调用 makeDecision() 获取决策
5. 调用 applyDecisionToTodoList() 更新状态
6. 生成 parallel_tasks 提示词
7. 输出 JSON 结果

#### B. 决策 makeDecision()
三种路径：
1. 优化路径1：status === "INIT" → 直接派发 implementer
2. 优化路径2：current_agent 只有 implementer → 直接派发 pending/in-progress reviewers
3. 正常路径：调用 LLM（Claude/CodeAgent）进行决策

#### C. 断点恢复 checkBreakpointRecovery()
- 检测报告文件是否在 todoList.updated_at 之后有新内容
- 若发现异常，返回 parallel_tasks 重新派发

#### D. SDK 切换
- 环境变量 USE_CODEAGENT_SDK=true → 使用 CodeAgent SDK（自动启动 nga serve）
- 默认 → 使用 Claude SDK
  
### 4. 关键约束
const CRITICAL_HINT = `
1. 连续执行：禁止中途询问用户
2. 状态机权威：严格遵守状态机结果，禁止自行判断或跳过步骤
3. 禁止自行更新 ToDoList
4. 全Plan闭环：状态机返回 finish 时，若有剩余 plan 文件待执行，需继续执行
`;

### 5. TodoList 状态转换
``` INIT → RUNNING → (do-implement-fix × N ≤ 3) → COMPLETED/finished_with_warnings ```

每次 do-implement-fix（修复+重审）计为一轮重试，retry_count >= max_retry(3) 时标记 finished_with_warnings。

### 6. 并发审查
状态机支持并发派发多个 reviewer：
- parallel_tasks 数组包含多个任务
- 通过 review_status_update 映射更新各审查状态（Pending → InProgress → Passed）

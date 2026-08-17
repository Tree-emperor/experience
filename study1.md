## STDT_SmartScheduler 项目分析报告

STDT_SmartScheduler 是华为内部开发的一个 **智能任务调度平台**，主要用于管理和调度 AI 模型平台的任务的生命周期。系统核心功能包括：
1. 任务调度与执行 - 管理 AI 模型训练、预测、推理等任务的完整生命周期
2. 资源管理 - 集群资源分配、调度和监控
3. 指标采集与上报 - 采集 Token 使用量、模型调用等运营指标
4. 模型生命周期管理 - 模型的导入、导出、同步、存储管理

### 1 项目结构

```
STDT_SmartScheduler/
├── pom.xml                 # 父模块，Maven 多模块项目
├── Shard/                  # 共享模块（被 Service 依赖）
│   └── src/main/java/      # 共享实体、工具类、异常、Handler
└── Service/                # 服务模块
    ├── metric-management/  # 指标管理子模块
    │   ├── application/    # REST接口层
    │   ├── business/       # 业务逻辑层
    │   └── infrastructure/ # 数据访问层
    └── task-management/    # 任务管理子模块
        ├── application/    # REST接口层
        ├── business/       # 业务逻辑层（Scheduler、Service、策略模式）
        └── infrastructure/ # 数据访问层
```
### 2 任务调度功能流程

任务调度流程
1. 创建任务 → SrconTaskController.createTask()
2. 启动任务 → SrconTaskController.startTask()
3. 定时调度 → SrconTaskScheduler（轮询任务状态）
4. **策略执行** → SchedulerFactory 根据任务状态选择对应 Strategy
5. 状态同步 → SrconTaskController.statusSync()
6. 结果存储 → TaskDaoImpl 持久化到数据库

#### 2.1 调度架构


```
┌─────────────────────────────────────────────────────────────────┐
│                     调度入口层                                   │
│  ┌─────────────────────┐    ┌──────────────────────────────┐   │
│  │ SrconTaskScheduler  │    │    TaskStatusExecutor        │   │
│  │ (轮询QUEUING任务)    │    │    (轮询RUNNING任务)         │   │
│  │ 10秒/次              │    │    Cron表达式配置             │   │
│  └─────────────────────┘    └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     策略执行层                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         SchedulerFactory (策略工厂)                      │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │ Success │ │ Queuing │ │Running  │ │ Failed  │ ...   │   │
│  │  │Strategy │ │Strategy │ │Strategy │ │Strategy │       │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     调用执行层                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         IInvoke (IntelligentInvoke)                     │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │           ParameterBuilder (策略+工厂)            │   │   │
│  │  │  TrainParameterBuilder / RcaInferParameterBuilder │   │   │
│  │  │  UemInferParameterBuilder / EvaluationParameter... │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.2 任务状态定义 (TaskStatus)
系统定义了 17 种任务状态，核心状态流转：

```
INIT → QUEUING → RUNNING → SUCCESS/FAILURE/STOP
                      ↓
                  PENDING (智算侧排队)
```

#### 2.3 调度器组件 (SrconTaskScheduler)

职责：处理 QUEUING（排队中） 状态的任务，为其分配集群资源

```
@Scheduled(fixedRate = 10000, initialDelay = 3000)
public void schedulePendingTasks()
```
核心流程：
1. 获取 Redis 分布式锁（防止多实例重复执行）
        ↓
2. 查询所有 QUEUING 状态的任务
        ↓
3. 按优先级 score 降序排序
        ↓
4. 遍历调度每个任务:
   a. 调用 `ClusterService.allocateClusterResource()` **分配集群资源**
   b. 如果分配成功，更新任务状态为 RUNNING
优先级调度：任务按照 priorityScore 倒序排列，优先调度高优先级任务


#### 2.4 集群资源分配 (ClusterServiceImpl)

职责：为任务分配合适的集群节点
算法：
1. 获取任务所需卡数 (cardCount)
        ↓
2. 查询所有集群的总卡数和当前使用量
        ↓
3. 计算每个集群的可用卡数 = 总卡数 - 已用卡数
        ↓
4. 选择可用卡数最多的集群分配


#### 2.5 任务状态执行器 (TaskStatusExecutor) ⭐

职责：处理 RUNNING（运行中） 状态的任务，同步任务状态

##### 2.5.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TaskStatusExecutor 架构                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  @Scheduled(cron) ──→ scheduler()                                           │
│         │                                                                   │
│         ├─ 获取Redis分布式锁                                                 │
│         ├─ 查询RUNNING状态的主任务列表                                        │
│         └─ doScheduler(list)  ──并行执行──┐                                 │
│                                           ↓                                  │
│                         ┌─────────────────────────────────┐                 │
│                         │         线程池并行处理            │                 │
│                         │  ┌─────────┐ ┌─────────┐       │                 │
│                         │  │execute()│ │execute()│ ...  │                 │
│                         │  │ task_1  │ │ task_2  │       │                 │
│                         │  └────┬────┘ └────┬────┘       │                 │
│                         │       │          │            │                 │
│                         │       ↓          ↓            │                 │
│                         │  ┌─────────────────────┐      │                 │
│                         │  │ SchedulerFactory    │      │                 │
│                         │  │ (策略工厂)           │      │                 │
│                         │  │                     │      │                 │
│                         │  │ StopStrategy        │      │                 │
│                         │  │ SuccessStrategy     │      │                 │
│                         │  │ QueuingStrategy     │      │                 │
│                         │  │ RunningStrategy     │      │                 │
│                         │  │ FailedStrategy      │      │                 │
│                         │  │ PendingStrategy     │      │                 │
│                         │  └─────────────────────┘      │                 │
│                         └─────────────────────────────────┘                 │
│                                           ↓                                  │
│                         CountDownLatch.await(30分钟)                         │
│                                           ↓                                  │
│                         超时取消未完成任务 futureCancel()                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

##### 2.5.2 完整时序图梳理


```
主线程                                              Worker线程
  │                                                   │
  ├─ scheduler() ──────────────────────────────────▶ │
  │   ├─ redisLock.lock()                            │
  │   ├─ queryRunningTasks()                         │
  │   └─ doScheduler(list)                           │
  │        ├─ latch = CountDownLatch(3)              │
  │        ├─ executorService = ThreadPoolFactory    │
  │        │                                          │
  │        ├─ runAsync(execute(task1)) ──────────────▶│ execute(task1)
  │        ├─ runAsync(execute(task2)) ──────────────▶│ execute(task2)
  │        ├─ runAsync(execute(task3)) ──────────────▶│ execute(task3)
  │        │                                          │
  │        └─ latch.await() ◀─────────────────────────┼─ latch.countDown()
  │             (阻塞等待)                             │   (task1完成)
  │                                                   │◀─ latch.countDown()
  │                                                   │   (task2完成)
  │             (继续执行) ◀─────────────────────────┼─ latch.countDown()
  │                                                   │   (task3完成)
  │   ├─ redisLock.unlock()                          │
  │   └─ 方法返回                                     │
  │                                                   │
  │                                                   │
Worker线程1 执行 execute(task1):
  │                                                   │
  ├─ getSecondSchedulers()                           │
  │   ├─ queryTaskPipelines() ──查不到──┐           │
  │   └─ createTaskPipeline() ◀─────────┘           │
  │        └─ 创建 [train, infer_rca, cbb]           │
  │            所有子任务 status = QUEUING           │
  │                                                   │
  ├─ buildSchedulerStrategy()                        │
  │   └─ 创建 SchedulerFactory + AtomicInteger(0)    │
  │                                                   │
  ├─ for train:                                      │
  │   ├─ preStatus = "QUEUING"                       │
  │   ├─ schedulerFactory.execute(train)             │
  │   │   └─ QueuingStrategy.handle()                │
  │   │        └─ invoke.start() ──HTTP──┐          │
  │   │                                   │          │
  │   ├─ train.setStatus("running")       │          │
  │   ├─ shouldBreakAfterUpdate() → true  │          │
  │   └─ BREAK;  // 退出循环               │          │
  │                                                   │
  ├─ atomicInteger.get() == 1 != 3                   │
  │   └─ 不更新主任务为SUCCESS                        │
  │                                                   │
  └─ latch.countDown()                               │
                                                      │
Worker线程2 执行 execute(task2):
  │                                                   │
  ... (同上，类似逻辑)                                 │
                                                      │

```

##### 2.5.3 入口方法 scheduler()


```java
@Scheduled(cron = "${smart-scheduler.cron.task-status-scheduler}")
public void scheduler() {
    try {
        // 1. 获取Redis分布式锁，防止多实例重复执行
        redisLock.lock(PERIOD_EXEC_KEY, EXPIRE_TIME);
        
        // 2. 查询所有RUNNING状态的主任务
        List<TaskInfo> list = taskDao.queryRunningTasks();
        if (CollectionUtils.isEmpty(list)) {
            return;
        }
        
        // 3. 并行执行任务调度
        doScheduler(list);
    } catch (Exception ex) {
        log.error("TaskStatusExecutor error", ex);
    } finally {
        // 4. 释放Redis锁
        redisLock.unLock(PERIOD_EXEC_KEY);
    }
}

```
##### 2.5.4 并行执行 doScheduler() (使用线程池)⭐


```java
private void doScheduler(final List<TaskInfo> list) {
    // ============================================================
    // 步骤1：初始化并行控制变量
    // ============================================================
    CountDownLatch latch = new CountDownLatch(list.size());  // 计数器=任务数量
    ExecutorService executorService = ThreadPoolFactory.INSTANCE.get();  // 获取线程池
    List<Future<?>> futures = new ArrayList<>();
    
    // ============================================================
    // 步骤2：为每个主任务提交异步执行任务
    // ============================================================
    for (TaskInfo taskInfo : list) {
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> execute(taskInfo, latch), executorService)
            // 设置单个任务超时：10分钟
            .orTimeout(SUB_TASK_TIME_OUT, TimeUnit.MINUTES)
            // 异常处理：如果超时或出错，标记任务为失败
            .exceptionally(ex -> {
                updateTaskStatus(taskInfo, TaskStatus.FAILURE.getStatus());
                log.error("=========> exec timeout or error: {}", JSON.toJSONString(taskInfo), ex);
                return null;
            });
        futures.add(future);
    }
    
    // ============================================================
    // 步骤3：等待所有任务完成
    // ============================================================
    try {
        // 等待所有任务完成，最多等待30分钟
        boolean allDone = latch.await(ALL_TASK_TIME_OUT, TimeUnit.MINUTES);
        if (!allDone) {
            // 超时未全部完成，记录警告取消并未完成任务
            log.warn("Some subtasks are not completed within the specified time.");
            futureCancel(futures);
        }
    } catch (Exception ex) {
        log.error("An interrupt exception occurred while waiting for the subtask to complete", ex);
    }
}

```



#### 2.6 策略模式实现 (SchedulerStrategy) ⭐
无论主任务还是子任务，根据任务状态匹配策略。
##### 2.6.1 execute() 方法完整流程

```java
private void execute(TaskInfo taskInfo, CountDownLatch latch) {
    try {
        log.info("======> execute taskInfo={}", JSON.toJSONString(taskInfo));
        
        // ============================================================
        // 步骤1：获取或创建子任务Pipeline
        // ============================================================
        List<SecondScheduler> secondSchedulers = getSecondSchedulers(taskInfo);
        log.info("=====> secondSchedulers : {}", JSON.toJSONString(secondSchedulers));
        
        // ============================================================
        // 步骤2：创建策略工厂（包含独立的成功计数器）
        // ============================================================
        AtomicInteger atomicInteger = new AtomicInteger(0);
        SchedulerFactory schedulerFactory = buildSchedulerStrategy(atomicInteger);
        
        // ============================================================
        // 步骤3：遍历执行每个子任务
        // ============================================================
        for (SecondScheduler scheduler : secondSchedulers) {
            String preStatus = scheduler.getStatus();  // 执行前的状态
            
            // 根据状态匹配策略并执行
            schedulerFactory.execute(scheduler);
            
            log.info("====> after execute: {}", JSON.toJSONString(scheduler));
            
            // ============================================================
            // 步骤4：判断是否中断遍历
            // ============================================================
            if (shouldBreakAfterStrategy(scheduler)) {
                break;  // 状态为UNKNOWN，中断
            }
            
            // 更新数据库
            updateSecondScheduler(scheduler, taskInfo, preStatus);
            
            if (shouldBreakAfterUpdate(scheduler)) {
                break;  // 状态不是SUCCESS，中断
            }
        }
        
        // ============================================================
        // 步骤5：判断是否所有子任务都成功
        // ============================================================
        if (atomicInteger.get() == secondSchedulers.size()) {
            updateTaskStatus(taskInfo, TaskStatus.SUCCESS.getStatus());
        }
    } finally {
        latch.countDown();  // 任务完成，计数器-1
    }
}

```

##### 2.6.2 任务 Pipeline 构建 (TaskGenerationFactory)
职责：根据任务类型创建对应的子任务 pipeline
###### 2.6.2.1 整体架构


```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     TaskGenerationFactory 工厂方法                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TaskStatusExecutor.getSecondSchedulers()                                   │
│         │                                                                   │
│         └─▶ TaskGenerationFactory.createTaskPipeline(taskInfo)              │
│                       │                                                     │
│                       ▼                                                     │
│              ┌─────────────────────┐                                        │
│              │  strategyMap        │  (Spring注入的策略Map)                  │
│              │  ─────────────────  │                                        │
│              │  "training" ────────┼──▶ TrainingTaskStrategy                │
│              │  "prediction" ──────┼──▶ PredictionTaskStrategy              │
│              │  "evaluation" ──────┼──▶ EvaluationTaskStrategy              │
│              │  "infer_uem" ───────┼──▶ UEMPredictionTaskStrategy           │
│              │  "effectiveEval..."─┼──▶ EffectiveEvaluationTaskStrategy     │
│              └─────────────────────┘                                        │
│                       │                                                     │
│                       ▼                                                     │
│              根据taskType获取对应Strategy                                    │
│                       │                                                     │
│                       ▼                                                     │
│              TaskCreateStrategy.createTaskPipeline()                        │
│                       │                                                     │
│                       ├──▶ 子类实现 ──返回 List<SecondScheduler>            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

```
###### 2.6.2.2 工厂实现


```java
@Component
@RequiredArgsConstructor
public class TaskGenerationFactory {
    // Spring自动注入：key=Component名称, value=Bean实例
    private final Map<String, TaskCreateStrategy> strategyMap;
    public List<SecondScheduler> createTaskPipeline(final TaskInfo taskInfo) {
        String taskType = taskInfo.getTaskType();
        // 根据taskType从Map中获取对应的策略，若不存在则抛异常
        TaskCreateStrategy taskCreateStrategy = Optional.ofNullable(strategyMap.get(taskType))
            .orElseThrow(() -> new RuntimeException("no such taskType " + taskType));
        return taskCreateStrategy.createTaskPipeline(taskInfo);
    }
}

```

###### 2.6.2.3 Pipeline 策略接口
接口定义：

```java
public interface TaskCreateStrategy {
    List<SecondScheduler> createTaskPipeline(TaskInfo taskInfo);
}

```
通用构建方法：

```java
public abstract class AbstractTaskCreateStrategy implements TaskCreateStrategy {
    protected final ITaskDao taskDao;
    // 通用构建方法：遍历parameter，调用invokeMapper转换后构建子任务
    protected List<SecondScheduler> buildSchedulersFromParameter(
            TaskInfo taskInfo, 
            Function<String, String> invokeMapper) {
        List<SecondScheduler> secondSchedulers = new ArrayList<>();
        AtomicInteger index = new AtomicInteger(1);
        JSONObject parameter = taskInfo.getParameter();
        
        parameter.forEach((key, value) -> {
            String invoke = invokeMapper.apply(key);  // 将key映射为invoke类型
            if (invoke != null && !invoke.isEmpty()) {
                int step = index.getAndIncrement();
                // 调用核心构建方法
                SecondScheduler scheduler = buildScheduler(taskInfo, invoke, step, (JSONObject) value);
                secondSchedulers.add(scheduler);
            }
        });
        return secondSchedulers;
    }
    // 核心构建逻辑：创建SecondScheduler，初始状态为QUEUING
    protected SecondScheduler buildScheduler(
            final TaskInfo taskInfo, 
            final String invoke, 
            int step,
            JSONObject parameter) {
        return SecondScheduler.builder()
            .taskId(taskInfo.getTaskId())
            .batchNo(taskInfo.getBatchNo())
            .proxy(taskInfo.getProxy())           // 分配的集群代理
            .taskType(taskInfo.getTaskType())
            .invoke(invoke)                       // 任务类型标识
            .parameter(parameter)                 // 任务参数
            .status(TaskStatus.QUEUING.getStatus())  // 初始状态
            .step(step)                           // 执行步骤序号
            .build();
    }
}

```
TrainingTaskStrategy (训练任务)：


```java
@Component("training")  // 注册到strategyMap，key="training"
public class TrainingTaskStrategy extends AbstractTaskCreateStrategy {
    
    @Override
    public List<SecondScheduler> createTaskPipeline(TaskInfo taskInfo) {
        // 使用父类的通用构建方法，传入invoke映射函数
        return buildSchedulersFromParameter(taskInfo, this::getInvoke);
    }
    
    // 根据parameter中的key映射到具体的invoke类型
    private String getInvoke(String modelName) {
        return switch (modelName) {
            case "uem" -> TaskTypeEnum.TRAIN_UEM.getTaskType();   // "train_uem"
            case "bslm" -> TaskTypeEnum.TRAIN_BSLM.getTaskType(); // "train_bslm"
            default -> StringUtils.EMPTY;
        };
    }
}

```
PredictionTaskStrategy (推理任务)：

```java
@Component("prediction")
public class PredictionTaskStrategy extends AbstractTaskCreateStrategy {
    // 固定的任务类型顺序
    private static final List<String> STDT_TASK_TYPES = ImmutableList.of("cbb", "T0", "srcon_agent");
    @Override
    public List<SecondScheduler> createTaskPipeline(TaskInfo taskInfo) {
        List<SecondScheduler> secondSchedulers = new ArrayList<>();
        JSONObject parameter = taskInfo.getParameter();
        
        // 1. 首先添加RCA推理任务
        JSONObject rca = parameter.getJSONObject("rca");
        SecondScheduler scheduler = buildScheduler(taskInfo, TaskTypeEnum.INFER_RCA.getTaskType(), 1, rca);
        secondSchedulers.add(scheduler);
        
        // 2. 根据explainable标志决定是否添加srcon_agent
        boolean explainable = parameter.getBoolean("explainable");
        int index = 2;
        for (String invoke : STDT_TASK_TYPES) {
            if (!explainable && invoke.equals("srcon_agent")) {
                continue;  // 不可解释时不添加srcon_agent
            }
            // 注意：cbb/T0/srcon_agent使用默认parameter
            SecondScheduler build = SecondScheduler.builder()
                .taskId(taskInfo.getTaskId())
                .batchNo(taskInfo.getBatchNo())
                .proxy(taskInfo.getProxy())
                .taskType(taskInfo.getTaskType())
                .invoke(invoke)
                .step(index++)
                .build();
            secondSchedulers.add(build);
        }
        return secondSchedulers;
    }
}

```

###### 2.6.2.4 完整时序图

![](./images/pipeline完整流程时序图.PNG)

###### 2.6.2.2 状态同步
非首次调度时需要进行状态同步。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        taskStatusSync 状态同步                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  输入：主任务状态 status, 子任务列表 secondSchedulers                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ for each scheduler in secondSchedulers:                             │    │
│  │     IF scheduler.status == SUCCESS:                                 │    │
│  │         continue;  // 已完成的子任务不处理                           │    │
│  │                                                                     │    │
│  │     IF 主任务状态 == RUNNING:                                        │    │
│  │         IF 子任务状态 == QUEUING 或 PENDING:                         │    │
│  │             break;  // 正常执行，不处理                              │    │
│  │         IF 子任务状态 == RUN_FAILED 或 STOP:                         │    │
│  │             子任务状态 = QUEUING;  // 重置为排队中                   │    │
│  │             isRetryScene = !taskInfo.isBreakpointRetry;             │    │
│  │             break;                                                   │    │
│  │                                                                     │    │
│  │     IF 主任务状态 == RUN_FAILED 或 STOP:                             │    │
│  │         IF 子任务状态 == QUEUING:                                    │    │
│  │             子任务状态 = FAILURE;  // 标记为失败                     │    │
│  │             break;                                                   │    │
│  │         IF 子任务状态 == PENDING 或 RUNNING:                         │    │
│  │             子任务状态 = STOP;  // 标记为停止                        │    │
│  │             break;                                                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  IF isRetryScene:                                                           │
│      taskParameterSync();  // 同步最新参数                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

```

##### 2.6.3 策略工厂
每次执行任务都创建新的 SchedulerFactory 。
###### 2.6.3.1 构建方法


```java
private SchedulerFactory buildSchedulerStrategy(AtomicInteger atomicInteger) {
    return new SchedulerFactory.Builder()
        // 策略注册顺序很重要！按优先级从高到低排列
        .add(new StopStrategy(iInvoke, taskDao))        // 1. 停止策略
        .add(new SuccessStrategy(iInvoke, atomicInteger)) // 2. 成功策略
        .add(new QueuingStrategy(iInvoke, taskDao))     // 3. 排队中策略
        .add(new RunningStrategy(iInvoke))              // 4. 运行中策略
        .add(new FailedStrategy(iInvoke))               // 5. 失败策略
        .add(new PendingStrategy(iInvoke))              // 6. 待处理策略
        .build();
}

```

###### 2.6.3.2 SchedulerFactory 实现


```java
public record SchedulerFactory(List<SchedulerStrategy> strategies) {
    
    /**
     * 执行策略：遍历所有策略，找到第一个有效的执行
     */
    public void execute(SecondScheduler scheduler) {
        for (SchedulerStrategy strategy : strategies) {
            if (strategy.isEffective(scheduler)) {  // 判断该策略是否适用于当前子任务
                strategy.handle(scheduler);          // 执行策略处理
                break;                                // 执行后立即break，不继续匹配
            }
        }
    }
}

```

###### 2.6.3.3 子任务遍历执行示意图


```
Pipeline子任务列表: [train(QUEUING), infer_rca(QUEUING), cbb(QUEUING)]
                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                     遍历执行第一个子任务 train                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  preStatus = "QUEUING"                                                       │
│       ↓                                                                     │
│  schedulerFactory.execute(scheduler)                                        │
│       ↓                                                                     │
│  遍历策略列表:                                                               │
│       ├─ StopStrategy: isStop(QUEUING) = false  → 跳过                      │
│       ├─ SuccessStrategy: isSuccess(QUEUING) = false → 跳过                │
│       ├─ QueuingStrategy: isQueuing(QUEUING) = TRUE  → 执行！               │
│       │      └─ invoke.start() → 启动任务                                   │
│       │      └─ scheduler.setStatus("running")                             │
│       │      └─ atomicInteger.getAndIncrement() (如果成功)                  │
│       └─ break; 不再匹配后续策略                                             │
│       ↓                                                                     │
│  shouldBreakAfterStrategy(running) → false → 不中断                         │
│       ↓                                                                     │
│  updateSecondScheduler() → 更新到数据库                                      │
│       ↓                                                                     │
│  shouldBreakAfterUpdate(running) → true → BREAK退出循环                     │
│       ↓                                                                     │
│  ⚠️ 注意：只处理了第一个子任务train，后面两个infer_rca和cbb未处理！           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

```
###### 2.6.3.4 各策略处理逻辑

| 策略  | 处理逻辑  |
| ------------ | ------------ |
| StopStrategy  | 调用 invoke.stop() 停止任务  |
| SuccessStrategy  |  计数器 +1，不做其他处理 |
| QueuingStrategy  | 调用 invoke.start() 启动任务，构造参数，处理断点续训  |
| RunningStrategy  |  调用 invoke.status() 查询状态，更新结束时间 |
| FailedStrategy  | 记录错误日志  |
| PendingStrategy  | 调用 invoke.status() 查询状态  |



为什么每个任务要创建独立的 SchedulerFactory？
因为 SuccessStrategy 内部引用了 AtomicInteger：

```java
public class SuccessStrategy extends AbstractSchedulerStrategy {
    private final AtomicInteger atomicInteger;  // 用于计数成功的子任务数
    
    public SuccessStrategy(final IInvoke invoke, AtomicInteger atomicInteger) {
        super(invoke);
        this.atomicInteger = atomicInteger;
    }
    
    @Override
    public void handle(final SecondScheduler scheduler) {
        atomicInteger.getAndIncrement();  // 子任务成功时+1
    }
}
```

#### 2.7 调用执行层 (IInvoke / IntelligentInvoke)

核心接口：

```
public interface IInvoke {
    String start(SecondScheduler scheduler, boolean isBreakpointRetry);  // 启动任务
    String status(SecondScheduler scheduler);  // 查询状态
    String stop(SecondScheduler scheduler);    // 停止任务
}
```
##### 2.7.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        IInvoke 工厂方法                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  QueuingStrategy.handle()                         │
│         │                                                                   │
│         └─▶ IntelligentInvoke.start(scheduler, isBreakpointRetry)          │
│                       │                                                     │
│                       ▼                                                     │
│           ┌────────────────────────┐                                        │
│           │  getBuilder(invoke)    │  工厂方法：根据invoke类型              │
│           │  ────────────────────  │  选择对应的ParameterBuilder           │
│           │  "train_uem" ──────────┼──▶ TrainParameterBuilder              │
│           │  "train_bslm" ─────────┼──▶ TrainParameterBuilder              │
│           │  "infer_rca" ──────────┼──▶ RcaInferParameterBuilder           │
│           │  "infer_uem" ──────────┼──▶ UemInferParameterBuilder           │
│           │  "evaluate_uem" ───────┼──▶ EvaluationParameterBuilder         │
│           │  "evaluate_bslm" ──────┼──▶ EvaluationParameterBuilder         │
│           │  "cbb" ────────────────┼──▶ CBBProcessParameterBuilder         │
│           │  "T0" ─────────────────┼──▶ CBBProcessParameterBuilder         │
│           │  "effectiveEvaluation"─┼──▶ CBBProcessParameterBuilder         │
│           │  "srcon_agent" ────────┼──▶ ExplainParameterBuilder            │
│           └────────────────────────┘                                        │
│                       │                                                     │
│                       ▼                                                     │
│           ParameterBuilder.buildStart(intelligentRecord, isContinue)       │
│                       │                                                     │
│                       ├──▶ 构建请求URL                                       │
│                       └──▶ 构建请求体 (JSONObject)                           │
│                                                                             │
│                       │                                                     │
│                       ▼                                                     │
│           RestRecord execRecord = new RestRecord(url, body, proxy)         │
│                       │                                                     │
│                       ▼                                                     │
│           RestUtil.sendPostJsonRequest(execRecord)  ──HTTP POST──▶ 智算侧   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

```

##### 2.7.2 start 方法完整流程


```java
@Override
public String start(final SecondScheduler scheduler, final boolean isBreakpointRetry) {
    // ============================================================
    // 步骤1：选择对应的ParameterBuilder
    // ============================================================
    ParameterBuilder builder = getBuilder(scheduler.getInvoke());
    
    // ============================================================
    // 步骤2：构建调用上下文记录
    // ============================================================
    IntelligentRecord intelligentRecord = new IntelligentRecord(
        scheduler, intelligentApi, taskDao, metricTaskDao, srconTaskDao);
    
    // ============================================================
    // 步骤3：构建请求URL和请求体
    // ============================================================
    ImmutablePair<String, JSONObject> pair = builder.buildStart(intelligentRecord, isBreakpointRetry);
    if (pair == null || StringUtils.isEmpty(pair.left) || pair.right == null) {
        log.error("Intelligent parameter build failed!!");
        TaskErrorCodeEnum errorCode = builder.buildErrno(scheduler.getInvoke(), true, false);
        updateErrno(scheduler, errorCode.getErrno());
        return TaskStatus.FAILURE.getStatus();
    }
    
    // ============================================================
    // 步骤4：构建HTTP请求记录
    // ============================================================
    RestRecord execRecord = new RestRecord(pair.left, pair.right, scheduler.getProxy());
    log.warn("invoke start: {}", JSON.toJSONString(pair));
    
    // ============================================================
    // 步骤5：发送HTTP请求到智算侧
    // ============================================================
    JSONObject response = RestUtil.sendPostJsonRequest(execRecord);
    
    // ============================================================
    // 步骤6：更新错误码
    // ============================================================
    updateStartErrno(scheduler, builder, response);
    
    // ============================================================
    // 步骤7：判断请求是否成功
    // ============================================================
    if (!isInvokeSuccess(response)) {
        log.error("Intelligent start error: {}", JSON.toJSONString(scheduler));
        return TaskStatus.FAILURE.getStatus();
    }
    
    // ============================================================
    // 步骤8：保存任务结果
    // ============================================================
    ModelTaskResult modelTaskResult = buildModelTaskResult(scheduler, response);
    taskDao.saveModelTaskResult(modelTaskResult);
    
    return TaskStatus.RUNNING.getStatus();
}

```

##### 2.7.3 完整时序图

![](./images/invoke完整时序图.PNG)




#### 2.8 调度场景分析


```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           正常任务调度完整流程                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  【用户操作阶段】                                                             │
│  createTask() ──→ INIT                                                     │
│  startTask()  ──→ QUEUING                                                  │
│                                                                             │
│  【第一调度周期】 (SrconTaskScheduler 每10秒)                                 │
│  queryTasksByStatus(QUEUING) ──→ allocateClusterResource() ──→ RUNNING     │
│                                   (分配集群资源)                              │
│                                                                             │
│  【第一调度周期】 (TaskStatusExecutor cron配置时间)                           │
│  queryRunningTasks() ──→ getSecondSchedulers()                              │
│                          ├─ 若无pipeline: createTaskPipeline()              │
│                          │     └─ 所有子任务 status=QUEUING                 │
│                          └─ 若有pipeline: taskStatusSync()                  │
│                                                                             │
│  遍历子任务:                                                                 │
│  ├─ 子任务1 (QUEUING) → QueuingStrategy.handle()                            │
│  │     ├─ buildSchedulerParameter()                                         │
│  │     ├─ invoke.start() ──→ HTTP到智算侧 ──→ "running"                    │
│  │     └─ 子任务状态 → RUNNING                                              │
│  │                                                                         │
│  └─ 子任务2 (QUEUING) → QueuingStrategy.handle()                            │
│        ├─ buildSchedulerParameter()                                         │
│        ├─ invoke.start() ──→ HTTP到智算侧 ──→ "running"                    │
│        └─ 子任务状态 → RUNNING                                              │
│                                                                             │
│  【后续调度周期】 (10秒后/下次cron)                                           │
│  queryRunningTasks() ──→ 主任务状态仍=RUNNING                                │
│                                                                             │
│  子任务状态同步:                                                             │
│  ├─ train (RUNNING) → RunningStrategy.handle()                              │
│  │     └─ invoke.status() ──→ 查询智算侧状态                                 │
│  │                                                                          │
│  ├─ infer_rca (QUEUING) → QueuingStrategy.handle()                          │
│  │     └─ invoke.start() ──→ 启动推理                                        │
│  │                                                                          │
│  └─ cbb (QUEUING) → QueuingStrategy.handle()                                │
│        └─ invoke.start() ──→ 启动后处理                                      │
│                                                                             │
│  【所有子任务成功时】                                                         │
│  主任务状态 ──→ SUCCESS                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

```

## 二、策略模式解决的核心问题
问题场景：多状态/多类型的分支处理
如果没有策略模式，糟糕的代码：大量if-else/switch，违反开闭原则
策略模式的优势

```java
// ✅ 使用策略模式：每个策略只关心自己的逻辑
public void handle(SecondScheduler scheduler) {
    // 策略自己判断是否适用
    for (SchedulerStrategy strategy : strategies) {
        if (strategy.isEffective(scheduler)) {  // 策略自己知道何时该处理
            strategy.handle(scheduler);          // 策略自己知道怎么处理
            break;
        }
    }
}
// ✅ 每个策略职责单一，易于维护
public class QueuingStrategy {
    @Override
    public boolean isEffective(SecondScheduler scheduler) {
        return TaskStatus.isQueuing(scheduler.getStatus());  // 只关心QUEUING状态
    }
    
    @Override
    public void handle(SecondScheduler scheduler) {
        // 处理QUEUING状态的逻辑，与其他策略隔离
    }
}
public class SuccessStrategy {
    @Override
    public boolean isEffective(SecondScheduler scheduler) {
        return TaskStatus.isSuccess(scheduler.getStatus());  // 只关心SUCCESS状态
    }
    
    @Override
    public void handle(SecondScheduler scheduler) {
        atomicInteger.getAndIncrement();  // 只做计数
    }
}
```
策略模式解决的具体问题

|问题|	解决方案|
| ------------ | ------------ |
|状态处理逻辑分散|	每个状态对应一个策略类，逻辑内聚|
|新增状态需要修改多处|	新增策略类，工厂注册，无需修改调用方|
|测试困难|	每个策略可独立单元测试|

---
## 三、工厂模式解决的核心问题
问题场景：对象创建与使用的耦合
如果没有工厂模式：
❌ 调用方需要知道所有策略的实现类

工厂模式的优势

```java
// ✅ 使用工厂模式：调用方只依赖接口
@Component
public class TaskGenerationFactory {
    private final Map<String, TaskCreateStrategy> strategyMap;
    
    public List<SecondScheduler> createTaskPipeline(TaskInfo taskInfo) {
        String taskType = taskInfo.getTaskType();
        TaskCreateStrategy strategy = Optional.ofNullable(strategyMap.get(taskType))
            .orElseThrow(() -> new RuntimeException("no such taskType " + taskType));
        return strategy.createTaskPipeline(taskInfo);
    }
}
依赖注入的实现：
// Spring自动将所有TaskCreateStrategy实现注入到Map中
// key = @Component的名称，value = Bean实例
@Component("training")      // key = "training"
class TrainingTaskStrategy extends AbstractTaskCreateStrategy { ... }
@Component("prediction")    // key = "prediction"
class PredictionTaskStrategy extends AbstractTaskCreateStrategy { ... }
@Component("evaluation")    // key = "evaluation"
class EvaluationTaskStrategy extends AbstractTaskCreateStrategy { ... }
```
工厂模式解决的具体问题
|问题	|解决方案|
| ------------ | ------------ |
|创建逻辑耦合	|工厂统一创建，调用方不关心创建细节|
|依赖具体类	|调用方只依赖接口，具体类由工厂选择|
|扩展困难	|新增策略只需加@Component，自动加入Map|
|隐藏实现差异	|调用方无需知道有多少种策略实现|
---
## 四、为什么不用状态模式 (State Pattern)?
状态模式的特点

```java
// 状态模式：状态对象持有，内部状态转换
class TaskContext {
    private TaskState state;
    
    public void handle() {
        state.handle(this);  // 状态对象处理并可能转换状态
    }
    
    public void changeState(TaskState newState) {
        this.state = newState;
    }
}
interface TaskState {
    void handle(TaskContext context);
}
class QueuingState implements TaskState {
    @Override
    public void handle(TaskContext context) {
        // 处理QUEUING状态的逻辑
        context.changeState(new RunningState());  // 状态转换
    }
}
```
为什么不适合这个场景
|状态模式特点|	项目场景|	冲突点|
| ------------ | ------------ |--|
|状态转换在对象内部控制	|状态转换由外部（智算侧）决定|	无法预测下一个状态|
|状态对象持有上下文|	需要访问DAO、配置、远程服务|	状态对象过于复杂|
|状态转换触发行为|	行为由调度器决定|	状态只是数据，行为由策略决定|
核心原因：

```
// 项目中：状态只是数据，行为由策略决定
scheduler.getStatus();  // 只是读取状态值
scheduler.setStatus("running");  // 由策略根据外部响应设置状态
// 状态模式中：状态本身决定行为
state.handle(context);  // 状态对象自己决定如何处理
context.changeState(newState);  // 状态对象决定转换到哪个状态
```
---
## 五、为什么不用责任链模式 (Chain of Responsibility)?
责任链模式的特点

```java
// 责任链：请求沿着链传递，直到某个处理器处理
abstract class Handler {
    private Handler next;
    
    public void handle(Request request) {
        if (canHandle(request)) {
            doHandle(request);
        } else if (next != null) {
            next.handle(request);
        }
    }
}
class AuthHandler extends Handler {
    @Override
    protected boolean canHandle(Request request) {
        return request.needAuth();
    }
}
class ValidationHandler extends Handler {
    @Override
    protected boolean canHandle(Request request) {
        return request.needValidation();
    }
}
```
为什么不适合这个场景，核心原因：

```java
// 责任链：每个处理器都有机会处理或传递
public void handle(Request request) {
    if (canHandle(request)) {
        doHandle(request);  // 处理后可能继续传递
    }
    if (next != null) {
        next.handle(request);  // 继续传递给下一个
    }
}
// 策略模式：只匹配一个，执行后立即退出
public void execute(SecondScheduler scheduler) {
    for (SchedulerStrategy strategy : strategies) {
        if (strategy.isEffective(scheduler)) {
            strategy.handle(scheduler);
            break;  // 执行后立即退出，不继续匹配
        }
    }
}
```
<span style="color:rgb(216,27,68)">**责任链核心语义是传递性，当前情况每个子任务只会被一个策略捕获执行，传递不了一点，所以不要用责任链模式**</span>！

---
## 六、为什么结合策略模式和工厂模式？
协作关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    策略模式 + 工厂模式 协作                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  调用方 (TaskStatusExecutor)                                                │
│       │                                                                     │
│       │  factory.createTaskPipeline(taskInfo)                               │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         工 厂 方 式                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  Map<taskType, TaskCreateStrategy>                          │    │   │
│  │  │  "training"      ───▶ TrainingTaskStrategy                  │    │   │
│  │  │  "prediction"    ───▶ PredictionTaskStrategy                │    │   │
│  │  │  ...                                                         │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       │  strategy.createTaskPipeline() 返回 List<SecondScheduler>         │
│       ▼                                                                     │
│  调用方使用返回的子任务列表                                                  │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  调用方 (QueuingStrategy.handle)                                            │
│       │                                                                     │
│       │  builder.buildStart(record, isContinue)                            │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         工 厂 方 式                                  │   │
│  │  getBuilder(invoke) 根据invoke类型创建ParameterBuilder               │   │
│  │       │                                                              │   │
│  │       ▼                                                              │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  switch(TaskTypeEnum.get(taskType))                         │    │   │
│  │  │  TRAIN_UEM/TRAIN_BSLM  ───▶ TrainParameterBuilder           │    │   │
│  │  │  INFER_RCA           ───▶ RcaInferParameterBuilder          │    │   │
│  │  │  ...                                                         │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       │  builder.buildStart() 构建请求参数                                 │
│       ▼                                                                     │
│  ParameterBuilder执行具体策略                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

设计原则的体现

```
单一职责原则 (SRP)
├─ TaskGenerationFactory: 只负责创建策略
├─ TaskCreateStrategy: 只负责构建Pipeline
└─ 各自职责清晰
开闭原则 (OCP)
├─ 新增策略: 只需新增策略类 + @Component
└─ 修改调用方: 无需修改
依赖倒置原则 (DIP)
├─ 调用方依赖接口 (TaskCreateStrategy)
└─ 具体实现由工厂注入
里氏替换原则 (LSP)
├─ 所有策略实现可互换
└─ 调用方无感知具体类型
```
---


## HourLevelExecutor 详解
### 一、类结构

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class HourLevelExecutor implements IScheduler {
    private static final String HOUR_LEVEL_EXECUTOR_KEY = "model.platform.scheduler.HourLevelExecutor";
    private static final int UEM_INFER_STEP = 3;
    private final ITaskDao taskDao;                    // 任务持久化
    private final DataFunctionBusiness dataFunctionBusiness;  // 数据功能
    private final DataSyncBusiness dataSyncBusiness;  // 数据同步
    private final ILock redisLock;                    // 分布式锁
    private final TaskRecordReport recordReport;      // 执行记录
    private final IntelligentInvokeBusiness invokeBusiness;   // 智能调用(启动推理)
}
```
---
### 二、核心方法
#### 1. schedule() - 调度入口

```java
@Override
public void schedule() {
    try {
        // 获取分布式锁，保证集群只有一个实例执行
        redisLock.lockAndWait(HOUR_LEVEL_EXECUTOR_KEY, ILock.LOCK_EXPIRE_TIME);
        
        // 查询所有可运行的小时级任务
        List<Map<String, Object>> maps = taskDao.queryCanRunningHourLevel();
        if (CollectionUtils.isEmpty(maps)) {
            return;
        }
        
        // 创建职责链（每次新建，避免状态污染）
        HourTaskHandleChain chain = new HourTaskHandleChain();
        
        for (Map<String, Object> obj : maps) {
            TaskKeyRecord task = JSON.to(TaskKeyRecord.class, obj);
            doSchedule(task, chain);
        }
    } finally {
        redisLock.unlock(HOUR_LEVEL_EXECUTOR_KEY);  // 必须释放锁
    }
}
```
设计亮点:
- Redis分布式锁防止多实例重复执行
- 每次调度创建新的 HourTaskHandleChain，保证状态干净
- try-finally 确保锁释放
---
#### 2. doSchedule() - 单任务调度

```java
private void doSchedule(final TaskKeyRecord task, final HourTaskHandleChain chain) {
    // 1. 查询该任务的所有小时级数据记录
    List<DataProcessHourInfo> hourInfos = taskDao.queryHourInfos(
        task.subTaskId(), task.batchNo(), task.rat());
    if (CollectionUtils.isEmpty(hourInfos)) {
        log.warn("====={} Non-flow processing method", JSON.toJSONString(task));
        return;
    }
    
    // 2. 检查子任务状态：终态或已停止 -> 处理错误
    String status = taskDao.querySubTaskStatusBySubTaskId(task.subTaskId());
    if (TaskStatus.isFinishStatus(status) || TaskStatus.isStop(status)) {
        handleWhenStreamError(task);
        return;
    }
    
    // 3. 检查任务是否被删除（防孤儿数据）
    TaskInfo taskInfo = taskDao.getTaskInfoBySubTaskId(task.subTaskId());
    if (taskInfo == null) {
        handleWhenStreamError(task);
        return;
    }
    
    // 4. 执行小时级管道处理
    int successCount = hourLeverPipelineProcess(task, chain, hourInfos, taskInfo);
    
    // 5. 全部成功则确定最终状态
    if (successCount == hourInfos.size()) {
        status = determineFinalStatus(taskInfo, task);
        updateHourLevelStatus(task, status);
    }
}
```
三重保护:
- 任务不存在 → 跳过
- 子任务已停止 → 处理流错误
- 任务被删除 → 防止处理孤儿数据
---
#### 3. hourLeverPipelineProcess() - 核心处理流程（最复杂）

```java
private int hourLeverPipelineProcess(final TaskKeyRecord task, final HourTaskHandleChain chain,
    final List<DataProcessHourInfo> hourInfos, final TaskInfo taskInfo) {
    
    // 1. 查询子任务信息
    SubTaskInfo subTaskInfo = taskDao.querySubTaskInfoBySubId(task.subTaskId());
    List<DataProcessHourInfo> processHourInfos = hourInfos;
    
    // 2. 对比模式：仅保留MR和PM数据源
    boolean comparisonMode = ApiConfigSingleton.INSTANCE.getDataProcessComparison();
    if (comparisonMode) {
        processHourInfos = filterComparisonSources(hourInfos);
    }
    
    // 3. 获取最后一个MR的序号（用于流控判断）
    Integer lastMRSerialNo = getLastSerialNoForMR(processHourInfos);
    
    // 4. 原子计数器记录成功数量
    AtomicInteger successCount = new AtomicInteger(0);
    
    // 5. 按序号分组，每批一起处理
    Map<Integer, List<DataProcessHourInfo>> map =
        processHourInfos.stream()
            .collect(Collectors.groupingBy(DataProcessHourInfo::getSerialNo));
    
    // 6. 遍历每个批次执行职责链
    for (Map.Entry<Integer, List<DataProcessHourInfo>> entry : map.entrySet()) {
        Integer index = entry.getKey();
        
        // 构建上下文（包含该批次所有需要的信息）
        TemporaryVariable temporaryVariable = new TemporaryVariable(
            index, task, subTaskInfo.getSftpMountPrefix(), lastMRSerialNo, 
            map, processHourInfos, successCount);
        TaskContext context = getTaskContext(temporaryVariable);
        
        // 执行职责链
        chain.process(context);
        
        // 检查是否需要进入错误等待
        Optional<List<DataProcessHourInfo>> optional = shouldSetErrorWaiting(context, lastMRSerialNo);
        if (optional.isPresent()) {
            setErrorWaiting(optional.get());
            continue;
        }
        
        // 检查是否有运行失败的
        if (hasRunFailed(context)) {
            handleWhenStreamError(task);
            break;
        }
    }
    
    // 7. 非对比模式执行UEM推理
    if (comparisonMode) {
        return successCount.get();
    }
    uemInfer(task, taskInfo, subTaskInfo.getSftpMountPrefix());
    redisLock.delete(task.subTaskId());  // 删除断点重试标记
    return successCount.get();
}
核心数据结构 TemporaryVariable (Java Record):
record TemporaryVariable(
    int index,                           // 当前批次序号
    TaskKeyRecord task,                  // 任务Key
    String sftpPrefix,                   // SFTP挂载前缀
    Integer lastMRSerialNo,              // 最后一个MR序号
    Map<Integer, List<DataProcessHourInfo>> map,  // 所有批次
    List<DataProcessHourInfo> totalHourInfos,     // 全部数据
    AtomicInteger successCount           // 成功计数器
) {}
```
---
#### 4. TaskContext - 执行上下文

```java
@Data
@Builder
public class TaskContext {
    private Map<Integer, List<DataProcessHourInfo>> map;      // 所有批次
    private List<DataProcessHourInfo> totalHourInfos;         // 全部数据
    private List<DataProcessHourInfo> currentHourInfos;       // 当前批次
    private int index;                                        // 批次序号
    private String subTaskId;
    private int batchNo;
    private ITaskDao taskDao;
    private DataFunctionBusiness dataFunctionBusiness;
    private DataSyncBusiness dataSyncBusiness;
    private String rat;                                       // 无线接入技术
    private TaskRecordReport recordReport;
    private String sftpPrefix;
    private Integer lastMRSerialNo;
    private AtomicInteger successCount;
}
```

设计意图: 将处理一个批次所需的所有信息封装在一起，通过 Builder 模式灵活构建。

---

#### 5. determineFinalStatus() - 确定最终状态

```java
private String determineFinalStatus(TaskInfo taskInfo, TaskKeyRecord task) {
    // 1. 非预测任务（如数据采集）直接成功
    if (!TaskEnum.isPrediction(taskInfo.getTaskType())) {
        return TaskStatus.SUCCESS.getStatus();
    }
    
    // 2. 预测任务需要等待UEM推理完成
    SubTaskPipeline subTaskPipeline = TaskUtil.buildUemPipeline(task, UEM_INFER_STEP);
    String status = taskDao.queryPipelineSecondTaskStatus(subTaskPipeline);
    if (StringUtils.isEmpty(status)) {
        return TaskStatus.RUNNING.getStatus();  // 还在跑
    }
    
    return TaskStatus.isFinishStatus(status) ? status : TaskStatus.RUNNING.getStatus();
}
```
关键逻辑:
- 数据采集任务不需要推理 → 直接成功
- 预测任务必须等 UEM 推理完成
---

#### 6. uemInfer() / startUemInfer() - UEM推理启动

```java
private void uemInfer(final TaskKeyRecord task, final TaskInfo taskInfo, final String sftpPreFix) {
    if (!TaskEnum.isPrediction(taskInfo.getTaskType())) {
        return;  // 非预测任务跳过
    }
    
    SubTaskPipeline subTaskPipeline = TaskUtil.buildUemPipeline(task, UEM_INFER_STEP);
    String status = taskDao.queryPipelineSecondTaskStatus(subTaskPipeline);
    
    if (TaskStatus.isSuccess(status)) {
        return;  // 已成功
    }
    
    if (StringUtils.isEmpty(status) || TaskStatus.isRunFailed(status) || TaskStatus.isStop(status)) {
        startUemInfer(task, taskInfo, subTaskPipeline, sftpPreFix);  // 启动
        return;
    }
    
    if (TaskStatus.isRunning(status)) {
        updateUemInfer(task, subTaskPipeline);  // 更新状态
    }
}
UEM推理启动条件（必须同时满足）:
private static boolean isMetStartCondition(final List<DataProcessHourInfo> hourInfos) {
    return !isFirstMrProcessed(hourInfos) || !isNoFailed(hourInfos);
}
// 首批MR已处理
private static boolean isFirstMrProcessed(final List<DataProcessHourInfo> hourInfos) {
    return hourInfos.stream()
        .filter(info -> Constants.MR.equals(info.getSource()))
        .filter(info -> TaskStatus.SUCCESS.getStatus().equals(info.getStatus()))
        .findFirst().isPresent();
}
// 没有失败的数据
private static boolean isNoFailed(final List<DataProcessHourInfo> hourInfos) {
    return hourInfos.stream()
        .map(DataProcessHourInfo::getStatus)
        .noneMatch(TaskStatus.FAILURE.getStatus()::equals);
}
```
---
### 三、错误处理机制
handleWhenStreamError() - 流错误处理

```java
private void handleWhenStreamError(TaskKeyRecord task) {
    log.error("handleWhenStreamError...{}", JSON.toJSONString(task));
    
    // 1. 更新流处理状态为错误
    taskDao.streamProcessWhenUemError(task.subTaskId(), task.batchNo(), task.rat());
    updateHourLevelStatus(task, TaskStatus.FAILURE.getStatus());
    
    // 2. 停止所有数据处理和同步
    List<Map<String, Object>> list = taskDao.queryStreams(task.subTaskId(), task.batchNo(), task.rat());
    stopDataProcessAndSync(task, list);
    // 3. 更新UEM管道状态为失败
    SubTaskPipeline subTaskPipeline = TaskUtil.buildUemPipeline(task, UEM_INFER_STEP);
    subTaskPipeline.setStatus(TaskStatus.FAILURE.getStatus());
    taskDao.updatePipelineSecondTaskStatus(subTaskPipeline);
}
```

---
### 四、特殊处理逻辑
错误等待 (ErrorWaiting)

```java
private Optional<List<DataProcessHourInfo>> shouldSetErrorWaiting(
    TaskContext context, Integer lastSerialNo) {
    // 如果最后一个MR正在运行，且当前批次有非MR数据失败
    if (isLastMRRunning(context.getTotalHourInfos(), lastSerialNo)) {
        List<DataProcessHourInfo> list = context.getCurrentHourInfos()
            .stream()
            .filter(info -> TaskStatus.isRunFailed(info.getStatus()))
            .filter(info -> !Constants.MR.equals(info.getSource()))  // 非MR
            .toList();
        if (CollectionUtils.isNotEmpty(list)) {
            return Optional.of(list);
        }
    }
    return Optional.empty();
}
```

设计意图: MR 是主数据源，当 MR 还在运行时，允许 PM、CDR 等其他数据源暂时失败，等待 MR 完成后自动恢复处理。

---
### 五、关键设计思想总结
设计思想	应用场景
分布式锁	保证集群环境下调度不重复
原子计数器	并发安全的成功计数
职责链模式	状态处理逻辑解耦
Builder模式	灵活构建TaskContext
Record类型	临时变量便捷封装
流控策略	MR完成后才允许其他源失败
对比模式	仅处理MR和PM数据源
---
### 六、调试关键日志

```java
log.warn("====={} Non-flow processing method", ...);    // 非流式处理
log.debug("--------->HourLevelExecutor context={}", ...); // 调试上下文
log.info("=====> Hour-level {}-{}-{}: {}", taskId, serialNo, source, status); // 核心执行
log.error("handleWhenStreamError...{}", ...);           // 错误处理
log.info("========> uem infer complete: {}-{}", ...);   // 推理完成
```
---
### 七、与 HourTaskHandleChain 的关系
```
HourLevelExecutor                         HourTaskHandleChain
      │                                          │
      │ doSchedule()                             │
      │   └─ hourLeverPipelineProcess()          │
      │         └─ chain.process(context) ──────→│ head.handle(context)
      │                                          │
      │              QueuingStatusHandler.handle()
      │              RunningStatusHandler.handle()
      │              FailedStatusHandler.handle()
      │              ErrorWaitingStatusHandler.handle()
      │              SuccessStrategy.handle()
```
HourLevelExecutor 负责调度和批次处理，将具体状态处理委托给 HourTaskHandleChain。



## HourTaskHandleChain 详解

### 一、整体架构
HourTaskHandleChain 是职责链模式的具体实现，用于处理小时级任务在不同生命周期阶段的状态流转。
类结构

```java
public class HourTaskHandleChain {
    private final HourTaskHandler head;
    public HourTaskHandleChain() {
        // 1. 创建各个处理器
        ErrorWaitingStatusHandler errorWaitingHandler = new ErrorWaitingStatusHandler();
        FailedStatusHandler failedHandler = new FailedStatusHandler();
        SuccessStatusHandler successHandler = new SuccessStatusHandler();
        QueuingStatusHandler queuingHandler = new QueuingStatusHandler();
        RunningStatusHandler runningHandler = new RunningStatusHandler();
        // 2. 构建责任链（注意顺序！）
        errorWaitingHandler.setNext(failedHandler);
        failedHandler.setNext(successHandler);
        successHandler.setNext(queuingHandler);
        queuingHandler.setNext(runningHandler);
        
        // 3. 链头是 ErrorWaitingStatusHandler
        this.head = errorWaitingHandler;
    }
    public void process(TaskContext context) {
        head.handle(context);
    }
}
```
链的顺序（重要！）

```
ErrorWaitingStatusHandler  →  FailedStatusHandler  →  SuccessStatusHandler  →  QueuingStatusHandler  →  RunningStatusHandler
        │                           │                       │                        │                        │
   错误等待状态                  失败状态                  成功状态                  排队状态                  运行状态
   (first check)              (second check)           (third check)            (fourth check)          (default handler)
```
设计巧妙之处: 每个 Handler 检查自己负责的状态，如果不符合就传递给下一个。这种"排除法"设计使得调试和扩展都很方便。

---

### 二、HourTaskHandler 接口

```java
public interface HourTaskHandler {
    // 流式数据源（MR和PM）
    Set<String> STREAM_SOURCE = ImmutableSet.of(Constants.MR, Constants.PM);
    
    String START = "start";
    String RUNNING = "run";
    String EMPTY = "empty";
    // 处理任务上下文
    void handle(TaskContext taskContext);
    
    // 设置下一个处理器
    void setNext(HourTaskHandler handler);
    
    // ---------- 工具方法 ----------
    
    // 判断是否是流式数据源
    default boolean isStreamData(final String source) {
        return STREAM_SOURCE.contains(source);
    }
    
    // 判断是否是MR流
    default boolean isMrStream(final DataProcessHourInfo hourInfo) {
        return Objects.equals(hourInfo.getSource(), Constants.MR);
    }
    
    // 生成任务ID
    default String getTaskId(final String taskId, final DataProcessHourInfo hourInfo) {
        Integer index = Constants.SOURCE_INDEX.get(hourInfo.getSource());
        return ModelFileUtil.getTaskId(taskId, hourInfo.getBatchNo()) 
            + Constants.SYMBOL + index + Constants.SYMBOL + hourInfo.getSerialNo();
    }
    
    // 判断是否有符合条件的状态
    default boolean isEffective(final TaskContext context, Predicate<String> p) {
        return context.getCurrentHourInfos().stream()
            .map(DataProcessHourInfo::getStatus).anyMatch(p);
    }
    
    // 获取结束标记
    default String getEndFlag(TaskContext context, DataProcessHourInfo hourInfo) {
        if (Objects.equals(hourInfo.getSource(), Constants.LABEL_CDR)) {
            return "_SUCCESS";
        }
        if (Objects.equals(hourInfo.getSource(), Constants.MR)) {
            int serialNo = hourInfo.getSerialNo();
            int count = Math.toIntExact(context.getTotalHourInfos().stream()
                .filter(this::isMrStream).count());
            return "MR-Event" + File.separator + count + "-" + (serialNo - 1) + ".end";
        }
        return StringUtils.EMPTY;
    }
    
    // 报告进度
    default void reportProgress(TaskRecordReport recordReport, 
        DataProcessHourInfo hourInfo, String type) {
        if (Objects.equals(hourInfo.getSource(), Constants.LABEL_CDR)) {
            return;
        }
        String key = String.format("record.stream.%s.%s.%s", 
            hourInfo.getInvoke(), type, hourInfo.getStatus());
        // ... 构建任务记录并报告
    }
}
```
---

### 三、各 Handler 详解
#### 1. QueuingStatusHandler（排队状态处理）
职责: 处理状态为 QUEUING（排队中）的 DataProcessHourInfo

```java
@Slf4j
public class QueuingStatusHandler implements HourTaskHandler {
    private HourTaskHandler next;
    @Override
    public void handle(final TaskContext context) {
        // 1. 如果当前批次没有排队状态，传递给下一个Handler
        if (!isEffective(context, TaskStatus::isQueuing)) {
            if (Objects.nonNull(next)) {
                next.handle(context);
            }
            return;
        }
        
        // 2. 遍历当前批次的每个小时数据
        for (DataProcessHourInfo hourInfo : context.getCurrentHourInfos()) {
            if (!TaskStatus.isQueuing(hourInfo.getStatus())) {
                continue;  // 跳过非排队状态
            }
            
            String taskId = getTaskId(context.getSubTaskId(), hourInfo);
            
            // 3. 根据调用类型分发处理
            if (Invoke.isUpload(hourInfo.getInvoke())) {
                // 3.1 上传类：执行数据上传
                handleUpload(context, hourInfo, taskId);
            } else if (isStreamData(hourInfo.getSource())) {
                // 3.2 流式数据源：执行流式数据处理
                updateStreamData(context, hourInfo, taskId);
            } else {
                // 3.3 非流式数据：检查前置条件后执行
                if (preRunFinish(context)) {
                    updateStatusAndInfo(context, hourInfo, taskId);
                }
            }
        }
    }
    
    private void handleUpload(TaskContext context, DataProcessHourInfo hourInfo, String taskId) {
        // CDR且未到结束流不处理
        if (Objects.equals(hourInfo.getSource(), Constants.LABEL_CDR) && !isEndStreamBegin(context)) {
            return;
        }
        
        // 构建同步记录并执行上传
        DataSyncBusiness dataSyncBusiness = context.getDataSyncBusiness();
        SecondScheduler scheduler = taskDao.querySecondSchedule(
            context.getSubTaskId(), context.getBatchNo(), context.getRat());
        String endFlag = getEndFlag(context, hourInfo);
        
        DataSyncRecord syncRecord = new DataSyncRecord(scheduler, taskId, 
            hourInfo.getSource(), endFlag, context.getSftpPrefix());
        
        TaskStatus uploadStatus = dataSyncBusiness.upload(syncRecord);
        hourInfo.setStatus(uploadStatus.getStatus());
        taskDao.updateDataProcessHourInfo(hourInfo);
        reportProgress(context.getRecordReport(), hourInfo, START);
    }
    
    private void updateStreamData(TaskContext context, DataProcessHourInfo hourInfo, String taskId) {
        // 检查前置条件：前一批次同源数据已处理完成
        if (!preNoneStreamProcessSuccess(context) || !preStreamSameSourceFinished(context, hourInfo)) {
            return;
        }
        
        // 老架构PM数据需要前置非流式处理完成
        boolean newArchitecture = ApiConfigSingleton.INSTANCE.getUseDaasArch();
        if (!newArchitecture && Objects.equals(hourInfo.getSource(), Constants.PM)) {
            if (!preNoneStreamProcessSuccess(context)) {
                return;
            }
        }
        
        updateStatusAndInfo(context, hourInfo, taskId);
    }
    
    // 检查前置非流式数据是否全部完成
    private boolean preNoneStreamProcessSuccess(TaskContext context) {
        return context.getTotalHourInfos()
            .stream()
            .filter(info -> info.getSerialNo() < context.getIndex())
            .filter(info -> !isStreamData(info.getSource()))
            .allMatch(info -> TaskStatus.isFinishStatus(info.getStatus()) 
                || (TaskStatus.isRunning(info.getStatus()) && Invoke.isUpload(info.getInvoke())));
    }
    
    // 检查前置同源数据是否完成
    private boolean preStreamSameSourceFinished(TaskContext context, DataProcessHourInfo currentHourInfo) {
        int preIndex = context.getIndex() - 1;
        Map<Integer, List<DataProcessHourInfo>> map = context.getMap();
        String currentSource = currentHourInfo.getSource();
        
        if (map.containsKey(preIndex)) {
            Optional<DataProcessHourInfo> optional = map.get(preIndex).stream()
                .filter(info -> Objects.equals(info.getSource(), currentSource))
                .findFirst();
            if (optional.isPresent()) {
                DataProcessHourInfo preHourInfo = optional.get();
                return TaskStatus.isSuccess(preHourInfo.getStatus()) 
                    || Invoke.isUpload(preHourInfo.getInvoke());
            }
        }
        return true;
    }
    
    // 核心执行方法
    private TaskStatus dataFunctionExecutor(TaskContext context, DataProcessHourInfo hourInfo, String taskId) {
        DataFunctionBusiness dataFunctionBusiness = context.getDataFunctionBusiness();
        SecondScheduler secondScheduler = taskDao.querySecondSchedule(
            context.getSubTaskId(), context.getBatchNo(), context.getRat());
        TaskData taskData = taskDao.queryModelTaskData(secondScheduler);
        
        taskData.setDataStartTime(hourInfo.getStartDataTime());
        taskData.setDataEndTime(hourInfo.getEndDataTime());
        
        TaskStatus execute = dataFunctionBusiness.execute(
            secondScheduler, taskData, taskId, hourInfo.getSource());
        
        log.info("=====> Hour-level {}-{}-{}: {}", taskId, hourInfo.getSerialNo(), hourInfo.getSource(), execute);
        return execute;
    }
    
    @Override
    public void setNext(HourTaskHandler handler) {
        this.next = handler;
    }
}
```

核心逻辑流程图:

```
QueuingStatusHandler.handle()
    │
    ├─→ isEffective(TaskStatus::isQueuing)? 
    │        │
    │        ├─ NO → next.handle(context)  // 传递
    │        │
    │        └─ YES → 遍历 currentHourInfos
    │                  │
    │                  ├─ Invoke.isUpload? → upload() 上传
    │                  │
    │                  ├─ isStreamData? → updateStreamData() 流式处理
    │                  │                        ├─ preNoneStreamProcessSuccess?
    │                  │                        └─ preStreamSameSourceFinished?
    │                  │
    │                  └─ 其他 → preRunFinish? → execute() 执行
```
---

#### 2. RunningStatusHandler（运行中状态处理）
**职责:** 处理状态为 `RUNNING`（运行中）的数据
**关键方法:**
- `updatePipelineStatus()` - 更新管道状态
- `updateTransferStatus()` - 更新传输状态
- `reportProgress()` - 报告进度
- `dataFunctionExecutor()` - 执行数据功能
- `getAfterStreamProcess()` - 获取后续流处理
- `processNextHourInfo()` - 处理下一批次数据
---
#### 3. FailedStatusHandler（失败状态处理）
职责: 处理失败状态，设置下一个处理器形成链式调用

```java
public class FailedStatusHandler implements HourTaskHandler {
    private HourTaskHandler next;
    @Override
    public void handle(TaskContext taskContext) {
        // 检查是否有失败状态
        if (!isEffective(taskContext, TaskStatus::isRunFailed)) {
            // 没有失败，传递给下一个
            if (Objects.nonNull(next)) {
                next.handle(taskContext);
            }
            return;
        }
        
        // 有失败状态 -> 链式调用下一个Handler继续处理
        if (Objects.nonNull(next)) {
            next.handle(taskContext);
        }
    }
    @Override
    public void setNext(HourTaskHandler handler) {
        this.next = handler;
    }
}
```
注意: FailedStatusHandler 不单独处理，而是将失败状态传递给下一个 Handler 继续处理。这是一种"链式组合"的设计。

---

#### 4. ErrorWaitingStatusHandler（错误等待处理）
职责: 处理 ERROR_WAITING 状态
特点:
- 作为链头，首先被调用
- 通常返回 true 表示已处理，不再传递给下一个
- 与 FailedStatusHandler 配合形成嵌套链
---
#### 5. SuccessStatusHandler（成功状态处理）
职责: 处理 SUCCESS 状态
设计简洁，成功状态只需要标记并继续往下传递。
---
### 四、处理流程总览

```
chain.process(context)
    │
    ├─→ ErrorWaitingStatusHandler.handle()
    │        │
    │        └─ is ERROR_WAITING?
    │              │
    │              ├─ YES → 处理后返回
    │              │
    │              └─ NO ↓
    │
    ├─→ FailedStatusHandler.handle()
    │        │
    │        └─ has FAILED?
    │              │
    │              ├─ YES → next.handle() 链式继续
    │              │
    │              └─ NO ↓
    │
    ├─→ SuccessStatusHandler.handle()
    │        │
    │        └─ is SUCCESS?
    │              │
    │              ├─ YES → 处理后返回
    │              │
    │              └─ NO ↓
    │
    ├─→ QueuingStatusHandler.handle()
    │        │
    │        └─ is QUEUING?
    │              │
    │              ├─ YES → 处理（上传/流式/执行）
    │              │
    │              └─ NO ↓ (传递给next)
    │
    └─→ RunningStatusHandler.handle()
             │
             └─ is RUNNING?
                   │
                   ├─ YES → 处理
                   │
                   └─ NO → 结束（兜底）
```
---
### 五、TaskContext 在链中的流动
每个 Handler 从 TaskContext 读取信息，处理后可能更新 context 中的某些状态：

```java
public class TaskContext {
    // 输入信息（只读）
    private Map<Integer, List<DataProcessHourInfo>> map;      // 所有批次数据
    private List<DataProcessHourInfo> totalHourInfos;         // 全部数据
    private List<DataProcessHourInfo> currentHourInfos;       // 当前批次数据
    private int index;                                        // 批次序号
    private String subTaskId;
    private int batchNo;
    private String rat;
    private String sftpPrefix;
    private Integer lastMRSerialNo;
    
    // 服务（只读）
    private ITaskDao taskDao;
    private DataFunctionBusiness dataFunctionBusiness;
    private DataSyncBusiness dataSyncBusiness;
    private TaskRecordReport recordReport;
    
    // 输出信息（可写）
    private AtomicInteger successCount;  // 成功计数
}
```

```
Handler 之间的协作模式:
Handler1 处理后可能修改 context 中的数据
    ↓
Handler2 读取修改后的 context 继续处理
    ↓
Handler3 ...
```
---
### 六、设计亮点
| 设计亮点  | 说明  |
| ------------ | ------------ |
| 职责链模式  |  状态处理逻辑解耦，每个Handler只负责一种状态 |
| 链头优先判断  |  ErrorWaiting在最前面，快速短路 |
| 链式组合  | FailedStatusHandler 不中断，继续传递给下一个  |
|  工具方法复用 | 接口提供 isEffective(), getTaskId() 等默认实现  |
|  可变顺序 | 通过 setNext() 动态调整处理器顺序  |
| 上下文传递  | TaskContext 作为数据载体在链中流动  |
	
---
### 七、扩展新 Handler 的方法
1. 实现 HourTaskHandler 接口
2. 在 HourTaskHandleChain 构造函数中添加
3. 设置 setNext() 建立链接

```java
public class NewStatusHandler implements HourTaskHandler {
    private HourTaskHandler next;
    
    @Override
    public void handle(TaskContext context) {
        if (!isEffective(context, TaskStatus::isNewStatus)) {
            if (Objects.nonNull(next)) {
                next.handle(context);
            }
            return;
        }
        // 处理逻辑...
    }
    
    @Override
    public void setNext(HourTaskHandler handler) {
        this.next = handler;
    }
}
// 在 HourTaskHandleChain 中添加
errorWaitingHandler.setNext(new NewStatusHandler());  // 插入到链中
newStatusHandler.setNext(failedHandler);  // 新Handler的下一个是failedHandler
```

## ErrorWaitingStatusHandler 和 SuccessStatusHandler 详解
### 一、ErrorWaitingStatusHandler（错误等待处理）
#### 1.1 类结构

```java
public class ErrorWaitingStatusHandler implements HourTaskHandler {
    private HourTaskHandler next;
    @Override
    public void handle(final TaskContext context) {
        // 1. 如果当前批次没有 ERROR_WAITING 状态，传递给下一个 Handler
        if (!isEffective(context, TaskStatus::isErrorWaiting)) {
            if (Objects.nonNull(next)) {
                next.handle(context);
            }
            return;
        }
        
        // 2. 检查 MR 是否完成
        if (!checkMRStatus(context)) {
            return;  // MR 还在跑，等待
        }
        
        // 3. MR 已完成，将所有 ERROR_WAITING 状态改为 FAILURE
        List<DataProcessHourInfo> currentHourInfos = context.getCurrentHourInfos();
        currentHourInfos.stream()
            .filter(info -> TaskStatus.isErrorWaiting(info.getStatus()))
            .forEach(hourInfo -> {
                hourInfo.setStatus(TaskStatus.FAILURE.getStatus());
                context.getTaskDao().updateDataProcessHourInfo(hourInfo);
            });
    }
}
```
#### 1.2 核心逻辑

```java
private static boolean checkMRStatus(final TaskContext context) {
    // 检查是否存在 MR 数据
    if (isNoneMrMatch(context)) {
        return false;
    }
    
    // 检查最后一批 MR 是否已完成（终态）
    return context.getCurrentHourInfos()
        .stream()
        .filter(info -> Constants.MR.equals(info.getSource()))
        .filter(info -> context.getLastMRSerialNo() == info.getSerialNo())
        .allMatch(info -> TaskStatus.isFinishStatus(info.getStatus()));
}
```
#### 1.3 处理流程图

```
ErrorWaitingStatusHandler.handle()
    │
    ├─→ isEffective(ERROR_WAITING)?
    │        │
    │        ├─ NO → next.handle(context)  // 不是错误等待状态，继续传递
    │        │
    │        └─ YES ↓
    │
    ├─→ checkMRStatus()  // 检查最后一批MR是否完成
    │        │
    │        ├─ MR 还在运行 → return（不做处理，等待下次调度）
    │        │
    │        └─ MR 已完成 ↓
    │
    └─→ 将所有 ERROR_WAITING → FAILURE
             └─ 更新到数据库
```
#### 1.4 设计意图
ErrorWaiting 状态的含义： 当最后一个 MR 还在运行时，允许其他数据源（如 PM、CDR）暂时失败，等待 MR 完成后统一处理。
例子：
- 假设有 5 个批次的数据，批次 5 是最后一个 MR
- 批次 5 中 PM 数据失败了，但它被标记为 ERROR_WAITING 而不是 FAILURE
- 因为 PM 可能依赖 MR 的某些结果，MR 还没跑完不能确定 PM 是否真的失败
- 等批次 5 的 MR 跑完后，如果 MR 成功，则 PM 的 ERROR_WAITING → FAILURE
- 如果 MR 也失败了，整个流式处理终止
---

### 二、SuccessStatusHandler（成功状态处理）
#### 2.1 类结构

```java
public class SuccessStatusHandler implements HourTaskHandler {
    private HourTaskHandler next;
    @Override
    public void handle(final TaskContext taskContext) {
        List<DataProcessHourInfo> currentHourInfos = taskContext.getCurrentHourInfos();
        
        // 1. 统计成功数量
        List<DataProcessHourInfo> successHourInfos = currentHourInfos.stream()
            .filter(hourInfo -> TaskStatus.isSuccess(hourInfo.getStatus()))
            .toList();
        
        // 2. 更新成功计数器
        if (CollectionUtils.isNotEmpty(successHourInfos)) {
            taskContext.getSuccessCount().addAndGet(successHourInfos.size());
        }
        
        // 3. 如果全部成功，不继续传递（短路）
        if (currentHourInfos.size() == successHourInfos.size()) {
            return;
        }
        
        // 4. 如果还有非成功状态，传递给下一个 Handler
        if (Objects.nonNull(next)) {
            next.handle(taskContext);
        }
    }
}
```
#### 2.2 处理流程图

```
SuccessStatusHandler.handle()
    │
    ├─→ 统计 currentHourInfos 中 SUCCESS 状态的记录
    │
    ├─→ successCount.addAndGet(success数量)
    │
    ├─→ currentHourInfos.size() == successHourInfos.size()?
    │        │
    │        ├─ YES（全部成功）→ return（短路，不传递给下一个）
    │        │
    │        └─ NO（有非成功状态）↓ → next.handle(context)
    │
    └─→ 传递给下一个 Handler（如 QueuingStatusHandler）
```
#### 2.3 设计意图
- 统计成功数量：用于 HourLevelExecutor 判断是否所有批次都成功
- 全部成功时短路：避免不必要的 Handler 调用
- 有非成功状态继续传递：让其他 Handler 处理（如排队中、运行中）
---
### 三、"处理任务"到底在处理什么？
#### 3.1 数据结构 DataProcessHourInfo

```java
@Data
@Builder
public class DataProcessHourInfo {
    private String subTaskId;    // 子任务ID
    private int batchNo;         // 批次号
    private int serialNo;        // 序号（批次内的序列号）
    private String startDataTime; // 数据开始时间
    private String endDataTime;   // 数据结束时间
    private String status;        // 状态
    private String createTime;    // 创建时间
    private String invoke;        // 调用类型（上传/执行等）
    private String source;        // 数据源（MR/PM/CDR等）
    private String rat;           // 无线接入技术（4G/5G等）
}
```
每个 DataProcessHourInfo 代表：一小时内的、某种数据源的、处理任务
#### 3.2 数据源类型（source）
![](./images/数据源类型.PNG)
#### 3.3 调用类型（invoke）
常见类型包括：
- UPLOAD - 上传数据
- EXECUTE - 执行数据处理
- 其他业务自定义类型
#### 3.4 "处理任务"的真实含义
一句话总结：处理的是"每小时每数据源"的数据处理任务记录。
一个任务示例：

```
subTaskId = "task_001"
batchNo = 1
serialNo = 1
source = "MR"        // 测量报告数据
invoke = "UPLOAD"    // 需要先上传
startDataTime = "2025-07-31 10:00:00"
endDataTime = "2025-07-31 11:00:00"
status = "QUEUING"   // 当前状态：排队中
rat = "5G"           // 5G网络
```
#### 3.5 处理流程
一个 DataProcessHourInfo 的生命周期：

```
1. 创建 → status = QUEUING
   │
   ├─→ QueuingStatusHandler 处理
   │        │
   │        ├─ invoke = UPLOAD → 上传数据文件
   │        │
   │        └─ invoke = EXECUTE → 执行数据功能
   │
   ├─→ status 变为 RUNNING
   │        │
   │        └─→ RunningStatusHandler 处理
   │
   ├─→ status 变为 SUCCESS / FAILURE / ERROR_WAITING
   │        │
   │        ├─→ SuccessStatusHandler（统计成功）
   │        │
   │        ├─→ FailedStatusHandler（处理失败）
   │        │
   │        └─→ ErrorWaitingStatusHandler（错误等待）
   │
   └─→ 完成
```
#### 3.6 Handler 处理的具体内容
![](./images/handler具体操作.PNG)
#### 3.7 与数据库的交互

```java
// 读取
List<DataProcessHourInfo> hourInfos = taskDao.queryHourInfos(subTaskId, batchNo, rat);
// 更新状态
taskDao.updateDataProcessHourInfo(hourInfo);
// 读取单个
DataProcessHourInfo info = taskDao.queryDataProcessHourInfo(id);
```
---
### 四、总结
#### 4.1 Handler 处理本质
不是"处理任务本身"，而是"处理 DataProcessHourInfo 这条数据记录的状态"。
每个 Handler 的工作流程：
1. 读取 context.getCurrentHourInfos() 获取当前批次的数据记录
2. 筛选 符合自己处理状态的记录（如 isQueuing、isSuccess）
3. 执行业务操作（上传、执行、更新状态）
4. 更新 DataProcessHourInfo 到数据库
5. 传递给下一个 Handler 或 短路返回
#### 4.2 状态机全景

```
                        ┌─────────────────────────────────────┐
                        │          DataProcessHourInfo         │
                        │  (每小时每数据源的处理任务记录)        │
                        └─────────────────────────────────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    ▼                       ▼                       ▼
              QUEUING                   RUNNING                ERROR_WAITING
                    │                       │                       │
                    ▼                       ▼                       │
      ┌─────────────────────┐     ┌─────────────────┐             │
      │ QueuingStatusHandler │     │RunningStatusHandler│        │
      │  - upload()          │     │  - updateStatus() │        │
      │  - execute()         │     │  - reportProgress()│       │
      └──────────┬──────────┘     └────────┬────────┘             │
                 │                         │                       │
                 └────────────┬────────────┘                       │
                              ▼                                     │
                        SUCCESS / FAILURE                          │
                              │                                     │
                              ▼                                     │
               ┌──────────────────────────────┐                    │
               │      SuccessStatusHandler     │                    │
               │  - 累加 successCount          │                    │
               │  - 全部成功则短路返回          │                    │
               └──────────────────────────────┘                    │
                                                          ┌───────┴───────┐
                                                          ▼               ▼
                                                    保持 ERROR_WAITING  → FAILURE
                                                    (MR未完成)        (MR已完成)
```

## 问


### 问题1：FailedStatusHandler 的意义是什么？

表面现象
看起来 FailedStatusHandler 什么都没做，只是传递给下一个 Handler。

深层设计意图
FailedStatusHandler 不是"处理失败"，而是"检测到失败后不中断链"


职责链的两种类型：
1. 中断型（Short-circuit）：Handler处理后就返回，不继续传递
      
```
if (canHandle()) {
       handle();
       return;  // 处理完就结束
   }
   next.handle();
```
   
2. 穿透型（Transparent）：不管是否处理，都继续传递
    
```
  if (canHandle()) {
       handle();  // 做点什么
   }
   next.handle();  // 一定传递下去
```
   
实际上 FailedStatusHandler 是个"看门人"

```
// FailedStatusHandler 的实际作用：
// 确保即使有失败状态，链也会继续传递下去
// 而不是在 SuccessStatusHandler 就短路返回
```

---
### 问题2：不会同时做 UPLOAD 和 EXECUTE 吧？
对，不会同时做。
每个 DataProcessHourInfo 只有一种 invoke 类型
// invoke 字段表示这个任务"应该如何被调用"
hourInfo.getInvoke()  // UPLOAD / EXECUTE / 其他
// 不会同时是 UPLOAD 和 EXECUTE
但是！一个批次内可能有多个 DataProcessHourInfo
// currentHourInfos 可能包含多条记录：

```
[
    { source: "MR", invoke: "UPLOAD", status: "QUEUING" },
    { source: "PM", invoke: "EXECUTE", status: "QUEUING" },
    { source: "CDR", invoke: "EXECUTE", status: "QUEUING" }
]
```
// 循环处理时：
// MR → 进入 handleUpload() 分支
// PM → 进入 updateStreamData() 分支
// CDR → 进入 updateStatusAndInfo() 分支
流程图

```
QueuingStatusHandler.handle()
    │
    └─→ 遍历 currentHourInfos (可能包含多个不同 source/invoke 的记录)
            │
            ├─→ { source: "MR", invoke: "UPLOAD" } → handleUpload()
            ├─→ { source: "PM", invoke: "EXECUTE" } → updateStreamData()
            └─→ { source: "CDR", invoke: "EXECUTE" } → updateStatusAndInfo()
```
---
### 问题3：为什么用责任链模式？
![](./images/模型平台-各模式对比.PNG)

为什么不用策略模式？

```
// 策略模式的问题：
interface HandlerStrategy {
    void handle(TaskContext context);
}
// 选择一个策略
HandlerStrategy strategy = selectStrategy(context);
strategy.handle(context);  // 只会执行一个
```
策略模式只选一个执行，但这里需要"依次检查所有状态"
<span style="color:rgb(216,27,68)">**这里是按批次处理，批次中有多个任务，可能会被多个责任处理，要传递，所以不要用策略模式**</span>。

为什么不用状态模式？

```
// 状态模式的问题：
class TaskContext {
    State state;  // 当前状态
    
    void handle() {
        state.handle(this);  // 状态自己决定下一步
    }
}
// 但这里的状态转换不是自动的
// 需要外部（Handler）根据业务逻辑决定是否转换
```
状态模式假设状态转换是内嵌的，但这里的处理逻辑更复杂（检查invoke、检查source、检查前置条件）

为什么用责任链模式？优势在哪？

1. 短路与穿透可以混合

```java
// SuccessStatusHandler：全部成功时短路
if (currentHourInfos.size() == successHourInfos.size()) {
    return;  // 短路，不传递给下一个
}
// FailedStatusHandler：无论是否处理，都传递
next.handle(taskContext);  // 一定传递
```
2. Handler 可以修改 Context

```java
// QueuingStatusHandler 处理后，可能修改了 hourInfo 的状态
hourInfo.setStatus(newStatus);
taskDao.updateDataProcessHourInfo(hourInfo);
// 下一个 Handler (RunningStatusHandler) 看到的是更新后的状态
next.handle(context);  // context 里已经是新状态了
```
3. 动态组合

```
// HourTaskHandleChain 可以在运行时决定链的顺序和内容
ErrorWaitingStatusHandler errorHandler = new ErrorWaitingStatusHandler();
FailedStatusHandler failedHandler = new FailedStatusHandler();
...
errorHandler.setNext(failedHandler);
// 可以轻松替换、新增、删除 Handler
```
4. 单一职责

```
// 每个 Handler 只管一种状态
QueuingStatusHandler      → 只管 QUEUING
RunningStatusHandler      → 只管 RUNNING
SuccessStatusHandler      → 只管 SUCCESS
ErrorWaitingStatusHandler → 只管 ERROR_WAITING
FailedStatusHandler       → 只管检测失败并保证链继续
责任链的核心优势：可插拔的检查和处理
// 可以理解为一系列"过滤器"
request → Filter1 → Filter2 → Filter3 → response
// 每个Filter决定：
// 1. 我处理（处理后继续 or 停止）
// 2. 我不处理（传递给下一个）
```
---
### 问题4：为什么传递 context 而不是直接传 DataProcessHourInfo？
Context 的内容

```java
public class TaskContext {
    // ========== 输入信息（只读）==========
    private Map<Integer, List<DataProcessHourInfo>> map;      // 所有批次
    private List<DataProcessHourInfo> totalHourInfos;         // 全部数据
    private List<DataProcessHourInfo> currentHourInfos;       // 当前批次
    private int index;                                        // 批次序号
    private String subTaskId;
    private int batchNo;
    private String rat;
    private String sftpPrefix;
    private Integer lastMRSerialNo;
    
    // ========== 服务（只读）==========
    private ITaskDao taskDao;
    private DataFunctionBusiness dataFunctionBusiness;
    private DataSyncBusiness dataSyncBusiness;
    private TaskRecordReport recordReport;
    
    // ========== 输出信息（可写）==========
    private AtomicInteger successCount;
}
```
Handler 需要的不只是 hourInfo
看 QueuingStatusHandler 需要什么：

```java
public void handle(final TaskContext context) {
    // 需要当前批次的所有信息
    for (DataProcessHourInfo hourInfo : context.getCurrentHourInfos()) {
        // 需要查询数据库
        SecondScheduler scheduler = taskDao.querySecondSchedule(...);
        TaskData taskData = taskDao.queryModelTaskData(secondScheduler);
        
        // 需要业务服务
        dataSyncBusiness.upload(syncRecord);
        dataFunctionBusiness.execute(...);
        
        // 需要SFTP前缀
        String sftpPrefix = context.getSftpPrefix();
        
        // 需要报告进度
        recordReport.recordReporting(taskRecord, 0);
    }
}
```

Context 的设计意图
Context 是依赖注入的"载体"

```
调用者 (HourLevelExecutor)                    Handler (QueuingStatusHandler)
        │                                              │
        │ doSchedule()                                 │
        │   ├─ 创建 context                           │
        │   │    ├─ 注入 taskDao (从Spring容器)        │
        │   │    ├─ 注入 dataFunctionBusiness         │
        │   │    ├─ 注入 dataSyncBusiness             │
        │   │    └─ 注入其他服务...                    │
        │   │                                          │
        │   └─ chain.process(context) ──────────────→ context 包含所有依赖
        │                                              │
        │                            Handler 直接从 context 取用，不需要自己获取
```

### 总结
|  问题 | 答案  |
| ------------ | ------------ |
|  FailedStatusHandler 意义 | "看门人"角色，确保即使有失败状态，链也不会短路，会继续传递下去让后续Handler感知到失败  |
| 不会同时 UPLOAD 和 EXECUTE  |  对，每条 DataProcessHourInfo 只有一个 invoke 类型；但一个批次内可能混合多种 invoke 类型的记录 |
|  为什么用责任链 | 可以混合短路/穿透、可修改Context传递状态、动态组合、单一职责；策略/工厂/状态模式都无法实现这种"依次检查并可能处理"的流程  |
|  为什么传 Context | Handler 需要的不只是数据（hourInfo），还需要服务（taskDao, dataFunctionBusiness）、配置（sftpPrefix）、批次信息等；Context 是依赖注入的载体  |

	
	
	
	
	

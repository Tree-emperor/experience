

# 任务创建详细流程
## 第一阶段：HTTP请求接收
入口: TaskOperatorController.addTask() (第73行)

```java
@PostMapping(value = "task-add", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public RestResponse addTask(@RequestPart(name = "taskInfo") String parameter,
    @RequestPart(name = "file", required = false) MultipartFile file) {
    return handleTaskOperation(parameter, file, ValidatorType.CREATE, taskOperatorService::createTask);
}
```
请求流程:
1. 接收 taskInfo (JSON字符串) 和可选的标注文件
2. 解析 taskInfo 为 TaskInfo 对象
3. 设置默认值:
   - creator: 从请求头获取当前用户名
   - storeLoc: 从配置获取存储位置
   - solutionType: 默认为 STANDARD
4. 执行校验链 ValidatorChain.validate()
5. 使用分布式锁 ReentrantLock 保证并发安全
6. 调用 taskOperatorService.createTask()
---
## 第二阶段：任务创建核心逻辑
核心方法: TaskOperatorService.createTask() (第114行)

```java
public RestResponse createTask(final MultipartFile file, final TaskInfo taskInfo) throws Exception {
    // 1. 生成taskId
    String taskId = taskInfo.getTaskId() == null || taskInfo.getTaskId().isEmpty()
        ? taskDao.generateTaskId()  // 分布式ID生成器
        : taskInfo.getTaskId();
    taskInfo.setTaskId(taskId);
    // 2. 执行检查并保存
    return checkAndSaveTask(file, taskInfo);
}
```
关键步骤1: 校验链检查 checkAndSaveTask() (第124行)

```java
private RestResponse checkAndSaveTask(final MultipartFile file, final TaskInfo taskInfo) throws Exception {
    RestResponse response = RestResponse.success();
    // 构建责任链检查
    TaskCheckChain checkChain = new TaskCheckChain.Builder()
        .add(new DataResourceCheckStrategy(gisRestfulUtil, configDao))      // 数据资源检查
        .add(new MapRoadGridCheckStrategy(gisRestfulUtil, taskDao))         // 路网网格检查
        .add(new LicenseCheckStrategy(licenseDao, licenseBusiness, licenseFactory, cellQueryBusiness))  // 许可证检查
        .build();
    checkChain.check(taskInfo, response);
    
    if (response.getResultCode() == RestResponse.SUCCESS_CODE) {
        saveMultipartFile(file, taskInfo);  // 保存标注文件
        taskDao.saveTask(taskInfo);          // 持久化任务
    }
    return response;
}
```
校验内容详解:
检查策略	作用
DataResourceCheckStrategy	检查数据源是否可用、 polygon 覆盖的基站数量等
MapRoadGridCheckStrategy	检查路网网格划分是否正确、是否有冲突
LicenseCheckStrategy	检查许可证额度是否足够、场景是否符合
关键步骤2: 文件保存 saveMultipartFile() (第143行)

```java
private void saveMultipartFile(final MultipartFile file, final TaskInfo taskInfo) throws IOException {
    if (Objects.isNull(file) || file.isEmpty()) {
        return;  // 可选文件，可能为空
    }
    String fileName = file.getOriginalFilename();
    if (HadoopUtil.isUseHdfs()) {
        saveToHdfs(file, taskInfo, fileName);
    } else {
        saveToLocal(file, taskInfo, fileName);  // 保存到本地磁阵共享目录
    }
}
```
关键步骤3: 任务持久化 taskDao.saveTask(taskInfo)
- 插入任务主表 (task_info)
- 初始化任务状态为 INIT
---
## 第三阶段：事件发布与异步处理
任务保存后，不会同步执行，而是发布 Spring Event 事件：
事件发布: 任务创建接口 本身不发布事件，事件在周期任务触发时发布
让我重新梳理正确流程：
正确的任务触发流程:
1. 手动创建任务: TaskOperatorService.createTask() → 保存到数据库，状态为 INIT
2. 周期任务调度: PeriodTaskExecutor.schedule() (第53行)

```java
      public void schedule() {
       List<TaskPeriodInfo> taskPeriodInfos = taskDao.queryPeriodInfos();  // 查询周期任务配置
       for (TaskPeriodInfo periodInfo : taskPeriodInfos) {
           TaskInfo taskInfo = taskDao.mainTaskInfo(periodInfo.getTaskId());
           // 检查是否需要执行
           if (isNeedExecutor(taskType, variableMap, capacityMap)) {
               processTaskPeriod(periodInfo);  // 处理周期信息
           }
       }
   }
```
   
3. 发布创建事件: processTaskPeriod() (第102行)

```java
      private void processTaskPeriod(PeriodVariable variable) {
       for (IPeriodTask periodTask : periodTasks) {
           if (periodTask.effective(executionMode)) {
               periodTask.build(periodInfo, taskInfo);
               break;
           }
       }
       if (periodInfo.isNeedExecutor()) {
           applicationEventPublisher.publishEvent(new ModelTaskCreateEvent(this, periodInfo));
       }
   }
```
   
---
## 第四阶段：异步消费与智算任务创建
消费者: ModelTaskCreateConsumer.consumer() (第62行)

```java
@Async("model-sync")  // 异步执行
@Override
@EventListener(value = ModelTaskCreateEvent.class)
public void consumer(final ModelTaskCreateEvent event) {
    submitTask(event.getTaskPeriodInfo());
}
```
核心方法: submitTask() (第81行)

```java
private void submitTask(TaskPeriodInfo taskPeriodInfo) throws Exception {
    // 1. 获取任务信息
    TaskInfo taskInfo = taskDao.mainTaskInfo(taskPeriodInfo.getTaskId());
    taskInfo.setStartDataTime(taskPeriodInfo.getStartDataTime());
    taskInfo.setEndDataTime(taskPeriodInfo.getEndDataTime());
    
    // 2. 读取Pipeline配置
    String modelLowerCase = taskInfo.getModelName().toLowerCase(Locale.ROOT);
    String fileName = "TaskPipeline-" + modelLowerCase + ".json";
    Map<String, Object> script = taskDao.getScript(modelLowerCase, FileModule.PIPELINE.getType(), fileName);
    
    // 3. 执行策略链
    TaskGenerationStrategyChain strategyChain = new TaskGenerationStrategyChain(...);
    strategyChain.execute(taskInfo, json);  // 核心创建逻辑
    
    // 4. 更新主任务状态为 RUNNING
    taskDao.updateMainTaskStatus(taskInfo.getTaskId(), TaskStatus.RUNNING.getStatus());
}
```

---
## 第五阶段：策略链构建Pipeline
策略链: TaskGenerationStrategyChain.execute() (第45行)

```java
public void execute(TaskInfo taskInfo, JSONObject config) throws Exception {
    // 责任链: RatStrategy → ModelStrategy → PipelineStrategy
    strategy.generateAndSubmitTasks(taskInfo, config, taskInfo.getRat(), taskInfo.getModels());
}
核心Pipeline策略: PipelineStrategy.generateAndSubmitTasks() (第90行)
public void generateAndSubmitTasks(final TaskInfo taskInfo, final JSONObject config, 
    final String rat, final JSONObject model) throws Exception {
    createSubTask(taskInfo, config, rat, model);
}
```

---
## 第六阶段：创建智算任务与二级调度器
核心方法: createSubTask() (第95行)

```java
private void createSubTask(final TaskInfo taskInfo, final JSONObject json, 
    final String rat, final JSONObject model) throws Exception {
    
    // ====== 关键步骤1: 调用智算侧API创建任务 ======
    ImmutableTriple<String, String, Long> intelligentTrip = createIntelligentTask(taskInfo);
    String subTaskId = intelligentTrip.left;  // 智算侧返回的subTaskId
    
    // ====== 构建Pipeline二级调度器 ======
    MutablePair<List<SecondScheduler>, List<TaskData>> pair = buildPipeline(taskInfo, json, subTaskId, model);
    List<SecondScheduler> secondSchedulers = pair.left;
    
    // ====== 数据筛选（根据配置）======
    if (taskInfo.getDatasetOnly()) {
        secondSchedulers.removeIf(...)  // 仅保留数据集生成流水线
    }
    if (!HadoopUtil.isUseHdfs()) {
        secondSchedulers.removeIf(Invoke::isDownload);  // 移除下载步骤
    }
    
    // ====== 插入数据库 ======
    taskDao.insertSecondSchedulers(secondSchedulers);  // 二级调度器列表
    taskDao.insertTaskData(pair.right);                 // 任务数据
    
    // ====== 创建子任务记录 ======
    SubTaskParameter subTaskParameter = new SubTaskParameter(taskInfo, intelligentTrip, rat, model, pair.right);
    taskDao.addSubTask(builderSubTaskInfo(subTaskParameter));
}
```

---
## 第七阶段：调用智算侧API
核心方法: createIntelligentTask() (第123行)

```java
private ImmutableTriple<String, String, Long> createIntelligentTask(final TaskInfo taskInfo) throws IOException {
    // 1. 构建请求参数
    JSONObject input = buildCreateTaskParameter(taskInfo);
    
    // 2. 调用智算侧API
    JSONObject response = ApiFabricUtil.sendPostJsonRequest(createTask, input, taskInfo.getTaskType());
    
    // 3. 解析响应
    if (resultCode == 0) {
        String subTaskId = data.getString("subTaskId");
        String sftpMountPrefix = data.getString("sftpMountPrefix");
        long intelligentSystemTime = data.getLongValue("systemTime");
        return new ImmutableTriple<>(subTaskId, sftpMountPrefix, intelligentSystemTime);
    } else {
        // 失败处理
        handlerWhenCreateIntelligentTaskFailed(taskInfo, subTaskId, errno, resultCode);
        throw new ModelException("createIntelligentTask failed");
    }
}
```
请求参数构建 buildCreateTaskParameter() (第170行):

```java
private JSONObject buildCreateTaskParameter(final TaskInfo taskInfo) {
    JSONObject cell = statisticsCellNumber(taskInfo.getPolygon(), taskInfo);
    return new JSONObject()
        .fluentPut("taskId", taskInfo.getTaskId())
        .fluentPut("taskType", taskInfo.getTaskType())
        .fluentPut("storeLoc", taskInfo.getStoreLoc())
        .fluentPut("polygonName", ...)      // 多边形名称
        .fluentPut("cellCount", cell)       // 基站统计
        .fluentPut("rat", taskInfo.getRat())
        .fluentPut("osType", ...)
        .fluentPut("model", taskInfo.getModels());
}
```
---
## 第八阶段：构建二级调度器
核心方法: buildPipeline() (第258行)

```java
private MutablePair<List<SecondScheduler>, List<TaskData>> buildPipeline(...) {
    JSONArray jsonArray = json.getJSONArray("pipeline");  // 从配置读取流水线定义
    List<SecondScheduler> pipelines = createSecondSchedulers(taskInfo, jsonArray, subTaskId);
    TaskData taskData = createTaskData(taskInfo, subTaskId, model);
    
    // 根据是否按Polygon分批，决定单批次或多批次
    boolean isSplitByPolygon = json.getBooleanValue("is-split-by-cell");
    return isSplitByPolygon
        ? createMultiBatchSchedulersAndTaskData(...)  // 多批次
        : createSingleBatchSchedulersAndTaskData(...); // 单批次
}
```
调度器创建 createSecondSchedulers() (第338行):

```java
private List<SecondScheduler> createSecondSchedulers(...) {
    for (JSONObject object : jsonArray) {
        SecondScheduler secondScheduler = JSON.to(SecondScheduler.class, object);
        secondScheduler.setTaskId(subTaskId);
        secondScheduler.setRat(taskInfo.getRat());
        secondScheduler.setBatchNo(1);
        secondScheduler.setModel(taskInfo.getModels());
        processSecondScheduler(taskInfo, secondScheduler, atomicInteger, pipelines);
    }
}
```
批量处理:
- 如果 isRatBatch=true: 按RAT分裂（如NR、LTE分开执行）
- 如果 isModelBatch=true: 按模型分裂（如uem、bslm分开执行）
---
## 完整时序图

```
用户请求 POST /task/v1/task-add
    │
    ▼
TaskOperatorController.addTask()
    │
    ▼
TaskOperatorService.createTask()
    │
    ├─→ taskDao.generateTaskId() 生成任务ID
    │
    ├─→ TaskCheckChain 检查
    │   ├─→ DataResourceCheckStrategy
    │   ├─→ MapRoadGridCheckStrategy
    │   └─→ LicenseCheckStrategy
    │
    ├─→ saveMultipartFile() 保存标注文件到HDFS/本地
    │
    └─→ taskDao.saveTask() 持久化任务（状态=INIT）
    │
    ▼
[周期调度器触发] PeriodTaskExecutor.schedule()
    │
    ▼
[发布事件] ModelTaskCreateEvent
    │
    ▼
[异步消费] ModelTaskCreateConsumer.consumer()
    │
    ├─→ taskDao.getScript() 读取Pipeline配置
    │
    ▼
TaskGenerationStrategyChain.execute()
    │
    ▼
PipelineStrategy.generateAndSubmitTasks()
    │
    ├─→ createIntelligentTask() 调用智算API创建任务
    │   └─→ ApiFabricUtil.sendPostJsonRequest(createTask, ...)
    │       └─→ 返回 subTaskId, sftpMountPrefix
    │
    ├─→ buildPipeline() 构建二级调度器
    │   ├─→ createSecondSchedulers() 解析pipeline配置
    │   ├─→ processIfProcessByHour() 流式处理
    │   └─→ processIfPredictionExplainable() 可解释性
    │
    ├─→ taskDao.insertSecondSchedulers() 保存调度器
    │
    ├─→ taskDao.addSubTask() 创建子任务记录
    │
    └─→ taskDao.updateMainTaskStatus(RUNNING)
    │
    ▼
[定时调度] TaskStatusExecutor.schedule()
    │
    ├─→ 查询未完成的子任务
    │
    └─→ SchedulerChain 处理
        ├─→ QueuingStrategy → IntelligentInvokeExecutor.execute()
        └─→ RunningStrategy → IntelligentInvokeExecutor.status()
```

## 概念澄清：主任务、子任务、Pipeline、SecondScheduler
### 1. TaskInfo（主任务）
定义: 用户提交的完整任务，包含所有执行所需参数

```
TaskInfo {
    taskId              // 任务ID（用户或系统生成）
    taskName            // 任务名称
    taskType            // 任务类型：TRAINING/PREDICTION/EVALUATION
    modelName           // 模型名称
    rat                 // 无线接入类型：NR/LTE/NR,LTE
    polygon             // 地理区域列表
    models              // 模型配置JSON（包含uem、bslm等模型版本）
    startDataTime       // 数据开始时间
    endDataTime         // 数据结束时间
    explainable         // 是否可解释
    datasetOnly         // 是否仅生成数据集
    ...
}
```
特点: 
- 一个用户请求创建一个 TaskInfo
- 是任务创建的入口实体
- 包含完整的业务参数
---
### 2. SubTaskInfo（子任务）
定义: 调用智算侧API后，智算侧返回的实际执行单元

```
SubTaskInfo {
    taskId              // 关联的主任务ID
    subTaskId           // 智算侧返回的子任务ID（全局唯一）
    subTaskStatus       // 子任务状态
    startDataTime       // 数据开始时间
    endDataTime         // 数据结束时间
    subModel            // 子任务使用的模型配置
    subRat              // 子任务使用的RAT类型
    sftpMountPrefix     // 智算侧返回的SFTP挂载路径
    ...
}
```
创建时机: PipelineStrategy.createIntelligentTask() (第123行)

```java
// 调用智算侧API
JSONObject response = ApiFabricUtil.sendPostJsonRequest(createTask, input, taskInfo.getTaskType());
// 智算侧返回 subTaskId
String subTaskId = data.getString("subTaskId");
```
什么时候一个主任务会产生多个子任务:
场景	触发条件	结果
按Polygon分裂	isSplitByCell=true 且基站的cell数量超过限制	每个分裂的polygon生成一个SubTaskInfo，但共用同一个subTaskId + 不同batchNo

**注意: 代码中实际上一个主任务通常只创建一个SubTaskInfo，但会在内部按 batchNo 分成多个批次**

#### 举例说明
假设任务需求：
- 覆盖区域有 3 个 Polygon（P1、P2、P3）
- 每个 Polygon 包含的基站数量总和超过了系统限制（cellLimit）
- Pipeline 配置：is-split-by-cell: true
分批过程

```java
// PipelineStrategy.splitPolygon() - 按cellLimit分裂Polygon
splitPolygon(taskInfo):
    polygon列表: [P1, P2, P3]
    cellLimit: 1000
    
    // 假设 P1有400基站，P2有500基站，P3有300基站
    // P1+P2=900 <= 1000，可以合并
    // P1+P2+P3=1200 > 1000，需要分裂
    
    结果: [[P1, P2], [P3]]  // 分成2个批次
```
生成的 SecondScheduler 结构

```
SubTaskInfo:
    taskId: "TASK_001"
    subTaskId: "SUB_123"  (智算侧返回，全局唯一)
```
    
SecondScheduler列表:

```
    ┌─────────────────────────────────────────────────────────────┐
    │ batchNo = 1                                                │
    │   SecondScheduler(subTaskId="SUB_123", batchNo=1, step=1)  │
    │   SecondScheduler(subTaskId="SUB_123", batchNo=1, step=2)  │
    │   SecondScheduler(subTaskId="SUB_123", batchNo=1, step=3)  │
    │   ...                                                      │
    ├─────────────────────────────────────────────────────────────┤
    │ batchNo = 2                                                │
    │   SecondScheduler(subTaskId="SUB_123", batchNo=2, step=1)  │
    │   SecondScheduler(subTaskId="SUB_123", batchNo=2, step=2)  │
    │   SecondScheduler(subTaskId="SUB_123", batchNo=2, step=3)  │
    │   ...                                                      │
    └─────────────────────────────────────────────────────────────┘
```

|维度	|说明|
|--|--|
|subTaskId|	全局唯一，智算侧识别一个任务的标识|
|batchNo|	本地分批编号，用于控制执行顺序和数据隔离|
|关系|	1个 subTaskId → N个 batchNo|
|执行	|串行执行，只有上一batch全部成功，下一batch才开始|
|原因	|控制单次处理的数据量，避免资源耗尽|

**一句话概括**: 智算侧只知道一个任务(subTaskId)，但本地平台为了可控执行，把这个任务分成了多个批次(batchNo)，每个批次处理一部分数据，必须按顺序执行。

---
### 3. Pipeline（流水线配置）
定义: 声明式的任务执行流程配置文件
存储位置: ModelTaskCreateConsumer 从数据库读取

```java
String fileName = "TaskPipeline-" + modelLowerCase + ".json";
Map<String, Object> script = taskDao.getScript(modelLowerCase, FileModule.PIPELINE.getType(), fileName);
```
配置：
Pipeline配置存储在JSON文件中，配置示例结构:

```json
{
  "training": {
    "is-multi-rat-task": true,
    "is-multi-model-task": true,
    "is-split-by-cell": false,
    "pipeline": [
      {
        "invoke": "dataSetProcessing",
        "descZh": "数据集加工",
        "descEn": "DataSetProcessing",
        "ratBatch": true,
        "modelBatch": false
      },
      {
        "invoke": "upload",
        "descZh": "上传",
        "descEn": "Upload",
        "ratBatch": true,
        "modelBatch": false
      },
      {
        "invoke": "modelTrain",
        "descZh": "增训",
        "descEn": "Train",
        "ratBatch": false,
        "modelBatch": true
      },
      {
        "invoke": "download",
        "descZh": "下载",
        "descEn": "download",
        "ratBatch": false,
        "modelBatch": false
      },
      {
        "invoke": "evaluateResult",
        "descZh": "评估结果处理",
        "descEn": "download",
        "ratBatch": false,
        "modelBatch": false
      }
    ]
  },
  "prediction": {
    "is-multi-rat-task": true,
    "is-multi-model-task": true,
    "is-split-by-cell": true,
    "pipeline": [
      {
        "invoke": "dataSetProcessing",
        "descZh": "数据集加工",
        "descEn": "DataSetProcessing",
        "ratBatch": true,
        "modelBatch": false,
        "isProcessByHour": true
      },
      {
        "invoke": "upload",
        "descZh": "上传",
        "descEn": "Upload",
        "ratBatch": true,
        "modelBatch": false,
        "isProcessByHour": true
      },
      {
        "invoke": "modelInference",
        "descZh": "推理",
        "descEn": "modelInference",
        "ratBatch": false,
        "modelBatch": false
      },
      {
        "invoke": "download",
        "descZh": "下载",
        "descEn": "download",
        "ratBatch": false,
        "modelBatch": false
      },
      {
        "invoke": "cbbProcessing",
        "descZh": "后置处理",
        "descEn": "cbbProcessing",
        "ratBatch": false,
        "modelBatch": false
      }
    ]
  },
  "evaluation": {
    "is-multi-rat-task": true,
    "is-multi-model-task": true,
    "is-split-by-cell": false,
    "pipeline": [
      {
        "invoke": "dataSetProcessing",
        "descZh": "数据集加工",
        "descEn": "DataSetProcessing",
        "ratBatch": true,
        "modelBatch": false
      },
      {
        "invoke": "upload",
        "descZh": "上传",
        "descEn": "Upload",
        "ratBatch": true,
        "modelBatch": false
      },
      {
        "invoke": "modelEvaluation",
        "descZh": "评估",
        "descEn": "Train",
        "ratBatch": false,
        "modelBatch": true
      },
      {
        "invoke": "download",
        "descZh": "下载",
        "descEn": "download",
        "ratBatch": false,
        "modelBatch": false
      },
      {
        "invoke": "evaluateResult",
        "descZh": "评估结果处理",
        "descEn": "download",
        "ratBatch": false,
        "modelBatch": false
      }
    ]
  }
}
```

---

### 4. SecondScheduler（二级调度器）
定义: Pipeline中每个步骤的执行单元，是**调度器实际处理的最小单位**

```
SecondScheduler {
    taskId              // 实际是subTaskId（智算侧返回）
    batchNo             // 批次号（按polygon分裂时有多批次）
    step                // 步骤序号（1, 2, 3...）
    invoke              // 执行类型：download/intelligent/upload/paramInit
    rat                 // 无线接入类型：NR/LTE
    model               // 模型配置JSON
    status              // 当前状态：QUEUING/RUNNING/SUCCESS/FAILURE
    ...
}
```
创建过程: PipelineStrategy.buildPipeline() (第258行)

```
Pipeline JSON配置
    │
    ▼
createSecondSchedulers() 解析pipeline数组
    │
    ▼
processSecondScheduler() 处理RAT分裂
    │
    ▼
processModelBatch() 处理模型分裂
    │
    ▼
createMultiBatchSchedulersAndTaskData() 处理Polygon分裂
    │
    ▼
最终 SecondScheduler 列表
```
---
#### 四者关系图

```
用户提交请求
    │
    ▼
┌─────────────────────────────────────┐
│           TaskInfo (主任务)          │
│  polygon: [P1, P2, P3]              │
│  rat: "NR,LTE"                      │
│  models: {uem: "v1", bslm: "v2"}    │
│  taskType: "TRAINING"               │
└─────────────────────────────────────┘
    │
    │ 调用智算侧API创建任务
    ▼
┌─────────────────────────────────────┐
│        SubTaskInfo (子任务)          │
│  taskId: 主任务ID                    │
│  subTaskId: 智算侧返回的ID           │
│  subModel: {uem: "v1", bslm: "v2"}  │
│  subRat: "NR,LTE"                   │
└─────────────────────────────────────┘
    │
    │ 读取Pipeline配置
    ▼
┌─────────────────────────────────────┐
│   Pipeline 配置                      │
│   isSplitByCell: true               │
│   pipeline: [                       │
│     {invoke: "paramInit", step: 1}, │
│     {invoke: "download", step: 2},  │
│     {invoke: "intelligent",         │
│      step: 3, isModelBatch: true},  │
│     {invoke: "upload", step: 4}     │
│   ]                                 │
└─────────────────────────────────────┘
    │
    │ 构建二级调度器（分裂规则应用）
    ▼
┌─────────────────────────────────────────────────────────────┐
│                   SecondScheduler 列表                       │
│                                                              │
│  batchNo=1:                                                  │
│    SecondScheduler(subTaskId, batch=1, step=1, invoke=paramInit)│
│    SecondScheduler(subTaskId, batch=1, step=2, invoke=download)│
│    SecondScheduler(subTaskId, batch=1, step=3, invoke=intelligent, model=uem)│
│    SecondScheduler(subTaskId, batch=1, step=4, invoke=intelligent, model=bslm)│
│    SecondScheduler(subTaskId, batch=1, step=5, invoke=upload) │
│                                                              │
│  batchNo=2:  (如果polygon需要分裂)                            │
│    SecondScheduler(subTaskId, batch=2, step=1, invoke=paramInit)│
│    SecondScheduler(subTaskId, batch=2, step=2, invoke=download)│
│    ...                                                       │
└─────────────────────────────────────────────────────────────┘
```
---
#### 分裂详解
1. Polygon分裂 (is-split-by-cell: true)

```java
// PipelineStrategy.createMultiBatchSchedulersAndTaskData() 第276行
for (int index = 0; index < polygons.size(); index++) {
    int batchNo = index + 1;
    for (SecondScheduler scheduler : pipelines) {
        SecondScheduler clone = scheduler.clone();
        clone.setBatchNo(batchNo);  // 设置批次号
        list.add(clone);
    }
}
```
场景: 如果一个任务覆盖的基站数量太多（超过cellLimit），会按Polygon分成多个batch并行执行
2. RAT分裂 (isRatBatch: true)

```java
// PipelineStrategy.processSecondScheduler() 第356行
if (secondScheduler.isRatBatch()) {
    for (String subRat : rat.split(",")) {
        SecondScheduler clone = secondScheduler.clone();
        clone.setRat(subRat);  // NR 或 LTE 分开
        processModelBatch(clone, atomicInteger, pipelines);
    }
}
```
3. 模型分裂 (isModelBatch: true)

```java
// PipelineStrategy.processModelBatch() 第370行
if (secondScheduler.isModelBatch()) {
    for (String key : collect) {
        JSONObject subModel = new JSONObject().fluentPut(key, model.getString(key));
        SecondScheduler clone = secondScheduler.clone();
        clone.setModel(subModel);  // uem 和 bslm 分开
        clone.setStep(atomicInteger.incrementAndGet());
        pipelines.add(clone);
    }
}
```
---
总结
|概念	|作用|	数量关系|
|--|--|--|
|TaskInfo	|用户提交的完整任务，保存到task_info表|	1个用户请求 = 1个TaskInfo|
|SubTaskInfo	|智算侧返回的实际执行单元，关联TaskInfo，保存到sub_task_info表|	1个TaskInfo = 1个或N个SubTaskInfo（按polygon分裂时）|
|Pipeline	|JSON配置文件，定义执行步骤流程|	被TaskInfo.taskType引用|
|SecondScheduler|	Pipeline步骤的执行单元，保存到second_scheduler表|	1个SubTaskInfo = N×M个SecondScheduler（N=batch数，M=步骤数×模型分裂数）|

**一句话总结**: 
TaskInfo是任务容器，SubTaskInfo是智算执行单元，Pipeline是执行步骤模板，SecondScheduler是步骤实例。Pipeline配置决定了SubTaskInfo要创建哪些SecondScheduler，以及是否按RAT/模型/Polygon进行分裂。


# 任务执行详细流程

## 调度器入口
TaskStatusExecutor 实现 IScheduler 接口，被 Spring 的 @Scheduled(cron = "0 */1 * * * *") 驱动，每分钟执行一次。

```java
// TaskStatusExecutor 第53行
@Override
public void schedule() {
    try {
        redisLock.lockAndWait(TASK_PERIOD_EXECUTOR_KEY, ILock.LOCK_EXPIRE_TIME);
        doSchedule();  // 核心逻辑
    } finally {
        redisLock.unlock(TASK_PERIOD_EXECUTOR_KEY);
    }
}
```
---
## 执行流程详解
### 步骤1：查询未完成的子任务

```java
// TaskStatusExecutor.doSchedule() 第102行
private void doSchedule() {
    // 查询所有未完成（状态非SUCCESS/FAILURE/STOP）的子任务
    List<SubTaskInfo> subTaskInfos = taskDao.queryNoCompleteSubTasks();
    
    if (CollectionUtils.isEmpty(subTaskInfos)) {
        return;  // 没有待处理任务，直接返回
    }
    
    // 线程池并行处理每个子任务
    ExecutorService executor = ThreadPoolFactory.INSTANCE.get();
    CountDownLatch latch = new CountDownLatch(subTaskInfos.size());
    
    for (SubTaskInfo subTaskInfo : subTaskInfos) {
        CompletableFuture.runAsync(() -> execute(subTaskInfo, latch), executor);
    }
}
```
---

### 步骤2：执行单个子任务

```java
// TaskStatusExecutor.execute() 第149行
private void execute(final SubTaskInfo subTaskInfo, final CountDownLatch latch) {
    try {
        // 查询该子任务下的所有SecondScheduler
        List<SecondScheduler> secondSchedulers = subTaskInfo.getSchedulerList();
        
        // 按batchNo分组
        Map<Integer, List<SecondScheduler>> map = groupSchedulersByBatchNo(secondSchedulers);
        
        // 构建策略链
        AtomicInteger atomicInteger = new AtomicInteger(0);
        SchedulerChain chain = buildSchedulerChain(atomicInteger);
        
        // 按批次顺序处理
        for (Map.Entry<Integer, List<SecondScheduler>> entry : map.entrySet()) {
            processBatch(subTaskInfo, entry.getValue(), chain, atomicInteger);
            
            // 如果当前批次失败，跳出不再处理后续批次
            if (shouldStopProcessing(subTaskInfo.getSubTaskId(), entry.getKey())) {
                break;
            }
        }
        
        // 更新子任务最终状态
        updateSubTaskStatus(subTaskInfo, atomicInteger, secondSchedulers);
    } finally {
        latch.countDown();
    }
}
```
---
### 步骤3：策略链处理单个批次

```java
// TaskStatusExecutor.processBatch() 第206行
private void processBatch(final SubTaskInfo subTaskInfo, 
    final List<SecondScheduler> batchPipelines, 
    final SchedulerChain chain,
    final AtomicInteger atomicInteger) {
    
    // 按step排序，确保步骤顺序执行
    batchPipelines.sort(Comparator.comparingInt(SecondScheduler::getStep));
    
    for (SecondScheduler secondScheduler : batchPipelines) {
        // 策略链处理（状态机驱动）
        chain.handle(secondScheduler);
        
        // 更新数据库状态
        updateSecondScheduler(secondScheduler, subTaskInfo.getTaskId());
        
        // 失败或UNKNOWN状态，中断批次
        if (shouldBreakAfterUpdate(secondScheduler)) {
            break;
        }
    }
}
```
---
### 步骤4：策略链状态机

```java
// SchedulerChain.handle() 第240行
public void handle(SecondScheduler secondScheduler) {
    for (SchedulerStrategy strategy : strategies) {
        if (!strategy.isEffective(secondScheduler)) {
            continue;  // 当前策略不适用，跳过
        }
        strategy.handle(secondScheduler);  // 执行策略
        if (strategy.isBreak(secondScheduler)) {
            break;  // 执行完就中断，不再尝试后续策略
        }
    }
}
```

策略链顺序及作用:
|顺序|	策略|	触发条件|	作用|
|--|--|--|--|
|1|	TimeoutStrategy|	任意状态	|检查是否超时，超时则标记失败|
|2|	QueuingStrategy|	status == QUEUING	|调用执行器的 execute() 方法开始执行|
|3|	RunningStrategy|	status == RUNNING	|调用执行器的 status() 方法查询状态|
|4|	PendingStrategy|	status == PENDING	|等待前置条件|
|5|	FailedStrategy|	status == FAILURE	|记录失败信息|
|6|	SuccessStrategy|	status == SUCCESS	|成功计数|

---
### 步骤5：执行器分发
QueuingStrategy 根据 invoke 类型分发到具体执行器：

```java
// QueuingStrategy.executeTask() 第101行
private TaskStatus executeTask(final SecondScheduler secondScheduler) {
    return executorFactory.get(secondScheduler.getInvoke()).execute(secondScheduler);
}
```

MExecutorFactory 的分发映射：

|invoke值|	执行器|	execute() 实际动作|
|--|--|--|
|paramInit|	TaskParamInitializingExecutor|	初始化任务参数|
|download|	DownloadExecutor|	从SFTP/HDFS下载数据|
|intelligent|	IntelligentInvokeExecutor|	调用智算侧API启动训练/推理|
|dataFunction|	DataFunctionExecutor|	执行数据处理（裁剪、转换等）|
|upload	|UploadExecutor	|上传结果到存储|
|cbbAppDB|	CBBAppDBExecutor|	CBB后处理结果入库|
|T0	|类似处理	|T0后处理|
---

### 步骤6：IntelligentInvokeExecutor 执行智算任务
这是最核心的执行器，以 invoke="intelligent" 为例：

```java
// IntelligentInvokeExecutor.execute() 第90行
@Override
public TaskStatus execute(final SecondScheduler scheduler) {
    String taskId = scheduler.getTaskId();
    int batchNo = scheduler.getBatchNo();
    
    // 1. 检查是否需要跳过（如前置步骤未完成）
    TaskStatus status = handleInferBeforeInvoke(taskInfo, taskId, batchNo);
    if (status != null) {
        return status;  // 需要等待
    }
    
    // 2. 构建调用参数
    Optional<JSONObject> optional = buildParameter(scheduler, taskInfo, isBreakpointRetry);
    
    // 3. 调用智算侧API
    boolean execute = invokeBusiness.execute(parameter, taskInfo.getTaskType(), taskId);
    
    // 4. 处理返回结果
    return execute ? TaskStatus.RUNNING : TaskStatus.FAILURE;
}
调用智算侧API IntelligentInvokeBusiness.execute():
// IntelligentInvokeBusiness.execute() 第65行
public boolean execute(JSONObject input, String taskType, String subTaskId) {
    // 调用 startTask API（如 model.platform.intelligent.start-task）
    JSONObject execute = ApiFabricUtil.sendPostJsonRequest(startTask, input, taskType);
    
    int resultCode = execute.getIntValue("resultCode");
    return resultCode == 0;
}
```
---
### 步骤7：状态查询与推进
当 execute() 返回 RUNNING 状态后，下一轮定时调度会：

```java
// RunningStrategy.handle() 第40行
@Override
public void handle(final SecondScheduler scheduler) {
    // 调用执行器的status()方法查询智算侧状态
    String status = executorFactory.get(scheduler.getInvoke()).status(scheduler);
    
    if (TaskStatus.isQueuing(status)) {
        status = TaskStatus.RUNNING.getStatus();  // 仍在队列中，算作运行中
    }
    
    // 更新状态
    scheduler.setStatus(status);
}
```
IntelligentInvokeExecutor.status() 查询智算侧：

```java
// IntelligentInvokeExecutor.status() 第152行
@Override
public String status(final SecondScheduler scheduler) {
    // 调用 searchTask API 查询智算侧状态
    ImmutablePair<String, List<ModelTaskInfo>> pair = 
        invokeBusiness.queryModelTaskInfo(task, taskType, step);
    
    return pair.left;  // 返回智算侧状态：SUCCESS/FAILURE/RUNNING
}
```
---
完整状态流转图

```
定时调度触发（每分钟）
    │
    ▼
TaskStatusExecutor.schedule()
    │
    ▼
查询未完成的 SubTaskInfo 列表
    │
    ▼
对每个 SubTaskInfo 执行:
    │
    ├─→ 按 batchNo 分组
    │
    ├─→ batchNo=1 的 SecondScheduler 列表
    │   │
    │   ├─→ step=1 (QUEUING) → QueuingStrategy.handle()
    │   │   └─→ executorFactory.get("paramInit").execute()
    │   │       └─→ 状态变为 RUNNING
    │   │
    │   ├─→ step=2 (QUEUING) → QueuingStrategy.handle()
    │   │   └─→ executorFactory.get("download").execute()
    │   │       └─→ 状态变为 RUNNING
    │   │
    │   ├─→ step=3 (QUEUING) → QueuingStrategy.handle()
    │   │   └─→ executorFactory.get("intelligent").execute()
    │   │       └─→ 调用智算侧API，状态变为 RUNNING
    │   │
    │   ├─→ step=4 (RUNNING) → RunningStrategy.handle()
    │   │   └─→ executorFactory.get("intelligent").status()
    │   │       └─→ 智算侧仍在运行，保持 RUNNING
    │   │
    │   ├─→ step=4 (RUNNING) → RunningStrategy.handle()
    │   │   └─→ executorFactory.get("intelligent").status()
    │   │       └─→ 智算侧完成，状态变为 SUCCESS
    │   │
    │   ├─→ step=5 (QUEUING) → QueuingStrategy.handle()
    │   │   └─→ executorFactory.get("upload").execute()
    │   │       └─→ 上传结果，状态变为 RUNNING → SUCCESS
    │   │
    │   └─→ batchNo=1 全部 SUCCESS
    │
    ├─→ batchNo=2 的 SecondScheduler 列表（开始执行）
    │   └─→ 同上流程...
    │
    └─→ 所有 batch 完成 → SubTaskInfo 状态变为 SUCCESS
```
---
关键点总结
|问题	|答案|
|--|--|
|定时调度多久触发|	每分钟 (@Scheduled(cron = "0 */1 * * * *"))|
|每次调度处理什么|	所有状态为 QUEUING/RUNNING 的 SecondScheduler|
|如何决定执行顺序|	按 batchNo → step 顺序执行|
|如何触发实际执行|	QueuingStrategy 调用执行器的 execute() 方法|
|如何知道执行结果|	RunningStrategy 调用执行器的 status() 方法查询|
|失败怎么办|	记录错误码，停止后续步骤和批次执行|

# Apache DolphinScheduler 深度技术文档

## 一、架构深度解析

### 1.1 去中心化 Master-Worker 架构

DolphinScheduler 采用完全去中心化的 Master-Worker 架构，所有服务通过 ZooKeeper 注册与发现，消除了传统调度系统（如 Airflow 的 CeleryExecutor）的单点瓶颈。

**架构示意图**：

```text
 ┌──────────────────────────────────────────────┐
 │                   UI (前端)                  │
 └──────────────────────┬───────────────────────┘
                        │ HTTP
 ┌──────────────────────▼───────────────────────┐
 │              API Server (多节点)             │
 └──────────────────────┬───────────────────────┘
                        │ 写入指令
 ┌──────────────────────▼──────────────┐
 │           ZooKeeper 集群            │
 │   (Master/Worker注册、分布式锁)     │
 └──────┬────────────────────▲─────────┘
        │ 监听任务分配       │ 上报状态
 ┌──────▼─────────┐   ┌─────┴───────────────┐
 │ Master (多节点) │   │   Worker (多节点)   │
 │  DAG 解析、切分 │   │   任务实际执行       │
 │  负载均衡	   │   │   (用户进程)         │
 └────────────────┘   └─────────────────────┘
```

**Master 核心职责**：

-   定期扫描数据库中的 `t_ds_command` 表，获取工作流启动指令（采用槽位机制减少 ZooKeeper 锁竞争：`id % master_count == current_index`）。
    
-   解析 DAG 结构，切分任务实例，写入任务队列。
    
-   通过 Netty 长连接将任务 **主动推送给 Worker**（Push 模型），而非 Worker 轮询数据库（Pull 模型）。
    
-   监听 Worker 的上报状态，更新任务实例生命周期。
    

**Worker 核心职责**：

-   接收 Master 通过 Netty 下发的任务执行命令。
    
-   创建进程执行任务（Shell、SQL、Spark 等），并通过 Logger Server 实时推送日志。
    
-   任务结束后向 Master 汇报最终状态，自身重启不影响任务信息（状态由 Master 持久化）。
    

**高可用保障**：

-   Master 和 Worker 均支持多节点，宕机时 ZooKeeper 通过临时节点移除自动触发故障转移。
    
-   任务状态持久化于数据库，Master 故障时其他 Master 可接管正在运行的工作流实例。
    
-   使用数据库乐观锁（版本号）防止重复提交。
    

---

## 二、核心概念与数据模型

### 2.1 概念层级关系

```text
租户 (Tenant) 	—— 对应 Linux 用户
  └─ 项目 (Project) 		—— 业务线隔离单元
       └─ 工作流定义 (Process Definition) 	—— DAG 图与版本
            └─ 任务定义 (Task Definition) 	—— 节点元数据
       └─ 工作流实例 (Process Instance) 	—— 一次调度触发
            └─ 任务实例 (Task Instance) 		—— 单个节点的执行记录
```

**数据库关键表组**：

| 表组  | 表名  | 关键字段 | 说明  |
| --- | --- | --- | --- |
| **工作流定义** | `t_ds_process_definition` | `id`, `name`, `version`, `release_state`, `project_code` | 存储 DAG 元数据，发布后 `release_state` 标记上线 |
| **任务定义** | `t_ds_task_definition` | `code`, `name`, `task_type`, `task_params` (JSON) | 任务参数（类型、SQL 语句、脚本等）以 JSON 存储 |
| **工作流实例** | `t_ds_process_instance` | `id`, `name`, `state`, `start_time`, `end_time` | 状态包括：0-提交成功, 1-运行中, 2-暂停, 3-停止, 4-重试, 5-失败, 6-成功, 7-需要容错, 8-杀死, 9-等待线程, 10-等待依赖 |
| **任务实例** | `t_ds_task_instance` | `id`, `task_code`, `state`, `retry_times`, `max_retry_times` | 任务状态标识：0-提交, 1-运行中, 2-准备暂停, 3-暂停, 4-准备停止, 5-停止, 6-失败, 7-成功, 8-需要容错, 9-杀死, 10-等待线程, 11-等待依赖 |
| **指令表** | `t_ds_command` | `id`, `command_type`, `process_definition_code`, `command_param` | Master 拉取指令：START\_PROCESS, RECOVER\_PROCESS 等 |
| **调度** | `t_ds_schedules` | `id`, `process_definition_code`, `crontab`, `start_time`, `end_time` | Cron 表达式和生效时间 |

### 2.2 任务状态机（关键）

任务实例从创建到结束的完整状态流转，直接影响调度逻辑和容错机制：

```text
                        ┌─────────┐
                        │  SUBMIT │  (初始)
                        └────┬────┘
                             │ Master 分发
                        ┌────▼────┐
                        │ DISPATCH│ -> Worker 领取
                        └────┬────┘
                             │ Worker 开始执行
                        ┌────▼────┐
                        │ RUNNING │
                        └─┬──┬──┬─┘
              成功───────┘  │  │  └───────失败(未超重试)
              │             │  │             │
         ┌────▼───┐   ┌───▼──▼───┐   ┌──────▼──────┐
         │SUCCESS│   │ PAUSE/STOP│   │ FAILURE/RETRY│
         └────────┘   └──────────┘   └──────┬──────┘
                                             │ 可重试且未达上限
                                        ┌────▼────┐
                                        │  DELAY  │ (等待重试)
                                        └─────────┘
```

**容错机制**：

-   失败的任务会根据 `max_retry_times` 和 `retry_interval` 自动重试。
    
-   若所有重试失败，工作流实例状态变为失败，但支持 **断点续跑**：重新运行工作流实例时，可选择从失败节点开始执行，避免重跑已成功的部分。
    

---

## 三、参数体系详解

### 3.1 参数类型与优先级

| 类型  | 作用域 | 示例  | 优先级 |
| --- | --- | --- | --- |
| **系统内置参数** | 全局  | `${system.biz.date}` | 最低（可被覆盖） |
| **全局参数** | 工作流定义 | `day=${system.biz.date}` | 较低  |
| **局部参数 (IN)** | 当前任务节点 | `input_path=/data/raw` | 中   |
| **局部参数 (OUT)** | 向下游传递 | 运行时由任务生成 | 高   |
| **上游传递参数** | 下游节点接收 | 由 OUT 参数自动注入 | 高于全局参数 |

### 3.2 衍生内置参数 `$[...]` 深度用法

衍生参数是解决**非 T-1 调度**和**跨天依赖**的核心工具，支持任意日期偏移和格式定制。

**语法规则**：

```text
$[format_string 偏移量]
```

-   格式符：`yyyy`, `MM`, `dd`, `HH`, `mm`, `ss` 等。
    
-   偏移量：`+N` 或 `-N` 表示天数偏移，支持周、小时、分钟。
    
-   月份操作：`$[add_months(format, offset)]`，自动处理大小月、闰年。
    

**高级示例**：

```bash
# 上个月的今天（处理月末差异）
$[add_months(yyyyMMdd, -1)]

# 上周同一天（7天前）
$[yyyyMMdd-7]

# 明天凌晨2点的时间戳（自定义组合）
$[yyyy-MM-dd+1] 02:00:00

# 上一小时的整点（用于小时级调度）
$[yyyyMMddHH-1/24]
```

**应用场景**：

-   **数据回刷**：补数时自动根据 `schedule_time` 计算对应分区。
    
-   **跨周期依赖**：今天任务依赖昨天23点的数据，使用 `$[HH-1/24]` 确定源分区。
    

### 3.3 工作流内参数传递

**限制规则**：

| 任务类型 | 能否向下游传递 (OUT) | 能否接收上游 (IN) |
| --- | --- | --- |
| **SQL** | ✅   | ✅   |
| **Shell** | ✅   | ✅   |
| **Python** | ❌   | ✅   |
| **Spark/Flink** | ❌   | ❌   |

**Shell 传递示例**：

```bash
# 计算目标分区并传递给下游
target_date=$(date -d "yesterday" +%Y%m%d)
echo "分区: ${target_date}"
# 关键语法：${setValue(key=value)}
echo '${setValue(partition=${target_date})}'
```

此时下游 SQL 节点可直接使用 `${partition}` 作为参数。

---

## 四、与 Hive 的集成实战

### 4.1 配置 Hive 数据源

在 DolphinScheduler 中使用 Hive 任务，需要先配置 **Hive 数据源**。

**步骤**：

1.  进入 **数据源中心** → **创建数据源**。
    
2.  选择 **HIVE** 类型。
    
3.  填写连接信息：
    
    ```text
    数据源名称：hive_production
    IP/主机名：hiveserver2-host
    端口：10000
    用户名：hive
    密码：(若启用认证)
    数据库名：default
    ```
    
4.  其他参数（可选）： `hive.server2.proxy.user=hadoop` （代理用户，解决 Kerberos 认证）
    
5.  点击 **测试连接** 通过后保存。
    

**Kerberos 集成**： 若 Hive 启用了 Kerberos，需在 `api-server/conf/common.properties` 中添加：

```properties
# 启用 Kerberos
security.authentication.type=kerberos
# 提供 keytab 和 principal
hive.kerberos.principal=hive/_HOST@REALM
hive.kerberos.keytab.path=/opt/dolphinscheduler/conf/hive.keytab
```

Worker 节点需同步 keytab 文件，并确保 `hive` 用户有权限。

### 4.2 Hive 任务节点配置

在工作流画布中添加 **HIVE** 节点：

**基本配置**：

-   **数据源**：选择上文创建的 `hive_production`。
    
-   **SQL 类型**：`0-查询`（返回结果不持久化）或 `1-非查询`（DDL/DML）。
    
-   **SQL 语句**：
    
    ```sql
    INSERT OVERWRITE TABLE dw.dwd_user
    PARTITION (dt='$[yyyyMMdd-1]')
    SELECT id, name, action
    FROM ods.ods_user_log
    WHERE dt = '$[yyyyMMdd-1]';
    ```
    
-   **前置 SQL**（可选）：执行前执行的 SQL，如 `set hive.exec.parallel=true;`。
    

**资源级联**： 如果 SQL 较大，可上传 `.sql` 文件至 **资源中心**，然后在节点中引用 `<filename>`。DolphinScheduler 会自动读取文件内容替换占位符。

### 4.3 Hive 集成常见问题

| 问题  | 原因  | 解决方案 |
| --- | --- | --- |
| `Could not open client transport with JDBC` | HiveServer2 未启动或连接信息错误 | 检查 HiveServer2 端口，测试 beeline 连接：`beeline -u jdbc:hive2://host:10000` |
| `User: hive is not allowed to impersonate hive` | Hadoop 代理用户配置问题 | 在 `core-site.xml` 中设置 `hadoop.proxyuser.hive.hosts` 和 `hadoop.proxyuser.hive.groups` 为 `*` |
| `Table not found` | SQL 中未指定库名或数据源默认库错误 | 在 SQL 中使用全限定表名 `database.table`，或在数据源中明确默认数据库 |
| Kerberos 认证失败 | keytab 路径错误、时间不同步 | 确保 Worker 时间与 KDC 同步（NTP），验证 `kinit -kt keytab principal` 可执行 |
| 任务长时间卡在 `RUNNING` | Hive 任务卡在 YARN，未产生输出 | 检查 YARN Application 是否 `ACCEPTED`（资源不足），或 MapReduce 任务阻塞 |
| 中文乱码 | 元数据编码或 JDBC 连接字符集问题 | HiveServer2 JDBC URL 加 `?useUnicode=true&characterEncoding=UTF-8` |
| `Failed to run task: java.lang.OutOfMemoryError: PermGen space` | Worker JVM 参数不足 | 增加 Worker 的 `-XX:MaxMetaspaceSize=256m`（Java 8+）或调整 `worker.exec.threads` 减少并发 |

### 4.4 补数示例：Hive ETL 历史数据处理

**场景**：需要重新计算 2025-01-01 至 2025-01-15 的历史数据，工作流定义使用 `$[yyyyMMdd-1]` 动态分区。

**操作流程**：

1.  进入 **工作流实例** 页面，点击 **补数**。
    
2.  设置起始日期 `2025-01-01`，结束日期 `2025-01-15`。
    
3.  选择 **串行执行** 或 **并行执行**（注意 Hive 资源，避免同时过多并发）。
    
4.  点击 **执行**，系统自动为每天生成一个工作流实例，每个实例中的 `${schedule_time}` 会被替换为对应的调度时间，从而 `$[yyyyMMdd-1]` 计算出的分区即为正确的历史日期。
    
5.  可在 **工作流实例** 列表查看每一天的执行状态和日志，失败实例可单独重跑或批量重跑。
    

**注意事项**：

-   补数前确保 Hive 中历史分区存在或脚本具备自动创建分区的逻辑。
    
-   若 Hive 表很大，应控制并行度，避免大量 MapReduce 同时提交。
    

---

## 五、核心源码深度解读

本节选取 DolphinScheduler 3.x 分支中的关键类，揭示调度核心流程。

### 5.1 调度入口：`MasterSchedulerService`

所属模块：`dolphinscheduler-master`**职责**：Master 的核心循环，负责扫描 Command 并提交工作流实例。

```java
// 伪代码简化逻辑
@Scheduled(fixedDelay = 1, timeUnit = TimeUnit.SECONDS)
public void scheduled() {
    // 1. 使用槽位机制从 t_ds_command 表获取属于自己的指令
    List<Command> commands = commandDao.queryCommandBySlot(currentSlot, masterCount);
    for (Command command : commands) {
        // 2. 根据指令类型执行不同逻辑：启动、恢复、重试、补数等
        switch (command.getType()) {
            case START_PROCESS:
                // 创建工作流实例，构建 DAG，切分任务
                processInstance = createProcessInstance(command);
                dag = buildDag(processInstance);
                submitTask(dag.getStartNodes());
                break;
            case RECOVER_PROCESS:
                // 从失败节点恢复
                recoverProcessInstance(command);
                break;
            // ...
        }
    }
}
```

**关键设计**：槽位机制 `id % master_count == current_index` 避免了多个 Master 抢占同一个 Command，同时减少了对 ZooKeeper 分布式锁的依赖。

### 5.2 任务分发模型：`WorkflowExecuteRunnable`

所属模块：`dolphinscheduler-master`**职责**：实际控制单个工作流实例的执行状态机。

```java
public class WorkflowExecuteRunnable implements Callable<Void> {
    private Map<String, TaskInstance> activeTaskProcess = new HashMap<>();
    private ProcessInstance processInstance;

    public void run() {
        while (!processInstance.isFinished()) {
            // 1. 检查可提交的任务（所有前置任务已完成）
            List<TaskInstance> readyTasks = getReadyToSubmitTasks();
            for (TaskInstance task : readyTasks) {
                // 2. 通过 Netty 将任务发送给 Worker
                TaskDispatchCommand dispatch = new TaskDispatchCommand(task);
                NettyClientManager.getInstance().send(dispatch);
            }
            // 3. 处理已完成任务的状态上报
            for (TaskInstance completed : completedQueue) {
                // 更新 DAG 状态，标记后续节点可执行
                updateTaskState(completed);
            }
        }
    }
}
```

**设计要点**：

-   采用 **事件驱动** 而非轮询，Worker 完成任务后主动通过 Netty 汇报，Master 立即更新状态并尝试提交下游任务。
    
-   使用 `completedQueue` 缓冲上报消息，解耦接收与处理。
    

### 5.3 Worker 执行器：`TaskExecuteProcessor`

所属模块：`dolphinscheduler-worker`**职责**：接收任务命令，创建具体执行器，启动进程并监控。

```java
public class TaskExecuteProcessor implements NettyRequestProcessor {
    @Override
    public void process(Channel channel, Command command) {
        TaskExecutionContext taskRequest = JSONUtils.parseObject(command.getBody(), TaskExecutionContext.class);
        // 根据任务类型获取对应的执行器，如 ShellTask, SqlTask
        AbstractTask task = TaskFactory.createTask(taskRequest);
        // 提交到线程池执行
        workerManagerService.submit(new TaskExecutionThread(task));
    }
}
```

`AbstractTask` 的核心方法：

```java
public abstract class AbstractTask {
    public void handle() {
        // 1. 构建进程执行命令
        String command = buildCommand();
        // 2. 创建进程
        Process process = Runtime.getRuntime().exec(command);
        // 3. 读取标准输出和错误输出
        // 4. 等待进程结束并获取退出码
        int exitCode = process.waitFor();
        // 5. 根据退出码判定成功或失败
        if (exitCode == 0) {
            setResult(SUCCESS);
        } else {
            setResult(FAILURE);
        }
    }
}
```

### 5.4 依赖处理：`DependentTaskProcessor`

**职责**：实现跨项目、跨工作流的依赖等待。

核心逻辑：

-   任务启动时，将依赖定义（待依赖的项目、工作流、时间范围）注册到一个全局的 `DependentTaskManager`。
    
-   `DependentTaskManager` 定期轮询数据库或通过事件监听，检查指定日期的工作流是否已成功完成。
    
-   一旦满足条件，唤醒对应的 `Dependent` 任务，使其状态变为成功，从而释放下游节点。
    

### 5.5 告警机制：`AlertSender`

**职责**：发送告警通知。采用插件化设计，支持邮件、钉钉、企业微信、HTTP 等。

```java
public class AlertSender {
    public void sendAlert(AlertInfo alertInfo, List<AlertPluginInstance> pluginInstances) {
        for (AlertPluginInstance instance : pluginInstances) {
            AlertChannel channel = AlertPluginManager.getChannel(instance.getPluginDefineId());
            channel.process(alertInfo); // 发送告警
        }
    }
}
```

告警规则在 `t_ds_alert` 表中定义，通过 `t_ds_alert_plugin_instance` 关联具体通道。

---

## 六、2025 年大厂面试高频题

### Q1：DolphinScheduler 和 Airflow / Azkaban 的架构区别？为什么选择 DS？

**考点**：去中心化架构、Push 模型、多租户。 **回答要点**：

-   DS 采用去中心化 Master-Worker，无单点故障；Airflow 的 Scheduler 存在单点。
    
-   DS 使用 Netty Push 下发任务，延迟毫秒级；Airflow 依赖 DB 轮询，延迟较高。
    
-   DS 支持原生多租户、UI 拖拽，更适合非 Python 技术栈的团队。
    

### Q2：Master 如何保证高可用？多个 Master 同时运行如何避免冲突？

**考点**：ZooKeeper 选主、槽位机制。 **回答要点**：

-   Master 通过 ZooKeeper 注册临时节点，宕机后自动移除，其他 Master 接管。
    
-   指令表 `t_ds_command` 的分配采用 **槽位机制**：`id % master_count == current_index`，避免了分布式锁竞争，每个 Master 只处理属于自己的指令。
    

### Q3：工作流实例执行过程中 Master 宕机会怎样？任务状态是否丢失？

**考点**：状态持久化、故障转移。 **回答要点**：

-   所有任务实例状态实时写入数据库。
    
-   存活的其他 Master 会检测到宕机 Master 失去 ZK 心跳，扫描其未完成的工作流实例，并接管执行。已完成的任务不会重跑，失败或未完成的任务继续重试或重新分发。
    

### Q4：你怎么设计一个支持动态分区的 Hive 调度任务？如何处理补数？

**考点**：参数系统、补数机制。 **回答要点**：

-   使用内置参数或衍生参数 `$[yyyyMMdd-1]` 动态生成分区。
    
-   补数时选择日期范围，系统自动为每个日期生成实例，`schedule_time` 被替换为对应时间，分区参数随之变化。需确保历史分区存在或脚本能创建。
    

### Q5：Shell 任务如何将参数传递给下游的 SQL 任务？

**考点**：参数传递 OUT/IN。 **回答要点**：

-   Shell 任务设置 `OUT` 参数，输出 `echo '${setValue(key=value)}'`。
    
-   SQL 任务设置同名 `IN` 参数，引用 `${key}`。
    
-   只有 Shell 和 SQL 支持向下游传递参数。
    

### Q6：如果 Hive 任务执行超长或卡住，你如何在 DS 中排查？

**考点**：日志定位、问题隔离。 **回答要点**：

-   在 DS UI 中查看任务实例的日志，看最后执行到的步骤。
    
-   登录 YARN 查看 Application 状态，若为 ACCEPTED 说明资源不足，调整并发或集群资源。
    
-   检查 HiveServer2 日志，看是否有锁竞争或 GC 问题。
    

### Q7：DolphinScheduler 如何集成 Kerberos 认证的 Hive？

**考点**：Kerberos 配置、keytab 使用。 **回答要点**：

-   在 `api/conf/common.properties` 中启用安全认证，指定 principal 和 keytab。
    
-   Worker 节点需同步 keytab，并 `kinit` 后才能正常连接 HiveServer2。
    
-   可在数据源配置中添加代理用户，避免权限问题。
    

### Q8：你觉得 DS 的 Worker 分组有什么用？什么场景下要用？

**考点**：资源隔离、业务分组。 **回答要点**：

-   Worker 分组可以将不同的物理节点绑定到特定作业类型，比如 Spark 大任务用 `spark` 组，Shell 轻量任务用 `default` 组。
    
-   避免资源争抢，确保核心任务稳定性。
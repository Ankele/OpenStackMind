### 什么是alarm告警
当收集的meter数据或event事件break定义的规则时，将触发告警。
告警由4个组件构成：
**1、aodh-api**
    跑在多个控制节点上，提供alarm信息的访问。
**2、aodh-evaluator**
    跑在多个控制节点上，决定是否触发alarm，由于相关统计趋势在规定的滑动时间窗口内超过阈值。
**3、aodh-listener**
    跑在一个控制节点上，决定何时触发alarm。根据ceilometer 服务收集的event事件信息判断。
**4、aodh-notifier**
    跑在多个控制节点上，允许基于samples设置alarm。
以上组件通过message bus通信。

告警服务aodh是提供 启用 基于已定义的规则 检查gnocchi收集到的度量数据或者事件数据 来触发相应动作 这一功能 的服务
[点我安装](https://docs.openstack.org/aodh/rocky/install/install-rdo.html)
Administration Guide
关于怎么部署、操作和升级aodh

* alarm的定义
    * 阈值规则告警
    * 复合规则告警
* alarm的尺寸标注
* alarm评估
    * alarm动作
    * 工作负载分区
* 使用alarms
    * 创建alarm
        * 基于阈值
        * 复合类型
        * 基于事件
    * 检索alarm
    * 更新alarm
    * 删除alarm
    * debug

### alarm的定义
alarm 监控即服务能确保您可通过编排服务自动扩展一个（组）实例，您还可以使用其来了解云资源的健康状况。
alarm遵循三态模型，OK、alarm、insufficient-data
alarm的定义提供了 何时应该发生状态转变以及要对状态转变采取的动作 的规则。每种类型的规则都有其特点。
##### 阈值告警规则
以下是常见的基于阈值的规则。会触发状态转变：
* 与一个镜头阈值比较，或大或小
* A statistic selection to aggregate the data
* 移动的时间窗口指示你想要窗口最近的过去 的距离
有效的阈值告警threshold alarms：
gnocchi_resources_threshold_rule,
gnocchi_aggregation_by_metrics_threshold_rule, gnocchi_aggregation_by_resources_threshold_rule.

##### 复合告警规则
允许多个触发条件来定义一个告警，使用`and` 或和 `or`来组合。

### alarm的尺寸标准
一个关键的相关概念是尺寸标注的概念，它定义了一组匹配的meter，用于报警评估。 回想一下，meter是per-resource-instance，因此在最简单的情况下，可以在应用于特定用户可见的所有资源的特定meter上定义警报。 然而，更有用的是明确选择您感兴趣的特定资源的选项。在一个极端，您可能具有范围很窄的警报，其中此选择将仅具有单个目标（由资源ID标识）。 在另一个极端，您可以拥有广泛的尺寸标注警报，其中此选择标识了聚合统计信息的许多资源。 例如，从特定镜像或具有匹配用户元数据的所有实例引导的所有实例（后者是Orchestration服务识别自动缩放组的方式）。

### alarm评估
alarm通过 `alarm-evaluator` 服务进行周期性评估，默认每分钟一次。
##### alarm actions
对于一个alarm来说，任何状态转换可能有一个或多个动作与之有关。这些操作有效地向消费者发送状态转换已发送的信号，并提供上下文。
状态转变被`alarm-evaluator` 发现，`alarm-notifier` 通知，
两种方式通知：
1、webhooks
这是telemetry事实上的通知类型，仅仅涉及发送到endpoint的HTTP POST请求，请求主题包含编码为JSON片段的状态转换的描述。
2、Log actions
这是webhooks的轻量级替代方案，其中状态转换仅由`alarm-notifier` 通知，主要用于测试。
##### 负载分区
请参阅 [点我](https://docs.openstack.org/aodh/rocky/admin/telemetry-alarms.html#workload-partitioning)

### Using alarms
##### 创建alarm
***阈值based alarm***
基于阈值的alarm ：threshold based alarm
![举子例子](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1548663308249&di=759e10e0c7435efcc95becd21322a155&imgtype=0&src=http%3A%2F%2F04img.mopimg.cn%2Fmobile%2F20180719%2F20180719162646_8c1bbbf82a09bea13a40ceb75792a0db_3.jpeg)
一个instance的CPU利用率上限
```
$ aodh alarm create \
--name cpu_hi \
--type gnocchi_resources_threshold \
--description 'instance running hot' \
--metric cpu_util \
--threshold 70.0 \
--comparison-operator gt \
--aggregation-method mean \
--granularity 600 \
--evaluation-periods 3 \
--alarm-action 'log://' \
--resource-id *instance_id* \
--resource-type instance
```

```
# CPU使用率
aodh alarm create --name zl_cpu_util222 --type gnocchi_resources_threshold --description "instance running hot" --metric cpu_util --threshold 2.0 --comparison-operator gt --aggregation-method mean --granularity 300 --evaluation-periods 3 --alarm-action 'log://' --resource-id 9cfa8956-fae2-4882-ae2f-82c745ec2275 --resource-type instance
```

```
# 物理机网络输出数据包
aodh alarm create --name zl_hardware_network_ip_outgoing_datagrams --type gnocchi_resources_threshold --description "hardware.network.ip.outgoing.datagrams desc" --metric hardware.network.ip.outgoing.datagrams --threshold 130000000.0 --comparison-operator gt --aggregation-method mean --granularity 300 --evaluation-periods 3 --alarm-action 'log://' --resource-id 7ceaa4e8-5d37-59db-99d3-d0bd85cb719b --resource-type host
```

在3个连续的600秒内，如果这个instance的CPU平均使用率高达70%，那么僵触发告警。告警通过log方式表达，当然最好是使用webhook咯。
> Note
> 同一项目中，alarm的name应该唯一。

该示例中，alarm 评估 的滑动时间窗口是30m。This window is not clamped to wall-clock time boundaries.rather it's anchored on the current time for each evaluation cycle,
and continually creeps forward as each evaluation sysle rolls around(by default, this occurs every minute).
> Note
> The alarm granularity 必须和 gnocchi 中metric的 粒度相等。否则，状态将为insufficient data。pipeline.yaml中可配置

其他的alarm 属性，在创建或更新中设置它们：
**state**
    初始alarm 状态 （default是**insufficient data**）
**description**
    alarm的自由文本描述（默认是概要）
**enabled**
    若要为alarm启用评估evaluate和操作action，则为True（default is True）
**repeat-actions**
    若alarm保持在目标状态，是否重复执行告警actions (defaults to False)
**ok-action**
    当alarm state转为OK的时候英执行的action
**insufficient-data-action**
    当alarm state转为insufficient-data时执行的action
**time-constraint**
    用于将alarm的评估时间限制为一天中的某些时间段或一周内的某些天（表示为带有可选时区的cron 表达式）

***复合alarm***
举例，两个basic rules的composite alarm
```
$ aodh alarm create \
   --name meta \
   --type composite \
   --composite-rule '{"or": [{"threshold": 0.8, "metric": "cpu_util", \    "type":   "gnocchi_resources_threshold",              "resource_id": INSTANCE_ID1, \    "resource_type": "instance", "aggregation_method": "last"}, \    {"threshold": 0.8, "metric": "cpu_util", \    "type": "gnocchi_resources_threshold", "resource_id": INSTANCE_ID2, \    "resource_type": "instance", "aggregation_method": "last"}]}' \
   --alarm-action 'http://example.org/notify'
```
当两个basic rule之一满足条件，将触发alarm。notification是webhook call。
> Note
> 此处的resource_id 和 resource_type 不是之前基于阈值的alarm的--resource-id 和--resource-type了

***基于事件的 alarm***
```
$ aodh alarm create \
   --type event \
   --name instance_off \
   --description ‘Instance powered OFF’\
   --event-type "compute.instalce.power_off.*" \
   --enable True \
   --query "traits.instance_id=string::INSTANCE_ID" \
   --alarm-action 'log://' \
   --ok-action 'log://' \
   --insufficient-data-action 'log://'
```

```
# instance关机事件
aodh alarm create --type event --name zl_instance_off --description "Instance powered OFF' --event-type "compute.instance.power_off.*" --enable True --query "traits.instance_id=string::9cfa8956-fae2-4882-ae2f-82c745ec2275 --alarm-action 'log://' --ok-action 'log://' --insufficient-data-action 'log://'
```

**event-type** 和 **traits**的有效值可以在**event_defintions.yaml**中找到。 **--query**包含mix of traits for example to create alarm 当instance power on 但状态为error。
```
aodh alarm create \
--type event \
--name instance_on_but_in_err_state \
--description 'Instance powered On but in error state' \
--event-type "compute.instance.power_on.*" \
--enable True \
--query "traits.instance_id=string::INSTANCE_ID;traits.state=string::error" \
--alarm-action 'log://' \
--ok-action 'log://' \
--insufficient-data-action 'log://'
```
> 开启event alarm 还需要 edit， see https://docs.openstack.org/aodh/latest/contributor/event-alarm.html#configuration

##### alarmretrieval
`aodh alarm list`
此例中，状态为 insufficient data 表明：
- 评估窗口还未收集关于此虚拟机的meter
- or, 指定的虚拟机对owning the alarm的user/project不可见
- or, 简单来说，自创建alarm以来，alarm 评估周期还没有启动（默认每分钟一次）
> alarms 的可见度由用户角色决定
> admin see all，no limit
> non-admin see only their own project

##### alarm update
突然觉得70%使用率太低了，调为75吧
`$ aodh alarm update ALARM_ID --threshold 75`
这个update将在下一个评估周期起效，默认每分钟一周期。
直接设置告警状态，喔~~~
`$ openstack alarm state set --state ok ALARM_ID`

随着时间推移，alarm的状态可能经常变化，特别是选择的阈值接近趋势值的时候。
您可以通过API跟踪其生命周期中的警报历史记录
`$ aodh alarm-history show ALARM_ID`

##### alarm deletion
可以disable
`$ aodh alarm update --enabled False ALARM_ID`
也可以直接delete
`$ aodh alarm delete ALARM_ID`



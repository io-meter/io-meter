---
title: AWS 上 Kylin 调度系统的设计
date: 2017-10-13 14:18:26
tags: [Kylin, AWS, Scala, Scheduler, Cloud, Big Data, Hadoop, EMR]
---

最近加入 [Strikingly](https://strikingly.com) / [上线了](https://sxl.cn) 光荣地成为了一名数据平台工程师，
投身于大数据平台开发的工作当中。这两个月来，通过设计和实现一个 AWS 的 Kylin 数据仓库调度系统，
收获很多，借此机会总结一下。

<!-- more -->

## 背景介绍

Strikingly 是一家为用户提供建站服务的初创企业，目前的数据平台主要处理的是用户所建立网站的访问者信息统计，
可以认为是一套简单的 [Google Analytics](https://analytics.google.com) 服务。这套系统使用 [Keen IO](https://keen.io/)
收集访问者信息，使用 Kylin、Hadoop、Hive 等技术处理海量数据，整套系统都部署在 [AWS](https://aws.amazon.com/) 上，
深度使用了 EC2、ECS、ELB、EMR 等 AWS 服务。除了收集访问者的 Page View 信息，
这套系统还会处理一些用户付费行为相关的 e-commerce 信息等。

### 数据处理流程

下图是目前系统的数据处理流程的一个简化版本:

![Data Processing Pipeline](/img/kylin-scheduler/data-processing-pipeline.png)

以用户的 Page View 数据处理为例，主要有以下几个步骤：

1. 数据首先由内嵌在页面当中的 JS 脚本收集到第三方服务 Keen IO 那里
2. 通过 Keen IO 的数据导出功能，将 Page View 数据以五分钟为单位，使用 JSON 格式打包放置在 AWS S3 上的指定目录
3. 通过为 Hive 配置 [JsonSerDe](https://github.com/rcongiu/Hive-JSON-Serde)，我们可以将第二步当中的指定目录直接虚拟成一个
Hive 表，从而可以在 Hive 上使用 SQL 语句进行查询操作
4. Kylin 控制 EMR 上的各种组件，如 Hive、Hadoop 等进行处理，将数据处理的结果保存在 HBase 表当中，以备之后取用

当数据被处理好保存在 HBase 上之后，我们就可以调用 Kylin 所提供的 API 对这些数据进行远超过以往的速度的访问，
从而支撑每个 Strikingly 或上线了用户对自己网站统计数据的查询要求。

### Kylin 简介

从上面的数据处理流程当中可以看出，Kylin 无疑是整套系统的核心: 一方面它控制了对 Hive 当中原始的数据进行处理并缓存到 HBase
的全过程，另一方面它还要服务用户对处理后的数据大量的查询请求。在这里有必要对 Kylin 及其基本概念进行一个简单的介绍。

[Apache Kylin](http://kylin.apache.org/) 是一个开源的分布式数据分析系统，它和 Hadoop、Spark 等一样，
都是 Apache 基金会的顶级项目。Kylin 可以被认为是一种 [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load)
工具，我把它主要完成的任务概括为以下几点:

1. 根据预先定义好的模型，将保存在 Hive 表或其他来源的数据提取出来
2. 根据预先定义好的模型关系，将数据按照一定的算法进行变换处理，使得数据以一种易于查询的方法存在
3. 当用户请求到来时，通过分析用户请求，将查询分解成可以作用于变换后数据格式的任务，从而快速的得到处理结果

在第 2 点当中 Kylin 使用的技术被称为 Cubiod。 这一技术有点像根据模型的定义，预先穷举用户查询的所有可能结果，
然后将这些结果以合适的方式组织起来。当然，在实际实现当中，为了更加有效地执行这一任务并将其抽象为可以运行在 Hadoop
集群上的 MapReduce 任务，Kylin 所实现的算法要复杂很多。在此我们不对 Kylin 的技术实现进行深入的介绍，
但是为了继续之后的内容，必须要了解 Kylin 当中四个重要的概念: Model、Cube、Job 和 Segment。

![Model, Cube, Segment, Job](/img/kylin-scheduler/kylin-model-cube-segment-job.png)

上图给出了这四个概念的一个关系说明，简单来讲：

* Model 是对数据来源的描述，比如说我们想要进行查询的数据表的主表是什么，想要和主表 Join 起来的辅助表是什么等等。
值得注意的是，在 Model 里，我们要为模型指定一个列(或者称为维度)作为这个模型的 Range Key(或者称为 Partition Key)。
所谓 Range Key，就是说根据这个维度，可以比较好地将数据划分为多个不相交的部分。这个 Key 在之后 Segment 的部分也会用到
* Cube 是对转化和处理数据的方法的描述。在 Kylin 里，我们并不需要手动编写转化的程序，而是通过指定需要进行操作的 Model，
提供一些用户可能进行的查询和统计的指标(比如用户可能在哪些字段上使用 `GROUP BY`，在哪些字段上进行 `COUNT` 或
`COUNT_DISTINCT` 统计等)，Kylin 会自动根据这些定义和数据模型本身的特点，构造出合理的执行流程
* Job 是在 Cube 的控制下，Kylin 执行的一次任务处理。它包括了数据处理过程当中，数据抽取、数据变换、数据储存、
垃圾清理等的全过程。Job 的类型包括 `BUILD`、 `REFRESH`、 `MERGE` 等
* Segment 是一个 Job 执行的结果。对于一个 Cube 来说，他的 Segment 可以根据 Range Key 划分为多个区间分别构建，
当某一区间的 Segment 构建成功之后，Kylin 就可以实现对这一区间当中数据的快速查询。而构建当中或者构建失败的 Segment
无法提供查询服务。对于横跨多个 Segment 的查询，Kylin 会分别执行对各个 Segment 的查询，再将结果合并起来。
我们可以 `REFRESH` 一个 Segment 以更新其中的数据，或者 `MERGE` 多个 Segment 以避免跨 Segment 查询带来的性能损失

当我们根据上述模型构建了 Segment 之后，Kylin 就可以通过 SQL 接口对 Model 指定的数据表进行快速的查询和分析了。
值得注意的是，我们还可以以集群的方式部署 Kylin。一个 Kylin 集群可以有一个 Job 节点和若干个 Query 节点组成，
其中 Job 节点单独用来做 Cube 构建的编排控制操作，Query 节点则用来承载用户的查询请求。

### 对调度系统的需求

尽管 Kylin 的存在使得我们承载用户 Page View 查询的数据处理平台成为可能，在系统的开发和部署当中仍然遇到一些问题，
导致开发一个集中式的调度系统成为必要。在实践当中遇到的需求可以总结为以下几条:

**定制化的任务调度**，在很多情况下，Kylin 自带的调度系统不能很好的满足我们的需求，
这是我们自行实现调度系统最本质的原因。我们遇到的问题主要包括以下几点:

1. 由于数据从收集到导入 Kylin 之间的步骤比较多，每个步骤都需要调度控制协调运行。
比如说，由于历史设计的原因，在激发 Kylin 构建一段时间区间内的数据时，我们需要先对应的 Hive 表进行一次刷新的操作，
这个操作不能由 Kylin 来控制
2. 在构建的时候，我们希望不同 Cube 的任务可以并行运行，但是同一个 Cube 的任务必须串行执行。
这是因为，在我们的系统设计中，有一类调度任务需要获取当前 Cube 的各项状态，用于规划之后的构建工作，
允许同一 Cube 的任务并行可能导致这类任务获取到的当前状态无效，简便起见我们希望避免这样的情况
3. 有一些特别的任务，如元数据的备份、HBase 表的清理等需要 Block 所有其他的任务。
这是因为在这些任务执行的时候需要对系统整体的元信息做一个查询或变更，
如果在这一过程中有其他任务正在执行会产生并发一致性问题，不但可能导致所得数据不完整，还可能损坏整个系统
4. 之前提到，横跨多个 Segment 的查询速度可能比较慢，一个可能的优化思路就是根据用户请求的特点，
对某一 Cube 的 Segment 实行更有针对性更积极的 Merge 策略。截至目前 Kylin 本身还没有提供对这一需求良好的支持。
因此自己实现 Merge 任务的调度也就比较有必要了

**运维自动化**，在 Strikingly，我们不但重度使用 AWS 的服务，还使用 [Terraform](https://www.terraform.io/)
和 [Docker 容器](https://www.docker.com/)等工具来实现运维自动化，
这些自动化工具要求我们系统的每个部分尽可能地无状态和可配置。尽管 Kylin 本身将自己的元数据保存在 HBase 上，
但是仍然有一些对运维自动化的挑战:

1. 包括 Kylin 在内的 Hadoop 组件深度依赖 XML 或 Java Properties 配置文件进行定义，
这对容器镜像的复用造成了一定的阻碍。比如说我们需要将这套系统分别部署在 AWS 的两个不同的 Region 中，
理想的情况是可以使用环境变量来定义 IP 地址等的变化，而不用构建不同的容器镜像
2. Kylin 目前集群部署的设计对于 Auto Scale 不友好。这主要是由于 Kylin 需要在集群的 Job 节点硬编码 Query
节点的 IP 地址和端口，并在 Cube 构建任务结束或者元信息变化之后通知 Query 节点，以便清理缓存更新数据。
这一点大大地限制了 ECS 服务在进行容器编排和 Auto Scale 时的灵活性。
3. 在实际中需要使用的一些 Kylin 功能(如元信息备份工具等)只能通过命令行调用，
而在在容器中使用简单的 crontab 脚本调度这些命令既笨重又容易出故障。

**系统的健壮性**，我们的数据平台系统在遇到故障或需要上线新的版本的时候不可避免的需要重启。
这样的情况下就可能出现本应被调度的一个事件不幸错过的现象。在以前，
我们需要人工地辨识这种情况并手动应用这些事件，但是在一个动态运行的系统当中，
手动的工作往往是费事且令人手忙脚乱的。理想状态下我们的调度系统应该具备一定的自我恢复能力，
能够在重启之后自主地发现错过的任务从而恢复他们。

综上所述，实现一个集中式、功能丰富的调度系统势在必行。

## 调度系统的设计

调度系统的整体设计如下图所示:

![Scheduler Overall Design](/img/kylin-scheduler/scheduler-overall-design.png)

图中箭头的方向指示了数据流动的方向和调度器控制组件的方向。为了满足前文提到的各种需求，
我们的调度系统有如下的特点:

1. 调度器集中地调度和控制除 Keen IO 之外的各个组件。在激发任务后监控并等待任务完成，以便使所有任务协调有序执行
2. Kylin 集群中，Job 节点不再与 Query 节点进行直接的交互，而是由 Scheduler 通过 AWS SDK 获得存在于指定 Target Group
中 Kylin 节点的地址信息，直接控制 Cache 刷新等工作
3. 调度器使用 DynamoDB 保存运行状态，因故重启之后自动读取记录恢复，从而获得更强的健壮性。保存在 DynamoDB
当中的记录还可以用来弥补任务调度当中出现的缺口
4. 调度器直接通过查询 Kylin Job 节点获得 Cube 和 Segment 的完整信息，自行连接 HBase 完成 Kylin
元信息表的备份和数据表的清理工作，从而避免执行命令行工具。备份数据将会被直接放置在 S3 上
5. 调度器和 Kylin 节点都使用 Docker 容器部署。对于 Kylin 节点来说，我们使用 Python 脚本自行编写了启动器，
这一脚本可以在启动时通过环境变量和预先提供好的配置模板，先进行字符串替换生成由环境变量定制过的配置文件，
再启动 Kylin 和其他 Hadoop 组件，从而可以在不同场合使用单个容器镜像

## 调度系统的实现

我们决定使用 Scala 编写和实现调度器，以便利用 Java/JVM 上的各种工具和库连接和管理包括 AWS、HBase 和 Hive 在内的各个组件。
除了 [AWScala](https://github.com/seratch/AWScala)、[Joda-Time](http://www.joda.org/joda-time/)、
[Spray-JSON](https://github.com/spray/spray-json) 等工具库之外，我们使用的最重要的框架是 [Akka](https://akka.io/)。

Akka 是一个 Scala/Java 下 [Actor 并发模型](https://en.wikipedia.org/wiki/Actor_model)的一个开源实现。
Akka 实现的 Actor 模型受 [Erlang](https://www.erlang.org/) 编程语言的启发，每个 Actor 相当于一个可以执行任务的对象，
这个对象执行的具体任务由它接收到的消息(Message)决定。这些对象都包含一个并发安全的队列作为信箱，
信箱中的每条消息将会被 Actor 串行地处理，Actor 接受到消息之后也可以再转发给其他的 Actor。
我们的调度器将会利用这些特性实现并发任务调度和管理。

Akka 还提供了一种名为`ConsistentHashingRouter`的组件。它本质上是一个专门用来分发消息的 Actor。
`ConsistentHashingRouter`可以利用消息本身提供的哈希键来分发消息到下游的 Actor，
它可以保证具有相同哈希键消息一定会被分发到同一个 Actor 上。我们可以利用这一特点来保证我们对于某一 Cube
的调度操作都是串行的。Akka 的 `ConsistentHashingRouter` 还支持包括 Auto Scale 在内更多丰富的功能，
在此不再赘述，下面是调度器的 Actor 系统的架构图:

![Actor System Structure](/img/kylin-scheduler/scheduler-actor-message-flow.png)

调度器的执行模块主要包括 5 个部分:

1. `ControlActor` 是调度器的重要组成部分，每个任务消息都会首先到达 `ControlActor`。`ControlActor`
会对消息进行一些预处理。所有的 `ControlMessage` 都会由 `ControlActor` 直接执行，不会被传递给下游
2. `ConsistentHashingRouter` 是一个分发器，它将接收到的 `TaskMessage` 以一致性哈希的方式分发给下游的 Actor。
这种分发保证同一个 Cube 的消息一定会在同一个 Actor 当中串行执行，同时也具备一定的并行性
3. `TaskActor` 是真正执行调度任务的 Actor。所有的 `TaskActor` 都是相同的，它们接收到消息之后也会进行一些预处理，
然后根据消息指定的 `Executor` 名称，选择正确的执行器进行执行。如果执行失败，`TaskActor`
也负责捕捉错误并进行一些错误恢复和 Log 的处理
4. `Executor` 是具体执行调度任务的代码逻辑所在的地方。一般来讲所有的 `Executor` 都是单例，
接收到对应的消息后，它们对消息的内容进行解析然后调用各种 `Service` 进行执行。`Executor`
根据自己的需要还可以进一步生成新的 `TaskMessage` 传输给 `ControlActor`， 从而 Spawn 出新的任务
5. `Scheduler` 是定时器，它定时地生成一些消息传递给 `ControlActor` 从而达到定期执行任务的目的

除了以上的几个部分，调度器还有一类模块称为 `Service`，用来抽象一些共享的代码逻辑(比如对 AWS 常用操作的整合等)。

首先可以看到，在调度器当中流动的任务消息有两种: `ControlMessage` 和 `TaskMessage`。

### ControlMessage

`ControlMessage` 是一类用来维护和管理调度器状态的消息，它指示 `ControlActor` 执行一些消息的维护过程。
比如说 `Recover` 消息指示 `ControlActor` 从 DynamoDB 当中抽取任务消息的状态以查看是否有需要恢复的任务。
如果有，它会将这些任务分发给下游执行。`ControlMessage` 只会由 `ControlActor` 执行且不会被记录到 DynamoDB 上。

`ControlMessage` 在 Scala 被定义为一系列的 Case Class。

### TaskMessage

`TaskMessage` 是描述实际调度任务的消息(如指示 Kylin 进行一次 Cube 构建等)。它的 Scala 类定义如下:

```scala
case class TaskMessage(
  uid: String,
  name: String,
  hashKey: String,
  data: Map[String, String]
) extends ConsistentHashable {
  def blocking = hashKey == null || hashKey.isEmpty

  override def consistentHashKey = if (hashKey == null) "" else hashKey
}
```

`TaskMessage` 包含一个 `uid` 用来全局唯一地标记一个消息，`name` 字段指定了 `Executor` 的名称，
`hashKey` 字段用来提供给 `ConsistentHashingRouter` 作为哈希的参考，一般是相关 Cube 的名称。
如果`hashKey` 的值是空的，我们就认为这个这个任务是阻塞的，也就是说在执行它之前，所有其他的任务都要结束，
在它执行完成之前，其他任务都需要等待。`data` 字段是一个 `Map[String, String]` 类型的字典，
用来保存一个任务消息的具体参数。

受函数式编程思想的影响，所有的 `TaskMessage` 都是 Immutable 的。也就是说一旦被构建出来，
一个 `TaskMessage` 所包含的任何字段都不会改变，因此可以保证在整个处理流程当中任务消息本身不存在副作用。

然而 `TaskMessage` 在运行过程中为了记录和恢复的需要，存在一个生命周期的概念。也就是说这类消息在生成之后，
会经历 `init`、`running`、`finish` 或 `error` 等几个生命周期。`ControlActor`
通过查询一个任务消息的生命周期状态来决定在重启恢复时是否需要恢复一个任务。下图展示了一个 `TaskMessage`
在处理流程各个部分生命周期的变化:

![Task Message Life Cycle](/img/kylin-scheduler/task-message-life-cycle.png)

简单来说，一个任务消息在进入 `ControlActor` 后会被转化为 `init` 状态，经过分发到达 `TaskActor` 后，
会在实际执行前被修改为 `running` 状态，在执行结束后根据执行状况可能被标记为 `finish` 或 `error` 状态。
这些状态会和消息定义本身一起被直接更新到 DynamoDB 当中以备之后查询利用。在这里，我们没有使用
[Write Ahead Logging](https://en.wikipedia.org/wiki/Write-ahead_logging) 的方式记录这些消息状态的变化，
因此在某些情况下仍然可能出现记录丢失等问题。但是考虑到实现的简洁性和实际需要，目前的解决方案应该足够稳定了。

在 `TaskMessage` 进入执行状态之前会先利用 `GlobalLockService` 获得执行锁。我们目前使用一个简单的读写锁实现我们的需求:
非阻塞任务将会尝试获得读锁，因此非阻塞任务可以并行执行。阻塞任务将会试图获得写锁，
因此阻塞任务和任何其他任务都是互斥的。

### 任务类型和 Executor

`TaskMessage` 的类型一一对应于不同的 `Executor`。接下来我们结合不同的 `Executor` 来介绍不同调度任务的类型。

任务调度最重要的两个类型是 `PlanDataRefresh` 和 `PlanCubeMaintenance`。
这两个任务并不具体执行需要完成的调度目标，而是通过请求 Kylin、AWS 等 `Service` 获得对当前系统状态的了解，
通过这些信息决定如何执行真正的调度任务。这两个任务在执行之后将会 Spawn 出新的消息任务导回 `ControlActor` 等待执行。

`PlanDataRefresh` 用来规划数据的刷新，它会先生成 `HiveTableRefresh` 任务用于刷新上一个小时进来数据的 Hive 表，
之后会根据当前时间片段覆盖的 Segment 来决定激发什么类型的 Cube 构建任务:

1. 当前处理的时间片段没有 Segment 覆盖，将会生成一个长度为 4 个小时的时间片段并在之上激发一个 `KylinCubeBuild` 任务
2. 当前处理的时间片段已经有 Segment 完全覆盖，将会针对覆盖这一时间片段所有的 Segment 执行 `KylinCubeRefresh` 任务
3. 在上一种情况之外，如果处理的时间片段有一部分没有被覆盖，将会在没有覆盖的部分激发 `KylinCubeBuild` 任务 

`PlanDataMaintenance` 任务用来扫描指定 Cube 的 Segment 和 `HiveTableRefresh` 任务执行的情况，
根据所得信息决定是否进行一系列的维护操作。主要进行的操作有:

1. 通过 Kylin API 获得 Cube 所有 Segment 的覆盖情况。如果存在没有被覆盖到的缺口，则激发 `KylinCubeBuild` 任务弥补缺口
2. 通过查询 DynamoDB 当中 `HiveTableRefresh` 任务近期执行的状况。如果发现没有执行过 Hive 表刷新的时间窗口，
则在这些窗口上执行 `HiveTableRefresh` 和 `KylinCubeRefresh` 等任务
3. 根据 Cube 所有 Segment 的时间片段分布情况，决定是否进行 Segment 的 `MERGE` 操作。如果需要进行合并，
则激发 `KylinCubeMerge` 任务

根据观察，Strikingly 的用户最常查询的数据为最近一周左右的，我们可以根据这一特点，按照过去距今时间的长短设定 Segment
数量的大小，以使得密集查询区间内的 Segment 比较少，又不至于过于频繁地激发合并操作。
Cube 的 Segment 数量采用如下的策略进行分配，以使得每一个 Cube 的 Segment 数量维持在 10 个左右。

![Segment Merge Strategy](/img/kylin-scheduler/segment-merge-strategy.png)

在每一个时间段内，决定如何合并 Segment 则是通过一个简单的贪心算法实现的。我们首先将时间段均匀划分为三个桶，
按照时间顺序尽可能多地向一个桶里放置 Segment，如果一个 Segment 不能被放进前面的桶，再将其放到新的桶中。
下图展示了这一过程:

![Segment Merge Algorithm](/img/kylin-scheduler/segment-merge-algorithm.png)

值得注意的是，像 `KylinCubeBuild`、`KylinCubeRefresh` 这样的任务会监控对应任务的执行状态，只有当 Kylin
当中对应的任务执行完成之后才会结束。由于 Kylin 的 API 是异步的，我们通过循环等待的方法等待任务的结束。
这样也可以保证任务在并行状态下的有序协调执行。同时，这些任务在结束时也会通过 AWS SDK 获得 Kylin 的 Query 节点的地址信息，
通过调用 Kylin 有关 Cache 管理的 API 直接广播通知状态的改变，从而取消了在 Kylin 的 Job 节点对 Query 节点地址信息的依赖，
使得 Query 节点可以 Auto Scale。

另外两个值得介绍的任务是 `KylinMetadataBackup` 和 `KylinHBaseTableCleanup`。这两个任务之前只能通过调用 Kylin
的命令行工具执行。通过查看 Kylin 的源代码，我们发现这两个任务较为简单，可以直接在我们的调度器当中实现。

`KylinMetadataBackup` 任务用于备份 Kylin 的元信息。Kylin 的元信息除了一些基本配置之外，还包括每一个 Cube、
Segment 和 Job 等的定义和构建信息等。默认情况下 Kylin 将所有的元信息保存在 HBase 上的 `kylin_metadata` 表中，
在调度器当中实现备份功能的原理是直接将 HBase 的 Client 集成进来，通过 Scan 这一 HBase 表，将每一条记录 Dump
下来。Kylin 的元信息都是以文件路径和 JSON 内容的形式存在的。我们直接以文件目录的方式将这些数据打包成 GZ 包。
最后生成的 GZ 包将会直接被上传到 S3。 

`KylinHBaseTableCleanup` 任务用于清理多余的 HBase 表。这是因为在某些情况下(如 Cube Refresh、系统故障退出等)
Kylin 构建任务产生的中间表或过期的表会遗留在 HBase 当中。而 HBase 存在表数过多会拖慢访问速度的问题，
因此必须定期清理这些表。Kylin 清理这些表的逻辑也很简单。那就是在没有任务执行的情况下扫描所有 Cube 和 Segment 等，
过滤出没有被任何 Cube、Segment 或 Job 引用的 Kylin 数据表，然后将这些表清理掉。我们在调度器当中实现了类似的功能，
通过 `HBaseAdmin` 和 Kylin API ，我们找出所有没有用的表，在第一次清理时先将这些表 Disable，第二次再删除。
这样可以留出一段时间作为缓冲，防止错误的数据删除。

显而易见，上述两个任务都需要阻塞所有其他任务的执行。因此它们的 `hashKey` 都为空。

有关任务类型方面最后值得一提的是，只有前面提到的 Plan 类型的任务和两个 Kylin 相关的 Backup 和 Cleanup
以及 `ControlMessage` 会被 `Scheduler` 定期生成。其他的任务都是由这些任务产生的衍生任务。

### 其他

除了调度器本身的实现，我们还使用了 [New Relic](https://newrelic.com/) 的 Application Performance Monitor 服务。
通过使用 New Relic 提供的 Java Agent，可以在云上对我们运行的 JVM 实例进行监控和 Profiling，
大大方便了我们搜集线上服务运行信息的效率。通过 New Relic 收集的数据，可以对各个任务执行的时间和代码执行瓶颈进行分析，
从而指导进一步的策略设计和代码优化工作。

## 总结与展望

上面就是我们为 Kylin 数据处理平台全新设计的集中式任务调度系统。它满足了我们对数据平台调度器定制化、
自动化和健壮性的需求，现在已经开始进行局部上线和测试。目前来看，
我们的调度系统和数据平台还有以下一些需要提高的地方:

1. 调度器目前无法手动提交任务。虽然新的调度器引入了对任务并发更好的控制，保证了相同和不同 Cube 的任务可以协调执行，
但有时手动执行任务仍然无法避免。更好的实践显然是将调度器作为提交任务的唯一入口。
一种可能的解决方案是为调度器实现 Web 接口，从而使得人工的操作也可以通过调度器执行
2. 目前使用 Keen IO 和 Hive 协调工作的解决方案过于复杂且健壮性差。由于网络延迟等原因，数据从 Keen IO
到达数据平台再到构建结束仍然具有很大的延迟和不确定性，晚到的数据本身也增加了系统实现的难度。
可能的解决方案包括精简 Hive 表流程、利用 [Kafka](https://kafka.apache.org/) 或
[AWS Kinesis](https://aws.amazon.com/kinesis/) 等消息队列服务传输信息以及自行收集用户访问数据等
3. 以 Kylin 为基础的数据平台虽然很好的解决了用户海量查询的问题，但是实时性较差。即使使用 Kylin 的 Kafka 
构建模式仍然无法实现亚分钟级别的实时数据查询。
为了解决这一问题，一种思路是引进 [Netflix Atlas](https://github.com/Netflix/atlas)、
[Facebook Beringei](https://github.com/facebookincubator/beringei) 或 [Yandex ClickHouse](https://clickhouse.yandex/)
这样的时间序列数据库，用于取代 Kylin 满足对短时数据的实时查询需求

就我个人来讲，在加入 Strikingly 转向数据平台相关的工作之后，接触并深入学习了 Scala、Hadoop、Kylin、HBase、Akka、AWS
等多种技术和工具，可谓是收获颇丰。希望以后能继续在数据平台这一方向取得更多进步。

## Presto ##
### 什么是Presto ###
Presto是一种旨在使用分布式查询有效查询大量数据的工具。 如果您使用TB或数PB的数据，您可能会使用与Hadoop和HDFS交互的工具。 Presto被设计为使用MapReduce作业（如Hive或Pig）管道查询HDFS的工具的替代方案，但Presto不限于访问HDFS。 Presto可以并且已经扩展到可以在不同类型的数据源上运行，包括传统的关系数据库和其他数据源，如Cassandra。

Presto旨在处理数据仓库和分析：数据分析，聚合大量数据和生成报告。 这些工作负载通常归类为在线分析处理（OLAP）。

Presto允许查询它所在的数据，包括Hive，Cassandra，关系数据库甚至专有数据存储。 单个Presto查询可以组合来自多个来源的数据，从而允许整个组织进行分析。

Presto针对的是那些期望响应时间从亚秒到分钟不等的分析师。 Presto打破了使用昂贵的商业解决方案进行快速分析或使用需要过多硬件的慢速“免费”解决方案之间的错误选择。

### Presto架构 ###

Presto是一个在一组机器上运行的分布式系统。 完整安装包括Cordinator和多个Worker。 查询从客户端（例如Presto CLI）提交给Cordinator。 协调器解析，分析和计划查询执行，然后将处理分发给Worker。

![](https://prestodb.github.io/static/presto-overview.png)

#### 服务器类型 ####

Presto服务器有两种类型：Cordinator和workers。 以下部分解释了两者之间的区别。

##### Cordinator #####
Presto Cordinator是负责解析语句，规划查询和管理Presto工作节点的服务器。 它是Presto安装的“大脑”，也是客户端连接以提交语句以供执行的节点。 每个Presto安装必须有一个Presto Cordinator和一个或多个Presto worker。 出于开发或测试目的，可以将单个Presto实例配置为执行这两个角色。

Cordinator跟踪每个Worker的活动并协调查询的执行。 Cordinator创建一个涉及一系列阶段的查询的逻辑模型，然后将其转换为在Presto工作集群上运行的一系列连接任务。

Cordinator使用REST API与Worker和客户进行通信。

##### Worker #####
Presto worker是Presto安装中的服务器，负责执行任务和处理数据。 工作节点从连接器获取数据并相互交换中间数据。 Cordinator负责从Worker那里获取结果并将最终结果返回给客户。

当Presto工作进程启动时，它会将自己通告给Cordinator中的发现服务器，这使Presto Cordinator可以执行任务。

Worker使用REST API与其他工作人员和Presto Cordinator进行通信。

#### 数据源 ####

在本文档中，您将阅读连接器，目录，架构和表等术语。 这些基本概念涵盖了Presto的特定数据源模型，将在下一节中介绍。

##### Connector #####
连接器将Presto适配到数据源（如Hive或关系数据库）。您可以将连接器视为与数据库驱动程序相同的方式。它是Presto SPI的一个实现，它允许Presto使用标准API与资源进行交互。

Presto包含几个内置连接器：一个用于JMX的连接器，一个提供对内置系统表的访问的系统连接器，一个Hive连接器和一个用于提供TPC-H基准数据的TPCH连接器。许多第三方开发人员都提供了连接器，以便Presto可以访问各种数据源中的数据。

每个目录都与特定连接器相关联。如果检查目录配置文件，您将看到每个目录配置文件都包含强制属性connector.name，目录管理器使用该属性为给定目录创建连接器。可以让多个目录使用相同的连接器来访问类似数据库的两个不同实例。例如，如果您有两个Hive集群，则可以在单个Presto集群中配置两个目录，这两个目录都使用Hive连接器，允许您查询来自两个Hive集群的数据（即使在同一SQL查询中）。

##### Catalog #####
Presto目录包含模式，并通过连接器引用数据源。 例如，您可以配置JMX目录以通过JMX连接器提供对JMX信息的访问。 在Presto中运行SQL语句时，您将针对一个或多个目录运行它。 目录的其他示例包括用于连接到Hive数据源的Hive目录。

在Presto中寻址表时，完全限定的表名始终以目录为根。 例如，hive.test_data.test的完全限定表名称将引用hive目录中test_data模式中的测试表。

目录在存储在Presto配置目录中的属性文件中定义。

##### Schema #####
模式是组织表的一种方式。 目录和模式一起定义了一组可以查询的表。 使用Presto访问Hive或MySQL等关系数据库时，模式会转换为目标数据库中的相同概念。 其他类型的连接器可以选择以对底层数据源有意义的方式将表组织成模式。

##### Table #####
表是一组无序行，它们被组织成具有类型的命名列。 这与任何关系数据库中的相同。 从源数据到表的映射由连接器定义。

#### 查询执行模型 ####
Presto执行SQL语句并将这些语句转换为在Cordinator和Worker线程的分布式集群中执行的查询。

##### Statement #####
Presto执行与ANSI兼容的SQL语句。 当Presto文档引用语句时，它指的是ANSI SQL标准中定义的语句，它由子句，表达式和谓词组成。

有些读者可能会好奇为什么本节列出了语句和查询的单独概念。 这是必要的，因为在Presto中，语句只是引用SQL语句的文本表示。 执行语句时，Presto会创建一个查询以及一个查询计划，然后该查询计划将分布在一系列Presto Worker中。

##### Query #####
当Presto解析语句时，它会将其转换为查询并创建分布式查询计划，然后将其实现为在Presto Worker上运行的一系列互连阶段。 在Presto中检索有关查询的信息时，您会收到生成结果集以响应语句所涉及的每个组件的快照。

语句和查询之间的区别很简单。 语句可以被认为是传递给Presto的SQL文本，而查询是指为实现该语句而实例化的配置和组件。 查询包含阶段，任务，拆分，连接器以及协同工作以生成结果的其他组件和数据源。

##### Stage #####
当Presto执行查询时，它会通过将执行分解为阶段层次结构来实现。 例如，如果Presto需要聚合存储在Hive中的10亿行的数据，则可以通过创建根阶段来聚合其他几个阶段的输出，所有这些阶段都旨在实现分布式查询计划的不同部分。

包含查询的阶段层次结构类似于树。 每个查询都有一个根阶段，负责聚合其他阶段的输出。 阶段是Cordinator用于建模分布式查询计划的阶段，但阶段本身不在Presto Worker上运行。

##### Task #####
如上一节所述，阶段为分布式查询计划的特定部分建模，但阶段本身不在Presto Worker上执行。 要了解阶段的执行方式，您需要了解阶段是通过Presto Worker网络分发的一系列任务来实现的。

任务是Presto架构中的“工作之马”，因为分布式查询计划被解构为一系列阶段，然后将这些阶段转换为任务，然后处理或处理拆分。 Presto任务具有输入和输出，正如一个阶段可以通过一系列任务并行执行，任务与一系列驱动程序并行执行。

##### Split #####
任务对拆分进行操作，拆分是较大数据集的一部分。 分布式查询计划的最低级别的阶段通过连接器的拆分检索数据，而分布式查询计划的更高级别的中间阶段从其他阶段检索数据。

当Presto正在安排查询时，Cordinator将查询连接器以获取可用于表的所有拆分的列表。 Cordinator跟踪哪些计算机正在运行哪些任务以及哪些任务正在处理哪些拆分。

##### Driver #####
任务包含一个或多个并行驱动程序。 驱动程序对数据进行操作并组合运算符以生成输出，然后由任务聚合，然后将其传递到另一个阶段中的另一个任务。 驱动程序是一系列操作符实例，或者您可以将驱动程序视为内存中的一组物理运算符。 它是Presto架构中最低级别的并行性。 驱动程序有一个输入和一个输出。

##### Operator #####
操作器消耗，转换和生成数据。 例如，表扫描从连接器获取数据并生成可由其他运算符使用的数据，并且过滤器运算符通过在输入数据上应用谓词来使用数据并生成子集。

##### Exchange #####
交换在Presto节点之间传输数据以用于查询的不同阶段。 任务使用交换客户端将数据生成到输出缓冲区并使用其他任务中的数据。

### 官方文档 ###

[https://prestodb.github.io/docs/current/index.html](https://prestodb.github.io/docs/current/index.html)
## HAWQ ##
### 什么是HAWQ ###
HAWQ是一个Hadoop本机SQL查询引擎，它结合了MPP数据库的关键技术优势和Hadoop的可扩展性和便利性。 HAWQ本地从HDFS读取数据并将数据写入HDFS。

HAWQ提供业界领先的性能和线性可扩展性。 它为用户提供了工具，可以自信地成功地与PB级数据集进行交互。 HAWQ为用户提供了完整的，符合标准的SQL界面。 更具体地说，HAWQ具有以下特征：

- 内部部署或云部署
- 强大的ANSI SQL合规性：SQL-92，SQL-99，SQL-2003，OLAP扩展
- 极高的性能 - 比其他Hadoop SQL引擎快很多倍
- 世界级的并行优化器
- 完整的事物能力和一致性保证：ACID
- 动态数据流引擎通过基于UDP的高速互连
- 基于按需虚拟段和数据位置的弹性执行引擎
- 支持多级分区和基于列表/范围的分区表。
- 多种压缩方法支持：snappy，gzip
- 多语言用户定义的函数支持：Python，Perl，Java，C / C ++，R
- 通过MADLib实现先进的机器学习和数据挖掘功能
- 动态节点扩展：以秒为单位
- 最先进的三级资源管理：与YARN和分层资源队列集成。
- 轻松访问所有HDFS数据和外部系统数据（例如，HBase）
- Hadoop Native：从存储（HDFS），资源管理（YARN）到部署（Ambari）。
- 身份验证和粒度授权：Kerberos，SSL和基于角色的访问
- 高级C / C ++访问库到HDFS和YARN：libhdfs3和libYARN
- 支持大多数第三方工具：Tableau，SAS等。
- 标准连接：JDBC / ODBC

HAWQ将复杂查询分解为小任务，并将它们分发到MPP查询处理单元以供执行。

HAWQ的基本并行单元是段实例。 商用服务器上的多个段实例一起工作以形成单个并行查询处理系统。 提交给HAWQ的查询被优化，分解为更小的组件，并分派到一起工作以提供单个结果集的段。 所有关系操作（例如表扫描，连接，聚合和排序）同时跨段执行。 来自动态管道中的上游组件的数据通过可伸缩的用户数据报协议（UDP）互连传输到下游组件。

基于Hadoop的分布式存储，HAWQ没有单点故障，并支持全自动在线恢复。 系统状态会持续受到监视，因此，如果某个段发生故障，它将自动从群集中删除。 在此过程中，系统继续提供客户查询，并且可以在必要时将这些段添加回系统。
### HAWQ架构 ###
在典型的HAWQ部署中，每个从节点都有一个物理HAWQ段，一个HDFS DataNode和一个NodeManager。 HAWQ，HDFS和YARN的主服务器托管在不同的节点上。

下图提供了典型HAWQ部署的高级体系结构视图。
![](http://hawq.apache.org/docs/userguide/2.3.0.0-incubating/images/hawq_high_level_architecture.png)

HAWQ与HARop资源管理框架YARN紧密集成，用于查询资源管理。 HAWQ在资源池中缓存来自YARN的容器，然后通过利用HAWQ自己对用户和组的更细粒度的资源管理来本地管理这些资源。 为了执行查询，HAWQ根据查询的成本，资源队列定义，数据位置和系统中的当前资源使用来分配一组虚拟段。 然后将查询分派给相应的物理主机，这些主机可以是节点的子集或整个集群。 每个节点上的HAWQ资源实施器监视和控制查询使用的实时资源，以避免资源使用违规。

下图提供了构成HAWQ的软件组件的另一个视图。
![](http://hawq.apache.org/docs/userguide/2.3.0.0-incubating/images/hawq_architecture_components.png)

#### HAWQ Master ####
HAWQ Master是系统的入口点。 数据库进程接受客户端连接并处理发出的SQL命令。 HAWQ Master解析查询，优化查询，将查询分派给段并协调查询执行。

最终用户通过主服务器与HAWQ进行交互，并可以使用客户端程序（如psql）或应用程序编程接口（API）（如JDBC或ODBC）连接到数据库。

主服务器是全局系统目录所在的位置。 全局系统目录是包含有关HAWQ系统本身的元数据的系统表集。 主服务器不包含任何用户数据; 数据仅驻留在HDFS上。 主服务器验证客户端连接，处理传入的SQL命令，在段之间分配工作负载，协调每个段返回的结果，并将最终结果呈现给客户端程序。

#### HAWQ Segment ####
在HAWQ中，段是同时处理数据的单元。

每台主机上只有一个物理段。每个段可以为每个查询切片启动许多查询执行器（QE）。这使得单个段像多个虚拟段一样，这使HAWQ能够更好地利用所有可用资源。

注意：在本文档中，当我们单独引用段时，我们指的是物理段。

虚拟段的行为类似于QE的容器。每个虚拟段对于查询的每个切片都有一个QE。使用的虚拟段的数量决定了查询的并行度（DOP）。

段与主服务器不同，因为它：

- 是无国籍的。
- 不存储每个数据库和表的元数据。
- 不在本地文件系统上存储数据。
- 主服务器将SQL请求与要处理的相关元数据信息一起分派给段。元数据包含所需表的HDFS URL。该段使用此URL访问相应的数据。

#### HAWQ Interconnect ####
互连是HAWQ的网络层。 当用户连接到数据库并发出查询时，将在每个段上创建进程以处理查询。 互连是指段之间的进程间通信，以及此通信所依赖的网络基础结构。 互连使用标准以太网交换结构。

默认情况下，互连使用UDP（用户数据报协议）通过网络发送消息。 HAWQ软件执行超出UDP提供的额外数据包验证。 这意味着可靠性等同于传输控制协议（TCP），并且性能和可扩展性超过TCP。 如果互连使用TCP，则HAWQ的可扩展性限制为1000个段实例。 使用UDP作为互连的当前默认协议，此限制不适用。

#### Resource Manager ####
HAWQ资源管理器从YARN获取资源并响应资源请求。 资源由HAWQ资源管理器缓冲，以支持低延迟查询。 HAWQ资源管理器也可以在独立模式下运行。 在这些部署中，HAWQ在没有YARN的情况下自行管理资源。

#### HAWQ Catalog Service ####
HAWQ目录服务存储所有元数据，例如UDF / UDT信息，关系信息，安全信息和数据文件位置。

#### HAWQ Fault Tolerance Service ####
HAWQ容错服务（FTS）负责检测段故障并接受来自段的心跳。

#### HAWQ Dispatcher ####
HAWQ调度程序将查询计划分派给选定的段子集并协调查询的执行。 调度程序和HAWQ资源管理器是负责动态调度查询和执行它们所需资源的主要组件。

### 官方文档 ###
[http://hawq.apache.org/docs/userguide/2.3.0.0-incubating/tutorial/overview.html](http://http://hawq.apache.org/docs/userguide/2.3.0.0-incubating/tutorial/overview.html)

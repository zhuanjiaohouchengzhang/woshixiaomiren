## 小米SparkSQL服务简介

本文为小米SparkSQL服务系列文章第一篇，主要介绍SparkSQL服务的的概念，原理，以及在原生SparkSQL上的一些改进。

### 什么是小米SparkSQL服务

小米SparkSQL服务是一个基于原生SparkSQL开发的SQL ad-hoc查询服务，底层依托于小米的资源调度管理服务Yarn, 并提供了多租户的功能。

#### 整体架构

小米SparkSQL服务的整体架构如下，相比于原生的SparkSQL ThriftServer, 小米SparkSQL服务通过增加SQLProxy以及ThriftServer与Yarn队列的绑定实现了多租户, 并且通过常驻型的ThriftServer来提供ad-hoc查询。

![sparksql-service-architecture](../img/sparksql-service-architecture.png)

如上图所示，小米SparkSQL服务架构主要分为两部分：

- **SQLProxy**: 负责在用户的yarn队列上启动Thriftserver, 接受客户端查询请求，并转发给对应的ThriftServer

- **Thriftserver**：真正的执行客户端查询，并将结果返回给SQLProxy，最终返回给客户端。每个ThriftServer与用户的队列是一一映射的。

概念解释：

- **常驻**:  服务启动之后一直存在，不会自动关闭, 避免启动的overhead。

- **ad-hoc查询**： 即席查询，一般是查询时间较短，根据用户当时的一些需求进行的一些查询。

- **多租户**： 不同团队的SparkSQL实例使用不同资源队列，不同队列之间资源互相隔离，互不影响。

#### 优势和特点

- **多租户**: 相比于原生的SparkSQL服务以及Impala，我们实现了多租户的方案，不同业务组之间可以做到资源隔离，解决了用户因为查询高峰期导致查询受影响的问题。

- **高可用**:  提供了HA的机制, 保证了SQLProxy的高可用, 避免因为单点宕机导致整个请求转发失败.

- **可扩展性**: 利用Yarn资源调度,极大减轻了运维部署相关的工作, 方便更大规模的服务扩展. 同时, 我们支持yarn-cluster模式，解决了driver受单机资源限制的问题.

- **完善的监控报警**: 提供了thriftserver异常失败报警，canary监控以及session/query监控和报警

- **统一入口**: 通过域名方式提供入口，实现迁移透明化

#### 使用场景

小米SparkSQL服务支持查询多种数据源，包括HDFS/Hive表/Kudu表，完全兼容Hive查询语法和Hive元数据，同时提供了对数据实时性要求更高的OLAP查询。如果你对OLAP查询感兴趣可以查看[OLAP文档](http://infra.d.xiaomi.net/olap/tutorial/introduction.html), 这里主要介绍SparkSQL服务整体的使用场景，以及和上下游之前的关系。

![sparksql-use-scenario](../img/sparksql-use-scenario.png)

如上图所示，SparkSQL服务支持标准的JDBC协议, 能够方便的接入上游各种各样的BI分析工具(例如superset/R等), 用户可以使用这些BI工具进行各种分析以及可视化, 当然, 也可以直接使用我们提供的命令行查询工具 - infra-client的beeline。SparkSQL服务下游是一些存储，比如HDFS文件/Hive表/Kudu表, 用户可以将原始数据(如实时数据流, MySQL数据库, 以及其他系统存量数据)导入到对应的存储系统然后利用上游BI工具进行相关的分析。

### 新feature和改进

我们基于小米内部的实际环境对SparkSQL做了一些相应的开发和改进，包括一些sql执行内部逻辑的优化和bugfix以及SQL服务化相关的改进。下面仅列出部分和用户使用直接相关的一些大的feature和改进。

- **完善的权限认证**

原生的SparkSQL是没有权限认证的，并且很难和小米内部的认证体系打通。一开始我们增加了简单的权限认证，原理就是直接把查询语句丢给Hive的解析接口，从而复用Hive的认证逻辑，但是有两个问题：1. SparkSQL的语法兼容Hive语法, 但是Hive的语法并不完全包含了SparkSQL的语法，所以会导致一些语法无法进行鉴权。2. 我们的OLAP服务发展迅速，OLAP服务底层也依赖SparkSQL，并且其鉴权逻辑和Hive并不是一套，所以直接复用Hive的逻辑会有问题。 基于上面的问题，我们基于Spark的逻辑执行计划实现了完善的权限认证。

- **扩展jar包动态更新**

扩展jar包是用户自定义的一系列udf函数和扩展包，早在Hive体系里面就存在了，但是Hive的扩展包的更新一直需要重启服务，这也是一直的一个痛点，因此，在SparkSQL服务上我们实现了自动拉取和动态载入扩展jar包。这里最大的难点就是如何避免类对象不被释放的问题，因为随着时间迁移，由于类的一些更新，加载的类对象必然越来越多，如果不释放会导致服务OOM。我们采用了替换classloader的方式实现了扩展jar包动态更新，同时解决了类对象不被释放的问题。

- **队列独立资源配置**

每个业务组对资源的需求以及能够承担的成本是不一样的，如果大家都用统一的资源配置可能出现小业务组承担不起资源成本，而大业务组觉得资源不足的问题。因此，我们支持了队列资源独立配置，方便各个业务组跟据各自的情况配置对应的队列资源，极大的改善了刚刚提到的问题。

- **OOM问题修复**

OOM问题一直是导致服务不稳定的一个重大隐患，因为服务端是不知道用户的查询情况的，很有可能因为用户的使用不当导致ThriftServer OOM。我们在实践和用户支持的过程发现有两点极易引起OOM的问题：

1. 用户select大量数据到客户端。 

2. 对于select a from table1 where a not in (select b from table2) 这种不带join key的join语法框架默认会走broadcastJoin, 如果table1和table2都较大时会导致OOM. 

对于问题1，我们自主实现了自适应数据收集算法，会对查询数据进行采样，跟据不同数据规模进行不同的处理，如果是大数据量会先将数据输出到hdfs然后迭代式返回给用户。对于问题2, 我们会对数据大小进行判断，如果大于broadcast的最大阈值，会直接返回异常提示给用户，告知用户相关的数据量大小信息和一些提示，这个issue提到社区，社区也初步提出了类似的修复方案，[SPARK-23124](https://issues.apache.org/jira/browse/SPARK-23124)

- **session/query监控报警**

监控和报警一直是线上服务特别重要的方面，除了一般的服务异常终止报警和canary监控报警，在SparkSQL上我们提供了session/query的监控和报警，

通过监听一些session的打开/关闭, query的执行/失败/完成事件，我们将这些数据进行一些计算后得到activeSession/activeQuery等信息，并将这些信息上报到falcon上，然后通过falcon实现告警通知。

另外, 我们会定期检查正在running的query的执行时间，如果执行时间过长也可以发送相关告警通知。

更多的信息和改进修复，可以查看我们的[ReleaseNote](http://infra.d.xiaomi.net/spark/modules/sql/releasenote.html)

### 现状与展望

#### 集群部署和用户接入情况

目前小米SparkSQL服务已经在国内和海外都部署了相应集群，国内集群主要是c3集群和zjy集群，海外集群目前只有新加坡集群，后续随着服务的发展和业务的增长会在越来越多的region进行部署。

目前已经接入小米SparkSQL服务的业务已经有10+个，包括小米广告，小米金融，小米精品生活电商，信息部，小米统计等等。

#### 业务支撑情况

目前每天我们SparkSQL所有集群的总查询数目在3k+, 波动较大，这也是ad-hoc查询的特点。另外，从目前的数据来看服务的可用性一直在99.99%以上。

#### 服务优化

目前整体SparkSQL服务运行状况良好，我们开发的一些patch也极大的解决了一些查询失败的问题（比如OOM问题），稳定性也经过了用户实际查询作业的检验。当然，在某些极端情况下，会有些慢或者排队甚至失败的情况。比如在资源不足的情况下，同队列并发查询容易出现排队慢的现象，不过，后续我们都会进行相关的优化，提升服务的稳定性，易用性和性能。同时我们会持续跟进社区发展, 及时引入一些好的feature和修复. 目前社区发展迅猛, 服务优化也是社区重点投入的方面, 相信站在巨人的肩膀上, 我们也会获益更多.

### 总结

本文从整体上介绍了小米SparkSQL服务的架构和特点以及做出的一些改进，希望能够给大家一个整体的认识。我们会继续专注于SparkSQL服务的优化，包括稳定性，易用性以及性能等各方面，请大家多多关注和支持。

如果对小米SparkSQL服务有兴趣或者任何问题，请直接联系我们：spark-help@xiaomi.com。
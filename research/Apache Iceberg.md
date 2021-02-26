# Apache Iceberg

Apache Iceberg is an open **table format** for huge analytic datasets，是一个轻量级的java库。

![image-20210224160257553](C:\Users\kafka\AppData\Roaming\Typora\typora-user-images\image-20210224160257553.png)

对于 table format，我认为主要包含4个层面的含义，分别是表 schema 定义（是否支持复杂数据类型），表中文件的组织形式，表相关统计信息、表索引信息以及表的读写 API 实现：

+ 表 schema 定义了一个表支持字段类型，比如 int、string、long 以及复杂数据类型等。

+ 表中文件组织形式最典型的是 Partition 模式，是 Range Partition 还是 Hash Partition。

+ Metadata 数据统计信息。

+ 封装了表的读写 API。上层引擎通过对应的API读取或者写入表中的数据。

设计时充分考虑对象存储设计。

## 来源

最初由Netfilx开源，后面合入Apache成为顶级开源项目。一开始Netfilx数据湖/仓也是基于Hive构建的，遇到了以下问题：

+ 依赖HDFS和Mysql（hiveMetaStore），查询时根据mysql的meta store找到相关的分区文件后，需要在HDFS上对分区进行文件系统的List操作，分区文件大时，操作非常耗时；
+ 元数据和数据文件分开存储在两个系统，写操作的原子性没办法保证；
+ Hive metaStore没有文件级别的统计信息，filter只能下推到分区，无法再下推到分区中的文件；
+ Hive对底层文件系统的语义依赖比较；文件系统切换成s3比较困难；
+ Hive对DDL语句支持比较费劲；

企业客户分析平台的lambda数据湖/仓更通用问题

业务数据来源 ---> 进入kafka --->  消费 Flink job / spark Streaming ---> 实时分析结果

​                                         ---> 导入到文件系统（parquet格式写HDFS或者s3，简配版数据湖）---> 导入数仓 ---->历史数据分析

+ 问题1：批量导入文件系统缺乏schama规范，导入数仓或直接分析的时候会碰到格式混乱的问题；
+ 问题2：数据写入文件系统没有ACID保证，用户可能读到中间数据的状态；exact-Once的语义比较难以保证；
+ 问题3：没有办法的高效update/delete, 列存文件一旦写入，更改就比较难了；
+ 问题4：频繁的导入会产生小文件；小文件多文件剪枝的代价就比较大；需要有定期小文件合并功能；

## Apache Iceberg的核心特性：

+ 支持ACID的事务；串行化的隔离级别：只有写写串行，其他并行，多版本支持，time travel
+ 支持识别parquet，orc标准列存文件，只有这些列存格式才能被table formats组织；
+ Schame支持DDL语句：add，drop，update，rename  -----只涉及到meta信息的修改和必要的列文件修改；

+ 流批接口支持：支持流式写入，表增量读取----意味着可以实现CDC；可以定义Flink Job增量消费Iceberg表中的数据；
+ 接口抽象化：相比与Delta lake和Hudi，写入和读取路径、底层存储高度抽象且设计优雅，做了非常好的解耦；
+ 查询性能优化：
  + 扫描执行计算快；
  + filter下推支持的比较好（文件级别）；
  + merge on read（on-going）；
  + 可以自定义flink-job对小文件进行定期合并；
+ 最新社区版本已经支持upsert
+ 不强调主键、使用方便；
+ 支持物化视图
+ 支持与Hive集成，直接读取hive中的表；避免现有数仓数据迁移；

## 现状和关键技术实现

现状：

![image-20210224181006750](C:\Users\kafka\AppData\Roaming\Typora\typora-user-images\image-20210224181006750.png)

AICD/MVCC隔离性

![image-20210224160514012](C:\Users\kafka\AppData\Roaming\Typora\typora-user-images\image-20210224160514012.png)

每一次写操作都会生成一个新的快照--->表再某个时刻的完整文件列表。

重分区不会破坏旧分区的数据组织方式；

## Parquet Orc列存

Parquet 标准列存格式

嵌套数据格式；

被多种查询引擎支持，和语言平台无关；该支持的谓词下推都支持；

Orc原生不支持复杂数据类型，需要特殊处理；压缩效率比parquet高一些；查询效率Orc比parquet稍微好一些；

## Flink And Iceberg

### 利用Flink+Iceberg可以构建实时的Data pipeLine；

![image-20210224175439188](C:\Users\kafka\AppData\Roaming\Typora\typora-user-images\image-20210224175439188.png)

业务端产生数据，被导入到 Kafka 消息队列等，运用 Flink 流计算引擎执行 ETL后，导入到 Apache Iceberg 原始表中（数据清洗）。有一些业务场景需要直接跑分析作业来分析原始表的数据，而另外一些业务需要对数据做进一步的提纯。可以再新起一个 Flink job从 Apache Iceberg 表中消费增量数据，经过处理之后写入到提纯之后的 Iceberg 表中（已经实现，但还没达到商用阶段；我们可以修改优化）。此时，可能还有业务需要对数据做进一步的聚合，可以继续在iceberg 表上启动增量 Flink 作业，将聚合之后的数据结果写入到聚合表中。

### Flink+Iceberg 来MySQL 等关系型数据库的 binlog 

![image-20210224175719471](C:\Users\kafka\AppData\Roaming\Typora\typora-user-images\image-20210224175719471.png)

Flink 已经原生地支持 CDC Mysql 数据解析。

Flink长远目标和Iceberg的长远目标一致：流批一体

## 和我们湖仓一体架构需求契合点

| 功能                     | Flink + Apache Iceberg                                       | clickhouse                                              |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------- |
| 事务支持                 | 支持                                                         | 有限支持； 社区 on-going                                |
| 体系内ELT                | 支持CDC，不成熟                                              | 无机制支持                                              |
| 支持BI直连               | 支持                                                         | 支持                                                    |
| 存储和计算分离           | 支持                                                         | 不支持                                                  |
| 开放                     | 标准列存parquet，除了我们自用的查询引擎，可以对接其他计算引擎 | 支持，性能不如自己的格式，差很多                        |
| 非结构化和半结构化的支持 | 支持                                                         | 支持                                                    |
| 支持不同的workloads      | 支持                                                         | 支持有限                                                |
| 流式ETL                  | 支持                                                         | 不支持                                                  |
| 数据安全                 | 不支持                                                       | 不支持                                                  |
| 调度与监控               | flink实现ETL有支持；flink或者计算引擎部署到k8s或者k3s上，坚定的拥抱k8s（云时代的操作系统）；监控部分没有； | 调度不支持；clickhouse on k8s不知道行不行；监控支持部分 |

## & PK Clickhouse

1. clickhouse的存储和计算分离我们自己实现难度比较大；且改造分离后性能到底如何，不好判断；
2. 我们基于clickhouse做不出很有特色的点？客户为什么不直接用clickhouse呢？公有云也有clickhouse的托管服务，他们对clickhouse也有对应的云上适配和修改
3. 表内无法实现CDC
4. clickhouse是一个“点”；snowflake是一个面；当然，也可以把clickhouse做成“面”；

如果选择clickhouse：我们以后技术研发的功能可以更单纯一些：

专注于改造存储与计算分离和CDC即可，其他的功能（SQL）和性能比较强大；

## 最后

基于Flink需要对iceberg功能进行测试，iceberg对接flink还不成熟（DDL无法通过SQL实现），是机会；

选择clickhouse还是选择Iceberg+flink，需要共同决定；

Iceberg：需要维护的组件比较多；

我们提供的计算引擎尽量也收敛于flink；需要在计算节点缓存源数据；

cloudera 也明确地选型 Apache Iceberg 来构建他们的商业版数据湖；他们将基于 Apache Iceberg 推出公有云服务，将给用户带来完善的 Flink、Spark、Hive、Impala 数据湖集成体验；


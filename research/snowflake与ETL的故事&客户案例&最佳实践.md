## snowflake与ETL的故事

对于客户数据抽取、转化和加载、数据清理等数据治理的需求，snowflake有三种处理方式：

+ 和一些可视化的ETL工具配套使用，在loading至snowfalke之前，在这些工具中处理好数据治理需求；很多ETL工具都已经内置支持了将转化结果直接上传到snowflake的功能（需要简单的配置）；
+ 强调ELT（抽取，加载，转化）而非ETL，内置了很多工具、方法来实现将数据loading时/至snowflake进行数据转化；
+ 直接支持半结构化数据的查询，可以减免很多半结构化数据的转化过程；

展开来说。

### 配套ETL tools 使用

snowflake可以和很多可视化ETL工具（ Informatica, Talend, Tableau, Matillion等）进行配套使用，很多数据整合工具/数据管道/数据源都提供了直连snowflake的功能（只需要简单的配置，积累的生态），就可以将转化后的结果自动持续上传到snowflake中。

### 强调ELT而非ETL， 数据转换在snowflake内部完成

ETL：抽取源数据，转化源数据（清理转化），导入转化后的数据至snowflake；

ELT：抽取源数据，导入源数据至snowfake时/后，在数据仓库中做数据转化；

+ 通过copy into <table> 命令（将stage的数据加载进table时使用）进行支持，可以指定：列重新排序，使用select语句只选择特定的列、可以根据配置截断某个长度的字符串，各种表达式函数，日期，源数据格式自定义等以及支持CSV、json、Avro、ORC、Parquet、XML等半结构化数据展平，拆分列，列填充等。
+ 支持创建Temporary Tables/stages（生命周期为session，session断开后，临时表即被删除）。用临时表可以存储ELT的中间数据。支持创建复杂的task，stream来（增量）自动定时处理table/Temporary Table 中变化的数据（insert，delete，update）;可以用snowpipe自动加载stage中的增量数据到table中；

+ 内置专门的data interation area，在这里即可以使用[snowflake认证的工具](https://docs.snowflake.com/en/user-guide/ecosystem-etl.html)进行ETL操作，也可以自己programming，支持的语言有Python，go，java，.net， js

![snowflake refence architectture](https://github.com/kafka1991/lakehouseResearch/blob/master/picture/snowflake%20Architecture.png)

当然，snowflake只是提供了这些可能的选择。具体到是使用ETL还是ELT，每个客户都可以根据经验和preference来选择。这些工具和自服务式的pipelines正在消除传统的需要人工coding进行的ETL和数据清理。snowflake声称入湖/入仓之前不需要任何的pre-transformations和预定义数据的schema。

### 半结构化数据的直接查询支持

Snowflake中支持3种类型的半结构化数据，其类型分别是：

- VARIANT
- OBJECT
- ARRAY

VARIANT可以存储其它数据类型，包含OBJECT或ARRAY本身，只要其大小压缩后小于16MB就可以。OBJECT和ARRAY是VARIANT的一种特例，类似于JSON中的对象和数组。而且VARIANT、OBJECT、ARRAY都支持嵌套。详细可以：https://docs.snowflake.com/en/sql-reference/data-types-semistructured.html

对于JSON数据的处理，我们可以直接设置目标列为 OBJECT 类型，然后就可以使用类似于： content:attribute::string 的方式来获取值了。使用起来非常方便。《The Snowflake Elastic Data Warehouse》中介绍对于半结构化数据的支持：

>存取部分，利用内部存储格式使得从 VARIANT（以及OBJECT、ARRAY)中存取数据都非常高效：
>
>无需额外的复制
>
>A child element is just a pointer inside the parent element通过指针加速
>
>Extraction is often followed by a cast of the resulting VARIANT value to a standard SQL type。同时，其内部存储格式也使得这些类型转化非常高效

Snowflake在存储Variant数据时，会大量利用算法来统计Variant数据中的模式信息，并利用历史查询记录，推算出该Variant列中，哪些访问路径是最常见的，把这些列单独存为物理列（列式存储），而当查询的时候，则再把这些列合并到VARIANT中。Snowflake论文中测试了其对半结构化数据的自动类型推导和自动索引的效率，对TPC-H同样的测试数据集做了两个不同的存储实验： 一种是各个列都存为原始的数据类型，另一种是把所有的列组装成一个 VARIANT（OBJECT） 类型，然后测试其性能区别。 据论文数据，大多数查询的性能区别都小于 10% 。

## 一些最佳实践&用户案例

+ trade me

  一个拍卖网站，snowflake的客户，具体[案例连接](https://medium.com/default-to-open/snowflake-the-details-of-our-first-data-warehousing-project-in-the-cloud-2e5d4435275c)。

  总结如下：

  + 数据源在aws的数据湖中，使用了snowpipe持续监听aws数据湖中的数据变化，自动推送变化的数据到snowflake中；---编程调用api的形式实现
  + 单点登录，打通和snowflake的鉴权系统；
  + ETL：使用perfect（定时调度工具）来调度ETL任务；调度任务是用python写的一系列任务集合，使用snowflake的python 连接器api来定义具体执行的工作；
  + 在snowflake内部对Json数据进行了定期ETL抽取成维表和实时表；
  + 通过JDBC和ODBC服务的形式向外提供服务，可视化与Power BI集成；

+ capital One

  信用库和借款的银行公司

  痛点：分析数据太慢了，本地环境宕机频繁，不能伸缩

  角色：可视化报表分析和其他application的的底层数据引擎、

+ overstock

  在线零售公司

  痛点：旧系统部署比较难，需要维护一堆组件和big data tools；不同的数据分散在不同的系统中；

  角色：统一的数据分析平台，在此上运行各种分析任务，可以直接将云上的业务数据持续对接到分析平台中。

## Snowflake的大概技术实现及其带来的影响

![snowflake Architecture](picture\snowflake Architecture.png)

### 存储如何支持修改&事务

对象存储的优点是：低成本、高可用、无限容量；缺点是：不支持标准的POSIX文件操作，而是只支持简单的 PUT、GET、COPY、DELETE、LIST操作，而且list和rename属于时间复杂度较高的操作。

在Snowflake的技术体系中，其它的都可以用不同的组件替换，唯一不能替换的是底层的对象存储（亚马逊云上是S3，阿里云是OSS，腾讯云是COS）。snowflake用对象存储来实现数仓需要的修改 & 事务（Transaction）支持的方式为：在对象存储的基础上，增加一层“事务管理”的服务层。具体实现为使用了 [FoundationDB](https://www.snowflake.com/blog/how-foundationdb-powers-snowflake-metadata-forward/) 这个分布式多租户NoSQL数据库来作为事务的支持。

FoundationDB的作用：

- 作为“元数据”，可以存储Snowflake中的用户、Database信息，表信息，vm计算资源信息等；
- 因为其支持ACID事务，可以作为各个操作的协调者
- 对象存储中虽然存储着表的实际数据（文件），但是如何解读这些数据（表结构），则需要FoundationDB中该表的Schema信息

对象存储和FoundationDB这两个搭档的工作流程是：

+ 用户首先使用SQL创建 Table1， 这时，Snowflake只要在FoundationDB中创建一条对应的Schema记录即可，无需涉及对象存储

+ 接下来用户使用Insert语句批量的插入了一批数据到 Table1，这时，vm会首先把这些记录写到一个列式存储文件中，并上传到对象存储，文件名可能为： STRING_1.parquet，然后在FoundationDB中更新Table1的Schema记录来关联到 STRING_1.parquet

+ 用户又执行另一批Insert语句到Table1。这时，我们不能修改 STRING_1.parquet 的内容，而是利用vm把 STRING_1.parquet 的内容和新的数据做一个合并操作，生成新的 STRING_2.parquet 文件并上传对象存储，最后在FoundationDB中更新Table1的Schema记录来关联到 STRING_2.parquet

所以，S3中的文件是写过后就不可变了（immutable），对于每个表具体对应于哪个S3文件则是在元数据层FoundationDB中记录的。这里简化了细节，同一个表的存储文件可能会分为多个分区，并且，也不可能每做一次修改就全量生成S3。类似的开源实现有：

+ [Apache Hudi](https://hudi.apache.org/)
+ [Databricks Delta Lake](https://delta.io/)
+ [Apache Iceberg](https://iceberg.apache.org/)

其中，公认以iceberg的实现最好，因为其与计算引擎彻底解耦。其他两个都以spark默认作为默认计算引擎。

### 充分利用对象存储

+ vm查询结果处理：

  传输结果集不用维护客户端到vm的网络连接。Snowflake本身是存储计算分离的，计算部分是通过动态启动的VW来实现的，而且Snowflake对VW支持非常灵活的： 自动挂起、自动恢复、扩容、缩容。这样，执行优化器对vm长时间保持网络链接来传输数据比较难， query的结果集不是通过vm直接返回到客户端（意味着需要维护vm到客户端的连接）。所以，Snowflake中的任何查询VW执行后都直接把查询的最终结果存储为S3中的文件。而该结果文件和查询的具体关联关系则通过一条FoundationDB中记录来存储。当用户端获取查询结果时，只需要经过Snowflake的Service层，Service层从FoundationDB读取到查询结果的S3路径，并转发S3文件中的内容到客户端，而无需涉及计算层VW。这样做了以后，对于一些数据库的修改操作（Merge），或者查询操作，其实对于VW来说都是一样的： 执行SQL，然后结果写到指定的S3路径下。

+ 避免vm节点间数据传输。

  对于我们MapReduce，多个m节点需要计算完成后需要将结果发送到r节点。但是对于snowflake来说，管理节点间的Shuffle数据，并做到网络负载平衡是非常难的任务。所以，Snowflake也把中间结果存储在S3上。其VW层执行时，并不用考虑该SQL是不是需要分布，其理解的任务都是一样的单机任务。它只接受3个参数：

  - 输入的数据文件位置
  - 要单机执行的SQL
  - 输出的数据文件位置

  这样，mr中的m节点会生成n个无状态的任务，并由worker执行。当n个任务执行结束后，写结果到s3文件中，Service层中的调度组件再发送一个汇总任务，由任一Worker执行。Snowflake借助S3的高可用和AWS的网络来实现更简单分布式处理。

+ 增加缓存层加速s3的读取写入。

  上面2个充分利用S3的功能，虽然能简化架构的设计，但是也带来了新问题： S3毕竟不是本地的SSD磁盘，其读取是有性能代价的。Snowflake也有一个针对S3的缓存，在读取某些S3文件时，会提前缓存用到的部分到VW本地的SSD硬盘上，来进行查询的加速。我感觉这部分和开源实现[Alluxio](https://github.com/Alluxio/alluxio)非常相似。

### 带来的好处

+ 多版本支持，快照的隔离级别&Time Travel。从对象存储如何支持修改&事务可以看出，，对于Snowflake的数据表的每个修改都会产生一个新的版本。而由于对象上的旧的文件并不会被修改，snowflake把这些历史S3文件和对应的Version记录下来，对于一个Table，就拥有了该历史版本的快照。Iceberg和hudi都有相同的开源实现。从而Snowflake可以做到 Time Travel 到该数据表90天内的任意一时刻（为什么是90天，应该是成本控制吧，不可能无限保存历史版本）。

+ 方便共享数据由于数据都是只读地存储在S3中，而如何解释这些文件则通过FoundationDB中的记录控制。那么，数据的复制和分享将变得简单。如果我们想把某个数据表复制到另一个数据库中，我们无需复制S3中的文件，而只需要把这些文件的信息关联到 FoundationDB 的一条新的表记录中。比如我们测试Snowflake的性能时，Snowflake已经把TPC-DS大规模测试数据集分享给我们了，方便了我们对Snowflake的性能测试。由于复制和分享数据无需增加额外的存储，以此可以衍生出很多新的使用方式（支持多环境、快速的基于线上的表搭建一个预发环境等）

## 附录

### snowflake四种收费版本

标准版：入门级产品，标准功能的无限次使用访问；

企业版：除了标准版的功能和服务，提供了一些大数据规模下的数据处理功能；

商业版：提供了高级的敏感数据加密保护；

私有云版：提供对金融数据的最高的安全级别；

### 具体编程语言

不好具体考证。但从具体的招聘要求和网上的面经对语言的要求来看：

安全方面：python、Perlg、go；

权限（用户、组、角色）方面：java、sql；

实时数据存储处理：java、sql；

queryEngine ，性能工程师：C or C++精通，java是加分项；

数据库工程师：Java or C++

Meching learning：Java, C# or C++（应用层）

query processing：Fluency in C++, C, or Java 

...

基本上可以推出计算引擎用c/c++；其他服务用java；安全加密用python、go、Perlg等。

### 我们能在云上制造出自己的“snowflake“吗？

很难。

云服务收费的三个大头：存储、计算和网络带宽。云厂商同一个区域内的主机和服务之间的网络通信是不收费的，但是要收取从该区域流出到外网或其它区域\云服务商的费用。这样，为了降低成本，用户的数据在哪个云服务商/区域上（离哪里的数据中心更近），snowflake就智能的选取同区域/云服务商的主机作为计算节点。意味着，snowflake本身已经在维护多个云厂商，多个区域的不同集群---你的数据在哪个云上，就用哪个云的集群，你在哪个区域，就用哪里的集群。----降低用户数据传输的网络费用和上传延迟

Snowflake很可能会和国内的某个云合作（在国产化软件的浪潮下，这个可能不是问题）。snowflake功能点很多，的做了很多细节的工作（创建物化视图，一个SQL可以查询不同维度下的小计聚合，半结构化的数据的查询加速  这只是是我体验到的），而且做得非常好，我们需要投入大量的人力和财力。

### 我们数仓的思考

云上构建数仓是非常复杂的一项工程...，我们也看到了。所以我们第一步必然是在帮助用户本地构建数仓解决方案。方向应该是：

+ 真正的帮用户降低成本
+ 真正的帮用户运维方便
+ 性能和稳定性

存储和计算分离真的重要吗？在云上构建数仓，当然重要，而且是必不可少的架构。如果在本地机房构建数仓，用户的计算资源和存储资源是分别采购的吗，用户机房的网络带宽真的能满足需求吗（最主要的，已加入调查问卷）？我们在构建系统，实施项目时要考虑这些问题。






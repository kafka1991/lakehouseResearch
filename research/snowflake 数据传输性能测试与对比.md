# snowflake 数据传输效率测试
## 测试目标
snowflake作为一款云数仓产品，为了更好的伸缩性，采用了计算与存储节点分离的方案。此测试目标为大致推算出数据在存储节点和计算节点之间的传输时间占query总执行时间的占比。
## 测试方法
在snowflake的官方文档和论文中，计算节点对于缓存的描述：
>每个vm内部的worker节点会在本地磁盘上缓存table的一部分已访问过的s3对象存储数据。为了更精确的缓存，node只缓存他们query查询用的列的数据。
>缓存目前使用的算法为LRU。为了更好的命中缓存，vm的查询优化器根据query查询计划用到的table的文件名，一致性hash到worker节点上。顺序或者并发的查询相同的表文件将落到同一个node节点上。
>缓存使用懒加载模式，query查询过的才会缓存起来，不会主动load数据。

*注：vm为Virtual Warehouse的缩写, 是snowFlake的计算节点；其中包含了worker nodes。根据包含的worker nodes数量不同，vm可配置为X-Small，Large，XX-Large等类似衣服size的容量配置。*

论文中只提到会缓存列的源数据（细节数据），对query结果集的缓存没有提及。但是测试发现，完全相同且复杂query，后一次执行会非常快，推断对结果集也有缓存。为了能在一次query中粗略的推算出传输使用时间，可以将一个query执行两次，如果无结果集缓存，则第一次执行时间减去第二次执行时间即为**query所需的列**由存储节点传输到计算节点的时间。为了排除结果集缓存的影响，可以将第一次query和第二次的query稍微改造一下，使其无法命中结果集缓存，同时又可以认为的计算花费时间是相同的。
例如：

```sql
query1 = select count(distinct concat("C_PHONE", '---1')) from CUSTOMER
query2 = select count(distinct concat("C_PHONE", '---2')) from CUSTOMER
```
传输时间t和占query查询时间占比为：
$$
cost(transfer) = cost(query1)−cost(query2)
$$

$$
t = \frac {cost(transfer)} {cost(query1)} * 100\%
$$

## 测试数据

### 自备数据

由于传输时间理论上和表文件存储大小呈正相关，而snowflake只提供了表级别的数据存储size的统计。因此，为了准确的反应存储空间对传输效率的影响，设计为每张表只有一列数据，表的存储空间大小即可认为是此列的存储空间的大小。

此测试针对string，date ，int，float四种数据类型设计了6张数据表，每张数据表只包含一列，其中string又分为string（50位长度），middleString（100位长度）和bigString(150位长度)，每个数据表行数都为1亿行。

| 数据库表名 | 列名      | DataType       | 本地文件大小 | description                                                  |
| :--------- | --------- | -------------- | ------------ | ------------------------------------------------------------ |
| STRING50   | string50  | VARCHAR(50)    | 4.7G         | 随机生成，csv格式，拆分为200个文件长传                       |
| STRING100  | string100 | VARCHAR(100)   | 9.7G         | 随机生成，csv格式，拆分为200个文件长传                       |
| STRING150  | string150 | VARCHAR(150)   | 14.5G        | 随机生成，csv格式，拆分为400个文件长传                       |
| DATE       | DATE      | DATE           | 1.2G         | 随机生成2000-01-01到2020-12-31之间的数据， csv格式，拆分为200个文件长传 |
| INT        | INT       | NUMBER(38, 0） | 1.14G        | 随机生成整型，csv格式，拆分为200个文件上传FLOAT              |
| FLOAT      | FLOAT     | FLOAT          | 658M         | 随机生成0-100之间的小数，保存两位有效数据。csv格式，拆分为200个文件上传 |

*注：snowflake上传数据时，最多允许50M数据文件上传，故拆分之。所有数据均为由本地Java代码随机生成。受网络限制，数据上传速度较慢。*

### snowflake提供的TPC-H数据

由于网络原因，自备数据上传速度较慢。snowflake官方提供了TPC-H亚马逊云上数据，可以先用相同的测试方式采集一部分测试结果。

## 测试环境

snowFlake亚马逊云存储，日本东京节点。（目前无中国区节点支持），vm配置为x-large。

## 测试结果

### 云上数据TPC-H测试结果

#### 测试1

表名：CUSTOMER

表结构：

| 列名         | DataType     |
| ------------ | ------------ |
| C_CUSTKEY    | NUMBER(38,0  |
| C_NAME       | VARCHAR(25)  |
| C_ADDRESS    | VARCHAR(40)  |
| C_NATIONKEY  | NUMBER(38,0) |
| C_PHONE      | VARCHAR(15)  |
| C_ACCTBAL    | NUMBER(12,2) |
| C_MKTSEGMENT | VARCHAR(10)  |
| C_COMMENT    | VARCHAR(117) |

表数据行数：15，000，000行

存储空间：1GB

测试结果：

vm配置为x-large时

| 查询SQL                                                      | 总执行时间 | 数据是否缓存在计算节点 |
| ------------------------------------------------------------ | ---------- | ---------------------- |
| `select avg("C_CUSTKEY" + 1), avg("C_ACCTBAL" + 1), `<br>`avg("C_NATIONKEY" + 1),"C_NAME", "C_ADDRESS", `<br>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 5.15s      | cold                   |
| `select avg("C_CUSTKEY" + 2), avg("C_ACCTBAL" + 2), `<br/>`avg("C_NATIONKEY" + 2),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4.91s      | hot                    |
| `select avg("C_CUSTKEY" + 3), avg("C_ACCTBAL" + 3), `<br/>`avg("C_NATIONKEY" + 3),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 3.94s      | hot                    |
| `select avg("C_CUSTKEY" + 4), avg("C_ACCTBAL" + 4), `<br/>`avg("C_NATIONKEY" + 4),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 3.72s      | hot                    |
| `select avg("C_CUSTKEY" + 5), avg("C_ACCTBAL" + 5), `<br/>`avg("C_NATIONKEY" + 5),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 3.13s      | hot                    |
| `select avg("C_CUSTKEY" + 6), avg("C_ACCTBAL" + 6), `<br/>`avg("C_NATIONKEY" + 6),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4.43s      | hot                    |

vm配置x-small时

| 查询SQL                                                      | 总执行时间 | 数据是否缓存在计算节点 |
| ------------------------------------------------------------ | ---------- | ---------------------- |
| `select avg("C_CUSTKEY" + 1), avg("C_ACCTBAL" + 1), `<br>`avg("C_NATIONKEY" + 1),"C_NAME", "C_ADDRESS", `<br>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 27.75s     | cold                   |
| `select avg("C_CUSTKEY" + 2), avg("C_ACCTBAL" + 2), `<br/>`avg("C_NATIONKEY" + 2),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 23.78s     | hot                    |
| `select avg("C_CUSTKEY" + 3), avg("C_ACCTBAL" + 3), `<br/>`avg("C_NATIONKEY" + 3),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 23.84s     | hot                    |
| `select avg("C_CUSTKEY" + 4), avg("C_ACCTBAL" + 4), `<br/>`avg("C_NATIONKEY" + 4),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 23.3s      | hot                    |
| `select avg("C_CUSTKEY" + 5), avg("C_ACCTBAL" + 5), `<br/>`avg("C_NATIONKEY" + 5),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 22.72s     | hot                    |
| `select avg("C_CUSTKEY" + 6), avg("C_ACCTBAL" + 6), `<br/>`avg("C_NATIONKEY" + 6),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 23.02s     | hot                    |

结果集大小：15,000,000行

#### 测试2

表名：同测试1

表结构：同测试1

表行数：150，000，000行

存储空间：10.1GB

测试结果

vm配置x-large时

| 查询SQL                                                      | 总执行时间 | 数据是否缓存在计算节点 |
| ------------------------------------------------------------ | ---------- | ---------------------- |
| `select avg("C_CUSTKEY" + 1), avg("C_ACCTBAL" + 1), `<br>`avg("C_NATIONKEY" + 1),"C_NAME", "C_ADDRESS", `<br>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 21.27s     | cold                   |
| `select avg("C_CUSTKEY" + 2), avg("C_ACCTBAL" + 2), `<br/>`avg("C_NATIONKEY" + 2),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 36.43s     | hot                    |
| `select avg("C_CUSTKEY" + 3), avg("C_ACCTBAL" + 3), `<br/>`avg("C_NATIONKEY" + 3),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 19.44s     | hot                    |
| `select avg("C_CUSTKEY" + 4), avg("C_ACCTBAL" + 4), `<br/>`avg("C_NATIONKEY" + 4),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 20.36s     | hot                    |
| `select avg("C_CUSTKEY" + 5), avg("C_ACCTBAL" + 5), `<br/>`avg("C_NATIONKEY" + 5),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 19.09s     | hot                    |
| `select avg("C_CUSTKEY" + 6), avg("C_ACCTBAL" + 6), `<br/>`avg("C_NATIONKEY" + 6),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 19.12s     | hot                    |

vm配置x-small时

| 查询SQL                                                      | 总执行时间 | 数据是否缓存在计算节点 |
| ------------------------------------------------------------ | ---------- | ---------------------- |
| `select avg("C_CUSTKEY" + 1), avg("C_ACCTBAL" + 1), `<br>`avg("C_NATIONKEY" + 1),"C_NAME", "C_ADDRESS", `<br>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4min22s    | cold                   |
| `select avg("C_CUSTKEY" + 2), avg("C_ACCTBAL" + 2), `<br/>`avg("C_NATIONKEY" + 2),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4min19s    | hot                    |
| `select avg("C_CUSTKEY" + 3), avg("C_ACCTBAL" + 3), `<br/>`avg("C_NATIONKEY" + 3),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4min15s    | hot                    |
| `select avg("C_CUSTKEY" + 4), avg("C_ACCTBAL" + 4), `<br/>`avg("C_NATIONKEY" + 4),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4min27s    | hot                    |
| `select avg("C_CUSTKEY" + 5), avg("C_ACCTBAL" + 5), `<br/>`avg("C_NATIONKEY" + 5),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4min24s    | hot                    |
| `select avg("C_CUSTKEY" + 6), avg("C_ACCTBAL" + 6), `<br/>`avg("C_NATIONKEY" + 6),"C_NAME", "C_ADDRESS", `<br/>`"C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER"` <br/>`group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"` | 4min16s    | hot                    |

结果集大小：150,000,000行

#### 测试3

表名：JORDERS

表结构：

| 列名   | DataType |
| ------ | -------- |
| ORDERS | VARIANT  |

表行数：15，000，000行

存储空间：4.4GB

测试结果：

vm配置x-large时

| 查询sql                                                      | 总执行时间 | 数据是否缓存在计算节点 |
| ------------------------------------------------------------ | ---------- | ---------------------- |
| `select count(distinct concat("ORDERS",'---1')) from "JORDERS"` | 19.78s     | cold                   |
| `select count(distinct concat("ORDERS",'---2')) from "JORDERS"` | 18.75s     | hot                    |
| `select count(distinct concat("ORDERS",'---3')) from "JORDERS"` | 19.77s     | hot                    |
| `select count(distinct concat("ORDERS",'---4')) from "JORDERS"` | 17.71s     | hot                    |
| `select count(distinct concat("ORDERS",'---5')) from "JORDERS"` | 18.84s     | hot                    |
| `select count(distinct concat("ORDERS",'---6')) from "JORDERS"` | 18.04s     | hot                    |

vm配置x-small时

| 查询sql                                                      | 总执行时间 | 数据是否缓存在计算节点 |
| ------------------------------------------------------------ | ---------- | ---------------------- |
| `select count(distinct concat("ORDERS",'---1')) from "JORDERS"` | 2min46s    | cold                   |
| `select count(distinct concat("ORDERS",'---2')) from "JORDERS"` | 2min40s    | hot                    |
| `select count(distinct concat("ORDERS",'---3')) from "JORDERS"` | 2min 42s   | hot                    |
| `select count(distinct concat("ORDERS",'---4')) from "JORDERS"` | 2min40s    | hot                    |
| `select count(distinct concat("ORDERS",'---5')) from "JORDERS"` | 2min 42s   | hot                    |
| `select count(distinct concat("ORDERS",'---6')) from "JORDERS"` | 2min 39s   | hot                    |

query 结果集大小：

| COUNT(DISTINCT CONCAT("ORDERS",'---X')) |
| --------------------------------------- |
| 150000000                               |

#### 测试4

表名：同测试3

表结构：同测试3

表行数：150，000，000

存储空间：50.6GB

测试结果：

vm配置x-large时

| 查询sql                                                      | 总执行时间 | 数据是否缓存在计算节点 |
| ------------------------------------------------------------ | ---------- | ---------------------- |
| `select count(distinct concat("ORDERS",'---1')) from "JORDERS"` | 2min33s    | cold                   |
| `select count(distinct concat("ORDERS",'---2')) from "JORDERS"` | 2min30s    | hot                    |
| `select count(distinct concat("ORDERS",'---3')) from "JORDERS"` | 2min37s    | hot                    |
| `select count(distinct concat("ORDERS",'---4')) from "JORDERS"` | 2min30s    | hot                    |
| `select count(distinct concat("ORDERS",'---5')) from "JORDERS"` | 2min30s    | hot                    |
| `select count(distinct concat("ORDERS",'---6')) from "JORDERS"` | 2min30s    | hot                    |

query 结果集大小：

| COUNT(DISTINCT CONCAT("ORDERS",'---X')) |
| --------------------------------------- |
| 1500000000                              |

vm配置x-small时，执行时间超长；

### 自备数据测试结果

数据上传的太慢了。4G数据上传了12小时，还不知什么时候结束。

## 结论

基于TPC-H数据的测试结论如下：

| vm大小  | 存储数据大小 | 表名     | 类SQL    | 执行时间cold | 执行时间hot（平均） | 传输时间 | 传输占比 |
| ------- | ------------ | -------- | -------- | ------------ | ------------------- | -------- | -------- |
| X-Large | 1GB          | CUSTOMER | {query1} | 5.15s        | 4.026s              | 1.124s   | 21.8%    |
| X-Large | 10.1GB       | CUSTOMER | {query1} | 21.27s       | 19.503s             | 1.767s   | 8.3%     |
| X-Large | 4.4GB        | JORDERS  | {query2} | 19.78s       | 18.62s              | 1.16s    | 5.9%     |
| X-Large | 50.6GB       | JORDERS  | {query2} | 2min15s      | 2min13.4s           | 1.6s     | 1.2%     |
| X-Small | 1GB          | CUSTOMER | {query1} | 27.75s       | 23.3s               | 4.45s    | 16%      |
| X-Smal  | 10.1GB       | CUSTOMER | {query1} | 4min22s      | 4min18.4s           | 3.6s     | 1.4%     |
| X-Smal  | 4.4GB        | JORDERS  | {query2} | 2min33s      | 2min30s             | 3s       | 2.0%     |

其中：

```sql
query1 = select avg("C_CUSTKEY" + x), avg("C_ACCTBAL" + x), avg("C_NATIONKEY" + x),"C_NAME", "C_ADDRESS", "C_PHONE", "C_MKTSEGMENT","C_COMMENT" from  "CUSTOMER" group by "C_NAME", "C_ADDRESS", "C_PHONE"`, `"C_MKTSEGMENT","C_COMMENT"
query2 = select count(distinct concat("ORDERS",'---x')) from "JORDERS"
```

*注：以上测试数据会丢掉抖动1.5倍以上的数据，且已扣除结果集传输时间。*

从测试结果可以看出，在vm的size为x-large情况下，无论需要传输的数据块的大小，其传输时间基本为固定的为1.3s左右。说明50GB的数据传输也未达到x-large所有work nodes的传输带宽上限。在vm的size为x-small的情况下，无论需要传输的数据大小(1GB-10GB)，传输时间基本固定在3-4s左右，其性能瓶颈主要在**vm的size配置**上。

vm size的配置对于整体查询性能影响巨大，基本上size每扩大或者缩小一倍，查询时间会有明显的减少或增加。

测试小存储容量的表数据（100M），几乎测不出传输的影响。其计算时间抖动的影响范围已经大于传输时间的影响。
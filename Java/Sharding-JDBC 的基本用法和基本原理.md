本文大纲如下

- [Sharding-JDBC 的基本用法和基本原理](#sharding-jdbc-----------)
- [前言](#--)
- [1. 我的出生和我的家族](#1----------)
- [2. 我统治的世界和我的职责](#2------------)
- [3. 召唤我的方式](#3-------)
- [4. 我的特性和我的工作方法](#4------------)
  * [4.2. 一些核心概念](#42-------)
    + [4.2.1. 逻辑表和物理表](#421--------)
    + [4.2.2. 分片键](#422----)
    + [4.2.3. 路由](#423---)
    + [4.2.4. 分片策略和分片算法](#424----------)
    + [4.2.5. 绑定表](#425----)
  * [4.3. 我处理 SQL 的过程](#43-----sql----)
    + [4.3.1. SQL 解析](#431-sql---)
    + [4.3.2. SQL 路由](#432-sql---)
    + [4.3.3. SQL 改写](#433-sql---)
    + [4.3.4. SQL 执行](#434-sql---)
    + [4.3.5. 结果归并](#435-----)
- [5. 结束语](#5----)


------

# 前言

这是一篇将“介绍 Sharding-JDBC 基本使用方法”作为目标的文章，但笔者却把大部分文字放在对 Sharding-JDBC 的工作原理的描述上，因为笔者认为原理是每个 IT 打工人学习技术的归途。

使用框架、中间件、数据库、工具包等公共组件来组装出应用系统是我们这一代 IT 打工人工作的常态。对于这些公共组件——比如框架——的学习，有些人的方法是这样的：避开复杂晦涩的框架原理，仅仅关注它的各种配置、API、注解，在尝试了这个框架的常用配置项、API、注解的效果之后，就妄称自己学会了这个框架。这种对技术的肤浅的认知既经不起实践的考验，也经不起面试官的考验，甚至连自己使用这些配置项、API、注解在干什么都没有明确的认知。

所以，打工人们，还是多学点原理，多看点源码，让优秀的设计思想、算法和编程风格冲击一下自己的大脑吧 :-)

因为 Sharding-JDBC 的设计细节实在太多，因此本文不可能对 Sharding-JDBC 进行面面俱到的讲解。笔者在本文中仅仅保留了对 Sharding-JDBC 的核心特性、核心原理的讲解，并尽量使用简单生动的文字进行表达，使读者阅读本文后对 Sharding-JDBC 的基本原理和使用有清晰的认知。为了使这些文字尽量摆脱枯燥的味道，文章采用了第一人称的讲述方式，让 Sharding-JDBC 现身说法，进行自我剖析，希望给大家一个更好的阅读体验。

但是，妄图不动脑子就能对某项技术产生深度认知是绝不可能的，你思考得越多，你得到的越多。这就印证了那句话：“我变秃了，也变强了。”

# 1. 我的出生和我的家族

我是 Sharding-JDBC，一个关系型数据库中间件，我的全名是 Apache ShardingSphere JDBC，我被冠以 Apache 这个贵族姓氏是 2020 年 4 月的事情，这意味着我进入了代码世界的“体制内”。但我还是喜欢别人称呼我的小名，Sharding-JDBC。

我的创造者在我诞生之后给我讲了我的身世：

> **“**
>
> 你的诞生是一个必然的结果。
>
> 在你诞生之前，传统软件的存储层架构将所有的业务数据存储到单一数据库节点，在性能、可用性和运维成本这三方面已经难于满足互联网的海量数据场景。
>
> 从性能方面来说，由于关系型数据库大多采用 B+树类型的索引，在数据量逐渐增大的情况下，索引深度的增加也将使得磁盘访问的 IO 次数增加，进而导致查询性能的下降；同时，高并发访问请求也使得集中式数据库成为系统的最大瓶颈。
>
> 从可用性的方面来讲，应用服务器节点能够随意水平拓展（水平拓展就是增加应用服务器节点数量）以应对不断增加的业务流量，这必然导致系统的最终压力都落在数据库之上。而单一的数据库节点，或者简单的主从架构，已经越来越难以承担众多应用服务器节点的数据查询请求。数据库的可用性，已成为整个系统的关键。
>
> 从运维成本方面考虑，随着数据库实例中的数据规模的增大，DBA 的运维压力也会增加，因为数据备份和恢复的时间成本都将随着数据量的增大而愈发不可控。
>
> 这样看来关系型数据库似乎难以承担海量记录的存储。
>
> 然而，关系型数据库当今依然占有巨大市场，是各个公司核心业务的基石。在传统的关系型数据库无法满足互联网场景需要的情况下，将数据存储到原生支持分布式的 NoSQL 的尝试越来越多。但 NoSQL 对 SQL 的不兼容性以及生态圈的不完善，使得它们在与关系型数据库的博弈中处于劣势，关系型数据库的地位却依然不可撼动，未来也难于撼动。
>
> 我们目前阶段更加关注在原有关系型数据库的基础上做增量，使之更好适应海量数据存储和高并发查询请求的场景，而不是要颠覆关系型数据库。
>
> 分库分表方案就是这种增量，它的诞生解决了海量数据存储和高并发查询请求的问题。
>
> 但是，单一数据库被分库分表之后，繁杂的库和表使得编写持久层代码的工程师的思维负担翻了很多倍，他们需要考虑一个业务 SQL 应该去哪个库的哪个表里去查询，查询到的结果还要进行聚合，如果遇到多表关联查询、排序、分页、事务等等问题，那简直是一个噩梦。
>
> 于是我们创造了你。你可以让工程师们以像查询单数据库实例和单表那样来查询被水平分割的库和表，我们称之为透明查询。
>
> 你是水平分片世界的神。
>
> **”**

这使我感到骄傲。

我被定位为一个轻量级 Java 框架，我在 Java 的 JDBC 层提供的额外服务，可以说是一个增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

- 我适用于任何基于 JDBC 的 ORM 框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template 或直接使用 JDBC。

- 我支持任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid, HikariCP 等。

- 我支持任意实现 JDBC 规范的数据库，目前支持 MySQL，Oracle，SQLServer，PostgreSQL 以及任何遵循 SQL92 标准的数据库。

我的创造者起初只创造了我一个独苗，后来为了我的家族的兴盛，我的两个兄弟——Apache ShardingSphere Proxy、Apache ShardingSphere Sidecar 又被创造了出来。前者被定位为透明化的数据库代理端，提供封装了数据库二进制协议的服务端版本，⽤于完成对异构语⾔的支持；后者被定位为 Kubernetes 的云原⽣数据库代理，以 Sidecar 的形式代理所有对数据库的访问。通过无中心、零侵⼊的⽅案提供与数据库交互的的啮合层，即 Database Mesh，又可称数据库⽹格。

因此，我们这个家族叫做 Apache ShardingSphere，旨在在分布式的场景下更好利用关系型数据库的计算和存储能力，而并非实现一个全新的关系型数据库。 我们三个既相互独立，又能配合使用，均提供标准化的数据分片、分布式事务和数据库治理功能。

# 2. 我统治的世界和我的职责

我是 Sharding-JDBC，我生活在一个数据水平分片的世界，我统治着这个世界里被水平拆分后的数据库和表。

在分片的世界里，数据分片有两种法则：垂直拆分和水平拆分。

按照业务拆分的方式称为垂直分片，又称为纵向拆分，它的核心理念是专库专用。在拆分之前，一个数据库由多个数据表构成，每个表对应着不同的业务。而拆分之后，则是按照业务将表进行归类，分布到不同的数据库中，从而将压力分散至不同的数据库。下图展示了根据业务需要，将用户表和订单表垂直分片到不同的数据库的方案。

![031](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cfd6ed215de6463dab3b209b85b85063~tplv-k3u1fbpfcp-zoom-1.image)

垂直分片往往需要对架构和设计进行调整。通常来讲，是来不及应对互联网业务需求快速变化的；而且，它也并无法真正的解决单点瓶颈。如果垂直拆分之后，表中的数据量依然超过单节点所能承载的阈值，则需要水平分片来进一步处理。

水平分片又称为横向拆分。相对于垂直分片，它不再将数据根据业务逻辑分类，而是通过某个字段（或某几个字段），根据某种规则将数据分散至多个库或表中，每个分片仅包含数据的一部分。例如：根据主键分片，偶数主键的记录放入 0 库（或表），奇数主键的记录放入 1 库（或表），如下图所示。

![032](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b9214d28cce40d5b3d250910a3b6930~tplv-k3u1fbpfcp-zoom-1.image)

水平分片从理论上突破了单机数据量处理的瓶颈，并且扩展相对自由，是分库分表的标准解决方案。我管辖的就是水平分片世界。

通过分库和分表进行数据的拆分来使得各个表的数据量保持在阈值以下，是应对高并发和海量数据系统的有效手段。此外，使用多主多从的分片方式，可以有效的避免数据单点，从而提升数据架构的可用性。

其实，水平分库本质上还是在分表，因为被水平拆分后的库中，都有相同的表分片。

分库和分表这项工作并不是我来做，我虽然是神，但我还没有神到能理解你们这些工程师的业务设计和架构设计，从而自动把你们的业务数据库和业务表进行分片。对哪部分进行分片、怎样分片、分多少份，这些工作全部由这些工程师进行。当这些分库分表的工作被完成后，你们只需要在我的配置文件中或者通过我的 API 告诉我这些拆分规则（这就是后文要提到的分片策略）即可，剩下的事情，交给我去做。

我是 Sharding-JDBC，我的职责是尽量透明化水平分库分表所带来的影响，让使用方尽量像使用一个数据库一样使用水平分片之后的数据库集群，或者像使用一个数据表一样使用水平分片之后的数据表。由于我的治理，每个服务器节点只能看到一个逻辑上的数据库节点，和其中的多个逻辑表，它们看不到真正存在于物理世界中的被水平分割的多个数据库分片和被水平分割的多个数据表分片。服务器节点看到的简单的持久层结构，其实是我苦心营造的幻象。

![033](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49f826ff593544f4956a2deb0e0a8604~tplv-k3u1fbpfcp-zoom-1.image)

而为了营造这种幻象，我在幕后付出了很多。

当一个 Java 应用服务器节点将一个查询 SQL 交给我之后，我要做下面几件事：

1）SQL 解析：解析分为词法解析和语法解析。我先通过词法解析器将这句 SQL 拆分为一个个不可再分的单词，再使用语法解析器对 SQL 进行理解，并最终提炼出解析上下文。简单来说就是我要理解这句 SQL，明白它的构造和行为，这是下面的优化、路由、改写、执行和归并的基础。

2）SQL 路由：我根据解析上下文匹配用户对这句 SQL 所涉及的库和表配置的分片策略（关于用户配置的分片策略，我后文会慢慢解释），并根据分片策略生成路由后的 SQL。路由后的 SQL 有一条或多条，每一条都对应着各自的真实物理分片。

3）SQL 改写：我将 SQL 改写为在真实数据库中可以正确执行的语句（逻辑 SQL 到物理 SQL 的映射，例如把逻辑表名改成带编号的分片表名）。

4）SQL 执行：我通过多线程执行器异步执行路由和改写之后得到的 SQL 语句。

5）结果归并：我将多个执行结果集归并以便于通过统一的 JDBC 接口输出。

![034](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef17248745aa4d418888d8959b09f4d2~tplv-k3u1fbpfcp-zoom-1.image)

如果你连读这段工作流程都很困难，那你就能明白我在这个水平分片的世界里有多辛苦。关于这段工作流程，我会在后文慢慢说给你听。

# 3. 召唤我的方式

我是 Sharding-JDBC，我被定位为一个轻量级数据库中间件，当你们召唤我去统治水平拆分后的数据库和数据表时，只需要做下面几件事：

1）引入依赖包。

maven 是统治依赖包世界的神，在他诞生之后，一切对 jar 包的引用就变得简单了。向 maven 获取我的 jar 包，咒语是：

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core</artifactId>
    <version>${latest.release.version}</version>
</dependency>
```

于是，我就出现在了这个项目中！

如果你们构建的项目已经被 Springboot 统治了（Springboot 是 Spring 的继任者，Spring 是统治对象世界的神，Springboot 继承了 Spring 的统治法则，并简化了 Spring 的配置），那么就可以向 maven 获取我的 springboot starter jar 包，咒语是：

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-spring-boot-starter</artifactId>
    <version>${shardingsphere.version}</version>
</dependency>
```

这样，我就能和 Springboot 神共存于同一个项目。

2）进行水平分片规则配置。

你们要把水平分片规则配置告诉我，这样我才能知道你们是怎样水平拆分数据库和数据表的。你们可以通过我提供的 Java API，或者配置文件告诉我分片规则。

如果是以 Java API 的方式进行配置，示例如下：

```java
// 配置真实数据源
Map<String, DataSource> dataSourceMap = new HashMap<>();
// 配置第 1 个数据源
BasicDataSource dataSource1 = new BasicDataSource();
dataSource1.setDriverClassName("com.mysql.jdbc.Driver");
dataSource1.setUrl("jdbc:mysql://localhost:3306/ds0");
dataSource1.setUsername("root");
dataSource1.setPassword("");
dataSourceMap.put("ds0", dataSource1);
// 配置第 2 个数据源
BasicDataSource dataSource2 = new BasicDataSource();
dataSource2.setDriverClassName("com.mysql.jdbc.Driver");
dataSource2.setUrl("jdbc:mysql://localhost:3306/ds1");
dataSource2.setUsername("root");
dataSource2.setPassword("");
dataSourceMap.put("ds1", dataSource2);
// 配置 t_order 表规则
ShardingTableRuleConfiguration orderTableRuleConfig 
    = new ShardingTableRuleConfiguration(
    "t_order", 
    "ds${0..1}.t_order${0..1}"
);
// 配置 t_order 被拆分到多个子库的策略
orderTableRuleConfig.setDatabaseShardingStrategy(
    new StandardShardingStrategyConfiguration(
        "user_id", 
        "dbShardingAlgorithm"
    )
);
// 配置 t_order 被拆分到多个子表的策略
orderTableRuleConfig.setTableShardingStrategy(
    new StandardShardingStrategyConfiguration(
        "order_id", 
        "tableShardingAlgorithm"
    )
);
// 省略配置 t_order_item 表规则...
// ...
// 配置分片规则
ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
shardingRuleConfig.getTables().add(orderTableRuleConfig);
// 配置 t_order 被拆分到多个子库的算法
Properties dbShardingAlgorithmrProps = new Properties();
dbShardingAlgorithmrProps.setProperty(
    "algorithm-expression", 
    "ds${user_id % 2}"
);
shardingRuleConfig.getShardingAlgorithms().put(
    "dbShardingAlgorithm", 
    new ShardingSphereAlgorithmConfiguration("INLINE", dbShardingAlgorithmrProps)
);
// 配置 t_order 被拆分到多个子表的算法
Properties tableShardingAlgorithmrProps = new Properties();
tableShardingAlgorithmrProps.setProperty(
    "algorithm-expression", 
    "t_order${order_id % 2}"
);
shardingRuleConfig.getShardingAlgorithms().put(
    "tableShardingAlgorithm", 
    new ShardingSphereAlgorithmConfiguration("INLINE", tableShardingAlgorithmrProps)
);

```

这段配置代码中涉及的 t_order 表（存储订单的基本信息）的表结构为：

| order_id | user_id | create_time | remarks | total_price |
| -------- | ------- | ----------- | ------- | ----------- |
|          |         |             |         |             |

t_order_item 表（存储订单的商品和价格明细信息）的结构为：

| order_id | production_code | count | price | discount |
| -------- | --------------- | ----- | ----- | -------- |
|          |                 |       |       |          |

这段配置代码描述了对 t_order 表进行的如下图所示的数据表水平分片（对 t_order_item 表也要进行类似的水平分片，但是这部分配置省略了）：

![035](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a77209c81134c1e82f6e4c4f2fc392c~tplv-k3u1fbpfcp-zoom-1.image)

在这段配置中，或许你们注意到了一些奇怪的表达式：

```groovy
ds$->{0..1}.t_order$->{0..1}
ds_${user_id % 2}
t_order_${order_id % 2}
```

这些表达式被称为 Groovy 表达式，它们的含义很容易识别：

1）对 t_order 进行两种维度的拆分：数据库维度和表维度数；

2）在数据库维度，t_order.user_id % 2 == 0 的记录全部落到 ds0，t_order.user_id % 2 == 1 的记录全部落到 ds1；（有人称这一过程为水平分库，其实它的本质还是在水平地分表，只不过依据表中 user_id 的不同把拆分的后的表放入两个数据库实例。）

3）在表维度，t_order.order_id% 2 == 0 的记录全部落到 t_order0，t_order.order_id% 2 == 1 的记录全部落到 t_order1。

4）对记录的读和写都按照这种方向进行，“方向”，就是分片方式，就是路由。

我允许你们这些工程师使用这种简洁的 Groovy 表达式告诉我你们设置的分片策略和分片算法。但是这种方式所能表达的含义是有限的。因此，我提供了分片策略接口和分片算法接口让你们利用 Java 代码尽情表达更为复杂的分片策略和分片算法。关于这一点，我将在《我的特性和工作方法》这一章详述。

而且在这里我要先告诉你，分片算法是分片策略的组成部分，分片策略设置=分片键设置+分片算法设置。上述配置里使用的策略是 Inline 类型的分片策略，使用的算法是 Inline 类型的行表达式算法，你或许不清楚我现在讲的这些术语，不要着急，我会在《我的特性和工作方法》这一章详述。

如果是以配置文件的方式进行配置，示例如下（这里以我的 springboot starter 包的 properties 配置文件为例）：

```properties
# 配置真实数据源
spring.shardingsphere.datasource.names=ds0,ds1
# 配置第 1 个数据源
spring.shardingsphere.datasource.ds0.type=org.apache.commons.dbcp2.BasicDataSource
spring.shardingsphere.datasource.ds0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds0.url=jdbc:mysql://localhost:3306/ds0
spring.shardingsphere.datasource.ds0.username=root
spring.shardingsphere.datasource.ds0.password=
# 配置第 2 个数据源
spring.shardingsphere.datasource.ds1.type=org.apache.commons.dbcp2.BasicDataSource
spring.shardingsphere.datasource.ds1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds1.url=jdbc:mysql://localhost:3306/ds1
spring.shardingsphere.datasource.ds1.username=root
spring.shardingsphere.datasource.ds1.password=
# 配置 t_order 表规则
spring.shardingsphere.rules.sharding.tables.t_order.actual-data-nodes=ds$->{0..1}.t_order$->{0..1}
# 配置 t_order 被拆分到多个子库的策略
spring.shardingsphere.rules.sharding.tables.t_order.database-strategy.standard.sharding-column=user_id
spring.shardingsphere.rules.sharding.tables.t_order.database-strategy.standard.sharding-algorithm-name=database_inline
# 配置 t_order 被拆分到多个子表的策略
spring.shardingsphere.rules.sharding.tables.t_order.table-strategy.standard.sharding-column=order_id
spring.shardingsphere.rules.sharding.tables.t_order.table-strategy.standard.sharding-algorithm-name=table_inline
# 省略配置 t_order_item 表规则...
# ...
# 配置 t_order 被拆分到多个子库的算法
spring.shardingsphere.rules.sharding.sharding-algorithms.database_inline.type=INLINE
spring.shardingsphere.rules.sharding.sharding-algorithms.database_inline.props.algorithm-expression=ds_${user_id % 2}
# 配置 t_order 被拆分到多个子表的算法
spring.shardingsphere.rules.sharding.sharding-algorithms.table_inline.type=INLINE
spring.shardingsphere.rules.sharding.sharding-algorithms.table_inline.props.algorithm-expression=t_order_${order_id % 2}
```

这段配置文件的语义和上面的 Java 配置代码同义。

3）创建数据源。

若使用上文所示的 Java API 进行配置，则可以通过 ShardingSphereDataSourceFactory 工厂创建数据源，该工厂产生一个 ShardingSphereDataSource 实例，ShardingSphereDataSource 实现自 JDBC 的标准接口 DataSource（所以 ShardingSphereDataSource 实例也是接口 DataSource 的实例）。之后，就可以通过 dataSource 调用原生 JDBC 接口来执行 SQL 查询，或者将 dataSource 配置到 JPA，MyBatis 等 ORM 框架来执行 SQL 查询。

```java
// 创建 ShardingSphereDataSource
DataSource dataSource = ShardingSphereDataSourceFactory.createDataSource(
    dataSourceMap, 
    Collections.singleton(shardingRuleConfig, new Properties())
);
```

若使用上文所示的基于 springboot starter 的 properties 配置文件进行分片配置，则可以直接通过 Spring 提供的自动注入的方式获得数据源实例 dataSource（同样，这也是一个 ShardingSphereDataSource 实例）。之后，就可以通过 dataSource 调用原生 JDBC 接口来执行 SQL 查询，或者将 dataSource 配置到 JPA，MyBatis 等 ORM 框架来执行 SQL 查询。

```java
/**
* 注入一个 ShardingSphereDataSource 实例
*/
@Resource
private DataSource dataSource;
```

有了 dataSource（以上两种方式产生的 dataSource 没有区别，都是 ShardingSphereDataSource 的一个实例，业务代码将 SQL 交给这个 dataSource，也就是交到了我的手中），就可以执行 SQL 查询了。

4）执行 SQL。这里给出 dataSource 调用原生 JDBC 接口来执行 SQL 查询的示例：

```java
String sql = "SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.user_id=? AND o.order_id=?";
try (
    Connection conn = dataSource.getConnection();
    PreparedStatement ps = conn.prepareStatement(sql)
) {
    ps.setInt(1, 10);
    ps.setInt(2, 1000);
    try (
        ResultSet rs = preparedStatement.executeQuery()
    ) {
        while(rs.next()) {
            // ...
        }
    }
}
```

在这个示例中，Java 代码调用 dataSource 的 JDBC 接口时，只感觉自己在对一个逻辑库中的两个逻辑表进行关联查询，并没有意识到物理分片的存在。而背后是我在进行 SQL 语句的解析、路由、改写、执行和结果归并！

# 4. 我的特性和我的工作方法

## 4.2. 一些核心概念

我是 Sharding-JDBC，我是统治水平分片世界的神，我要向你们解释我的特性和治理方法。在此之前，我要给出一系列用于描述我的术语。

### 4.2.1. 逻辑表和物理表

例如，订单表根据主键尾数被水平拆分为 10 张表，分别是 t_order0 到 t_order9，它们的逻辑表名为 t_order，而 t_order0 到 t_order9 就是物理表。

### 4.2.2. 分片键

例如，若根据订单表中的订单主键的尾数取模结果进行水平分片，则订单主键为分片键。订单表既可以根据单个分片键进行分片，也同样可以根据多个分片键（例如 order_id 和 user_id）进行分片。

### 4.2.3. 路由

应用程序服务器将针对逻辑表编写的 SQL 交给我，我在执行前，要找到 SQL 语句里包含的查询条件（where ......）所对应的分片（物理表），然后再针对这些分片进行查询，这个找分片的过程叫做路由。

而怎样找分片，是由你们在分片策略中告诉我的。

### 4.2.4. 分片策略和分片算法

在上文的配置示例中，有如下的一段：

```java
......
// 配置 t_order 被拆分到多个子库的策略
orderTableRuleConfig.setDatabaseShardingStrategy(
    new StandardShardingStrategyConfiguration(
        "user_id", 
        "dbShardingAlgorithm"
    )
);
// 配置 t_order 被拆分到多个子表的策略
orderTableRuleConfig.setTableShardingStrategy(
    new StandardShardingStrategyConfiguration(
        "order_id", 
        "tableShardingAlgorithm"
    )
);
......
ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
shardingRuleConfig.getTables().add(orderTableRuleConfig);
// 配置 t_order 被拆分到多个子库的算法
Properties dbShardingAlgorithmrProps = new Properties();
dbShardingAlgorithmrProps.setProperty(
    "algorithm-expression", 
    "ds${user_id % 2}"
);
shardingRuleConfig.getShardingAlgorithms().put(
    "dbShardingAlgorithm", 
    new ShardingSphereAlgorithmConfiguration("INLINE", dbShardingAlgorithmrProps)
);
// 配置 t_order 被拆分到多个子表的算法
Properties tableShardingAlgorithmrProps = new Properties();
tableShardingAlgorithmrProps.setProperty(
    "algorithm-expression", 
    "t_order${order_id % 2}"
);
shardingRuleConfig.getShardingAlgorithms().put(
    "tableShardingAlgorithm", 
    new ShardingSphereAlgorithmConfiguration("INLINE", tableShardingAlgorithmrProps)
);
......
```

它们表达的就是对 t_order 表进行的分片策略和分片算法的配置。

上文说到，我允许你们这些工程师使用简洁的 Groovy 表达式告诉我你们设置的分片策略和分片算法。但是这种方式所能表达的含义是有限的。因此，我提供了分片策略接口和分片算法接口让你们利用灵活的 Java 代码尽情表达更为复杂的分片策略和分片算法。

所谓分片策略，就是分片键和分片算法的组合，由于分片算法的独立性，我将其独立抽离出来，由你们自己实现，也就是告诉我数据是怎么根据分片键的值找到对应的分片，进而对这些分片执行 SQL 查询。

当然我也提供了一些内置的简单算法的实现。上面提到的基于 Groovy 表达式的分片算法就是我内置的一种算法实现，你们只要给我一段语义准确无误的 Groovy 表达式，我就能知道怎么根据分片键的值找到对应的分片。

我的分片策略有两个维度，如下图所示，分别是数据源分片策略（databaseShardingStrategy）和表分片策略（tableShardingStrategy）。数据源分片策略表示数据被路由到目标物理数据库的策略，表分片策略表示数据被路由到目标物理表的策略。表分片策略是依赖于数据源分片策略的，也就是说要先分库再分表，当然也可以只分表。

![036](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/320780a972314e60aeb2944864b050a8~tplv-k3u1fbpfcp-zoom-1.image)

我目前可以提供如下几种分片（无论是对库分片还是对表分片）策略：标准分片策略（使用精确分片算法或者范围分片算法）、复合分片策略（使用符合分片算法）、Hint 分片策略（使用 Hint 分片算法）、Inline 分片策略（使用 Grovvy 表达式作为分片算法）、不分片策略（不使用分片算法）。

我的 Jar 包源码里的策略类和算法接口如下：

![037](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f96374813ab402880f339c9454c5200~tplv-k3u1fbpfcp-zoom-1.image)

![038](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f399995451124a78bfacb39c01ff1808~tplv-k3u1fbpfcp-zoom-1.image)

**一、标准分片策略**

标准分片策略 StandardShardingStrategy 的源代码（部分）如下，这是一个 final class。

```java
package org.apache.shardingsphere.core.strategy.route.standard;

......

public final class StandardShardingStrategy implements ShardingStrategy {
    
    private final String shardingColumn;
    
    /**
    * 要配合 PreciseShardingAlgorithm 或 RangeShardingAlgorithm 使用
    * 标准分片策略
    */
    private final PreciseShardingAlgorithm preciseShardingAlgorithm;
    
    private final RangeShardingAlgorithm rangeShardingAlgorithm;
    
    public StandardShardingStrategy(
        // 传入分片配置
        final StandardShardingStrategyConfiguration standardShardingStrategyConfig
    ) {
        ......
        
        // 从配置中提取分片键
        shardingColumn = standardShardingStrategyConfig.getShardingColumn();
        // 从配置中提取分片算法
        preciseShardingAlgorithm = standardShardingStrategyConfig.getPreciseShardingAlgorithm();
        rangeShardingAlgorithm = standardShardingStrategyConfig.getRangeShardingAlgorithm();
    }
    
    @Override
    public Collection<String> doSharding(
        // 所有可能的分片表(或分片库)名称
        final Collection<String> availableTargetNames, 
        // 分片键的值
        final Collection<RouteValue> shardingValues, 
        final ConfigurationProperties properties
    ) {
        RouteValue shardingValue = shardingValues.iterator().next();
        Collection<String> shardingResult 
            = shardingValue instanceof ListRouteValue
                // 处理精确分片
                ? doSharding(availableTargetNames, (ListRouteValue) shardingValue) 
                // 处理范围分片
                : doSharding(availableTargetNames, (RangeRouteValue) shardingValue);
        Collection<String> result = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
        result.addAll(shardingResult);
        
        // 根据分片键的值，找到对应的分片表(或分片库)名称并返回
        return result;
    }
    
    /**
    * 处理范围分片
    */
    @SuppressWarnings("unchecked")
    private Collection<String> doSharding(
        // 所有可能的分片表(或分片库)名称
        final Collection<String> availableTargetNames, 
        // 分片键的值
        final RangeRouteValue<?> shardingValue
    ) {
        ......
        // 调用 rangeShardingAlgorithm.doSharding()根据分片键的值找到对应的
        // 分片表(或分片库)名称并返回，rangeShardingAlgorithm.doSharding()
        // 由你们自己实现
        return rangeShardingAlgorithm.doSharding(
            availableTargetNames, 
            new RangeShardingValue(
                shardingValue.getTableName(), 
                shardingValue.getColumnName(), 
                shardingValue.getValueRange()
            )
        );
    }
    
    /**
    * 处理精确分片
    */
    @SuppressWarnings("unchecked")
    private Collection<String> doSharding(
        // 所有可能的分片表(或分片库)名称
        final Collection<String> availableTargetNames, 
        // 分片键的值
        final ListRouteValue<?> shardingValue
    ) {
        Collection<String> result = new LinkedList<>();
        for (Comparable<?> each : shardingValue.getValues()) {
            // 调用 preciseShardingAlgorithm.doSharding()根据分片键的值找到对应的
            // 分片表(或分片库)名称并返回，preciseShardingAlgorithm.doSharding()
            // 由你们自己实现
            String target 
                = preciseShardingAlgorithm.doSharding(
                availableTargetNames, 
                new PreciseShardingValue(
                    shardingValue.getTableName(), 
                    shardingValue.getColumnName(), 
                    each
                )
            );
            if (null != target) {
                result.add(target);
            }
        }
        return result;
    }
    
    /**
    * 获取所有的分片键
    */
    @Override
    public Collection<String> getShardingColumns() {
        Collection<String> result = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
        result.add(shardingColumn);
        return result;
    }
}
```

其中 PreciseShardingAlgorithm（接口）和 RangeShardingAlgorithm（接口）的源代码分别为：

```java
package org.apache.shardingsphere.api.sharding.standard;

......

public interface PreciseShardingAlgorithm<T extends Comparable<?>> 
    extends ShardingAlgorithm {
    
    /**
     * @param 所有可能的分片表(或分片库)名称
     * @param 分片键的值
     * @return 根据分片键的值，找到对应的分片表(或分片库)名称并返回
     */
    String doSharding(
        Collection<String> availableTargetNames, 
        PreciseShardingValue<T> shardingValue
    );
}
```

```java
package org.apache.shardingsphere.api.sharding.standard;

......

public interface RangeShardingAlgorithm<T extends Comparable<?>> 
    extends ShardingAlgorithm {
    
    /**
     * @param 所有可能的分片表(或分片库)名称
     * @param 分片键的值
     * @return 根据分片键的值，找到对应的分片表(或分片库)名称并返回
     */
    Collection<String> doSharding(
        Collection<String> availableTargetNames, 
        RangeShardingValue<T> shardingValue
    );
}

```

标准分片策略提供对 SQL 语句中的操作符 =、>、 <、>=、<=、IN 和 BETWEEN AND 的分片操支持。

标准分片策略只支持单分片键，例如对 t_order 表只根据 order_id 分片。标准分片策略提供 PreciseShardingAlgorithm（接口）和 RangeShardingAlgorithm（接口）两个分片算法。PreciseShardingAlgorithm（接口）顾名思义用于处理操作符=和 IN 的精确分片。RangeShardingAlgorithm （接口）顾名思义用于处理操作符 BETWEEN AND、>、<、>=、<= 的范围分片。

我举个例子帮助你理解以上两段话的含义。以 t_order 为例，假如你使用 order_id 作为 t_order 的分片键，并设计了以下的分片策略：

```
策略一：设置 6 个分片
t_order.order_id % 6 == 0 的查询分片到 t_order0
t_order.order_id % 6 == 1 的查询分片到 t_order1
t_order.order_id % 6 == 2 的查询分片到 t_order2
t_order.order_id % 6 == 3 的查询分片到 t_order3
t_order.order_id % 6 == 4 的查询分片到 t_order4
t_order.order_id % 6 == 5 的查询分片到 t_order5

策略二：设置 2 个分片
t_order.order_id % 6 in (0,2,4) 的查询分片到 t_order1
t_order.order_id % 6 in (1,3,5) 的查询分片到 t_order1

策略三：经过估算订单不超过 60000 个，设置 6 个分片
t_order.order_id between 0 and 10000 的查询分片到 t_order0
t_order.order_id between 10000 and 20000 的查询分片到 t_order1
t_order.order_id between 20000 and 30000 的查询分片到 t_order2
t_order.order_id between 30000 and 40000 的查询分片到 t_order3
t_order.order_id between 40000 and 50000 的查询分片到 t_order4
t_order.order_id between 50000 and 60000 的查询分片到 t_order5

策略四：经过估算订单不超过 20000 个，设置 2 个分片
t_order.order_id <=10000 的查询分片到 t_order0
t_order.order_id >10000 的查询分片到 t_order1

......
```

那你就可以把以下三项：

1）分片键 order_id

2）描述以上分片策略内容的 PreciseShardingAlgorithm（接口）的实现类或 RangeShardingAlgorithm（接口）的实现类

3）前两项（即分片策略）的作用目标 t_order 表

写到分片配置里（无论是通过配置 API 还是通过配置文件），那我就能知道如何去路由 SQL，即根据分片键的值，找到对应的分片表(或分片库)。

有了这些配置，我就能帮你们透明处理如下 SQL 语句，不管实际的物理分片是怎样的：

```sql
-- 注：使用 t_order.order_id 作为 t_order 表的分片键
SELECT o.* FROM t_order o WHERE o.order_id = 10;
SELECT o.* FROM t_order o WHERE o.order_id IN (10, 11);
SELECT o.* FROM t_order o WHERE o.order_id > 10;
SELECT o.* FROM t_order o WHERE o.order_id <= 11;
SELECT o.* FROM t_order o WHERE o.order_id BETWEEN 10 AND 12;
......
INSERT INTO t_order(order_id, user_id) VALUES (20, 1001);
......
DELETE FROM t_order o WHERE o.order_id = 10;
DELETE FROM t_order o WHERE o.order_id IN (10, 11);
DELETE FROM t_order o WHERE o.order_id > 10;
DELETE FROM t_order o WHERE o.order_id <= 11;
DELETE FROM t_order o WHERE o.order_id BETWEEN 10 AND 12;
......
UPDATE t_order o SET o.update_time = NOW() WHERE o.order_id = 10;
......
```

**二、复合分片策略**

复合分片策略 ComplexShardingStrategy 的源代码（部分）如下，这是一个 final class。

```java
package org.apache.shardingsphere.core.strategy.route.complex;

......

public final class ComplexShardingStrategy implements ShardingStrategy {
    
    @Getter
    private final Collection<String> shardingColumns;
    
    /**
    * 要配合 ComplexKeysShardingAlgorithm 使用复合分片策略
    */
    private final ComplexKeysShardingAlgorithm shardingAlgorithm;
    
    public ComplexShardingStrategy(
        // 传入分片配置
        final ComplexShardingStrategyConfiguration complexShardingStrategyConfig
    ) {
        ......
        // 从配置中提取分片键
        shardingColumns = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
        shardingColumns.addAll(
            Splitter
            .on(",")
            .trimResults()
            .splitToList(complexShardingStrategyConfig.getShardingColumns())
        );
        // 从配置中提取分片算法
        shardingAlgorithm = complexShardingStrategyConfig.getShardingAlgorithm();
    }
    
    @SuppressWarnings("unchecked")
    @Override
    public Collection<String> doSharding(
        // 所有可能的分片表(或分片库)名称
        final Collection<String> availableTargetNames, 
        // 分片键的值
        final Collection<RouteValue> shardingValues, 
        final ConfigurationProperties properties
    ) {
        Map<String, Collection<Comparable<?>>> columnShardingValues 
            = new HashMap<>(shardingValues.size(), 1);
        Map<String, Range<Comparable<?>>> columnRangeValues 
            = new HashMap<>(shardingValues.size(), 1);
        String logicTableName = "";
        
        // 提取多个分片键的值
        for (RouteValue each : shardingValues) {
            if (each instanceof ListRouteValue) {
                columnShardingValues.put(
                    each.getColumnName(), 
                    ((ListRouteValue) each).getValues()
                );
            } else if (each instanceof RangeRouteValue) {
                columnRangeValues.put(
                    each.getColumnName(), 
                    ((RangeRouteValue) each).getValueRange()
                );
            }
            logicTableName = each.getTableName();
        }
        Collection<String> shardingResult 
            // 调用 shardingAlgorithm.doSharding()根据分片键的值找到对应的
            // 分片表(或分片库)名称并返回，shardingAlgorithm.doSharding()
            // 由你们自己实现
            = shardingAlgorithm.doSharding(
            availableTargetNames, 
            new ComplexKeysShardingValue(
                logicTableName, 
                columnShardingValues, 
                columnRangeValues)
        );
        Collection<String> result = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
        result.addAll(shardingResult);
        
        // 根据分片键的值，找到对应的分片表(或分片库)名称并返回
        return result;
    }
}
```

其中 ComplexKeysShardingAlgorithm（接口）的源代码为：

```java
package org.apache.shardingsphere.api.sharding.complex;

......

public interface ComplexKeysShardingAlgorithm<T extends Comparable<?>> 
    extends ShardingAlgorithm {
    
    /**
     * @param 所有可能的分片表(或分片库)名称
     * @param 分片键的值
     * @return 根据分片键的值，找到对应的分片表(或分片库)名称并返回
     */
    Collection<String> doSharding(
        Collection<String> availableTargetNames, 
        ComplexKeysShardingValue<T> shardingValue
    );
}
```

复合分片策略提供对 SQL 语句中的操作符 =、>、<、>=、<=、IN 和 ETWEEN AND 的分片操作支持。

复合分片策略支持多分片键，例如对 t_order 表根据 order_id 和 user_id 分片。复合分片策略提供 ComplexKeysShardingAlgorithm（接口）分片算法。

我举个例子帮助你理解以上两段话的含义。以 t_order 为例，假如你使用 order_id 和 user_id 作为 t_order 的分片键，并设计了以下的分片策略：

```
策略一：设置 4 个分片
t_order.order_id % 2 == 0 && t_order.user_id % 2 == 0 的查询分片到 t_order0
t_order.order_id % 2 == 0 && t_order.user_id % 2 == 1 的查询分片到 t_order1
t_order.order_id % 2 == 1 && t_order.user_id % 2 == 0 的查询分片到 t_order2
t_order.order_id % 2 == 1 && t_order.user_id % 2 == 1 的查询分片到 t_order3

策略二：经过估算订单不超过 60000 个、用户不超过 1000 个，设置 4 个分片
t_order.order_id between 0 and 40000 && t_order.user_id between 0 and 500 的查询分片到 t_order0
t_order.order_id between 0 and 40000 && t_order.user_id between 500 and 1000 的查询分片到 t_order1
t_order.order_id between 40000 and 60000 && t_order.user_id between 0 and 500 的查询分片到 t_order2
t_order.order_id between 40000 and 60000 && t_order.user_id between 500 and 1000 的查询分片到 t_order3

......
```

那你就可以把以下三项：

1）分片键 order_id 和 user_id

2）描述以上分片策略内容的 ComplexKeysShardingAlgorithm（接口）的实现类

3）前两项（即分片策略）的作用目标 t_order 表

写到分片配置里（无论是通过配置 API 还是通过配置文件），那我就能知道如何去路由 SQL，即根据分片键的值，找到对应的分片表(或分片库)。

有了这些配置，我就能帮你们透明处理如下 SQL 语句，不管实际的物理分片是怎样的：

```sql
-- 注：使用 t_order.order_id、t_order.user_id 作为 t_order 表的分片键
SELECT o.* FROM t_order o WHERE o.order_id = 10;
SELECT o.* FROM t_order o WHERE o.order_id IN (10, 11);
SELECT o.* FROM t_order o WHERE o.order_id > 10;
SELECT o.* FROM t_order o WHERE o.order_id <= 11;
SELECT o.* FROM t_order o WHERE o.order_id BETWEEN 10 AND 12;
......
INSERT INTO t_order(order_id, user_id) VALUES (20, 1001);
......
DELETE FROM t_order o WHERE o.order_id = 10;
DELETE FROM t_order o WHERE o.order_id IN (10, 11);
DELETE FROM t_order o WHERE o.order_id > 10;
DELETE FROM t_order o WHERE o.order_id <= 11;
DELETE FROM t_order o WHERE o.order_id BETWEEN 10 AND 12;
......
UPDATE t_order o SET o.update_time = NOW() WHERE o.order_id = 10;
......
SELECT o.* FROM t_order o WHERE o.order_id = 10 AND user_id = 1001;
SELECT o.* FROM t_order o WHERE o.order_id IN (10, 11) AND user_id IN (......);
SELECT o.* FROM t_order o WHERE o.order_id > 10 AND user_id > 1000;
SELECT o.* FROM t_order o WHERE o.order_id <= 11 AND user_id <= 1000;
SELECT o.* FROM t_order o WHERE (o.order_id BETWEEN 10 AND 12) AND (o.user_id BETWEEN 1000 AND 2000);
......
INSERT INTO t_order(order_id, user_id) VALUES (21, 1002);
......
DELETE FROM t_order o WHERE o.order_id = 10 AND user_id = 1001;
DELETE FROM t_order o WHERE o.order_id IN (10, 11) AND user_id IN (......);
DELETE FROM t_order o WHERE o.order_id > 10 AND user_id > 1000;
DELETE FROM t_order o WHERE o.order_id <= 11 AND user_id <= 1000;
DELETE FROM t_order o WHERE (o.order_id BETWEEN 10 AND 12) AND (o.user_id BETWEEN 1000 AND 2000);
......
UPDATE t_order o SET o.update_time = NOW() WHERE o.order_id = 10 AND user_id = 1001;
......
```

注：在《召唤我的方式》这一章，我给出了一段配置，这段配置表明先依照 user_id % 2 对 t_order 进行水平拆分（到不同的子库），再依照 order_id % 2 对 t_order 进行水平拆分（到不同的子表）。但这并不是说使用了复合分片策略，而是使用了两个两个维度的标准分片策略。两个维度，分别是数据源分片策略（DatabaseShardingStrategy）和表分片策略（TableShardingStrategy），且在数据源分片策略上使用 user_id 作为单分片键、在表分片策略上使用 order_id 作为单分片键。

**三、Hint（翻译为暗示） 分片策略**

Hint 分片策略对应 HintShardingStrategy 这个 final class，同标准分片策略和符合分片策略的代码类似，HintShardingStrategy 中包含一个 HintShardingAlgorithm 接口的实例，并调用它的 doSharding()方法。你们要自己去实现这个 HintShardingAlgorithm 接口中的 doSharding()方法，这样我就能知道如何根据分片键的值，找到对应的分片表(或分片库)。此处不在展示 HintShardingStrategy 和 HintShardingAlgorithm 的源码。

Hint 分片策略是指通过 Hint 指定分片值而非从 SQL 中提取分片值的方式进行分片的策略。简单来讲就是我收到的 SQL 语句中不包含分片值（像上面给出的几段 SQL 就是包含分片值的 SQL），但是工程师会通过我提供的 Java API 将分片值暗示给我，这样我就知道怎样路由 SQL 查询到具体的分片了。就像下面这样：

```java
String sql = "SELECT * FROM t_order";
try (
    // HintManager 是使用“暗示”的工具，它会把暗示的分片值放入
    // 当前线程上下文（ThreadLocal）中，这样当前线程执行 SQL 的
    // 时候就能获取到分片值
    HintManager hintManager = HintManager.getInstance();
    Connection conn = dataSource.getConnection();
    PreparedStatement pstmt = conn.prepareStatement(sql);
) {
    hintManager.setDatabaseShardingValue(3);
    try (ResultSet rs = pstmt.executeQuery()) {
        // 若 t_order 仅仅使用 order_id 作为分片键，则这里根据暗
        // 示获取了分片值，因此上面的 SQL 的实际执行效果相当于：
        // SELECT * FROM t_order where order_id = 3
        while (rs.next()) {
            //...
        } 
    } 
}
```

**四、不分片策略**

对应 NoneShardingStrategy，这是一个 final class。由于我并不要求所有的表（或库）都进行水平分片，因此当工程师要通过我执行对不分片表（或库）的 SQL 查询时，就要使用这个不分片策略。NoneShardingStrategy 的源码为：

```java
package org.apache.shardingsphere.core.strategy.route.none;

......

@Getter
public final class NoneShardingStrategy implements ShardingStrategy {
    
    private final Collection<String> shardingColumns = Collections.emptyList();
    
    @Override
    public Collection<String> doSharding(
        // 所有可能的分片表(或分片库)名称
        final Collection<String> availableTargetNames, 
        // 分片键的值
        final Collection<RouteValue> shardingValues, 
        final ConfigurationProperties properties
    ) {
        
        // 不需要任何算法，不进行任何逻辑处理，直接返回
        // 所有可能的分片表(或分片库)名称
        return availableTargetNames;
    }
}
```

**五、Inline 分片策略**

Inline 分片策略，也叫做行表达式分片策略。Inline 分片策略对应 InlineShardingStrategy。Inline 分片策略是为用 Grovvy 表达式描述的分片算法准备的分片策略。文章开始展示的两段配置中就使用了 Inline 分片策略。InlineShardingStrategy 把 Grovvy 表达式当做分片算法的实现，因此 HintShardingStrategy 中不包含算法域变量，这一点有别于 StandardShardingStrategy 等 final class。这里不再展示 InlineShardingStrategy 的源码。

我知道，这段关于分片策略和分片算法的表述很难理解。不过我还是想让你们明白，无论对某个逻辑表（或库）进行怎样的分片策略配置，这些策略不过都是在告诉我怎样处理分片，也就是告诉我如何根据分片键的值，找到对应的分片表(或分片库)。只不过我的创造者把这个简单的过程翻出了很多花样，也就是你们在上面看到的各种策略，以提供使用上的灵活性。

### 4.2.5. 绑定表

指分片规则一致的主表和子表。例如 t_order 是主表，存储订单的基本信息； t_order_item 是子表，存储订单中的商品和价格明细。若两张表均按照 order_id 分片，并且配置了两个表之间的绑定关系，则此两张表互为绑定表。绑定表之间的多表关联查询不会出现笛卡尔积关联，关联查询效率将大大提升。举例说明，如果 SQL 为：

```sql
SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.order_id IN (10, 11);
```

在不配置绑定表关系时，假设分片键 order_id 将数值 10 路由至第 0 片，将数值 11 路由至第 1 片，那么路由后的 SQL 应该为 4 条，它们呈现为笛卡尔积，这种情况是我最不愿意处理的，我要考虑所有可能的分组合，它的工作量实在太大了：

```sql
SELECT i.* FROM t_order0 o JOIN t_order_item0 i ON o.order_id=i.order_id WHERE o.order_id IN (10, 11);
SELECT i.* FROM t_order0 o JOIN t_order_item1 i ON o.order_id=i.order_id WHERE o.order_id IN (10, 11);
SELECT i.* FROM t_order1 o JOIN t_order_item0 i ON o.order_id=i.order_id WHERE o.order_id IN (10, 11);
SELECT i.* FROM t_order1 o JOIN t_order_item1 i ON o.order_id=i.order_id WHERE o.order_id IN (10, 11);
```

而在配置绑定表关系后，路由的 SQL 只有 2 条：

```sql
SELECT i.* FROM t_order0 o JOIN t_order_item0 i ON o.order_id=i.order_id WHERE o.order_id IN (10, 11);
SELECT i.* FROM t_order1 o JOIN t_order_item1 i ON o.order_id=i.order_id WHERE o.order_id IN (10, 11);
```

而我也提供了这种绑定关系配置的 API 和配置项，例如在 properties 配置文件中可以这么写：

```properties
# 设置绑定表
sharding.jdbc.config.sharding.binding-tables=t_order, t_order_item
```

## 4.3. 我处理 SQL 的过程

我是 Sharding-JDBC，我是水平分片世界的神。我的职责是透明化水平分库分表所带来的影响，让使用方尽量像使用一个数据库一样使用水平分片之后的数据库集群，或者像使用一个数据表一样使用水平分片之后的数据表。

我的法力，来源于我的创造者为我设计的内核，它把 SQL 语句的处理分成了 SQL 解析 =>SQL 路由 => SQL 改写 => SQL 执行 => 结果归并五个主要流程。

![039](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ebb192b796b4850aa0531dd68520b60~tplv-k3u1fbpfcp-zoom-1.image)

当一个应用服务器节点将一个面向逻辑表编写的查询 SQL 交给我之后，我要做下面几件事：

1）SQL 解析（由我内核中的解析引擎完成）：先通过词法解析器将逻辑 SQL 拆分为一个个不可再分的单词，再使用语法解析器对 SQL 进行理解，并最终提炼出解析上下文。

2）SQL 路由（由我内核中的路由引擎完成）：根据解析上下文匹配用户配置的分片策略（关于用户配置的分片策略，我后文会慢慢解释），并生成路由路径，路由路径指示了 SQL 最终要到哪些分片去执行。

3）SQL 改写（由我内核中的改写引擎完成）：将 面向逻辑表 SQL 改写为在真实数据库中可以正确执行的语句（逻辑 SQL 到物理 SQL 的映射）。

4）SQL 执行（由我内核中的执行引擎完成）：通过多线程执行器异步执行路由和改写之后得到的 SQL 语句。

5）结果归并（由我内核中的归并引擎完成）：将多个执行结果集归并以便于通过统一的 JDBC 接口输出。

### 4.3.1. SQL 解析

SQL 解析 SQL 解析分为词法解析和语法解析。

我的解析引擎先通过词法解析器将这句 SQL 拆分为一个个不可再分的单词，再使用语法解析器对 SQL 进行理解，并最终提炼出解析上下文。解析上下文包括表、选择项、排序项、分组项、聚合函数、分页信息、查询条件以及可能需要修改的占位符的标记。简单来说就是我要理解这句 SQL，明白它的结构和意图。所幸，SQL 是一个语法简单的语言，SQL 解析这件事情并不复杂。

我先使用解析引擎的词法解析器用于将 SQL 拆解为不可再分的原子符号，我把它们叫做 Token，并将其归类为关键字、表达式、字面量、操作符，再使用解析引擎的语法解析器将 SQL 转换为抽象语法树。

例如，以下 SQL：

```sql
SELECT id, name FROM t_user WHERE status = 'ACTIVE' AND age > 18
```

被我的词法解析器和语法解析器解析之后得到的抽象语法树为：

![040](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea5406c9ee2d4205830880c2139c174f~tplv-k3u1fbpfcp-zoom-1.image)

在上图中，为了便于理解，抽象语法树中的关键字和操作符的 Token 用绿⾊表示，字面量的 Token 用红⾊表示，灰⾊表示需要进一步拆分。

最后，我通过对抽象语法树的遍历去提炼分片所需的上下文，并标记有可能需要改写的位置。供分片使用的解析上下文包含查询选择项（Select Items）、表信息（Table）、分片条件（Sharding Condition）、自增主键信息（Auto increment Primary Key）、排序信息（Order By）、分组信息（Group By）以及分页信息（Limit、Rownum、Top）。

SQL 解析是下面的路由、改写、执行和归并的基础。

### 4.3.2. SQL 路由

我的内核在这一阶段根据 SQL 的解析上下文匹配数据库和表的分片策略（还记得吗，我在《一些核心概念》这一节说过，分片策略=分片键+分片算法，分片策略会指示我如何根据分片键的值，找到对应的分片表或分片库），找到对应的分片表或分片库，并生成路由后的 SQL。

对于携带分片键的 SQL，我会根据分片键值的不同可以划分为单片路由 (比如分片键的操作符是=)、多片路由 (比如分片键的操作符是 IN、BETWEEN AND、>、<、>=、<=，或者多表关联查询)。单片路由生成针对某个分片进行查询的 SQL，多片路由生成针对某些分片进行查询的 SQL。

不携带分片键的 SQL 则采用全路由（全路由是一种特殊的多片路由），即生成针对所有分片进行查询的 SQL。但如果这条 SQL 能够匹配 Hint 分片策略，我就知道工程师会通过我的 API 把分片键值暗示给我，这时候我从 API 拿到分片键值后也会去做单片或者多片路由。

这里的单片路由、多片路由或者全库路由是对路由划分的一种角度，它反映了我最终执行 SQL 的路径有几条：若 SQL 解析上下文最终被计算出存在单片路由，在一个数据源内我只需要针对一个分片上去执行 SQL；若 SQL 解析上下文最终被计算出存在多片路由，在一个数据源内我需要针对多个分片上去执行 SQL。若 SQL 解析上下文最终被计算出存在全路由，在一个数据源内我就要针对全部分片去执行 SQL。

下面是一些实例：

```sql
-- 若仅以 user_id 作为分片键对 t_user 进行分片，且分片算法为 user_id % 5，则以下 SQL 在一个数据源内会针对一个特定分片执行：
SELECT * FROM t_user WHERE user_id = 1009 --路由到 t_user4 执行

-- 若仅以 user_id 作为分片键对 t_user 进行分片，且分片算法为 user_id % 5，则以下 SQL 在一个数据源内会针对多个分片执行：
SELECT * FROM t_user WHERE user_id in (1002, 1003, 1009) --路由到 t_user2、t_user3、t_user4
SELECT * FROM t_user WHERE user_id > 1002 AND user_id <= 1004 --路由到 t_user3、t_user4
SELECT * FROM t_user WHERE user_id between 1002 and 1004 --路由到 t_user2、t_user3

-- 若仅以 user_id 作为分片键对 t_user 进行分片，且分片算法为 user_id % 5，则以下 SQL 在一个数据源内会针对所有的分片执行：
SELECT count(1) FROM t_user --路由到 t_user0、t_user1、t_user2、t_user3、t_user4
SELECT * FROM t_user where age < 18 --路由到 t_user0、t_user1、t_user2、t_user3、t_user4
```

### 4.3.3. SQL 改写

⼯程师交给我处理的 SQL 是面向逻辑表书写的 SQL，并不能够直接在数据库中执行，所以我的内核要完成 SQL 改写，将面向逻辑表的 SQL 改写面向物理表的 SQL。SQL 改写分为标识符改写、补列、分页修正、批量拆分。

![041](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a20ac0518fe14670ae06e6aa3394e8d3~tplv-k3u1fbpfcp-zoom-1.image)

**一、标识符改写**

在水平分片的场景中，需要将 SQL 中的逻辑表名改写为路由之后所对应的物理分片表名，索引名称以及 Schema 名称也要进行逻辑名到物理名的改写。

1）表名称改写

表名称改写是指将找到逻辑表名在原始 SQL 中的位置，并将其改写为真实分片表名的过程。比如，若逻辑 SQL 为：

```sql
SELECT order_id FROM t_order WHERE order_id=1;
```

假设该 SQL 配置分片键 order_id，并且 order_id=1 的情况，将路由至分片表 1。那么改写之后的 SQL 应该为：

```sql
SELECT order_id FROM t_order1 WHERE order_id=1;
```

你或许会以为只要通过字符串查找和替换就可以达到 SQL 改写的效果，但事实并非如此，例如：

```sql
SELECT t_order.order_id FROM t_order AS t_order WHERE t_order.order_id=1 AND remarks='备注 t_order xxx';
```

SQL 改写则仅需要改写表名称就可以了，别名“t_order”、备注字段内容“t_order”均无需改写：

```sql
SELECT t_order.order_id FROM t_order_1 AS t_order WHERE t_order.order_id=1 AND remarks='备注 t_order xxx';
```

因此表名称改写是一个典型的需要对 SQL 进行词法和语法解析的场景，它依赖于 SQL 解析上下文，即依赖于对 SQL 语义的理解，而不是简单的字符串替换！对于包含索引和 Schema 的 SQL 改写也是一样。

2）索引名称改写

索引名称是另一个有可能改写的标识符。在某些数据库中（如 MySQL、SQLServer），索引是以表为维度创建的，在不同的表中的索引是可以重名的；而在另外的一些数据库中（如 PostgreSQL、Oracle），索引是以数据库为维度创建的，即使是作用在不同表上的索引，它们也要求其名称的唯一性。这些琐碎的规则都要纳入我的索引改写算法的考量之中。

3）Schema 名称改写

我对于 Schema（Schema 这个词语的含义是 DBMS 系统中的数据库实例，上文讲的 ds0、ds1 就是两个数据库实例） 管理的方式与管理表的方式如出一辙，即采用逻辑 Schema 去管理一组数据源。因此，对于包含 Schema 的 SQL，我需要将用户在 SQL 中书写的逻辑 Schema 改写为真实的数据库分片 Schema。但我目前还不支持在 DQL（数据查询语言，SELECT）和 DML（数据操纵语言，INSERT、UPDATE、DELETE 等）语句中使用 Schema，我只能改写数据库管理语句中的 Schema，例如：

```sql
SHOW COLUMNS FROM t_order FROM order_ds;
```

我对这句数据库管理语句的处理的方式是，将逻辑 Schema 改写为随机查找到的一个正确的真实 Schema。这很简单粗暴，但合理，因为每个 Schema 中的 t_order 表的 COLUMNS 都是一样的。

**二、补列**

1）排序补列

如下所示的一个 SQL 语句，查询逻辑表 t_order 中的 order_id 和 user_id，并且得到的结果根据 user_id 降序排列，这个语句经过路由和改写之后在我的内核的执行阶段执行起来显然没有什么问题。

```sql
SELECT order_id, user_id FROM t_order ORDER BY user_id;
```

但如果 SQL 语句是：

```sql
SELECT order_id FROM t_order0 ORDER BY user_id;
```

我的内核在执行阶段就无法执行，因为这个语句查询的结果只有 order_id，但却要按照每个 order_id 对应的 user_id 排列 order_id，而结果集中没有 user_id 列。所以，我的内核在补列阶段要对这个 SQL 补充一列 user_id，补列的结果为：

```sql
SELECT order_id, user_id FROM t_order0 ORDER BY user_id;
```

再比如：

```sql
-- 补列前（结果集 o.* 中不包含排序键 order_item_id）
SELECT o.* FROM t_order o, t_order_item i WHERE o.order_id=i.order_id ORDER BY user_id, order_item_id;

-- 补列后（结果集 o.* 中包含排序键 order_item_id）
SELECT o.*, order_item_id FROM t_order o, t_order_item i WHERE o.order_id=i.order_id ORDER BY user_id, order_item_id;
```

2）分组补列

和排序补列类似，分组补列的目的是在结果字段中补全分组键，比如：

```sql
-- 补列前（结果集 order_id 中不包含分组键 user_id）
SELECT order_id FROM t_order GROUP BY user_id

-- 补列后（结果集 order_id 中包含分组键 user_id）
SELECT order_id, user_id FROM t_order GROUP BY user_id
```

3）聚合补列

分组和排序补列是简单的补列处理情形。复杂的补列情形如处理使用 AVG 等聚合函数的 SQL 语句的补列。

将逻辑表 t_order 仅使用 order_id 为分片键水平分片成 3 个物理表 t_order0、t_order1、t_order2。使用 avg1 + avg2 + avg3 / 3 计算逻辑表的某列的平均值并不正确，正确的算法为 (sum1 + sum2 + sum3) / (count1 + count2 + count3)。这就需要将包含 AVG 的 SQL 改写为 SUM 和 COUNT，并在结果归并时重新计算平均值。例如以下 SQL：

```sql
SELECT AVG(age) FROM t_user WHERE age>=18;
```

会被补列处理成：

```sql
SELECT COUNT(age) AS AVG_DERIVED_COUNT, SUM(age) AS AVG_DERIVED_SUM FROM t_user WHERE age>=18;
```

再经过路由和改写，最终执行的 SQL 为：

```sql
SELECT COUNT(age) AS AVG_DERIVED_COUNT, SUM(age) AS AVG_DERIVED_SUM FROM t_user0 WHERE age>=18;

SELECT COUNT(age) AS AVG_DERIVED_COUNT, SUM(age) AS AVG_DERIVED_SUM FROM t_user1 WHERE age>=18;

SELECT COUNT(age) AS AVG_DERIVED_COUNT, SUM(age) AS AVG_DERIVED_SUM FROM t_user2 WHERE age>=18;
```

最后，按照 (sum1 + sum2 + sum3) / (count1 + count2 + count3)在结果归并时计算出正确的平均值。

这很好理解，打个比方，一个学校四年级学生全部有 400 人，被水平分片到 4 个班级，分别是四（1）班、四（2）班、四（3）班、四（4）班，各班人数 100 左右。一次期末考试之后，统计整个四年级的平均成绩，一定是：

```
(
    四（1）班总分 +
    四（2）班总分 +
    四（3）班总分 +
    四（4）班总分
) / (
    四（1）班人数 +
    四（2）班人数 +
    四（3）班人数 +
    四（4）班人数
)
```

而不会是：

```
(
    四（1）班平均分 +
    四（2）班平均分 +
    四（3）班平均分 +
    四（4）班平均分
) / 4
```

4）自增主键补列

还有一种补列发生在执行 INSERT 的 SQL 语句时。

INSERT 语句如果使用数据库自增主键，是无需写入主键字段的，依靠数据库实例本身自动产生自增主键。但单个数据库实例产生的自增主键是无法满足数据表多分片场景下的主键的唯一性要求的，因此我提供了分布式自增主键的生成算法（如雪花算法），并且可以通过补列，让使用方无需改动现有代码，即可将数据库现有的自增主键透明地替换成分布式自增主键。举例说明，假设表 t_order 的主键是 order_id，原始的 SQL 为：

```sql
INSERT INTO t_example (`field1`, `field2`) VALUES (10, 1);
```

可以看到，上述 SQL 中并未包含自增主键，是需要数据库自行填充的，如果我不干预，数据库会使用一个局部自增主键来填充，这可能会造成全局范围内的多个 t_order 分片表里包含重复主键。但有我在，我就不会让数据库使用它自己的局部自增主键，而是使用我提供的分布式自增主键。因此，SQL 将被改写为：

```sql
INSERT INTO t_example (id, `field1`, `field2`) VALUES (snow_flake_id, 10, 1);
```

上述 SQL 中的 snow_flake_id 表示自动生成的分布式全局自增主键值。

显然，所有的补列都是基于 SQL 语义进行的，有赖于 SQL 的词法和语法分析。因此，我还是要重复那句话：SQL 解析是 SQL 路由、改写、执行和归并的基础。

**三、分页修正**

从多个表分片中获取分页数据与单表的场景是不同的。假设每 10 条数据为一页，要从一个逻辑表中查询 2 页数据。在分片环境下从每个物理分片中获取 LIMIT 10, 10，归并之后再根据排序条件取出前 10 条数据是不正确的。

举例说明，假设 t_order 根据 order_iid % 2 分成两片，若对逻辑表 t_order 分页查询的 SQL 为：

```sql
SELECT age FROM t_user ORDER BY age DESC LIMIT 1, 2;
```

若直接路由并改写成：

```sql
SELECT age FROM t_user0 ORDER BY age DESC LIMIT 1, 2;
SELECT age FROM t_user1 ORDER BY age DESC LIMIT 1, 2;
```

得到的结果会出乎你的预料，下图展示了不进行 SQL 的改写的分页执行结果。

![042](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90979e7386c94734a7efb8daa7e342f8~tplv-k3u1fbpfcp-zoom-1.image)

通过图中所示，想要取得两个分片表中共同的按照分数排序的第 2 条和第 3 条数据，应该是 95 和 90。由于执行的 SQL 只能从每个表中获取第 2 条和第 3 条数据，即从 t_user0 表中获取的是 90 和 80；从 t_user1 表中获取的是 85 和 75。因此进行结果归并时，只能从获取的 90，80，85 和 75 之中进行归并，那么结果归并无论怎么实现，都不可能获得正确的结果。

正确的做法是将分页条件改写为 LIMIT 0, 3，取出所有前两页数据，再结合排序条件计算出正确的数据。即：

```sql
SELECT age FROM t_user ORDER BY age DESC LIMIT 0, 3;
```

路由并改写之后的结果为：

```sql
SELECT age FROM t_user0 ORDER BY age DESC LIMIT 0, 3;
SELECT age FROM t_user1 ORDER BY age DESC LIMIT 0, 3;
```

下图展示了进行正确的 SQL 改写之后的分页执行结果：

![043](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28bb941a66384795a696991023721fa3~tplv-k3u1fbpfcp-zoom-1.image)

在这种做法下，获取数据的偏移量位置越靠后，使用 LIMIT 分页方式的效率就越低。但有很多方法可以避免使用 LIMIT 进行分页。比如使用上次分页数据结尾 ID 作为下次查询条件的分页方式等（我会在后文给出示例）。

**四、批量拆分**

1）批量插入拆分

在处理批量插入的 SQL 时，如果插入的数据是跨分片的，那么需要对 SQL 进行改写来防止将多余的数据写入到数据库中。举例说明，如下 SQL：

```sql
INSERT INTO t_order (order_id, xxx) VALUES (1, 'xxx'), (2, 'xxx'), (3, 'xxx');
```

假设数据表 t_order 仍然是按照 order_id 的奇偶值分为两片的，仅将这条 SQL 中的表名进行修改，然后发送至数据库完成 SQL 的执行，则两个分片都会写入相同的记录。虽然只有符合分片查询条件的数据才能够被查询语句取出，但存在冗余数据的实现方案并不合理。因此我需要将路由后的 SQL 改写为：

```sql
INSERT INTO t_order0 (order_id, xxx) VALUES (2, 'xxx');
INSERT INTO t_order1 (order_id, xxx) VALUES (1, 'xxx'), (3, 'xxx');
```

2）In 查询拆分

使用 IN 的批量查询与批量插入的情况相似，不过使用 IN 的批量查询操作并不会导致数据查询结果错误（批量插入操作与批量查询操作的不同之处在于，查询语句中即使用了不存在于当前分片的分片键值，也不会对结果产生影响。因此对批量查询 SQL 进行拆分并不是必须的，而插入操作则必须将多余的分片键值删除）。

因此对于如以下 SQL 的批量拆分改写，我偷了个懒：

```sql
SELECT * FROM t_order WHERE order_id IN (1, 2, 3);
```

直接路由并改写为：

```sql
SELECT * FROM t_order0 WHERE order_id IN (1, 2, 3);
SELECT * FROM t_order1 WHERE order_id IN (1, 2, 3);
```
实际上，更好的改写结果是：

```sql
SELECT * FROM t_order0 WHERE order_id IN (2);
SELECT * FROM t_order1 WHERE order_id IN (1, 3);
```

这样可以进一步的提升查询性能，但我的创造者给我设计的内核并没有进行这种优化。虽然 SQL 的执行结果是正确的，但并未达到最优的查询效率。

### 4.3.4. SQL 执行

在完成 SQL 解析、改写和路由之后，我终于要执行 SQL 了！但这也是我的内核最复杂的工作部分。

我拥有一个自动化的 SQL 执行引擎，它负责将改写和路由完成之后的真实 SQL 安全且高效发送到底层数据源执行。它不是简单地将 SQL 通过 JDBC 直接发送至数据源执行，也并非直接将执行请求放入线程池去并发执行，而是采用了复杂的控制策略。我的执行引擎的工作目标是平衡资源占用（资源包括数据库连接、内存和线程）与执行效率（时间）。

在讲解我的执行引擎执行 SQL 的过程之前，我要先向各位介绍我的执行引擎的连接模式。

一个面向逻辑表编写的 SQL 交到我的手中，会被我路由、改写成面向多个物理分片表的 SQL（也可以称为真实 SQL）。执行多个真实 SQL，最理想的情况是为每个分片 SQL 查询创建一个数据库连接，且每个连接交由一个专门的线程来处理。但是计算机系统所能提供的资源是有限的，不可能让进程无限创建数据库连接和线程。

从资源控制的角度看，业务方访问数据库的连接数量应当有所限制（你们常用的数据库连接池就在做这件事）。它能够有效地防止某一业务操作过多地占用资源，从而将数据库连接的资源耗尽，以致于影响其他业务的正常访问。特别是，在一个数据库实例中存在较多分片表的情况下，一条不包含分片键的逻辑 SQL 经过路由过程将产生大量落在同库不同分片表的真实 SQL，如果每条真实 SQL 都占用一个独立的连接，那么一次查询无疑将会占用过多的资源。

从执行效率的角度看，为每个分片查询维持一个独立的数据库连接，可以更加有效的利用多线程来提升执行效率，因为若为每个数据库连接开启独立的处理线程，可以并行处理查询结果集。而且，为每个分片查询维持一个独立的数据库连接，还能够避免过早的将查询结果集加载至数据库客户端（我，Sharding-JDBC，数据库中间件，运行在应用程序所在的 JVM 上，就是一个数据库客户端）的内存，代以流式处理方式来处理。若为每个分片查询维持一个独立的数据库连接，能够持有查询结果集游标位置的引用，在需要获取相应数据时移动游标即可。以结果集游标下移进行结果归并的方式，称之为流式归并，它无需将结果数据全数加载至数据库客户端内存，可以有效的节省数据库客户端内存资源，进而减少数据库客户端垃圾回收的频次（说的简单些，即先将查询结果集保留在数据库服务器的缓冲区内，然后客户端这边采用流式处理方式一点点获取数据来处理。避免一次性将结果集送到客户端，占用客户端太多内存）。当无法保证每个分片查询持有一个独立数据库连接时，则需要在复用该数据库连接获取下一个分片查询的结果集之前，将当前的分片查询结果集全数加载至内存。因此，即使可以采用流式归并，在此场景下也将退化为内存归并。

综上所述，我的执行引擎一方面想控制数据库连接的数量；另一方面想为每个分片查询维持一个独立的数据库连接，以采用更优的流式归并模式达到对数据库客户端内存资源的节省。如何处理好两者之间的关系，是我的执行引擎需要解决的问题。

举个例子，如果一条逻辑 SQL 在经过我的路由和改写处理之后，需要操作某数据库实例下的 200 张分表。那么，是选择创建 200 个连接并行执行，还是选择创建一个连接串行执行呢？效率与资源控制又应该如何抉择呢？针对上述场景，我的执行引擎提供了一种解决思路。它提出了连接模式（Connection Mode）的概念，将其划分为内存限制模式（MEMORY_STRICTLY）和连接限制模式（CONNECTION_STRICTLY）这两种类型。内存限制模式要求更多的连接，但占用更少的客户端内存；而连接限制模式要求更少的连接，但占用更多的客户端内存。

**一、内存限制模式**

在这种模式下，我的执行引擎对一次操作所耗费的数据库连接数量不做限制。如果实际执行的 SQL 需要对某数据库实例中的 200 张分片表做操作，则对每张分片表创建一个新的数据库连接，并通过多线程的方式并发处理，以达成执行效率最大化。并且在 SQL 满足条件情况下，优先选择流式归并，以防止数据库客户端出现内存溢出或避免频繁垃圾回收情况。

**二、连接限制模式**

在这种模式下，我的执行引擎严格控制对一次操作所耗费的数据库连接数量。如果实际执行的 SQL 需要对某数据库实例中的 200 张分片表做操作，那么只会创建唯一的数据库连接，并对其 200 张分片表串行处理。如果一次操作中的分片散落在不同的数据库，仍然采用多线程处理对不同库的操作，但每个库的每次操作仍然只创建一个唯一的数据库连接。这样即可以防止对一次请求对数据库连接占用过多所带来的问题。该模式始终选择内存归并。

内存限制模式适用于 OLAP（以读操作为主）操作，可以通过放宽对数据库连接的限制提升系统吞吐量；连接限制模式适用于 OLTP （以写操作为主）操作。OLTP 通常带有分片键，会路由到单一的分片，因此严格控制数据库连接，以保证在线系统数据库资源能够被更多的应用所使用。

我最初想将使用何种模式的决定权交由你们这些工程师来配置，让你们依据自己业务的实际场景需求选择使用内存限制模式或连接限制模式。这种想法将两难的选择的决定权交由用户，使得用户必须要了解这两种模式的利弊，并依据业务场景需求进行选择。而且这种静态的连接模式配置，缺乏灵活性。

在实际的使用场景中，面对不同的逻辑 SQL，每次的路由结果是不同的。这就意味着某些操作可能需要使用内存归并，而某些操作则可能选择流式归并更优，具体采用哪种方式不应该由用户在我启动之前配置好，而是应该根据具体的逻辑 SQL，来动态地决定连接模式。

为了降低用户的使用成本，以及让连接模式能够动态变化，我的执行引擎在其内部消化了连接模式概念（可我还是认为应该告诉你们这些被屏蔽的东西，毕竟技术的原理和优秀的设计思想是促进你们进步的重要因素），根据当前场景自动选择最优的执行方案。

我的执行引擎将连接模式的选择粒度细化至每一次逻辑 SQL 请求。针对每次逻辑 SQL 请求，我的执行引擎都将根据其路由结果，进行实时的演算和权衡，并自主地采用恰当的连接模式执行，以达到资源控制和效率的最优平衡。

针对这种自动化的执行引擎，用户只需配置 maxConnectionSizePerQuery 即可，该参数表示进行一次逻辑查询时每个数据库所允许使用的最大连接数，这是我的执行引擎进行演算和权衡的重要参数。

好了，我的执行引擎提供的连接模式讲完了，我可以给你们讲我的执行引擎执行 SQL 的过程了。我的执行引擎把执行 SQL 分为准备和执行两个阶段。

**一、准备阶段**

准备阶段分为结果集分组和执行单元创建两个步骤。

结果集分组是实现内化连接模式（向使用我的工程师屏蔽内存限制模式或连接限制模式的选择）概念的关键，结果集分组的工作，一言以蔽之，就是决定每个连接要处理的查询请求/要执行的 SQL。结果集分组具体步骤如下：

1）先将 SQL 的路由结果按照数据源的名称进行分组；

2）然后通过下图的公式，可以获得每个数据库实例在 maxConnectionSizePerQuery 的允许范围内，每个连接需要执行的 SQL 路由结果组，并计算出本次请求的最优连接模式。

![044](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab95a4c1e7524e328ac6c4241e3ad3ef~tplv-k3u1fbpfcp-zoom-1.image)

在 maxConnectionSizePerQuery 允许的范围内，当一个连接需要执行的请求数量大于 1 时，意味着当前的数据库连接无法持有相应的分片结果集，则必须采用内存归并；反之，当一个连接需要执行的请求数量等于 1 时，意味着当前的数据库连接可以持有相应的分片结果集，则可以采用流式归并。每一次的连接模式的选择，是针对每一个物理数据库的。也就是说，在同一次查询中，如果该查询被路由至一个以上的数据库，每个数据库的连接模式不一定一样，它们可能是混合存在的形态。

通过上一步骤获得的路由分组结果创建执行的单元，执行单元包括连接+该连接上要执行的 SQL。当数据源使用数据库连接池等控制数据库连接数量的技术时，在获取数据库连接时，如果不妥善处理并发，则有一定几率发生死锁。在多个请求相互等待对方释放数据库连接资源时，将会产生饥饿等待，造成交叉的死锁问题。举例说明，假设一次查询需要在某一数据源上获取两个数据库连接，并路由至同一个数据库的两个分表查询。则有可能出现查询 A 已获取到该数据源的 1 个数据库连接，并等待获取另一个数据库连接；而查询 B 也已经在该数据源上获取到的一个数据库连接，并同样等待另一个数据库连接的获取。如果数据库连接池的允许最大连接数是 2，那么这 2 个查询请求将永久的等待下去。下图描绘了死锁的情况。

![045](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0b44ec65d654c8cb4428f91eef64a26~tplv-k3u1fbpfcp-zoom-1.image)

我为了避免死锁的出现，在获取数据库连接时进行了同步处理。具体来说就是在创建执行单元时，以原子性的方式一次性获取本次 SQL 请求所需的全部数据库连接，杜绝了每次查询请求获取到部分资源的可能。由于这样做会导致每次获取数据库连接时都进行连接锁定，这会降低我执行 SQL 的并发度。因此，我在这⾥进行了 2 点优化：

1）避免锁定一次性只需要获取 1 个数据库连接的操作。因为每次仅需要获取 1 个连接，则不会发生两个请求相互等待的场景，无需锁定。对于大部分 OLTP 的操作，都是使用分片键路由至唯一的数据节点，这会使得系统变为完全无锁的状态，进一步提升了并发效率。

2）仅针对内存限制模式时才进行资源锁定。在使用连接限制模式时，所有的查询结果集将在装载至内存之后释放掉数据库连接资源，因此不会产生死锁等待的问题。

**二、执行阶段**

该阶段用于真正地执行 SQL，它分为分组执行和查询结果集生成两个步骤。

1）分组执行：分组执行将准备执行阶段生成的执行单元分组下发至我的底层执行引擎，并针对执行过程中的每个关键步骤发送事件。如：执行开始事件、执行成功事件以及执行失败事件。我的执行引擎仅关注事件的发送，它并不关心事件的订阅者。我的其他模块，如：分布式事务、调用链路追踪等，会订阅感兴趣的事件，并进行相应的处理。我通过在执行准备阶段的获取的连接模式，生成内存查询结果集或流式查询结果集，并将其传递至结果归并引擎，以进行下一步的⼯作。

我的执行引擎的整体工作流如下图所示。

![046](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a124764939814139b5cd7ef3cf68efe0~tplv-k3u1fbpfcp-zoom-1.image)

2）归并结果集：请看下一节。

### 4.3.5. 结果归并

我建议你好好看看这一节，它里面包含了很多数据结构的知识。

我将从各个数据分片上获取的结果集，组合成为一个总的结果集并正确的返回至请求客户端，这个过程就是结果归并。

我支持的结果归并从结构上划分，可分为流式归并、内存归并和装饰者归并：

1）流式归并

流式归并是指在实施归并的时候，不需要将所有分片上的查询结果全部都加载进客户端内存，只需要把每个分片的查询结果一点点地取到内存里面进行归并处理，最终能够逐条产生归并的结果。后文要讲的遍历归并、排序归并以及流式分组归并都属于流式归并。

2）内存归并

内存归并则是指需要将所有的分片结果集加载到内存中，再通过统一的分组、排序以及聚合等计算之后，再将其封装成为能被请求客户端逐条访问的归并结果集返回。

3）装饰者归并

装饰者归并是指对常规的结果集归并利用装饰者模式进行功能增强，目前装饰者归并有分页装饰归并和聚合装饰归并这 2 种类型。我在前文讲过，包含聚合函数的 SQL 经过改写之后要在归并阶段重新计算聚合，这就是装饰者归并要做的事情；同样，包含分页信息的 SQL 经过改写之后要在归并阶段重新进行分页计算，这也是装饰者归并要做的事情。

我支持的结果归并从功能上分为遍历、排序、分组、分页和聚合 5 种类型：

1）遍历归并

它是最为简单的归并方式。只需将多个分片结果集合并为一个单向链表即可。在遍历完成链表中当前分片结果集之后，将链表元素后移一位，继续遍历下一个分片结果集即可。

例如，逻辑表 t_user 在单个数据源（不做分库）中根据 user_id % 3 的结果分成三片 t_user0、t_user1 和 t_user2，当查询的逻辑 SQL 为：

```sql
SELECT age FROM t_user where age < 18
```

它被路由和改写之后的结果为：

```sql
SELECT age FROM t_user0 where age < 18
SELECT age FROM t_user1 where age < 18
SELECT age FROM t_user2 where age < 18
```

显然它最终产生三个分片结果集，对这三个结果集进行归并，只需将他们串联成链表返回给请求客户端即可。请求客户端读取总的归并结果集，也就是按照链表元素次序，一个分片结果集读完后，再到下一个分片结果集去读取。显然这个过程是可以使用流式处理方式的，即不需要事先把三个分片结果集一次性全部加载到内存。

2）排序归并

例如，逻辑表 t_user 在单个数据源（不做分库）中根据 user_id % 3 的结果分成三片 t_user0、t_user1 和 t_user2，当查询的逻辑 SQL 为：

```sql
SELECT age FROM t_user order by age DESC
```

它被路由和改写之后的结果为：

```sql
SELECT age FROM t_user0 order by age DESC
SELECT age FROM t_user1 order by age DESC
SELECT age FROM t_user2 order by age DESC
```

由于在 SQL 中存在 ORDER BY 语句，因此每个分片结果集自身是有序的，因此只需要将分片结果集当前游标指向的数据值进行排序即可。这相当于对多个有序的数组进行排序，归并排序是最适合此场景的排序算法。

我在对带 ORDER BY 语句的分片查询结果进行归并时，会将每个结果集的当前数据值进行比较，并将其放入优先级队列。每次获取下一条数据时，只需将队列顶端结果集的游标下移，并根据新游标重新进入优先级排序队列找到自己的位置即可。

下图展示了 3 张分片表返回的分片结果集，每个分片结果集已经根据分数排序完毕，但是 3 个分片结果集之间是无序的。将 3 个分片结果集的当前游标指向的数据值进行排序，并放入优先级队列，t_user0 的第一个数据值最大，t_user2 的第一个数据值次之，t_user1 的第一个数据值最小，因此优先级队列根据 t_user0、t_user2 和 t_user1 的方式排序队列。

![047](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac0911c436cf4df7b3da26de45e4c813~tplv-k3u1fbpfcp-zoom-1.image)

下图则展现了进行 next 调用的时候，排序归并是如何进行的。通过下图你们可以看到，当进行第一次 next 调用时，排在队列首位的 t_user0 将会被弹出队列，并且将当前游标指向的数据值（也就是 100）返回至查询客户端，并且将游标下移一位之后，重新放入优先级队列。而优先级队列也会根据 t_user0 的当前数据结果集指向游标的数据值（这⾥是 90）进行排序，根据当前数值，t_user0 排列在队列的最后一位。之前队列中排名第二的 t_user2 的分片结果集则自动排在了队列首位。

![048](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01ea9e7984d545a99fc42b81e57ba4de~tplv-k3u1fbpfcp-zoom-1.image)

在进行第二次 next 时，只需要将目前排列在队列首位的 t_user2 弹出队列，并且将其数据结果集游标指向的值返回至客户端，并下移游标，继续加入队列排队，以此类推。当一个结果集中已经没有数据了，则无需再次加入队列。

可以看到，对于每个数据结果集中的数据有序，而多数据结果集整体无序的情况下，我无需将所有的数据都加载至内存即可排序，我使用的是流式归并的方式，每次 next 仅获取唯一正确的一条数据，极大的节省了内存的消耗。

3）分组归并

分组归并的情况最为复杂，它分为流式分组归并和内存分组归并。流式分组归并要求 SQL 的排序项与分组项的字段必须保持一致，否则只能通过内存归并才能保证其数据的正确性。

举例说明，假设逻辑表 t_socre（表结构中包含考生的姓名 name、科目 subject 和分数 score，且为了简单起见，不考虑重名的情况）根据科目分成 3 片：t_socre_java、t_socre_go、t_socre_python（后文插图中的分片表均未展示科目字段，只展示姓名和分数字段）。现在要通过 SQL 获取每位考生的总分：

```sql
SELECT name, SUM(score) as sum_score FROM t_score GROUP BY name ORDER BY name asc;
```

以上 SQL 被路由和改写之后的结果为：

```sql
SELECT name, SUM(score) as sum_score FROM t_score_java GROUP BY name ORDER BY name asc;
SELECT name, SUM(score) as sum_score FROM t_score_go GROUP BY name ORDER BY name asc;
SELECT name, SUM(score) as sum_score FROM t_score_python GROUP BY name ORDER BY name asc;
```

在分组项与排序项完全一致的情况下，在三个分片表中取得的数据都是按照 name 字段升序排列的，每个分组所需的数据全部存在于各个分片结果集的当前游标所指向的数据值中，即每个 name 的分数全部存在于各个分片结果集的当前游标所指向的数据值中，因此可以采用流式归并。如下图所示：

![049](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ab63ed5e4bf48039e807c6b45e29916~tplv-k3u1fbpfcp-zoom-1.image)

进行归并时，过程与排序归并类似。下图展现了进行 next 调用的时候，流式分组归并是如何进行的。

![050](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69eede0bd8934c019a2f25410e698068~tplv-k3u1fbpfcp-zoom-1.image)

通过上一张图你们可以看到，当进行第一次 next 调用时，按照当前游标所指记录的 name 升序排列，排在队列首位的 t_score_java 分片将会被弹出队列，并且将 name 同为“Jetty”的其他分片结果集中的数据一同弹出队列。在获取了所有的 name 为“Jetty”的同学的分数之后，进行累加操作，得到“Jetty”的总分。与此同时，所有的分片结果集中的游标都将下移至数据值“Jetty”的下一个不同的数据值，并且根据分片结果集的当前游标所指记录的 name 值进行重排序。因此，包含名字“John”的相关数据结果集则排在的队列的前列。

对于分组项与排序项不一致的情况，由于在每个分片结果集中分组字段的值并非有序的，因此无法使用流式归并，需要将所有的分片结果集数据加载至内存中进行分组和聚合。例如，若通过以下 SQL 获取每位考生的总分并按照分数从高至低排序：

```sql
SELECT name, SUM(score) as sum_score FROM t_score GROUP BY name ORDER BY score DESC;
```

那么各个分片结果集中的数据如下图所示，显然是无法像上图那样进行流式归并的，不信你按照上一张图的过程动笔画一下试试 :-)

![051](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f8fcc05f6ff4cca8340087345fe883c~tplv-k3u1fbpfcp-zoom-1.image)

当 SQL 中只包含分组语句时，我会通过 SQL 改写，自动给 SQL 增加与分组项一致的排序项，这一点我在讲述 SQL 改写的没有说，我放在这里说你会更加明白我的意图：这能够使得这句 SQL 的归并阶段从消耗内存的内存分组归并方式转化为流式分组归并方式。

4）聚合归并

聚合函数可以分为比较、累加和求平均值这 3 种类型。

比较类型的聚合函数是指 MAX 和 MIN。它们需要对每一个同组的结果集数据进行比较，并且直接返回其最大或最小值即可。

举例说明，假设逻辑表 t_socre（表结构中包含考生的姓名 name、科目 subject 和分数 score，且为了简单起见，不考虑重名的情况）根据科目分成 3 片：t_socre_java、t_socre_go、t_socre_python（后文插图中的分片表均未展示科目字段，只展示姓名和分数字段）。现在要通过 SQL 获取每位考生的单科最高分：

```sql
SELECT name, MAX(score) FROM t_score GROUP BY name;
```

以上 SQL 被路由和改写之后的结果为：

```sql
--当 SQL 中只包含分组语句时，我会通过 SQL 改写，自动增加与分组项一致的排序项，这能够使得这句 SQL 的归并阶段从消耗内存的内存分组归并方式转化为流式分组归并方式
SELECT name, MAX(score) as max_score FROM t_score_java GROUP BY name ORDER BY name ASC;
SELECT name, MAX(score) as max_score FROM t_score_go GROUP BY name ORDER BY name ASC;
SELECT name, MAX(score) as max_score FROM t_score_python GROUP BY name ORDER BY name ASC;
```

![052](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6847e33791e44c34898a0c60780625c5~tplv-k3u1fbpfcp-zoom-1.image)

通过下一张图你们可以看到，当进行第一次 next 调用时，按照当前游标所指记录的 name 升序排列，排在队列首位的 t_score_java 分片将会被弹出队列，并且将 name 同为“Jetty”的其他分片结果集中的数据一同弹出队列。在获取了所有的 name 为“Jetty”的同学的分数之后，找出最大值，得到“Jetty”的单科最高分。与此同时，所有的分片结果集中的游标都将下移至数据值“Jetty”的下一个不同的数据值，并且根据分片结果集的当前游标所指记录的 name 值进行重排序。因此，包含名字“John”的相关数据结果集则排在的队列的前列。

显然，这一过程属于流式归并。

![053](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/861aa2b4ef804c33b2ccdd3b7975e61c~tplv-k3u1fbpfcp-zoom-1.image)

以上是 MAX 函数的聚合方式，MIN 函数的聚合方式类似，不再赘述。

累加类型的聚合函数是指 SUM 和 COUNT。它们需要将每一个同组的结果集数据进行累加，在前面那个“获取每位考生的总分并按照分数从高至低排序”的实例中你们已经见识过了，不再赘述。这一过程可以流式归并方式。

求平均值的聚合函数只有 AVG。这必须通过 SQL 改写出的 SUM 和 COUNT 进行计算，相关内容已在 SQL 改写的内容中涵盖，不再赘述。这一过程可以流式归并方式。

无论是流式分组归并还是内存分组归并，对聚合函数的处理都是一致的，因此，聚合归并是在之前介绍的归并过程之上追加的归并能力，即装饰。实际上我的创造者正是通过装饰者模式赋予我聚合归并能力的。

5）分页归并

上文所述的所有归并类型都可能进行分页。分页也是追加在其他归并类型之上的装饰过程，我的创造者通过装饰者模式赋予我对数据结果集的分页能力。若逻辑 SQL 要查询第 M 页的数据，查询结果集会包含 N（N=路由后的 SQL 数量）个页的数据，分页归会将无需获取的数据过滤掉，最终得到逻辑表的第 M 页的数据。

在分片场景中，将 LIMIT 10000000, 10 改写为 LIMIT 0, 10000010，才能保证其数据的正确性，这一点我在 SQL 改写的分页修正部分讲过。我的分页功能比较容易让使用者误解，用户通常认为分页归并会占用大量内存。用户非常容易产生我会将大量无意义的数据加载至内存中，造成内存溢出风险的错觉。其实，通过流式归并的原理可知，会将数据全部加载到内存中的只有内存分组归并这一种情况。除了内存分组归并这种情况之外，其他情况都可以通过流式归并获取数据结果集，因此我会通过结果集的 next 方法将无需取出的数据全部跳过，并不会将其存入内存。

但同时需要注意的是，由于排序的需要，大量的数据仍然需要传输到我所在 JVM 的内存空间（只不过我丢掉无用的数据，如上段所述）。因此，采用 LIMIT 这种方式分页，并非最佳实践。由于 LIMIT 并不能通过索引查询数据，因此如果可以保证 ID 的连续性，通过 ID 进行分页是比较好的解决方案，例如：

```sql
SELECT * FROM t_order WHERE id > 100000 AND id <= 100010 ORDER BY id;
```

或通过记录上次查询结果的最后一条记录的 ID 进行下一页的查询，例如：

```sql
SELECT * FROM t_order WHERE id > 10000000 LIMIT 10;
```

# 5. 结束语

我是 Sharding-JDBC，一个数据库水平分片中间件。当你们把逻辑 SQL 交给我处理时，作为中间件，我把 SQL 解析、路由、改写、执行、归并的复杂工作统统对你们屏蔽了。而你们要做的就是执行数据库和数据表水平拆分（无论是手动拆分还是自动化拆分均可，不过拆分是你们的工作，不是我的）、实现我提供的分片算法接口，告诉我怎么根据分片键的值找到对应的分片、在配置文件或者配置 API 中描述分片策略。

再让你们看一眼我提供的各种 ShardingAlgorithm 接口中的 doSharding()方法吧，这是你们使用我时接触得最多的一个方法，这也是你们使用我时唯一需要动脑筋的地方：

```java
/**
* 所有的分片算法 interface 都包含该方法
*
* @param 所有可能的分片表(或分片库)名称
* @param 分片键的值
* @return 根据分片键的值，找到对应的分片表(或分片库)名称并返回
*/
Collection<String> doSharding(
    Collection<String> availableTargetNames, 
    ComplexKeysShardingValue<T> shardingValue
);
```

我并非法力无边，我还有很多局限。在单片路由和多片路由的场景下，我全面支持 DML、DDL、DCL、TCL 和部分 DAL，支持分页、去重、排序、分组、聚合、不跨数据库的关联查询等操作。但在多片路由的场景下，我不支持 HAVING、UNION 等操作，对子查询的支持也有限。其他种种细节，一篇文章，难以详述。

我是 Sharding-JDBC，关于我的基本用法和基本原理，我说完了，你秃了吗？

最后欢迎大家关注我的公号，加我好友:「geekoftaste」,一起交流，共同进步！

![](https://user-gold-cdn.xitu.io/2019/12/29/16f51ecd24e85b62?w=1002&h=270&f=jpeg&s=59118)
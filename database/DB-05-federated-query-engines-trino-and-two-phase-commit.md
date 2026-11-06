# DB-05 什么是联邦查询引擎？Trino 与 2PC 有什么区别？

**标签**

`#Database` `#FederatedQuery` `#Trino` `#DistributedSQL` `#SQLServer` `#PostgreSQL` `#MongoDB` `#TwoPhaseCommit` `#XA` `#DistributedTransaction`

## 问题

联邦查询引擎如何用一条 SQL 查询多个数据库？Trino 是什么，与服务聚合模式有什么区别？SQL Server 有哪些跨库查询能力？跨库 JOIN 与 2PC 分别解决什么问题？SQL Server、PostgreSQL、MongoDB 能否直接加入同一个外部 2PC 事务，协调器应如何选择？

## 面试回答

联邦查询引擎提供统一的 SQL 入口，通过连接器访问多个独立数据源，规划查询、尽可能下推过滤等操作，并完成跨源 JOIN 和聚合。应用提交一条 SQL，底层仍可能执行多个远端请求和分布式计算任务。

Trino 是开源的分布式 SQL 查询引擎，主要用于分析查询、报表和跨数据源查询。业务数据保存在原数据库或数据湖中；Trino 通过 Connector 和 Catalog 接入数据源，由 Coordinator 规划和调度，再由 Worker 并行执行。

2PC（Two-Phase Commit，两阶段提交）是协调分布式事务提交的协议。它要求参与者先准备，再遵从协调器持久化的提交或回滚决定，用于实现跨资源事务的原子提交。

因此，跨库查询能力与跨库事务能力应分别判断。一条跨库 SQL 不自动意味着多个数据库能够原子提交，也不自动提供跨数据源的统一时间点快照。2PC 本身也不单独决定事务的隔离级别。

异构数据库是否能参加同一个 2PC，取决于数据库、驱动和协调器的协议及恢复支持。SQL Server 和 PostgreSQL 提供相应的外部协调能力；MongoDB 内部事务使用 2PC，并不等于它提供原生 XA 参与者接口。

## 详细解释

### 1. Federation 的不同语境

在部分系统设计资料中，Federation 指按业务功能拆库，例如用户库、订单库、商品库，也称 Functional Partitioning。

在联邦数据库或联邦查询语境中，重点是为多个独立数据源提供统一访问方式。一个查询可以关联多个来源的数据。这些来源可以因为业务拆分而独立，也可以原本就是不同系统。

此外，“垂直分区”也有不同用法：

| 用法 | 含义 |
| --- | --- |
| 垂直分表 | 按列拆分一个实体，通常保留共同的关联键 |
| 垂直分库 / 功能分区 | 按业务领域将数据放入不同数据库 |
| 联邦查询 | 通过统一查询入口访问多个独立数据源 |

按业务拆库与提供统一查询入口可以组合，但不能仅凭“Federation”这个名称推断系统已经具备跨库 SQL 能力。

### 2. 一条 SQL 如何查询多个数据库？

首先配置数据源的连接信息、认证方式、访问权限及表的映射。引擎据此知道逻辑表名对应哪个物理数据源。

假设在 Trino 中配置两个 Catalog：

| Trino 表名 | 实际数据源 |
| --- | --- |
| pg.public.users | PostgreSQL 某个已配置数据库中，public Schema 下的 users 表 |
| mysql.shop.orders | MySQL 的 shop 数据库中的 orders 表 |

pg 和 mysql 是本例配置的 Catalog 名称，可以自行命名。它们不是要求源数据库必须使用的固定名称。

假设两张表可以通过用户 ID 关联，应用向 Trino 提交：

```sql
SELECT
    u.id AS user_id,
    u.name,
    o.id AS order_id,
    o.total
FROM pg.public.users AS u
JOIN mysql.shop.orders AS o
    ON u.id = o.user_id
WHERE u.id = 123;
```

以上是说明表名和关联方式的示例，需要先存在相应连接配置、表结构和权限，不是实测执行结果。

其过程可以概括为：

```text
应用 / BI 工具提交一条 SQL
            ↓
Trino 解析、检查元数据、优化执行计划
            ├─ PostgreSQL Connector → 获取用户数据
            └─ MySQL Connector      → 获取订单数据
            ↓
执行跨源 JOIN、投影及其他处理
            ↓
返回结果集
```

连接器负责适配源系统的访问接口。对于 SQL 数据源，可以下发适合该系统的 SQL；对于其他类型的数据源，则使用对应接口。

过滤、投影、聚合或部分 JOIN 能否下推，由连接器能力和优化器决定。跨源 JOIN 通常需要在查询引擎中交换并处理数据。

统一 SQL 入口通常不要求预先把所有业务数据复制到 Trino，但查询执行仍有网络传输、中间结果和计算开销。

### 3. Trino 的角色与组成

Trino 将存储与查询计算分开。数据保存在源数据库、数据湖等系统中，Trino 负责规划和执行分布式 SQL。

| 组件 | 职责 |
| --- | --- |
| Coordinator | 接收 SQL、生成执行计划、调度任务并组织结果返回 |
| Worker | 读取数据、执行 JOIN / 聚合等任务，并交换中间结果 |
| Connector | 适配特定数据源的访问方式与类型映射 |
| Catalog | 某个数据源的具体配置，包括连接器、连接地址及认证等 |

Trino 的分布式能力体现在一条查询可以拆成多个任务，由多个 Worker 并行处理。

常见用途包括：

- 报表、BI 和临时分析。
- 大规模过滤、JOIN、聚合。
- 联合查询关系型数据库、部分 NoSQL 数据源和数据湖中的表。

Trino 主要面向分析工作负载。订单创建、库存扣减等 OLTP 业务事务通常仍由事务数据库负责。Trino 也可以通过支持的连接器执行写操作和事务，具体能力与范围需要按连接器核对，不能把它概括为只能 SELECT。

### 4. 与服务聚合模式有什么区别？

服务聚合模式由应用服务分别调用其他服务或数据库，再用业务代码组合面向用户的模型。

```text
聚合服务
   ├─ 用户服务 / 用户库
   └─ 订单服务 / 订单库
            ↓
应用代码组合成页面 DTO
```

联邦查询将 SQL 的解析、跨源访问与关系运算交给查询引擎：

```text
应用 → 一条 SQL → 联邦查询引擎 → 多个数据源
                         ↓
                      结果集
```

| 对比 | 服务聚合 | 联邦查询 |
| --- | --- | --- |
| 输入 | 业务请求及参数 | SQL |
| 数据获取 | 多次 API 调用或数据库查询 | 引擎通过连接器访问数据源 |
| 组合逻辑 | 聚合服务中的业务代码 | 查询计划中的 JOIN、聚合等算子 |
| 常见输出 | 面向页面或业务的 DTO | 查询结果集 |

两者可以配合使用：聚合服务内部执行联邦查询，再将结果转换为 DTO。查询引擎并不会自动替代各业务服务的业务规则。

### 5. SQL Server 如何跨库查询？

同一个 SQL Server 实例内，可以使用三段名称访问不同数据库的表：

```sql
SELECT u.Name, o.OrderId, o.Total
FROM UserDB.dbo.Users AS u
JOIN OrderDB.dbo.Orders AS o
    ON u.UserId = o.UserId;
```

它要求数据库、表及相应访问权限已配置，使用的是同一实例内的跨库名称解析能力。

如果订单库位于另一个实例，可以先配置 Linked Server，再使用四段名称：

```sql
-- 假设 OrderServer 已配置为链接服务器，
-- 且远端连接、登录映射和数据访问权限已经就绪。
SELECT u.Name, o.OrderId, o.Total
FROM UserDB.dbo.Users AS u
JOIN OrderServer.OrderDB.dbo.Orders AS o
    ON u.UserId = o.UserId;
```

四段名称为：

```text
链接服务器名.数据库名.Schema名.对象名
```

SQL Server 根据链接服务器定义访问远端数据并执行分布式查询。某些异构数据源也能通过合适的 OLE DB Provider 接入，具体能力需要验证。

这里主要讨论 SQL Server 数据库引擎。Azure SQL Database 不提供 Linked Server，不能直接套用这一示例。

Linked Server 解决访问远端数据的问题；需要跨实例事务时，还要单独配置并核对分布式事务支持。

### 6. Trino 与 2PC 分别协调什么？

| 对比 | Trino / 联邦查询 | 2PC |
| --- | --- | --- |
| 类型 | 查询引擎及其执行机制 | 分布式事务提交协议 |
| 核心目标 | 统一访问并计算多个数据源的数据 | 让事务参与者遵从同一提交或回滚决定 |
| 典型操作 | 跨库 SELECT、JOIN、聚合 | 跨库事务的 Prepare、Commit、Rollback |
| 协调者职责 | 规划查询、调度任务、组织结果 | 记录参与者和决议，推进提交及故障恢复 |
| 主要能力边界 | 受连接器、源系统和执行计划影响 | 受资源管理器、驱动、日志和恢复协议影响 |
| 是否自动保证跨库原子写入 | 不能由跨库查询能力推断 | 协议及恢复条件成立时提供原子提交 |
| 是否自动保证统一读取快照 | 不能由一条 SQL 推断 | 原子提交本身也不单独提供该保证 |

例如：

```text
报表需求：
    读取用户信息 + 读取订单 → JOIN → 统计结果
    主要需要跨源查询能力

下单需求：
    订单库创建订单 + 库存库扣减库存
    若要求共同提交，需要相应的跨库事务保证
```

SELECT、JOIN 与原子提交可以出现在同一个系统中，但需要分别设计。不能通过一个查询引擎支持 COMMIT，就推断它支持任意数据库组合的 XA 事务。

### 7. 2PC 的两个阶段与恢复

前提是各参与者能够加入同一个全局事务，并支持准备状态及恢复。各库的业务修改先在尚未最终提交的本地事务中执行。

第一阶段：准备。

```text
事务协调器
   ├─ 请求订单库 Prepare → 准备完成，投票 Yes
   └─ 请求库存库 Prepare → 准备完成，投票 Yes
```

参与者需要保证投票 Yes 后，即使进程重启，也能恢复到等待最终决议的状态。准备阶段可能继续持有事务锁等资源。

第二阶段：决议。

```text
全部参与者准备成功
       ↓
协调器持久化全局 COMMIT 决定
       ↓
通知各参与者提交
       ↓
重试未完成的通知，直到各参与者完成决议
```

如果提交条件没有满足，协调器可以在作出全局提交决定前选择回滚。全局 COMMIT 决定一旦可靠记录，就应继续完成该决定，不能因某个节点暂时不可达而改成回滚。

已准备的参与者如果无法得知最终决定，可能需要等待协调器或其恢复机制，不能为了释放锁就随意猜测回滚。这是经典 2PC 的阻塞问题之一。

2PC 不表示所有数据库在完全相同的瞬间完成提交，也不单独提供全局可串行化或跨库读取快照。隔离与原子提交需要分别考虑。

### 8. 异构数据库如何选择 2PC 协调器？

选择顺序是先检查资源与驱动的事务接口、准备状态持久化和恢复能力，再按技术栈选择事务管理器。

XA 是事务管理器与资源管理器之间的标准接口规范，覆盖准备、提交、回滚以及未决事务恢复等操作。

| 数据库 | 相关能力 | 需要确认的条件 |
| --- | --- | --- |
| SQL Server | DTC，以及通过相应驱动提供的 XA 接入 | 服务端 DTC / XA 配置、驱动及部署环境 |
| PostgreSQL | Prepared Transactions；JDBC 提供 PGXADataSource | 启用所需准备事务容量，配置驱动及恢复访问 |
| MongoDB | 自身集群内的事务；没有原生 XA 参与者接口 | 不能直接假定它可加入外部 XA / MSDTC 事务 |

PostgreSQL 的 PREPARE TRANSACTION 面向外部事务管理器。生产方案应让协调器跟踪并及时完成已准备事务，不能仅发出 PREPARE 后就丢弃其状态。

常见协调器路径：

- **Java / JTA / XA**：可以评估 Narayana、Atomikos 等事务管理器，接入兼容的 XA 驱动并配置恢复。
- **Windows / .NET**：对于支持加入相应分布式事务的资源，可以评估 MSDTC，通过 System.Transactions / TransactionScope 接入。

TransactionScope 是事务编程接口。数据库驱动是否自动加入、事务是否升级为分布式事务、资源能否恢复，都需要具体支持。

特别是 Npgsql 的官方文档说明：它对 .NET 分布式事务提供部分支持，没有实现恢复部分，并不建议完全依赖该机制。因此，不能把正常提交测试通过视为已验证故障恢复。

### 9. MongoDB 内部 2PC 与外部协调的区别

MongoDB 的跨分片事务可由其内部协调器执行 2PC，参与者是 MongoDB 集群内部的分片。其内部 prepareTransaction 命令，不是一个可直接当作通用 XA 接口使用的应用 API。

因此，以下两种能力不能画等号：

```text
MongoDB 内部：
    MongoDB 协调器 → MongoDB 分片 A、B

异构外部事务：
    外部协调器 → SQL Server + PostgreSQL + MongoDB
```

某些产品提供针对非 XA 资源的特殊整合。例如 Oracle MicroTx 文档介绍 LLR（Logging Last Resource），允许一个非 XA 资源参与全局事务，并以 MongoDB 为例。

这种机制依赖额外适配、将提交记录与本地业务事务一起保存等恢复要求；具体三库及驱动组合必须按产品支持范围验证。它不意味着 MongoDB 提供了原生 XA 接口。

还要区分 LLR 与 LRC：MicroTx 文档明确说明 LRC 不提供与 XA 相同的 ACID 保证，不能把两者作为同等可靠的替代方案。该产品对这类整合还限制为一个非 XA 参与者。

如果无法建立满足要求的原子事务，可以：

- 将必须原子修改的数据集中到同一个事务数据库。
- 若其他数据库仅提供查询模型，通过 Outbox、消息或 CDC 更新它们。
- 若多个业务参与者需要独立提交且允许最终一致性，使用持久化 Saga、幂等操作和必要的业务补偿。

Saga 的业务补偿不是数据库级原子回滚。相关内容参见 [DIST-01 多数据库更新中的宕机恢复与 Saga](../distributed-systems/DIST-01-multi-db-crash-recovery-saga.md)；可靠事件发布参见 [MQ-02 Outbox 与 Publisher Confirm](../message-queue/MQ-02-outbox-and-publisher-confirm.md)。

## 故障场景

### 场景 1：跨源 JOIN 拉取了大量数据

源数据库过滤不足，或连接器无法下推相关操作，导致大量数据跨网络传输，并在查询引擎中交换。

应检查执行计划、数据源扫描量、下推能力及中间数据量。跨源 JOIN 的统一语法不会消除传输和计算成本。

### 场景 2：一个数据源变慢或不可达

跨源查询可能因源数据库、网络或连接器故障而超时或失败。

需要区分源系统执行、远端访问、Trino 调度及计算耗时；不能只增加 Worker 就认为远端瓶颈一定消失。

### 场景 3：把顺序提交多个本地事务当成 2PC

```text
提交订单库成功
       ↓
提交库存库前应用宕机
       ↓
订单已提交，库存未修改
```

分别调用多个数据库的 BeginTransaction 和 Commit，不会自动构成 2PC。需要资源共同加入受支持的全局事务协议，或者采用具有不同一致性语义的业务流程。

### 场景 4：2PC 已准备，但协调器崩溃

参与者可能保持未决状态并继续持锁。正常路径可用，不代表协调器和资源在重启后能完成恢复。

应验证协调器的持久化日志、资源恢复接口和未决事务处理。已有可靠 COMMIT 决定时，应继续提交；不能仅根据超时就改为回滚。

### 场景 5：一条查询被误认为统一时间点快照

不同源数据库可能使用不同快照、隔离级别或落后的只读副本。关联结果是否对应统一读取时点，不能只由“一条 SQL”判断。

对相关业务保证，需要单独验证引擎、连接器、源系统和副本读取策略。2PC 的原子提交也不能单独替代这项读取保证。

## 面试追问

- 功能分区与联邦查询为什么可能都被称为 Federation？
- 联邦查询中的逻辑表名如何映射到物理数据源？
- Trino 的 Coordinator、Worker、Connector 和 Catalog 各负责什么？
- 为什么一条 SQL 仍可能产生多个远端请求？
- 什么是过滤、投影和 JOIN 下推？为什么不能假定全部可以下推？
- 服务聚合模式与联邦查询能否结合使用？
- SQL Server 的三段名称和 Linked Server 四段名称分别适用于什么情况？
- Trino 为什么主要面向分析查询，而非通用 OLTP？
- 查询协调器与 2PC 事务协调器的职责有什么区别？
- 2PC 的 Prepare 与最终 Commit 有什么区别？
- 为什么已准备事务不能仅因超时就随意回滚？
- 2PC 是否自动保证全局可串行化或统一读取快照？
- SQL Server、PostgreSQL、MongoDB 能否直接参加同一个 XA 事务？
- TransactionScope、数据库驱动和 MSDTC 分别承担什么角色？
- 为什么异构数据库选型必须验证故障恢复，而不只验证正常提交？
- Saga / Outbox 与 2PC 的一致性保证有什么区别？

## 关键结论

- 联邦查询提供统一 SQL 入口，由引擎访问多个数据源并完成关系运算。
- Trino 是分布式 SQL 查询引擎，主要面向分析和跨源查询；数据仍由源系统存储。
- 一条 SQL 不代表底层只有一个请求，也不代表查询没有跨网络数据交换。
- 服务聚合由应用代码组合结果，联邦查询由查询引擎执行关系运算，两者可以配合。
- Linked Server 等跨库访问能力与分布式事务能力需要分别配置和验证。
- Trino 与 2PC 分别关注查询计算和事务原子提交，不是互相替代的方案。
- 2PC 依赖可恢复的准备状态、可靠决议和最终完成机制，故障期间可能阻塞。
- 跨库原子提交、全局隔离和读取快照是不同的保证。
- 异构 2PC 需要兼容的数据库、驱动与协调器；MongoDB 内部事务不能直接等同于外部 XA。
- 特殊非 XA 整合要按协议和恢复条件评估；Saga / Outbox 也不能被描述为原子提交。

## 参考资料

- [Trino：分布式 SQL 查询引擎与联邦查询](https://trino.io/)
- [Trino：Coordinator、Worker、Connector、Catalog 与执行模型](https://trino.io/docs/current/overview/concepts.html)
- [Trino：分析场景及与通用事务数据库的定位差异](https://trino.io/docs/current/overview/use-cases.html)
- [Trino：PostgreSQL 连接器的查询、写入与下推能力](https://trino.io/docs/current/connector/postgresql.html)
- [Trino：START TRANSACTION](https://trino.io/docs/current/sql/start-transaction.html)
- [Microsoft：Linked Server 与四段名称](https://learn.microsoft.com/en-us/sql/relational-databases/linked-servers/linked-servers-database-engine?view=sql-server-ver17)
- [Microsoft：两阶段提交](https://learn.microsoft.com/en-us/dotnet/framework/data/transactions/committing-a-transaction-in-single-phase-and-multi-phase)
- [PostgreSQL：PREPARE TRANSACTION 与准备状态](https://www.postgresql.org/docs/current/sql-prepare-transaction.html)
- [Microsoft：SQL Server JDBC 的 XA 接入](https://learn.microsoft.com/en-us/sql/connect/jdbc/understanding-xa-transactions?view=sql-server-ver17)
- [PostgreSQL JDBC：PGXADataSource](https://jdbc.postgresql.org/documentation/publicapi/org/postgresql/xa/PGXADataSource.html)
- [Npgsql：System.Transactions 与分布式事务的恢复限制](https://www.npgsql.org/doc/basic-usage.html#systemtransactions-and-distributed-transactions)
- [Atomikos：XA 接口、2PC 与恢复](https://www.atomikos.com/Documentation/WhatIsXa)
- [Narayana：事务管理与 XA 恢复](https://www.narayana.io/docs/project/index.html)
- [MongoDB：内部跨分片 2PC 与准备事务](https://github.com/mongodb/mongo/blob/master/src/mongo/db/repl/README.md)
- [Oracle MicroTx：单个非 XA 资源的 LLR / LRC 整合及保证差异](https://docs.oracle.com/en/database/oracle/transaction-manager-for-microservices/26.1/tmmdv/logging-last-resource-transaction.html)
- [Microsoft：垂直分区与功能分区](https://learn.microsoft.com/en-us/azure/well-architected/design-guides/partition-data)
- [System Design Primer：Federation / Functional Partitioning 的用法](https://github.com/donnemartin/system-design-primer#federation)

# DB-08 主从复制实现 Failover 需要哪些节点与组件？

**标签**

`#Database` `#Replication` `#Failover` `#PrimaryReplica` `#Quorum` `#Witness` `#SQLServer` `#MongoDB` `#Patroni` `#HighAvailability`

## 问题

为了实现主从故障切换，通常有哪些节点和组件？主库、从库、投票成员、仲裁或见证、HA 管理组件与访问入口分别负责什么？它们是否必须各占一台机器？为什么有的部署是三个数据节点，有的是两个数据库节点加见证，还有的需要独立协调服务？

## 面试回答

主从复制通常先区分主库与从库两类数据角色，再提供故障检测、仲裁、角色切换、隔离与客户端访问入口等高可用能力。

这些能力不一定对应互相独立的服务器。主库和从库本身可以是投票成员；监控和切换组件可以与数据库同机；见证也可能是文件共享或云资源，而不是第三台数据库。

常见拓扑包括：三个数据成员组成的主从副本集，或在产品支持时使用两个数据成员加仲裁或见证。使用独立协调服务时，数据库副本数与协调服务成员数要分别计算。

投票决定选主或服务资格，数据复制确认决定写入的持久性条件。纯仲裁不保存业务副本，也不能提升成主库，所以“能选出新主”不等于“满足所有写入确认要求”。

## 详细解释

### 1. 应按角色理解，而不是固定列出几台服务器

| 角色或组件 | 职责 | 是否保存业务副本 | 是否一定单独部署 |
| --- | --- | --- | --- |
| Primary / 主库 | 接受业务写入，向副本发送修改 | 是 | 一个数据库实例承担该角色 |
| Secondary / Standby / 从库 | 保存和应用副本数据，满足资格后可提升 | 是 | 一个或多个数据库实例 |
| 投票成员 | 按协议参与选主、成员或提交协调 | 可以保存数据，也可以仅承担特定仲裁角色 | 可能与数据节点重叠 |
| Arbiter / Witness / 仲裁或见证 | 在产品支持时帮助仲裁 | 通常不保存完整业务副本 | 可能是节点，也可能是资源 |
| HA 管理组件 | 健康监控、租约维护、角色切换及隔离协作 | 其角色不要求保存业务副本 | 经常运行在数据库主机上 |
| Listener / VIP / 代理 | 为客户端提供有效主库的访问入口 | 否 | 可能是逻辑资源或独立服务 |

它们不是六类互斥的物理服务器。一个节点可以同时运行数据库与 HA Agent，也可以同时具有数据成员和投票成员身份。

数据副本是否可读、能否晋升、有没有投票权，需要按产品、版本和成员配置判断。不是每个复制目标都必然能自动成为新主。

### 2. 主库与从库分别负责什么？

主库接收正常业务写入，从库接收并应用复制数据。

```text
客户端写入
     ↓
Primary
     ├─ 复制 → Secondary 1
     └─ 复制 → Secondary 2
```

故障切换时，合格从库完成必要恢复并取得主角色。旧主恢复后，按产品要求校验并同步，以从库角色重新加入。

复制本身不一定自动提供监控、选主、旧主隔离和客户端重连；这些能力需要数据库自身或配套高可用组件完成。

从库用于承接读取时，还要单独判断复制延迟和读取一致性。写后读取策略参见 [DB-04 读己之写与读主库窗口](DB-04-read-your-writes-and-primary-routing-window.md)。

### 3. 仲裁或见证是否就是第三个数据库？

不一定。

一个完整数据副本可以参与投票，也可以在满足条件时接替主库。纯仲裁或见证则通常不保存业务数据，也不具备接替数据主库的能力。

仲裁的作用是按协议决定哪个成员或分区具备继续服务的资格。它不一定是每次写入都必须单独联系的中央审批者。

如果采用固定成员、每节点一票的多数派模型，三个投票成员通常需要两票；实际产品也可能使用动态仲裁、租约或独立协调服务。不能把某一种模型直接推广到所有部署。

相关票数与网络分区示例参见 [DB-06 主主复制、脑裂与多数派仲裁](DB-06-multi-primary-split-brain-quorum-and-network-partitions.md)。

### 4. 常见拓扑一：三个完整数据成员

MongoDB 的常见三成员副本集可以是：

```text
Primary + Secondary 1 + Secondary 2

三个成员都保存数据
正常投票配置下，共同参与选举
符合资格的 Secondary 可以接替 Primary
```

MongoDB 官方推荐三名数据成员的拓扑。它保留更多数据副本，但实际写入与切换保证仍取决于复制、写关注和成员状态。

此例用于说明主从副本集的角色组合，不代表所有数据库都采用 MongoDB 的选举协议或成员规则。

### 5. 常见拓扑二：两个数据成员加仲裁或见证

在产品支持的情况下，可以采用：

```text
Primary + Secondary + Arbiter / Witness
   数据       数据          仲裁
```

MongoDB 的 Primary-Secondary-Arbiter（PSA）拓扑中，Arbiter 参加选举、不保存数据集，也不能成为 Primary。

这意味着：

- 主库不可用时，剩余数据成员可能借助仲裁取得选主资格。
- 仲裁节点不能补齐缺失的业务数据，也不能确认一份它没有保存的数据。
- 要求多份数据确认的写入仍可能无法完成。

MongoDB 官方文档特别说明，PSA 的数据从节点不可用或落后时，majority 写关注可能出现可用性或性能问题。不能把“还可以选主”简单等同于“所有写入照常完成”。

此外，MongoDB Arbiter 与 Windows 集群见证是不同产品机制，只是在角色说明上都属于非完整业务数据副本，不能交换使用。

### 6. SQL Server Always On AG（Windows）的例子

常见部署可以包含：

```text
数据库主机 A：
    SQL Server 主副本
    WSFC 相关组件

数据库主机 B：
    SQL Server 辅助副本
    WSFC 相关组件

见证资源：
    文件共享见证或云见证等
    不保存该业务数据库的数据副本
```

这种部署可以只有两个 SQL Server 数据库节点，不要求额外安装第三台 SQL Server 作为见证。

WSFC 在各集群节点上运行相关服务，见证提供辅助仲裁信息或投票。较新的 Windows Server 还会动态管理节点与见证的票，不能将所有场景都简化为永远固定的三票。

SQL Server 与集群还需要协调主角色。是否满足自动切换条件，要结合副本同步状态、集群仲裁和故障策略判断。

见证不会作为第三个数据副本接管业务数据库，也不需要为每笔 SQL 事务提供数据落盘确认。

### 7. 常见拓扑三：数据库与协调服务分开

PostgreSQL + Patroni 可以采用两个数据库节点：

```text
数据库节点 A：
    PostgreSQL Primary + Patroni

数据库节点 B：
    PostgreSQL Standby + Patroni

独立 DCS：
    例如三个 etcd 成员
```

Patroni 官方文档说明，数据库节点与 DCS 节点可以解耦。一主一从的数据库配置可以支持自动切换，而 DCS 为了共识和容错通常使用三个或五个成员。

这里应分别计算：

| 对象 | 示例数量 | 承担的工作 |
| --- | --- | --- |
| PostgreSQL 数据库实例 | 2 | 数据复制和业务访问 |
| Patroni 进程 | 2 | 在对应数据库主机上执行 HA 控制 |
| DCS 成员 | 3 | 维护协调状态并执行自身共识协议 |

这是逻辑成员示例，不是固定要求五台完全分离的物理机器。若合机部署，需要同时计算共享主机故障对不同成员集合的影响。

不能把两个 PostgreSQL 实例加一个 DCS 服务入口，直接算成同一个三票组。DCS 服务本身有自己的成员与仲裁规则。

### 8. HA 管理组件和 Fencing 由谁承担？

实际实现可能是：

- 数据库内置选举与成员管理机制。
- SQL Server 配合 WSFC 或 Linux 下相应的集群管理器。
- PostgreSQL 配合 Patroni 及其 DCS。
- 配套的 Watchdog、电源或虚拟机管理等隔离设施。

纯见证的职责不等于执行主库关闭命令。故障检测、角色变更和节点隔离由相应的高可用协议与组件协作完成。

主库假死时，还需要可靠防止旧主恢复后继续有效写入。相关过程参见 [DB-07 主库假死、租约与 Fencing](DB-07-primary-hang-leases-fencing-and-safe-failover.md)。

### 9. 客户端入口是否也算一个节点？

Listener、VIP、代理或服务发现负责让客户端找到有效主库。它们的物理部署方式不同：

- Listener / VIP 可以是集群管理的逻辑资源。
- 代理可能作为独立进程或多个实例运行。
- 客户端驱动也可能具备拓扑发现和重连能力。

这些入口通常不等于仲裁成员。入口认为某个地址可达，也不代表该数据库一定拥有合法主角色。

故障切换后，已有连接和进行中事务通常需要由驱动与应用正确处理；入口切换并不自动重放业务操作。

## 故障场景

### 场景 1：把三种角色理解成必须三台服务器

把主库、从库、HA Agent 分别当成三个独立物理节点，忽略 Agent 通常与数据库同机运行。

应先识别数据、仲裁、控制和访问入口的职责，再映射到实例、进程及物理故障域。

### 场景 2：只部署主从复制，没有完整切换机制

从库能收到数据，却没有角色管理、旧主隔离和有效客户端入口。

复制能力并不自动包含完整自动 Failover。需要确认这些高可用功能分别由谁承担。

### 场景 3：仲裁正常，但缺少数据确认

见证帮助某个数据节点取得主角色，但其他数据副本不可用。

如果写入要求额外数据副本确认，仍可能超时或无法完成。见证不替代数据副本。

### 场景 4：两个数据库互通，但外部协调服务失去仲裁

两个数据库节点都在线，却不能维持产品要求的外部领导资格。

数据库节点数不能直接替代 DCS 多数派；应按外部协调服务的成员与租约机制处理。

## 面试追问

- 为什么节点角色、进程数量和物理服务器数量不是同一概念？
- 主库、从库是否可以同时也是投票成员？
- 所有从库都一定能自动提升为主库吗？
- Arbiter / Witness 为什么不能代替完整业务数据副本？
- 能选出新主，是否就能完成 majority 写入？
- HA Agent 是否必须运行在独立服务器上？
- SQL Server 的文件共享见证或云见证是否需要第三台 SQL Server？
- Windows 动态仲裁与固定成员多数派模型有什么区别？
- Patroni 的数据库成员与 DCS 成员为什么要分别计算？
- Listener、VIP、代理是否参与数据库选主？
- 谁负责防止旧主恢复后继续写入？
- 自动故障切换需要哪些超出数据复制本身的能力？

## 关键结论

- 主从架构的数据角色主要是主库与从库，高可用还需要检测、仲裁、切换和隔离能力。
- 投票是角色，未必是一类独立机器；数据节点可以同时承担投票。
- 见证通常不保存完整业务副本，不能晋升为数据主库或提供数据复制确认。
- HA 管理组件与访问入口也不一定各自独占一个节点。
- 三个数据成员与两个数据成员加见证，具有不同的数据保护和写入可用性条件。
- SQL Server Windows 集群的见证可以是资源，不必是第三台数据库。
- 外部协调服务有自己的成员和多数派，不能与数据库实例数量混算。
- 有复制、有选主资格、能确认写入和能让客户端重连，是需要分别验证的能力。

## 参考资料

- [MongoDB：三成员副本集、P-S-S 与 P-S-A](https://www.mongodb.com/docs/manual/core/replica-set-architecture-three-members/)
- [MongoDB：Arbiter 的职责与 majority 写入限制](https://www.mongodb.com/docs/manual/core/replica-set-arbiter/)
- [Microsoft：Windows 集群见证与动态仲裁](https://learn.microsoft.com/en-us/windows-server/failover-clustering/deploy-quorum-witness)
- [Microsoft：SQL Server AG 的数据库与副本角色](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server?view=sql-server-ver17)
- [Microsoft：AG 的自动切换条件](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-and-failover-modes-always-on-availability-groups?view=sql-server-ver17)
- [Patroni：数据库节点与 DCS 节点规划](https://patroni.readthedocs.io/en/latest/README.html#planning-the-number-of-postgresql-nodes)
- [Patroni：Watchdog 保护](https://patroni.readthedocs.io/en/latest/watchdog.html)
- [Microsoft：SqlClient 的重连与事务处理边界](https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/sqlclient-support-high-availability-disaster-recovery?view=sql-server-ver17)

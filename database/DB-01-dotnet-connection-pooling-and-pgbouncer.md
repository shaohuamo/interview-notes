# DB-01 .NET 数据库连接池与 PgBouncer 如何管理和复用连接？

**标签**

`#Database` `#DotNet` `#ADONET` `#ConnectionPooling` `#EFCore` `#PostgreSQL` `#PgBouncer` `#SQLServer` `#MySQL`

## 问题

在 .NET 中，数据库连接池由框架、数据库驱动还是第三方中间件管理？使用 PgBouncer 后，多个应用实例如何共享较少的数据库连接？客户端连接、后端连接、连接池个数与 Pool Size 分别是什么意思？SQL Server 和 MySQL 有哪些相关代理？

## 面试回答

.NET 中的数据库连接池通常由 **ADO.NET 数据库驱动**管理，例如 SQL Server 的 `Microsoft.Data.SqlClient`、PostgreSQL 的 `Npgsql`。EF Core 使用底层驱动的连接池，自身的 `DbContext` 池则是另一种对象复用机制。

驱动连接池通常位于应用进程内，多个应用实例各自维护自己的池。若 20 个实例每个只有一个相关池，且每池最大 100 条连接，合计最多可能建立 2,000 条连接；并不代表已经建立 2,000 条。

PgBouncer 是 PostgreSQL 的独立连接池代理：

```text
应用中的 Npgsql 连接池
          ↓ 客户端连接
       PgBouncer
          ↓ 后端连接
      PostgreSQL
```

采用 Transaction pooling 时，后端连接在事务期间分配给客户端，事务结束后可供其他客户端复用。因此在后端总额度为 100 的配置下，多个应用可以共享最多 100 条到 PostgreSQL 的实际连接。额度用完后，其他事务等待，而不是让所有事务同时执行。

## 详细解释

### 1. .NET 中由谁管理连接池？

| 层次 | 职责 |
| --- | --- |
| 业务代码 / EF Core | 发起数据库操作，管理连接与事务的使用范围 |
| 数据库驱动 | 管理池内物理连接的创建、借出、归还和清理 |
| 可选的外部池化代理 | 在应用与数据库之间复用后端连接，例如 PgBouncer |
| 数据库服务器 | 接收数据库会话并执行 SQL |

在启用池化、连接仍可复用的正常情况下：

```text
创建 Connection 对象 → 创建一个逻辑连接对象
Open / OpenAsync      → 从池中借用连接，必要时建立物理连接
Close / Dispose       → 归还连接供后续操作复用
```

所以 `new SqlConnection(...)` 不等于立即建立新的 TCP 连接，`Dispose()` 也不一定立即断开物理连接。

应在需要时打开连接，用完及时释放。不要为复用连接而把一条打开的 Connection 作为所有请求共用的全局对象。Npgsql 的 `NpgsqlDataSource` 则通常应长期复用；它代表数据源及池管理，不等于一条借出的连接。

### 2. DbContext 池不是数据库连接池

| 池 | 复用内容 | 管理者 |
| --- | --- | --- |
| 数据库连接池 | 底层数据库连接 | SqlClient、Npgsql 等驱动 |
| DbContext 池 | EF Core 的 DbContext 对象 | EF Core |

不使用 `AddDbContextPool()`，数据库连接池仍然可以工作。开启 DbContext 池也不会为每个 Context 永久分配一条数据库连接。

EF Core 的 Context 池容量主要控制保留多少 Context 对象用于复用，不是限制数据库同时执行多少请求的开关。

### 3. 连接池个数、池大小与实际连接数

假设每个应用实例只有一个相关连接池：

| 项目 | 数量或含义 |
| --- | --- |
| 应用实例数 | 20 |
| 连接池数量 | 20 个独立的池 |
| 每池 Max Pool Size | 最多 100 条连接 |
| 所有池的连接上限合计 | 最多可能有 2,000 条连接 |
| 当前实际连接数 | 随负载、最小池大小、空闲清理等变化 |

同一进程也可能有多个池，不能简单认为“一进程永远只有一个池”。例如 SqlClient 的连接字符串、认证身份等会影响池划分；多个 NpgsqlDataSource 通常也会产生多个池。

计算总额度时，需要汇总所有应用、所有相关池以及滚动发布期间新旧实例的额度。

### 4. Windows 和 Linux 下如何设置数据库池大小？

数据库连接池由驱动管理，使用相同驱动时，Windows 和 Linux 上通常采用相同的配置方式，而不是修改操作系统的统一“DB 连接池”参数。

SQL Server / SqlClient 连接字符串示例：

```text
Server=sql.example.com;Database=AppDb;User ID=app_user;Password=<password>;Encrypt=True;Pooling=True;Min Pool Size=0;Max Pool Size=100;
```

PostgreSQL / Npgsql 直连示例：

```text
Host=pg.example.com;Port=5432;Database=app;Username=app_user;Password=<password>;Pooling=true;Minimum Pool Size=0;Maximum Pool Size=100;
```

连接池上限不是 Web Server 的连接上限，也不是线程池大小。下面三者应分别配置：

```text
浏览器 → .NET 服务：HTTP 连接与请求并发
.NET 服务 → 数据库入口：数据库驱动连接池
数据库入口 → 数据库：直连，或由代理管理的后端连接
```

关于请求并发与延迟，可参见 [SYS-01 In-flight work 和长尾延迟](../system-design/SYS-01-in-flight-work-and-tail-latency.md)。

### 5. PgBouncer 的客户端连接与后端连接

这两个名称都以 PgBouncer 为参照：

| 名称 | 两端 | 含义 |
| --- | --- | --- |
| 客户端连接 / Frontend connection | 应用 ↔ PgBouncer | 数据库驱动连接到代理的连接 |
| 后端连接 / Backend connection | PgBouncer ↔ PostgreSQL | 实际连接数据库的连接 |

“后端连接”中的后端不是指 .NET 后端应用。“客户端连接”也不是浏览器到应用的 HTTP 连接。

例如所有应用经由同一个 PgBouncer 访问数据库，采用事务池模式：

```text
20 个 .NET 应用实例
每个实例一个 Npgsql 池，每池上限 100
              │
              │ 合计最多 2,000 条客户端连接
              ▼
          PgBouncer
              │
              │ 后端总额度配置为最多 100 条
              ▼
          PostgreSQL
```

假设当前实际建立了 1,000 条客户端连接，也可以只使用不超过 100 条后端连接。某个客户端的事务完成后，后端连接可以交给其他客户端，连接没有固定平均分给 20 个应用。

这描述的是连接复用能力，不是吞吐量保证。后端连接全部被占用时，新的工作需要等待，实际行为还受超时等设置影响。

### 6. PgBouncer 的三种池化模式

| 模式 | 后端连接何时归还 | 特点 |
| --- | --- | --- |
| Session pooling | 客户端连接断开后 | 默认模式；会话兼容性好，客户端连接存续期间通常占用后端连接 |
| Transaction pooling | 事务结束后 | 同一事务保持同一条后端连接，不同事务可能换连接 |
| Statement pooling | 语句完成后 | 不允许跨多条语句的事务 |

需要在多个长连接客户端之间共享较少后端连接时，Transaction pooling 是重点评估对象。自动提交模式下，单条语句通常形成自己的事务；长事务则会长时间占用后端连接。

事务池模式不能假设跨事务保留同一数据库会话。会话级锁、LISTEN、跨事务使用的临时表及会话参数等需要检查兼容性。预处理语句也要区分协议级机制与 SQL PREPARE，不能一概认为全部不支持，应按 PgBouncer 版本、配置和驱动行为验证。

### 7. 为什么可以同时保留 Npgsql 池和 PgBouncer？

两层池管理的是两段连接：

```text
Npgsql 池：复用 应用 → PgBouncer 的连接
PgBouncer 池：复用 PgBouncer → PostgreSQL 的连接
```

应用释放 Npgsql 连接时，通常只是把它归还应用内部的池，应用到 PgBouncer 的物理连接仍然打开。

- Session pooling：该客户端连接仍存在，后端连接可能继续被它占用。
- Transaction pooling：即使客户端连接保持打开，事务完成后也可归还后端连接。

因此“两层都启用连接池”并不自动意味着后端连接大幅减少，必须结合池化模式。也不必仅因使用 PgBouncer 就默认关闭 Npgsql 池。

### 8. PgBouncer 的 100 条上限如何理解？

以下只是展示连接预算的配置片段，监听、认证和 TLS 等配置需另外按环境设置：

```ini
[databases]
app = host=pg.example.com port=5432 dbname=app

[pgbouncer]
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 100
reserve_pool_size = 0
max_db_connections = 100
```

| 参数 | 含义 |
| --- | --- |
| max_client_conn | 此 PgBouncer 实例允许的客户端连接数量上限 |
| default_pool_size | 默认每个“数据库＋用户”池的后端连接上限 |
| reserve_pool_size | 池耗尽后可按条件使用的额外连接额度 |
| max_db_connections | 每个 PgBouncer 数据库入口的后端连接总上限，跨该入口的不同用户汇总 |

在这个只有一个数据库入口、没有额外备用额度的示例中，该 PgBouncer 实例为这个入口提供最多 100 条后端连接。

仅配置 `default_pool_size=100`，不能推断整个部署最多只有 100 条后端连接。多个用户、数据库入口或 PgBouncer 实例都可能使总量增加；同一物理数据库配置成多个入口，也要累计预算。绕过代理的连接、运维连接及其他应用连接还需另外计算。

### 9. SQL Server、MySQL 有哪些相关方案？

| 方案 | 定位 | 与连接池的关系 |
| --- | --- | --- |
| Microsoft.Data.SqlClient | SQL Server 的 .NET 驱动 | 提供进程内连接池，不跨应用进程共享 |
| Heimdall Proxy | 支持 SQL Server 等数据库的代理产品 | 提供外部池化及后端连接限制，接入需验证认证、事务和会话兼容性 |
| SQL Relay | 多数据库代理与连接管理方案 | 支持 SQL Server 后端，但自有 ADO.NET 接入方式不是原生 SqlClient 的透明替换 |
| ProxySQL | 支持 MySQL 等数据库的协议代理 | 提供连接池及满足条件时的连接复用 |
| Mycat | 主要面向 MySQL 生态的分布式数据库中间件 | 具备后端连接池，也负责分片路由、读写分离等 |

Mycat 不应仅理解为连接池代理。其后端各数据源可能分别维护连接池，总量应按实际部署汇总。

ShardingCore 等应用内分片库、Azure SQL 的分片路由工具，主要解决数据路由问题，也不能直接视为跨进程共享后端连接的池化代理。

“支持 SQL Server 后端”还不等于“前端接受 SqlClient 的 TDS 协议”。评估代理时，要同时检查前端协议、后端适配和所用 ORM 的兼容性。

从代表性项目看，MySQL 的开源代理与分片方案选择较多，但这是特定领域的生态观察，不宜扩展成所有第三方工具的数量结论。

### 10. Pool Size 应该如何估算？

在稳定负载、统计边界一致时，可用 Little’s Law 估算平均连接占用：

```text
平均忙碌连接数
≈ 每秒连接借用次数 × 平均每次持有连接的时间
```

例如每秒借用 1,000 次、平均持有 50 ms：

```text
1,000 × 0.05 = 平均约 50 条忙碌连接
```

应使用连接实际被持有的时间，而不是整个 HTTP 请求耗时；也不能把 P99 直接代入计算平均占用。

50 是平均需求，不是建议直接设置的上限。还应结合突发负载、数据库处理能力、连接等待时间和长尾延迟压测。PgBouncer 事务池的后端占用边界则是事务，不一定等于 Npgsql 的借用边界。

如果数据库已经饱和，盲目增加连接通常会加重争用；如果数据库尚有余量，再评估增大池。减少慢查询和长事务有时比增加池容量更有效。

## 故障场景

### 场景 1：连接未释放，应用池耗尽

连接长期未归还，新操作在驱动池中等待，最后可能超时。应先检查 Connection、DataReader 和事务的释放范围，再决定是否增加池容量。

### 场景 2：应用扩容导致数据库连接总数上升

20 个实例扩成 40 个，若每个实例仍只有一个上限 100 的相关池，潜在连接总量就从 2,000 增至 4,000。部署扩容、滚动更新以及代理副本数都应纳入预算。

### 场景 3：PgBouncer 后端额度耗尽

```text
客户端事务到达
       ↓
后端连接全部被占用
       ↓
在 PgBouncer 等待
       ↓
等待增大，端到端延迟升高或超时
```

有代理不等于数据库处理能力无限。应区分驱动池等待、代理等待和 SQL 执行耗时，结合代理连接状态与数据库活动会话定位瓶颈。

### 场景 4：事务已开始，但应用长时间不提交

即使暂时没有 SQL 正在执行，事务池中的后端连接仍可能被占用。应缩短事务范围，避免在事务中等待用户输入或进行耗时的外部调用。

## 面试追问

- 创建 Connection 对象是否等于创建物理连接？
- Close / Dispose 后为什么数据库连接仍存在？
- DbContext 池与数据库连接池有什么区别？
- 一个应用实例是否一定只有一个连接池？
- 20 个实例每池上限 100，为什么最多可能产生 2,000 条连接？
- PgBouncer 的客户端连接与后端连接分别连接哪两端？
- 事务池模式下，哪些边界保证使用同一条后端连接？
- 为什么 Session pooling 配合驱动长连接池时，可能无法明显减少后端连接？
- default_pool_size=100 为什么不等于整个部署最多 100 条连接？
- 数据库连接池与 HTTP 连接上限、请求并发上限有什么区别？
- SQL Server 的驱动池、外部代理与应用分片库分别解决什么问题？
- Mycat 为什么不能只理解为轻量级连接池代理？

## 关键结论

- .NET 通常由数据库驱动管理连接池，EF Core 依赖底层驱动。
- DbContext 池、数据库连接池、请求并发限制是不同机制。
- Pool Size 是池的容量参数，不是连接池个数，也不是当前实际连接数。
- 驱动池通常不跨进程共享；外部代理可以管理另一段后端连接。
- PgBouncer 客户端连接是应用到代理，后端连接是代理到 PostgreSQL。
- 事务池模式按事务复用后端连接；池满后需要等待，不会增加数据库本身的处理能力。
- 全局连接预算需要累计所有池、入口、用户、应用实例及代理实例。
- 池化代理、分片中间件和负载均衡不是同一个概念。

## 参考资料

- [Microsoft：SqlClient 连接池](https://learn.microsoft.com/en-us/sql/connect/ado-net/sql-server-connection-pooling?view=sql-server-ver17)
- [Microsoft：EF Core 连接池与 DbContext 池](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics#connection-pooling-considerations)
- [Npgsql：基本使用与连接池](https://www.npgsql.org/doc/basic-usage.html)
- [Npgsql：连接字符串参数](https://www.npgsql.org/doc/connection-string-parameters)
- [PgBouncer：池化模式与兼容性](https://www.pgbouncer.org/features.html)
- [PgBouncer：连接额度配置](https://www.pgbouncer.org/config.html)
- [Heimdall：SQL Server 代理与连接池](https://www.heimdalldata.com/microsoft/)
- [SQL Relay：前后端兼容性](https://sqlrelay.sourceforge.net/)
- [SQL Relay：ADO.NET 接入](https://sqlrelay.sourceforge.net/programming/ado.net.html)
- [ProxySQL：连接复用](https://proxysql.com/documentation/multiplexing/)
- [Mycat：官方项目](https://github.com/MyCATApache/Mycat-Server)
- [Mycat：连接池设计](https://www.mycat.org.cn/document/mycat-definitive-guide.pdf)

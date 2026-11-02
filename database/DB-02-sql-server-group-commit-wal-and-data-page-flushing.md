# DB-02 SQL Server 的 Group Commit、WAL 与数据页落盘是什么关系？

**标签**

`#Database` `#SQLServer` `#GroupCommit` `#WAL` `#TransactionLog` `#Durability` `#Checkpoint` `#CrashRecovery`

## 问题

SQL Server 是否使用 Group Commit（组提交）？组提交中的“提交日志”是否就是 Write-Ahead Log？事务提交时，实际数据什么时候写入数据文件？如果日志与数据页的落盘时机不同，数据库如何保证崩溃后的正确性？

## 面试回答

SQL Server 使用组提交这类批量日志处理机制，将多个事务的日志处理开销分摊到较少的 I/O 中。Always On 的组提交优化以及 In-Memory OLTP 的日志批处理都有明确的官方说明，但不同引擎和场景的具体实现不能混为一谈。

**Group Commit 合并的是日志处理，不是把多个业务事务合成一个事务。**在完整持久化模式下，每个事务仍要等其提交所需的日志持久化，才能向客户端确认提交成功。它与“先返回成功、稍后才刷日志”的 Delayed Durability 不同。

以普通磁盘表为例，SQL Server 的事务日志采用 WAL 机制。WAL 要求：**数据页写入持久化数据文件之前，该页相关修改的日志必须先持久化。**

执行 UPDATE 时，数据通常先在 Buffer Pool 中修改并成为脏页，同时生成相应日志。COMMIT 成功不要求所有脏页已经写入数据文件；Checkpoint、Lazy Writer 等机制会在其他时机写回数据页。数据页也可能在事务提交前落盘，只要满足 WAL 的顺序约束。

## 详细解释

### 1. Group Commit 解决什么问题？

如果每个小事务都单独承担一次日志持久化的固定成本，高并发下容易受到日志 I/O 限制。组提交让提交时间接近的事务共享部分日志写入与完成通知开销。

概念示意：

```text
T1 的提交所需日志 ─┐
T2 的提交所需日志 ─┼→ 批量写入并持久化 → 各事务分别获得提交确认
T3 的提交所需日志 ─┘
```

这里是一组独立事务。T1、T2、T3 的原子性边界不改变，也不是应用把三个事务放进一个更大的 BEGIN/COMMIT。

这幅图用于解释原理，不表示 SQL Server 固定凑齐三个事务才刷盘，也不保证任意三个提交总能归并为一次物理写入。批量大小及收益取决于并发、日志生成速度和具体机制；吞吐量收益与等待延迟之间可能存在取舍。

### 2. SQL Server 中有哪些明确的组提交场景？

- **Always On Availability Groups**：多个提交记录可以组合进较少的日志块，减少日志捕获、发送和副本处理的开销。
- **In-Memory OLTP / Hekaton**：微软论文描述了将多个事务日志记录组合为较大 I/O 的组提交思路。

同步提交的可用性组还需要满足相应的副本日志持久化确认条件；这不等于等待副本把全部数据页写入数据文件。

SQL Server 2025 提供了针对可用性组的组提交时间配置。该设置有明确适用范围，不能据此推断“所有 SQL Server 本地事务提交都固定等待同一个时间”。

.NET 或 EF Core 应用通常正常管理事务即可，不需要额外安装一个 Group Commit 中间件。

### 3. “提交日志”、事务日志与 WAL 的区别

| 术语 | 含义 |
| --- | --- |
| Transaction Log / 事务日志 | 数据库记录修改、事务结束等信息的日志，常见物理文件扩展名为 .ldf |
| Commit Record / 提交记录 | 标记某事务提交的日志记录，是事务日志的一部分 |
| WAL / Write-Ahead Logging | 约束日志与数据页持久化顺序的机制，也常用来指采用该机制的日志 |
| Data Files / 数据文件 | 保存普通磁盘表和索引数据页的文件，常见扩展名为 .mdf、.ndf |

“提交日志”在口头描述中通常指提交需要持久化的事务日志，不是 SQL Server 为每笔提交另建一个文件。

事务日志也不是简单保存 SQL 原文的审计日志。它包含恢复修改所需的信息；普通查询不需要在每次读数据时把日志重新执行一遍。

### 4. 执行 UPDATE 和 COMMIT 时发生什么？

例如：

```sql
BEGIN TRANSACTION;

UPDATE dbo.Accounts
SET Balance = Balance - 100
WHERE Id = 1;

COMMIT;
```

下面是一种常见时序，以普通磁盘表、写事务和完整持久化为前提：

| 阶段 | 内存中的数据页 | 日志 | 数据文件 |
| --- | --- | --- | --- |
| 执行 UPDATE | 相关页被修改，标记为脏页 | 在日志缓冲区产生修改记录 | 不要求立即更新 |
| 执行 COMMIT | 脏页可能仍在内存中 | 产生提交记录，等待相关日志持久化 | 不要求所有修改页已写入 |
| 返回提交成功 | 可继续使用缓冲区中的数据页 | 提交所需日志已持久化 | 可能仍有尚未写回的修改 |
| 后台写回 | 将需要写回的脏页写出 | 必须已满足该页的 WAL 条件 | 保存写回的数据页 |

以上不是所有数据页都必须遵循的完整时间排序。日志也可能因为缓冲区填满、数据页写回等原因，在 COMMIT 前就已经部分刷出。

“数据库已保存事务结果”与“所有修改页已经写入数据文件”是两种不同状态。完整持久化提交依靠可恢复的日志保证前者，而不是强制所有修改页同步写回。

### 5. WAL 真正要求的顺序是什么？

WAL 约束的是相关修改的持久化顺序：

```text
修改 X 对应的日志持久化
            ↓
包含修改 X 的数据页才允许落盘
```

不是下面两种过度简化：

```text
必须先把日志落盘，才能修改内存中的数据
```

```text
必须等事务提交后，才允许把数据页写入数据文件
```

事务提交前，一个包含未提交修改的脏页也可能被写回。此时该修改对应的日志已经持久化，但事务提交记录可能尚不存在。恢复机制需要能够消除未完成事务的效果。

### 6. 实际数据页什么时候写回？

| 机制 | 主要作用 |
| --- | --- |
| Checkpoint | 推进数据页持久化，减少崩溃恢复需要处理的工作 |
| Lazy Writer | 回收不常用的缓冲页，为后续工作提供可用缓冲区；脏页需先写回 |
| Eager Write | 用于部分最小日志记录操作，例如某些批量写入场景 |

不存在一个通用的“每次提交后固定多少毫秒，所有数据才落盘”的规则。负载、内存压力、检查点设置和 I/O 能力都会影响写回时机。

普通查询可以从 Buffer Pool 读取符合隔离级别要求的数据，不必等待所有修改写入数据文件。能够读取哪些版本由事务隔离控制，而不是仅由页面是否落盘决定。

Checkpoint 也不是“只有已提交事务的数据才写入”的边界，不能据此认定文件中不存在未提交修改。

### 7. Group Commit 与 Delayed Durability

| 对比 | Group Commit，采用完整持久化语义 | Delayed Durability |
| --- | --- | --- |
| 优化重点 | 多笔事务分摊日志处理和持久化开销 | 让提交不等待日志持久化 |
| 返回成功的前提 | 该事务提交所需日志已经持久化 | 日志可能仍只在内存中 |
| 是否改变业务事务边界 | 不改变 | 不改变 |
| 崩溃时是否可能丢失尚未刷日志的已确认提交 | 完整持久化提交不因仅有缓冲区丢失而丢失 | 可能 |
| 与数据页写回的关系 | 不要求提交时全部数据页写回 | 同样不代表数据页已经写回 |

SQL Server 默认使用完整持久化。**不需要启用 Delayed Durability，才能使用组提交思路。**

“延迟把脏页写入数据文件”和“延迟把事务日志持久化”不是同一件事。前者可以在完整持久化模式下正常发生；后者会影响提交确认时的持久性保证。

以上讨论以数据库引擎及存储正常兑现持久化语义为前提；存储介质永久丢失与崩溃恢复是不同问题。

## 故障场景

### 场景 1：COMMIT 已确认，部分数据页尚未写回时崩溃

```text
修改内存页
    ↓
提交所需日志持久化
    ↓
返回提交成功
    ↓
部分数据页尚未写回
    ↓
进程崩溃
```

恢复过程根据日志重做需要恢复的修改（Redo）。因此，数据文件中尚未反映全部修改，并不意味着已提交事务丢失。

### 场景 2：未提交修改已经写入数据文件时崩溃

```text
修改记录持久化
    ↓
包含未提交修改的数据页写回
    ↓
事务未完成就崩溃
```

恢复过程识别未完成事务并撤销其效果（Undo）。是否提交不是仅靠数据文件中有没有对应值来判断。

Redo / Undo 是解释恢复职责的抽象。具体恢复实现可能因引擎与 ADR 等功能而异，不应把这里的示意当作所有版本逐条执行的算法。

### 场景 3：启用延迟持久化，确认成功后日志仍未持久化

如果此时进程或系统崩溃，仍在内存中的相关日志可能丢失，从而丢失这些已确认提交。不能把这种风险归因于普通的组提交。

### 场景 4：日志持久化完成，但客户端没有收到成功响应

连接在成功响应返回前断开，客户端看到超时或连接错误，却不能据此断定事务一定回滚。数据库可能已经提交。

重试涉及写操作时，应结合业务唯一键、幂等请求标识或结果查询，避免把“结果未知”误当作“执行失败”。

## 面试追问

- Group Commit 为什么能减少小事务的日志 I/O 开销？
- 多笔事务组提交后，是否变成同一个原子事务？
- Commit Record 与整个事务日志是什么关系？
- WAL 的“先写”是相对内存修改，还是数据页落盘？
- COMMIT 成功时，是否要求所有修改页写入 .mdf/.ndf？
- 未提交数据为什么可能出现在数据文件中？
- Checkpoint、Lazy Writer 与事务提交有什么区别？
- 日志已持久化、数据页未写回时，崩溃后如何恢复？
- 数据文件中已有未提交修改时，如何恢复正确状态？
- Group Commit 与 Delayed Durability 有什么区别？
- 同步副本确认日志持久化，是否等于副本所有数据页已经落盘？
- 客户端提交超时为什么不一定代表回滚？

## 关键结论

- SQL Server 存在组提交及批量日志处理机制，具体实现应区分场景。
- 组提交分摊日志处理开销，不合并业务事务的原子性边界。
- 普通磁盘表采用 WAL：相关日志必须先于数据页持久化。
- 内存中的数据页可以在日志刷盘前修改。
- 数据页可以在事务提交前写回，提交后也可能尚未全部写回。
- 完整持久化提交等待相关日志持久化，不强制所有修改页同步写回。
- Redo / Undo 等恢复机制处理日志与数据文件状态之间的差异。
- Delayed Durability 改变提交确认时的持久性保证，不能与组提交混淆。

## 参考资料

- [Microsoft：SQL Server 事务日志架构与恢复](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17)
- [Microsoft：事务日志与未完成事务恢复](https://learn.microsoft.com/en-us/sql/relational-databases/logs/the-transaction-log-sql-server?view=sql-server-ver17)
- [Microsoft：Checkpoint 的作用与触发](https://learn.microsoft.com/en-us/sql/relational-databases/logs/database-checkpoints-sql-server?view=sql-server-ver17)
- [Microsoft：数据页写入与 WAL](https://learn.microsoft.com/en-us/sql/relational-databases/writing-pages?view=sql-server-ver17)
- [Microsoft：完整持久化与延迟持久化](https://learn.microsoft.com/en-us/sql/relational-databases/logs/control-transaction-durability?view=sql-server-ver17)
- [Microsoft：Always On 组提交配置与适用范围](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/availability-group-commit-time-server-configuration-options?view=sql-server-ver17)
- [Microsoft Research：Hekaton 日志与组提交（第 7.1 节）](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/06/Hekaton-Sigmod2013-final.pdf)

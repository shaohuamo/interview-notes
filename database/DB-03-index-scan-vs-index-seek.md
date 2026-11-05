# DB-03 Index Scan 与 Index Seek 有什么区别？

**标签**

`#Database` `#SQLServer` `#PostgreSQL` `#Index` `#ExecutionPlan` `#QueryOptimization` `#索引`

## 问题

Index Scan 和 Index Seek 有什么区别？查询已经使用索引，为什么仍然出现 Scan？Seek 一定比 Scan 快吗？如何根据执行计划判断实际开销？PostgreSQL 的 Index Scan 与 SQL Server 的命名是否相同？

## 面试回答

以 SQL Server 的普通行存 B+ tree 索引为例，Index Seek 先利用索引树定位目标键或范围，再读取候选记录；Index Scan 则遍历索引叶层，通常读取较多甚至全部索引数据。两者都使用索引，Scan 不等于没有使用索引。

Seek 可以是单值查找，也可以是范围访问，不保证只返回一行，也不保证读取的所有候选行都满足最终过滤条件。范围很大、额外过滤很多或产生大量回表时，Seek 仍然可能很贵。

优化器会根据查询、统计信息、索引覆盖情况等估算成本。需要读取大量数据时，扫描较小的覆盖索引，可能比 Seek 后逐行回表更合适。判断性能应结合实际读取行数、逻辑读、回表次数及执行时间，不能只看算子名称。

不同数据库的命名需要分别理解。PostgreSQL 的 Index Scan 也可以利用 Index Cond 定位少量记录，不能直接按 SQL Server 的 Index Scan 解释为全索引扫描。

## 详细解释

### 1. Seek 和 Scan 如何访问索引？

普通行存 B+ tree 索引包含根、中间层和叶层。索引树可以帮助定位键值，叶层保存索引记录。

| 对比 | Index Seek | Index Scan |
| --- | --- | --- |
| 访问方式 | 沿索引树定位目标键或范围，再读取候选记录 | 遍历索引叶层 |
| 常见适用场景 | 有效过滤后只需访问少量记录或较窄范围 | 需要读取大量记录，或缺少有效定位条件 |
| 读取范围 | 可以很小，也可以很大 | 通常较大，可能遍历整个索引 |
| 返回行数 | 可以是一行、多行或零行 | 可以是多行、少量行或零行 |
| 是否使用索引 | 是 | 是 |
| 是否一定更快 | 不一定 | 不一定 |

Scan 的实际读取也可能因 TOP、上层算子提前结束等情况而停止，不能把算子名称理解为“无论查询怎样，都必定读完整个索引”。

Seek 定位后仍需读取对应叶层记录。不要把 Seek 理解成没有扫描或没有 I/O 的一次常数时间查询。

### 2. SQL Server 示例

假设 dbo.Users 已存在，Email 是可索引的普通字符串列，并建立以下非聚集索引：

```sql
CREATE NONCLUSTERED INDEX IX_Users_Email
ON dbo.Users(Email);
```

如果邮箱条件只匹配很少的记录，下面的查询具备采用 Index Seek 的条件：

```sql
SELECT Email
FROM dbo.Users
WHERE Email = 'alice@example.com';
```

数据库可以定位到对应邮箱的索引范围，读取其中的记录。

如果需要全部邮箱：

```sql
SELECT Email
FROM dbo.Users;
```

优化器可能选择扫描这个非聚集索引。因为所需列就在索引中，读取它可能比扫描包含更多字段的整张表更省。

这两个示例用于解释适用场景，不保证所有数据量和配置下都产生同一种执行计划。小表、数据分布、其他索引以及统计信息都可能影响选择。

### 3. 范围查询也可以是 Seek

Seek 不只适用于等值条件。

例如，若有合适的 CreatedAt 索引，下面的条件可能形成范围 Seek：

```sql
SELECT CreatedAt
FROM dbo.Orders
WHERE CreatedAt >= '20261101'
  AND CreatedAt <  '20261201';
```

数据库先定位范围起点，再读取范围中的记录。如果这个月包含表中的大部分数据，即使使用 Seek，仍可能读取大量记录。

因此，“Seek”描述访问方式，并不直接代表访问的数据一定少。

### 4. Seek Predicate 与额外过滤条件

SQL Server 的 Seek 可能同时有两类条件：

| 执行计划属性 | 作用 |
| --- | --- |
| Seek Predicates | 用来定位索引中的键或范围 |
| Predicate | 对读取的候选记录继续判断，也称额外或残余过滤条件 |

例如，某个 Seek 定位到了一个较大的候选范围，但额外过滤最终只留下几条记录：

```text
利用 Seek 条件定位范围
            ↓
读取大量候选索引记录
            ↓
应用额外过滤条件
            ↓
只返回少量记录
```

最终返回少量行，不代表只读取了少量行。应比较实际读取行数与实际输出行数，结合额外过滤条件检查索引是否有效缩小了访问范围。

### 5. 为什么 Seek 不一定比 Scan 快？

主要需要考虑：

- **读取范围**：Seek 命中很大的范围时，叶层读取仍然很多。
- **回表次数**：非聚集索引缺少查询所需字段，可能需要再访问基础数据。
- **索引宽度与覆盖**：扫描较窄且覆盖查询的索引，可能比访问基础表更省。
- **执行次数**：Nested Loops 等计划中，一个小 Seek 可能被重复执行很多次。
- **成本估算**：统计信息与参数对应的数据分布会影响优化器判断。

SQL Server 中，非聚集索引后的基础数据访问可能表现为：

| 基础表组织方式 | 常见算子 |
| --- | --- |
| 聚集表 | Key Lookup |
| 堆表 | RID Lookup |

如果需要为大量结果逐行 Lookup，总成本可能超过一次扫描。

覆盖索引可以减少特定查询的 Lookup，但增加索引列也会增加空间与写入维护成本，应结合实际负载设计。

### 6. Index Scan 与 Table Scan 的关系

在 SQL Server 中，需要区分：

| 算子 | 访问对象 |
| --- | --- |
| Index Scan | 非聚集索引 |
| Clustered Index Scan | 聚集索引；普通行存聚集索引的叶层就是表数据 |
| Table Scan | 没有聚集索引的堆表 |

扫描索引和扫描基础数据不一定具有同样的读取量。一个很窄的非聚集索引，可能远小于基础表。

同样，执行计划还区分 Index Seek 与 Clustered Index Seek，后者使用聚集索引进行定位。

### 7. 如何判断真正的性能问题？

应在具有代表性的数据和参数下查看实际执行计划，重点关注：

| 信息 | 需要判断的问题 |
| --- | --- |
| 实际读取行数与输出行数 | 是否读取很多候选数据，却只返回少量结果？ |
| Seek Predicates / Predicate | 哪些条件负责定位，哪些只负责后续过滤？ |
| Lookup 的次数与读取量 | 是否产生大量基础数据访问？ |
| 算子的实际执行次数 | 是否因循环而累计很大成本？ |
| 估算行数与实际行数 | 优化器是否明显低估或高估数据量？ |
| 逻辑读、CPU 与耗时 | 哪个计划在真实负载下更合适？ |

SQL Server 可以结合实际执行计划与 SET STATISTICS IO、SET STATISTICS TIME 查看读取量和时间。逻辑读反映数据页访问，并不等同于每次都发生物理磁盘读取。

有 WHERE 条件不代表一定能 Seek。索引键顺序、条件可搜索性、函数、隐式转换等都可能影响可用访问路径；应结合具体表达式与计划判断。

优化目标是减少总工作量并满足业务延迟要求，而不是消灭所有 Scan。

### 8. PostgreSQL 为什么也叫 Index Scan？

PostgreSQL 的执行计划命名不同。例如，一个使用索引定位邮箱的查询，可能出现下面这样的简化计划：

```text
Index Scan using ix_users_email on users
  Index Cond: (email = 'alice@example.com')
```

这是命名与条件的示意，不是本笔记实测的执行结果。这里的 Index Cond 可以用于定位索引中的候选记录，算子名中的 Scan 不代表必定遍历整个索引。

分析 PostgreSQL 应结合 Index Cond、Filter、实际行数、loops 和缓冲区访问情况。需要实际数据时，可使用 EXPLAIN (ANALYZE, BUFFERS)；其中 ANALYZE 会真实执行查询。

不能把不同数据库中的同名算子直接视为同一种访问行为。

## 故障场景

### 场景 1：看到 Scan 就认为索引失效

查询需要绝大部分记录，优化器选择扫描覆盖索引。仅因存在 Scan 就重建索引或强制 Seek，可能反而增加开销。

应先检查查询需要读取的数据比例及索引是否覆盖。

### 场景 2：看到 Seek 就停止排查

计划包含一个大范围 Seek，并在后续产生大量 Lookup，或被重复执行很多次。

应检查实际累计读取量与整条执行路径，而不是只检查算子标签。

### 场景 3：只返回几条结果，却读取大量数据

Seek 定位范围不够窄，额外过滤丢弃了大多数候选记录。

应检查实际读取行数、输出行数以及 Seek Predicates 与 Predicate 的分工。

### 场景 4：把 PostgreSQL 的 Index Scan 当成 SQL Server 的全索引扫描

查询通过 Index Cond 已经精确缩小访问范围，但被误认为没有利用索引定位。

应按对应数据库的算子定义解读，并结合实际执行统计核验。

## 面试追问

- Index Scan 是否代表没有使用索引？
- Index Seek 能否返回多行，或读取大范围数据？
- 为什么有 WHERE 条件仍然可能 Scan？
- Seek Predicates 与 Predicate 有什么区别？
- 为什么返回少量行，仍然可能有大量逻辑读？
- Key Lookup 与 RID Lookup 分别出现在什么情况下？
- 覆盖索引为什么可能改善查询，又有什么维护成本？
- 小表或低选择性的查询为什么可能适合扫描？
- Index Scan、Clustered Index Scan 和 Table Scan 有什么区别？
- 为什么要看算子的累计执行次数？
- PostgreSQL 的 Index Scan 能否对应索引定位访问？
- EXPLAIN ANALYZE 与普通 EXPLAIN 的执行行为有什么区别？

## 关键结论

- 本文主要比较 SQL Server 的普通行存 B+ tree 索引访问方式。
- Seek 利用索引定位键或范围；Scan 遍历索引，二者都使用索引。
- Seek 不一定只读少量数据，Scan 也不一定执行到索引末尾。
- 返回行数不能代替实际读取量，额外过滤可能丢弃大量候选行。
- 大范围读取、大量 Lookup 或重复 Seek 都可能产生很高成本。
- Scan 可以是合理方案，应以代表性负载中的总成本判断性能。
- 分析时结合实际计划、逻辑读、回表、执行次数和耗时。
- PostgreSQL 的 Index Scan 需要按 Index Cond 等信息解读，不能只按 SQL Server 的名称类比。

## 参考资料

- [Microsoft：Index Scan、Index Seek、Lookup 等执行计划算子](https://learn.microsoft.com/en-us/sql/relational-databases/showplan-logical-and-physical-operators-reference?view=sql-server-ver17)
- [Microsoft：索引结构、覆盖索引与设计取舍](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide?view=sql-server-ver17)
- [Microsoft：Seek 残余过滤与读取行数诊断](https://support.microsoft.com/en-au/topic/kb3190762-update-to-improve-diagnostics-for-query-execution-plans-that-involve-residual-predicate-pushdown-in-sql-server-2016-cd819214-360f-f3e3-ff86-606b8cbc36cf)
- [Microsoft：逻辑读与物理读统计](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-statistics-io-transact-sql?view=sql-server-ver17)
- [Microsoft：CPU 与执行时间统计](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-statistics-time-transact-sql?view=sql-server-ver17)
- [PostgreSQL：EXPLAIN、索引访问与实际执行统计](https://www.postgresql.org/docs/current/using-explain.html)

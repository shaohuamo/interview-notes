# Interview Notes

面试题知识库，按技术领域分类，并使用统一编号与标签管理。

## 当前题库

### Message Queue

- [MQ-01 消费者消费消息过程中宕机如何处理？](message-queue/MQ-01-consumer-crash-and-redelivery.md)
- [MQ-02 使用 Outbox Pattern 后是否还需要 Publisher Confirm？](message-queue/MQ-02-outbox-and-publisher-confirm.md)

### Distributed Systems

- [DIST-01 消费者需要更新多个 DB 且不能使用同一事务时，中途宕机如何处理？](distributed-systems/DIST-01-multi-db-crash-recovery-saga.md)

### System Design

- [SYS-01 In-flight work 和长尾延迟是什么意思？](system-design/SYS-01-in-flight-work-and-tail-latency.md)
- [SYS-02 gRPC 如何进行版本控制与 Protobuf 兼容演进？](system-design/SYS-02-grpc-versioning-and-protobuf-compatibility.md)

### API Design

- [API-01 REST API 中的资源、URI、参数位置与请求体如何设计？](api-design/API-01-rest-resources-uri-parameters-and-request-body.md)

### Database

- [DB-01 .NET 数据库连接池与 PgBouncer 如何管理和复用连接？](database/DB-01-dotnet-connection-pooling-and-pgbouncer.md)
- [DB-02 SQL Server 的 Group Commit、WAL 与数据页落盘是什么关系？](database/DB-02-sql-server-group-commit-wal-and-data-page-flushing.md)
- [DB-03 Index Scan 与 Index Seek 有什么区别？](database/DB-03-index-scan-vs-index-seek.md)
- [DB-04 如何用写后读主库实现读己之写？60 秒窗口如何设置？](database/DB-04-read-your-writes-and-primary-routing-window.md)
- [DB-05 什么是联邦查询引擎？Trino 与 2PC 有什么区别？](database/DB-05-federated-query-engines-trino-and-two-phase-commit.md)

### Redis

- [REDIS-01 Redis 如何实现 Rate Limit（限流）？](redis/REDIS-01-distributed-rate-limiting-with-lua.md)
- [REDIS-02 Redis Lua 可以跨多个 Key 实现原子操作吗？](redis/REDIS-02-lua-multi-key-atomicity-and-cluster-slots.md)

## 编号规则

- `MQ`：Message Queue / RabbitMQ / Message Broker
- `DIST`：Distributed Systems / Saga / Distributed Transaction
- `REDIS`：Redis
- `DB`：Database / EF Core / PostgreSQL
- `DOTNET`：.NET / ASP.NET Core
- `DOCKER`：Docker / Docker Compose
- `K8S`：Kubernetes / AKS
- `OBS`：OpenTelemetry / Logging / Metrics / Tracing
- `SYS`：System Design
- `API`：API Design / REST / GraphQL / gRPC

## 归档规则

- 只有明确的面试题才进入本仓库。
- 新面试题按所属领域递增编号。
- 已有题目的后续追问默认合并到原题，不创建新编号。
- 每道题统一包含：问题、标签、面试回答、详细解释、故障场景、追问、关键结论。

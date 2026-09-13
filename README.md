# Interview Notes

面试题知识库，按技术领域分类，并使用统一编号与标签管理。

## 当前题库

### Message Queue

- [MQ-01 消费者消费消息过程中宕机如何处理？](message-queue/MQ-01-consumer-crash-and-redelivery.md)
- [MQ-02 使用 Outbox Pattern 后是否还需要 Publisher Confirm？](message-queue/MQ-02-outbox-and-publisher-confirm.md)

### Distributed Systems

- [DIST-01 消费者需要更新多个 DB 且不能使用同一事务时，中途宕机如何处理？](distributed-systems/DIST-01-multi-db-crash-recovery-saga.md)

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

## 归档规则

- 只有明确的面试题才进入本仓库。
- 新面试题按所属领域递增编号。
- 已有题目的后续追问默认合并到原题，不创建新编号。
- 每道题统一包含：问题、标签、面试回答、详细解释、故障场景、追问、关键结论。

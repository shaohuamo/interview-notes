# MQ-01 消费者消费消息过程中宕机如何处理？

**标签**

`#MessageBroker` `#RabbitMQ` `#消费者宕机` `#ACK` `#消息重投递` `#幂等性`

## 问题

在使用 Message Broker 进行异步通信时，生产者端使用 Outbox Pattern 确保消息不会因为数据库事务与消息发送不一致而丢失；消费者端通过状态机或 Inbox/ProcessedMessages 机制实现幂等。如果消费者在消费 Message 的过程中宕机，该如何处理？

## 面试回答

消费者应采用 **Manual ACK**，并且只在业务处理和本地数据库事务成功提交之后才向 RabbitMQ ACK。

- 如果消费者在数据库事务提交前宕机，本地事务回滚，RabbitMQ 因为没有收到 ACK，会重新投递该消息，消费者重新处理即可。
- 如果数据库事务已经提交，但消费者在发送 ACK 之前宕机，RabbitMQ 仍会重新投递同一条消息。这时消费者必须根据 `MessageId`、Inbox Pattern、ProcessedMessages 或状态机判断该消息已经处理完成，并跳过业务逻辑，只执行 ACK。

因此系统通常采用：

`At-Least-Once Delivery + Idempotent Consumer = Effectively-Once Processing`

## 详细解释

标准消费流程：

```text
RabbitMQ
   |
   | Message
   v
Consumer
   |
   | 1. 检查 MessageId / Inbox 状态
   | 2. 执行业务逻辑
   | 3. 提交业务数据 + 消费状态
   | 4. ACK
   v
Done
```

关键原则是：**ACK 必须发生在业务事务成功提交之后。**

## 故障场景

### 场景 1：收到消息后、业务尚未执行就宕机

消息尚未 ACK。连接断开后，RabbitMQ 会重新投递该消息。

### 场景 2：业务执行过程中宕机，数据库事务尚未 Commit

数据库事务回滚，消息仍未 ACK。RabbitMQ 重新投递后，消费者从头重新执行。

### 场景 3：数据库已 Commit，但 ACK 前宕机

这是最典型的重复消费窗口：

```text
DB Commit
   ↓
Consumer crash
   ↓
ACK 未发送
   ↓
RabbitMQ redelivery
```

此时消费者应检查消息处理记录：

```text
MessageId | Status
----------|----------
1001      | Completed
```

如果状态已经是 `Completed`，则不重复执行业务逻辑，直接 ACK。

## Processing 状态的详细说明

`Processing` 表示：**消费者已经认领了这条消息，并且业务处理正在进行中，但还不能认为这条消息已经成功完成。**

例如：

```text
MessageId | Status      | UpdatedAt              | RetryCount
----------|-------------|------------------------|-----------
1001      | Processing  | 2026-09-13 22:00:00    | 1
```

它的作用通常包括：

- 标记某条消息正在被某个消费者实例处理；
- 避免多个消费者同时对同一个 `MessageId` 重复执行；
- 为异常恢复提供状态依据；
- 记录处理时间、重试次数、错误信息等运行状态。

但必须注意：`Processing` **不是成功状态，也不能等价于“已经处理过”**。

### 为什么不能看到 Processing 就直接跳过？

假设消费者先把状态更新为 `Processing` 并提交：

```text
1. MessageState = Processing
2. COMMIT
3. Update Order
4. Update Inventory
5. Consumer crash
```

此时数据库中仍然是：

```text
MessageId = 1001
Status    = Processing
```

RabbitMQ 因为没有收到 ACK，会重新投递该消息。

如果新的消费者简单地执行：

```csharp
if (state != null)
{
    Ack(message);
    return;
}
```

或者：

```csharp
if (state.Status == Processing)
{
    Ack(message);
    return;
}
```

那么这条消息就会被错误地认为“已经处理完成”，实际上业务可能只执行了一部分甚至完全没有执行，从而造成数据缺失。

所以正确的语义应该是：

```text
Completed   → 已经成功处理，可以直接 ACK
Processing  → 正在处理，或上一次处理异常中断，需要进一步判断
不存在记录  → 第一次处理
```

### Processing 遗留问题：Stale Processing

如果 `Processing` 状态已经单独提交，但消费者随后宕机，就可能永久留下一个 `Processing` 记录，这通常称为：

`stale processing`

例如：

```text
MessageId | Status      | UpdatedAt
----------|-------------|---------------------
1001      | Processing  | 2026-09-13 21:00:00
```

当前时间已经是：

```text
2026-09-13 22:00:00
```

而正常处理一条消息只需要几秒钟，那么这个 `Processing` 很可能已经失效。

常见做法是额外保存：

```text
MessageId
Status
UpdatedAt
RetryCount
LastError
WorkerId / InstanceId
```

然后定义一个处理超时，例如 5 分钟：

```text
Status = Processing
AND UpdatedAt < Now - 5 minutes
```

则认为该记录已经 stale，可以重新认领并再次处理。

示意流程：

```text
收到 Message 1001
      ↓
查询 MessageState
      ↓
Processing
      ↓
检查 UpdatedAt
      ↓
是否已经超时？
   ├─ No  → 可能仍有消费者正在处理，暂不重复执行
   └─ Yes → 认为上一次 Consumer 已宕机，允许重新处理
```

### Processing 和并发消费者

除了宕机恢复，`Processing` 还可以帮助解决并发消费问题。

例如 RabbitMQ 因重投递、网络抖动或应用逻辑导致相同 `MessageId` 在短时间内被两个消费者看到：

```text
Consumer A → Message 1001
Consumer B → Message 1001
```

理想状态是只有一个消费者能够成功把状态从：

```text
Pending / NotExists
```

切换为：

```text
Processing
```

这一步需要数据库层面的并发控制，例如：

- `MessageId` 唯一约束；
- Optimistic Concurrency；
- 条件 `UPDATE`；
- 行锁；
- `INSERT ... ON CONFLICT`；
- compare-and-set 风格的状态更新。

例如语义上类似：

```sql
UPDATE MessageState
SET Status = 'Processing', UpdatedAt = NOW()
WHERE MessageId = @MessageId
  AND Status IN ('Pending', 'Failed');
```

只有成功更新到记录的消费者才获得处理权。

### Processing 是否一定需要持久化？

不一定。

如果：

```text
Processing
业务数据更新
Completed
```

全部放在同一个本地数据库事务中：

```text
BEGIN TRANSACTION

Insert MessageState = Processing
Update Order
Update Inventory
Update MessageState = Completed

COMMIT
```

那么如果消费者在 Commit 前宕机，整个事务都会回滚。

外部观察者通常只会看到两种稳定状态：

```text
没有 MessageState 记录
```

或者：

```text
Status = Completed
```

此时 `Processing` 只是事务内部的短暂状态，实际上不会作为持久化恢复状态暴露出来。

这种设计更简单，因为不需要处理 stale processing。

### 什么时候需要真正持久化 Processing？

当业务处理无法放进一个短事务中时，`Processing` 才更有价值，例如：

- 一个消息处理需要几十秒或几分钟；
- 需要跨多个数据库；
- 需要调用多个外部服务；
- 需要通过状态机分步骤执行；
- 需要支持 worker 接管和恢复；
- 需要记录长时间运行任务的 checkpoint。

这时 `Processing` 往往不只是一个布尔状态，而是一个真正的工作流状态，例如：

```text
Received
   ↓
ProcessingOrder
   ↓
ProcessingInventory
   ↓
ProcessingPayment
   ↓
Completed
```

同时需要配合：

- Timeout
- RetryCount
- LastError
- Checkpoint
- Idempotency
- Compensation

### Processing 状态不能解决外部副作用原子性

即使使用了 `Processing`，仍然存在这种故障窗口：

```text
Status = Processing
   ↓
调用 Payment API
   ↓
Payment 成功
   ↓
Consumer crash
   ↓
Status 仍然是 Processing
```

重新消费后，消费者并不知道 Payment 到底有没有成功。

因此 `Processing` 只能记录本地工作流状态，不能自动保证外部系统副作用的 Exactly Once。

外部操作仍然必须自己支持幂等，例如：

```http
Idempotency-Key: <MessageId>
```

或者由目标服务维护：

```text
ProcessedOperations

MessageId | Operation       | Result
----------|-----------------|---------
1001      | ProcessPayment  | Success
```

### 推荐判断逻辑

可以把消费者状态判断理解为：

```text
No Record
   ↓
开始处理

Processing + 未超时
   ↓
认为可能仍在执行，不重复处理

Processing + 已超时
   ↓
认为上一个 Consumer 已失败
   ↓
重新认领 / Retry

Completed
   ↓
跳过业务逻辑
   ↓
ACK

Failed
   ↓
根据 RetryCount / ErrorType 决定 Retry 或 DLQ
```

因此，`Processing` 的核心意义不是“这条消息已经处理过”，而是：

> **这条消息的处理已经开始，但最终结果尚未确定。系统必须结合事务边界、超时、并发控制、重试和幂等机制，判断它是仍在正常执行，还是上一次处理已经失败并需要恢复。**

## 状态机注意事项

如果使用 `Processing / Completed` 状态机，要注意 `Processing` 不能简单理解为“已经处理过”。如果中间状态是独立提交的，则消费者宕机后可能遗留 stale processing 状态，需要超时恢复、重试次数、更新时间等机制。

如果 `Processing`、业务数据更新和 `Completed` 都位于同一个数据库事务中，则事务提交前宕机会整体回滚，数据库通常只会观察到“无记录”或 `Completed` 两种可靠状态，设计更简单。

## 外部副作用

如果业务处理中调用外部系统，例如支付 API，本地数据库事务无法覆盖外部副作用：

```text
Payment API 成功
   ↓
Consumer crash
   ↓
本地 Completed 尚未保存
```

重新消费后可能再次调用支付接口。因此外部操作本身也应支持幂等，例如使用：

```http
Idempotency-Key: <MessageId>
```

## 面试追问

- 为什么必须使用 Manual ACK？
- 为什么 ACK 要放在 DB Commit 之后？
- DB Commit 后 ACK 前宕机会发生什么？
- `Processing` 为什么不能直接当成“已经处理过”？
- stale processing 应该如何恢复？
- `Processing` 是否一定要持久化？
- 多消费者同时处理同一 MessageId 时如何争抢处理权？
- Inbox Pattern 和 ProcessedMessages 如何实现？
- Exactly Once Delivery 和 Effectively Once 有什么区别？
- 外部 HTTP 调用如何实现幂等？

## 关键结论

- Broker 侧依赖未 ACK 消息重新投递来恢复消费者宕机。
- 消费者侧必须允许重复投递，但不能产生重复业务副作用。
- `Processing` 代表处理中，而不是处理成功。
- 独立持久化 `Processing` 时必须考虑 stale processing、超时恢复和并发认领。
- 如果业务数据与 `Completed` 可以在同一个本地事务中提交，通常可以避免长期持久化 `Processing`，从而简化设计。
- 本地业务数据和消费完成状态应尽可能在同一数据库事务中提交。
- 外部系统调用也需要幂等键或等价机制。

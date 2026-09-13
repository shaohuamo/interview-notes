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
- Inbox Pattern 和 ProcessedMessages 如何实现？
- Exactly Once Delivery 和 Effectively Once 有什么区别？
- 外部 HTTP 调用如何实现幂等？

## 关键结论

- Broker 侧依赖未 ACK 消息重新投递来恢复消费者宕机。
- 消费者侧必须允许重复投递，但不能产生重复业务副作用。
- 本地业务数据和消费完成状态应尽可能在同一数据库事务中提交。
- 外部系统调用也需要幂等键或等价机制。

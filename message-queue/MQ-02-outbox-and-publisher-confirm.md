# MQ-02 使用 Outbox Pattern 后是否还需要 Publisher Confirm？

**标签**

`#OutboxPattern` `#RabbitMQ` `#PublisherConfirm` `#生产者可靠性` `#消息不丢失`

## 问题

生产者端已经使用 Outbox Pattern，是否还需要使用 RabbitMQ Publisher Confirm（生产者确认）？

## 面试回答

需要。Outbox Pattern 和 Publisher Confirm 解决的是两个不同层次的问题。

- **Outbox Pattern**：解决“业务数据库事务成功，但消息没有成功发送”这一类数据库与消息系统之间的一致性问题。
- **Publisher Confirm**：解决生产者如何确认消息已经被 RabbitMQ Broker 接收的问题。

典型流程：

```text
业务事务
  ↓
Business Data + Outbox 同一事务提交
  ↓
Outbox Publisher 读取 Pending 消息
  ↓
Publish 到 RabbitMQ
  ↓
等待 Publisher Confirm
  ↓
收到 Broker ACK
  ↓
将 Outbox 标记为 Sent
```

只有收到 Publisher Confirm 后，Outbox Dispatcher 才应该把对应消息标记为已发送。

## 为什么仅有 Outbox 还不够

调用 RabbitMQ 的 Publish API 并不等于 Broker 一定已经成功接收消息。网络中断、Broker 故障等情况都可能使生产者无法确定消息最终状态。

Publisher Confirm 提供如下反馈：

```text
Publish
  ↓
Broker ACK ?
 ├─ Yes → Outbox = Sent
 └─ No / Unknown → 保持 Pending，之后重试
```

## 为什么仍然可能产生重复消息

即使使用了 Publisher Confirm，也存在一个经典故障窗口：

```text
RabbitMQ 已接收消息
        ↓
Publisher Confirm 返回 ACK
        ↓
Producer 尚未更新 Outbox=Sent
        ↓
Producer crash
```

Producer 重启后发现 Outbox 仍为 `Pending`，于是会再次发布同一条消息。

因此：

```text
Outbox + Publisher Confirm
```

保证可靠发送，但不能天然保证消息只发送一次。消费者仍然必须实现幂等。

## 两种 ACK 的区别

### Publisher Confirm

```text
Producer ← RabbitMQ
```

表示 Broker 对生产者发布结果的确认。

### Consumer ACK

```text
RabbitMQ ← Consumer
```

表示消费者已经成功处理消息，可以让 Broker 删除该消息。

两者方向、目的和生命周期都不同。

## 面试追问

- Outbox Pattern 具体解决什么问题？
- Publisher Confirm 和 Consumer ACK 有什么区别？
- 为什么 Publisher Confirm 后仍可能重复发送？
- 如何设计 Outbox Dispatcher 的重试机制？
- Outbox 表什么时候可以安全标记为 Sent？
- 为什么消费者仍然必须做幂等？

## 关键结论

- Outbox 解决业务数据库和消息发布的一致性。
- Publisher Confirm 解决 Producer → Broker 的可靠发送确认。
- Confirm 成功与 Outbox 状态更新之间仍存在 crash window。
- 因此整个系统通常是 At-Least-Once，而不是天然 Exactly Once。
- Consumer Idempotency 是最终可靠性的必要组成部分。

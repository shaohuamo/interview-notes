# DIST-01 消费者需要更新多个 DB 且不能使用同一事务时，中途宕机如何处理？

**标签**

`#SagaPattern` `#多数据库` `#分布式事务` `#幂等性` `#故障恢复` `#补偿事务`

## 问题

消费者处理一条消息时需要进行多个数据库更新，而这些更新无法放在同一个本地事务中。如果 Consumer 在其中某一步之后宕机，该如何保证系统最终一致性？

## 面试回答

如果多个更新位于同一个数据库，应优先使用本地事务，将业务更新和 Inbox/MessageState 放在同一个事务中。

如果涉及多个数据库或多个微服务，无法依赖单个数据库事务，应使用 **Saga Pattern + 持久化状态 + 幂等操作 + 补偿事务**。

Saga 会记录每一步是否完成。Consumer 宕机并收到 RabbitMQ 重新投递后，根据已持久化状态跳过已经完成的步骤，只继续执行尚未完成的步骤。

例如：

```text
MessageId = 1001

Order       = Done
Inventory   = Done
Payment     = Pending
Status      = Processing
```

重新消费后：

```text
Order       Done    → Skip
Inventory   Done    → Skip
Payment     Pending → Execute
                    ↓
                 Completed
                    ↓
                   ACK
```

但是状态机本身不能完全消除重复副作用，因为还存在“业务操作已经成功，但状态尚未保存时宕机”的窗口。因此每个业务步骤本身也必须具备幂等性。

## 为什么简单的 MessageState 不够

假设处理流程为：

```text
1. Update Order       ✓
2. Update Inventory   ✓
3. Update Payment     ← crash
4. MessageState=Completed
5. ACK
```

RabbitMQ 会因为未 ACK 而重新投递消息。如果重新从第 1 步执行，可能造成重复扣库存、重复扣款等副作用。

因此需要保存更细粒度的执行进度，或者将流程拆成多个独立的事件驱动步骤。

## Checkpoint / Resumable Workflow

一种实现方式是为每个步骤持久化状态：

```text
SagaId: 1001
OrderStatus      = Completed
InventoryStatus  = Completed
PaymentStatus    = Pending
OverallStatus    = Processing
```

伪代码：

```csharp
if (!state.OrderUpdated)
{
    await UpdateOrder();
    state.OrderUpdated = true;
    await SaveState();
}

if (!state.InventoryUpdated)
{
    await UpdateInventory();
    state.InventoryUpdated = true;
    await SaveState();
}

if (!state.PaymentCompleted)
{
    await ProcessPayment();
    state.PaymentCompleted = true;
    await SaveState();
}

state.Status = Completed;
await SaveState();
Ack(message);
```

## 为什么每一步仍然必须幂等

考虑以下故障窗口：

```text
ProcessPayment()
      ↓
支付已成功
      ↓
Consumer crash
      ↓
PaymentCompleted=true 尚未保存
```

重新投递后，状态仍显示 Payment 未完成，因此系统可能再次调用支付服务。

解决方案是让 Payment Service 使用幂等键，例如：

```http
Idempotency-Key: <SagaId-or-MessageId>
```

Inventory、Order 等服务也应采用相同思想，使相同业务操作重复执行时不会产生第二次业务效果。

## Compensation

如果流程无法继续完成，Saga 通常不是执行数据库级 Rollback，而是执行业务上的补偿操作。

例如：

```text
Create Order        ✓
Reserve Inventory   ✓
Payment             ✗
```

补偿流程：

```text
Release Inventory
       ↓
Cancel Order
```

常见正向/补偿操作：

```text
ReserveInventory ↔ ReleaseInventory
CreateOrder      ↔ CancelOrder
```

## 更符合微服务的拆分方式

通常不建议一个 Consumer 直接跨多个服务修改多个数据库：

```text
Consumer
 ├─ Order DB
 ├─ Inventory DB
 └─ Payment DB
```

更推荐事件驱动 Saga：

```text
OrderCreated
     ↓
Inventory Service
     ↓
InventoryReserved
     ↓
Payment Service
     ↓
PaymentCompleted
     ↓
Order Service
     ↓
OrderConfirmed
```

每个服务只保证自己的：

```text
Local Transaction
+ Inbox
+ Outbox
```

Saga 负责跨服务的最终一致性。

## 面试追问

- 什么情况下应该使用本地事务，什么情况下使用 Saga？
- Saga Choreography 和 Orchestration 有什么区别？
- 状态机为什么不能替代幂等设计？
- 业务操作成功但状态写入失败时怎么办？
- 补偿事务和数据库 Rollback 有什么区别？
- Saga 中 ACK 应该在什么时候发送？
- 如何避免一个 Consumer 长时间持有 RabbitMQ 的 unacked message？

## 关键结论

- 单数据库优先使用 Local Transaction。
- 多数据库/多服务采用 Saga 实现最终一致性。
- Saga 状态需要持久化，以支持 crash recovery 和 resume。
- 每个步骤必须幂等，因为“副作用成功、状态保存失败”的窗口无法靠状态机消除。
- 无法继续完成时使用 Compensating Transaction。
- 微服务之间更推荐通过事件拆分，而不是一个 Consumer 直接跨多个数据库更新。

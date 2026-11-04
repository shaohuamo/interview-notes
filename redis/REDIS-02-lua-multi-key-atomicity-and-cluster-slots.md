# REDIS-02 Redis Lua 可以跨多个 Key 实现原子操作吗？

**标签**

`#Redis` `#Lua` `#原子性` `#RedisCluster` `#HashSlot` `#HashTag` `#Rollback`

## 问题

Redis Lua 脚本能否一次操作多个 Key 并保持原子性？单实例与 Redis Cluster 有什么区别？为什么多个 Key 要放在同一个 hash slot？脚本中途报错是否会自动回滚？

## 面试回答

可以。Redis Lua 脚本可以在一次执行中读取、判断并更新多个 Key。正常执行期间，其他客户端的命令不能穿插到脚本内部的操作之间，因此可以避免把“读取—判断—写入”拆成多个客户端请求所产生的并发竞争。

单实例可以操作该实例中的多个 Key。标准 Redis Cluster 的常规 EVAL / EVALSHA 调用要求声明的 Key 映射到同一个 hash slot；可以使用相同的 hash tag 把相关 Key 放到同槽。Lua 不提供跨多个独立 Redis 实例的分布式事务。

原子执行需要与错误回滚区分。脚本中途发生运行时错误时，已经成功执行的写入不会自动撤销，应尽量先完成参数、数据类型和业务条件检查，再修改状态。

## 详细解释

### 1. “跨多个 Key 原子执行”是什么意思？

假设一次业务判断依赖两个 Key，处理顺序是：

```text
读取 Key A 和 Key B
          ↓
判断业务条件
          ↓
更新 Key A 和 Key B
```

如果把这些步骤拆成多个客户端命令，另一个客户端可能在读取之后、更新之前修改数据，使原先的判断失效。

把它们放入同一个 Lua 脚本后，其他客户端命令不会在这些步骤之间插入执行。这是 Lua 适合实现条件更新、限流与多 Key 状态协调的原因。

这个保证针对一次脚本执行。客户端分别发送两次 GET 时，两次读取之间仍可能发生一次完整的脚本执行；不能把多个独立读取自动当成同一时刻的快照。

### 2. 单实例和 Redis Cluster 的区别

| 部署与数据位置 | 常规多 Key 脚本 |
| --- | --- |
| 单个 Redis 实例中的多个 Key | 支持 |
| Redis Cluster 中同一个 hash slot 的多个 Key | 支持；请求需要路由到负责该槽的节点 |
| Redis Cluster 中不同 hash slot 的 Key | 常规调用不支持，通常返回 CROSSSLOT |
| 多个独立 Redis 实例 | 一个 Lua 脚本不能跨实例执行 |

Redis Cluster 将 Key 分配到 16,384 个 hash slot，再由不同节点负责这些槽。

**同槽比同节点更严格。**即使两个不同槽当前碰巧位于同一节点，常规调用也不能把它们当作同槽 Key；扩缩容和迁移还可能改变槽所属的节点。

这里讨论的是标准 Redis Cluster 的常规调用约束。不同托管产品或代理支持的跨分片命令能力，需要按产品文档单独确认。

### 3. 如何通过 hash tag 把相关 Key 放到同槽？

例如：

```text
user:{123}:tokens
user:{123}:last_refill
```

这两个 Key 都包含相同的非空 hash tag `{123}`。Redis 使用大括号内的 `123` 计算槽，因此它们会进入同一个 hash slot。

注意区别：

| 命名方式 | 是否保证同槽 |
| --- | --- |
| `user:123:tokens` 与 `user:123:last_refill` | 不保证；相同前缀不代表同槽 |
| `user:{123}:tokens` 与 `user:{123}:last_refill` | 保证；使用相同 hash tag |
| `user:{123}:tokens` 与 `user:{456}:tokens` | 不保证；hash tag 不同 |

hash tag 应围绕需要一起操作的业务范围设计，例如某个用户或某个订单。不要为了方便，把所有用户的数据都放入同一个固定 tag，否则容易把负载集中到一个槽。

Lua API 中的 `allow-cross-slot-keys` 是特殊脚本标志，官方仍要求脚本声明的 Key 映射到同一槽；它不提供跨节点的分布式事务能力。常规业务设计应优先让相关数据同槽。

### 4. 多 Key Lua 示例与参数传递

下面用两个 SET 展示多 Key 脚本的执行方式：

```lua
-- KEYS[1]: user:{123}:name
-- KEYS[2]: user:{123}:status
-- ARGV[1]: Alice
-- ARGV[2]: active

if #KEYS ~= 2 or #ARGV ~= 2 then
    return redis.error_reply("ERR expected 2 keys and 2 values")
end

redis.call("SET", KEYS[1], ARGV[1])
redis.call("SET", KEYS[2], ARGV[2])

return 1
```

相应的 EVAL 调用示意：

```text
EVAL "<上面的完整 Lua 脚本>" 2 user:{123}:name user:{123}:status Alice active
```

| 参数部分 | 含义 |
| --- | --- |
| 脚本内容之后的 `2` | 声明后面两个参数是 Key |
| `user:{123}:name` | KEYS[1] |
| `user:{123}:status` | KEYS[2] |
| `Alice` | ARGV[1] |
| `active` | ARGV[2] |

脚本访问的 Key 应显式作为 Key 参数传入，不应隐藏在 ARGV 中，也不应在脚本里根据数据内容临时拼接出未声明的 Key。

本例仅用于展示参数与执行边界；如果只是设置多个字符串值，也可以考虑原生多 Key 命令。Lua 更适合需要结合读取结果进行条件判断和更新的逻辑。

### 5. 原子执行是否意味着报错会自动回滚？

不意味着。执行期间不被其他命令穿插，与发生错误后撤销已经完成的写入，是不同的保证。

例如，假设 KEYS[2] 已存在且为字符串：

```lua
redis.call("SET", KEYS[1], "updated")

-- 对字符串执行列表操作，产生 WRONGTYPE 错误
redis.call("LPUSH", KEYS[2], "item")

return 1
```

第二条命令报错后：

- 使用 redis.call 产生的未处理运行时错误会终止脚本，并向调用者返回错误。
- 第一条 SET 已经成功，其修改仍然保留。
- 其他客户端在脚本结束后，可以看到这个只完成部分预期写入的结果。

因此，不能把 Lua 原子性理解为 SQL 事务式的“任意一步失败，全部自动撤销”。

redis.pcall 可以让脚本接收并处理命令错误，但它也不会自动撤销之前的写入。补偿逻辑如果需要，应由业务明确设计。

### 6. 如何组织需要共同更新的状态？

常见做法：

1. 在首次写入前检查参数、Key 数据类型及业务条件。
2. 将必须一起操作的 Key 按业务维度放入同一 hash slot。
3. 如果多个字段属于同一个实体，也可以考虑把状态放进同一个 Hash Key，再在脚本中操作不同字段。
4. 保持脚本短小，避免大范围扫描或长循环阻塞该节点上的其他请求。

一个脚本的原子执行不能把 Redis 与 SQL 数据库的更新合并成同一事务，也不直接保证故障切换不丢失写入。执行隔离、错误回滚与故障持久性需要分别考虑。

限流中的应用场景参见 [REDIS-01 Redis 如何实现 Rate Limit（限流）？](REDIS-01-distributed-rate-limiting-with-lua.md)。

## 故障场景

### 场景 1：单实例正常，迁移到 Cluster 后报 CROSSSLOT

脚本访问的多个 Key 在单实例中可用，但映射到不同槽。

应检查 Key 命名和调用参数，通过业务相关的 hash tag 实现同槽布局，而不是只检查它们当前是否位于同一节点。

### 场景 2：把用户 Key 与租户 Key 当作天然同槽

例如同时检查 `ratelimit:user:123` 与 `ratelimit:tenant:456`，它们没有共同的 hash tag，不能假设一个 Cluster 脚本就能原子操作。

需要先明确配额共享的业务范围，再决定数据布局。跨用户、跨租户的状态不一定适合强行放在一个槽中。

### 场景 3：第二条命令报错，以为第一条修改也被撤销

第一条写入成功，第二条因类型不匹配报错。脚本停止，但前面的修改保留。

应在写入前完成可以提前做的校验，并把命令错误与业务拒绝分别处理。

### 场景 4：客户端超时后直接重试非幂等脚本

客户端没有收到结果，并不代表脚本未执行。例如脚本已经完成扣减，重试后可能再次扣减。

对需要避免重复执行的操作，应结合请求标识与业务幂等规则设计。原子性本身不提供“恰好执行一次”的保证。

## 面试追问

- Lua 的原子执行具体防止了哪一种并发竞争？
- 原子执行是否保证脚本中途报错后自动回滚？
- redis.call 与 redis.pcall 对错误的处理有什么区别？
- 同一节点与同一 hash slot 为什么不是同一个条件？
- 相同 Key 前缀是否能保证同槽？
- hash tag 的大括号有什么作用？
- 为什么不应该把所有 Key 都使用同一个固定 hash tag？
- KEYS 与 ARGV 分别用于传递什么？
- 是否可以在脚本内临时拼接未声明的 Key？
- 多个 Key 改成一个 Hash Key 有什么取舍？
- Lua 是否可以实现跨 Redis 实例或跨 Redis 与 SQL 的事务？
- 原子脚本为什么仍需要考虑幂等和超时重试？

## 关键结论

- Lua 可以在一次执行中操作多个 Key，其他客户端命令不能插入脚本内部。
- 单实例支持本实例中的多 Key 操作；标准 Redis Cluster 的常规调用要求声明的 Key 同槽。
- 同一 hash slot 比当前同一节点更严格，hash tag 可以用于约束槽位置。
- 脚本访问的 Key 应通过 Key 参数显式传入，普通值使用 ARGV。
- 运行时错误不会自动撤销已经完成的写入，redis.pcall 也不提供自动回滚。
- 参数、类型与业务条件应尽量在写入前校验，脚本应保持短小。
- Lua 不是跨实例分布式事务，也不自动提供故障持久性或恰好执行一次。

## 参考资料

- [Redis：Lua 原子执行与 Key 参数化](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis：Lua API、错误处理与脚本标志](https://redis.io/docs/latest/develop/programmability/lua-api/)
- [Redis：Cluster 同槽约束与 hash tag](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis：多 Key 操作在不同部署中的行为](https://redis.io/docs/latest/develop/using-commands/multi-key-operations/)
- [Redis：事务与回滚说明](https://redis.io/blog/you-dont-need-transaction-rollbacks-in-redis/)
- [Redis：复制与故障切换的保证](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)

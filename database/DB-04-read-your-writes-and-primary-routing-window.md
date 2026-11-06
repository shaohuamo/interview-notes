# DB-04 如何用写后读主库实现读己之写？60 秒窗口如何设置？

**标签**

`#Database` `#Replication` `#ReadYourWrites` `#MonotonicReads` `#ReadWriteSplitting` `#Cookie` `#Session` `#Redis`

## 问题

主从异步复制时，如何让用户在写入后立即读到自己的修改？Cookie、浏览器 localStorage 和服务端 Session 中需要记录什么？60 秒的过期时间如何设置，后续写入或读取是否需要延长？固定时间窗口能否严格保证读己之写？

## 面试回答

可以在数据库事务成功提交后，为当前用户或会话记录“最近写入时间”或“读主库截止时间”。在截止时间前，将相关读取路由到主库；超过窗口后，再允许读从库。例如：

```text
readPrimaryUntil = 写入成功后的当前时间 + 60 秒

当前时间 < readPrimaryUntil → 主库
否则                        → 允许读从库
```

Cookie 可以通过 Max-Age=60 设置有效期；localStorage 没有内置 TTL，需要自行保存并检查截止时间；服务端 Session 可以保存时间戳，也可以使用 Redis 中带 60 秒 TTL 的独立标记。

每次新的成功写入都应重新计算窗口，普通读取不延长窗口。登录 Session 的有效期与这个路由标记的有效期应分别管理。

60 秒是应用选择的经验值，并不代表从库一定在 60 秒内追上。需要严格保证读己之写时，应在允许读从库前确认它已应用本次写入对应的复制位点，并让查询使用足够新的快照。

## 详细解释

### 1. 为什么写入成功后可能读不到修改？

主库提交事务与从库应用该事务之间存在时间差：

```text
用户更新职位头衔
        ↓
主库提交成功，返回成功响应
        ↓
用户立即刷新页面
        ↓
请求被路由到尚未追上的从库
        ↓
仍然显示旧头衔
```

“读己之写”（Read Your Writes）要求当前会话后续的读取能反映自己此前成功写入的结果。其他用户可以暂时继续从从库读取较旧的数据。

写后暂时读主库，是应用层实现该体验的一种策略。还需要保证读取使用合适的事务快照，并避免返回尚未更新的缓存结果。

### 2. 标记中应该保存什么？

可以选择以下形式：

| 保存形式 | 示例 | 判断方式 |
| --- | --- | --- |
| 最近成功写入时间 | lastWriteAt | 当前时间与它的差小于 60 秒 |
| 读主库截止时间 | readPrimaryUntil | 当前时间早于截止时间 |
| 带 TTL 的标记 | read-primary:user:123 = 1 | 标记在有效期内存在就读主库 |

单独保存一个永不失效的布尔值，无法判断何时恢复读从库。布尔标记需要搭配 TTL，或另外保存时间。

应先明确作用范围：

- 按会话保存：通常只覆盖携带该会话标识的请求。
- 按用户保存：同一账号的多个设备可以共享状态，但所有应用实例都要能访问该状态。
- 按用户与分片或资源保存：可以缩小强制读主库的范围，需要更细的路由逻辑。

### 3. Cookie 如何设置 60 秒？

写入事务成功提交后，服务端可以返回下面的响应头。此片段假设使用 HTTPS：

```http
Set-Cookie: read_primary=1; Max-Age=60; Path=/; HttpOnly; Secure; SameSite=Lax
```

含义如下：

| 属性 | 作用 |
| --- | --- |
| read_primary=1 | 表示相关读取暂时使用主库 |
| Max-Age=60 | 浏览器收到后，Cookie 最多保留 60 秒 |
| Path=/ | 在匹配的站点路径下携带 |
| HttpOnly | 不允许 JavaScript 读取；浏览器仍可自动随匹配请求携带 |
| Secure | 仅通过安全连接发送 |
| SameSite=Lax | 设置跨站请求下的携带规则 |

后端在收到有效标记时选择主库；标记过期后，再按默认策略选择从库。跨域调用还需满足 Cookie 的作用域及凭据配置，不能假设任意请求都会携带它。

严格来说，未设置 Expires 和 Max-Age 的 Cookie 才是 Session Cookie。也可以使用 Session Cookie 携带截止时间，由服务端比较时间；Cookie 本身不必在 60 秒后删除。

如果截止时间由客户端携带并交给服务端验证，可以使用受保护的令牌或签名 Cookie。身份认证与权限校验仍使用原有机制，路由标记只负责选择读取位置。

Cookie 的具体过期行为参见 [MDN：Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)。

### 4. localStorage 如何设置 60 秒？

localStorage 没有内置 TTL，应用可以保存截止时间：

```javascript
// 在收到写入成功响应后记录；60_000 的单位是毫秒。
localStorage.setItem(
  "readPrimaryUntil",
  String(Date.now() + 60_000)
);

// 后续读取请求发出前判断。
const until = Number(
  localStorage.getItem("readPrimaryUntil") ?? "0"
);
const usePrimary = Date.now() < until;

if (!usePrimary) {
  localStorage.removeItem("readPrimaryUntil");
}
```

这是有效期判断示例，不是完整的请求路由实现。前端还需通过约定的请求头等方式通知后端，再由后端选择数据库。

localStorage 不会自动随 HTTP 请求发送，也不负责选择 DB 连接。超过截止时间后，即使数据尚未删除，应用也应将标记视为失效，无须依赖持续运行的倒计时或 setTimeout。

该方式依赖浏览器状态和客户端时钟。若希望由服务端统一控制有效期，可以把截止时间保存在服务端。

参见 [MDN：localStorage 没有自动过期时间](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)。

### 5. 服务端 Session 或 Redis 如何设置？

服务端 Session 可以保存截止时间：

```text
事务提交成功后：
    session.readPrimaryUntil = 服务端当前时间 + 60 秒

收到读取请求时：
    当前时间 < session.readPrimaryUntil → 主库
    否则                               → 允许读从库
```

Session 可以继续保存其他信息，只让 readPrimaryUntil 字段在业务逻辑上失效。不要为了路由标记而把整个登录 Session 的有效期改为 60 秒。

如果使用 Redis，可以将标记保存为独立的 key：

```redis
SET read-primary:user:123 1 EX 60
```

后端根据已认证的用户身份构造 key，并在读取前检查：

```redis
EXISTS read-primary:user:123
```

- 返回 1：暂时读主库。
- 返回 0：允许按默认策略读从库。

EX 60 表示 60 秒有效期。每次成功写入后再次执行 SET ... EX 60，会更新标记并重新设置 TTL；普通读取只检查标记，不刷新它。

这个 Redis key 是路由状态。数据库仍负责存储业务数据和完成事务。多应用实例需要使用一致的 key 规则和共享状态源。

参见 [Redis：SET 的 EX 参数](https://redis.io/docs/latest/commands/set/)。

### 6. 60 秒从什么时候开始，是否会重置？

从成功提交写入后开始设置窗口，不应在尚未确认提交成功时就开始倒计时。

```text
12:00:00 写入成功 → 读主库截止时间为 12:01:00
12:00:40 再次写入 → 更新为 12:01:40
12:00:50 普通读取 → 仍为 12:01:40
12:01:40 窗口结束 → 允许按默认策略读从库
```

因此它是“距离最近一次成功写入的窗口”，不按第一次登录或第一次读取计时。

只在新的成功写入时重置，可以避免用户持续刷新页面导致主库窗口一直延长。Cookie 方案可以重新下发同名、同作用域的 Cookie；localStorage 和 Session 可以更新截止时间；Redis 可以重新设置 TTL。

多个应用实例使用服务端时间戳时，应保持时钟同步。Cookie 的 Max-Age 是从浏览器收到时计算，通常适合近似时间窗口；需要统一的截止时刻时，可以使用服务端维护的时间戳。

### 7. 路由标记如何与连接池配合？

标记决定本次读取选择哪个数据库入口，连接池仍负责连接复用：

```text
读取请求
   ↓
检查当前用户或会话的路由状态
   ├─ 窗口内 → 主库连接配置 → 对应连接池
   └─ 窗口外 → 从库连接配置 → 对应连接池
```

不需要为每个用户创建独立连接池，也不需要让某条物理连接保持 60 秒打开。读取完成后及时释放连接，后续请求再次根据标记选择目标。

如果使用 ORM 的自动读写分离，需要让这条规则覆盖其默认的查询走从库策略。

连接池的详细说明参见 [DB-01 .NET 数据库连接池与 PgBouncer](DB-01-dotnet-connection-pooling-and-pgbouncer.md)。

### 8. 60 秒能否严格保证读己之写？

固定时间窗口只是在假设“从库大概率能在窗口内追上”的前提下工作：

```text
主库写入版本 100
    ↓
60 秒内读取主库，可读到版本 100 或后续状态
    ↓
窗口结束，从库仍只有版本 95
    ↓
切回从库后，仍可能读到旧数据
```

窗口长度应根据实际复制延迟及业务需求选择，并监控异常延迟。异步复制并不天然提供固定的最大延迟。

严格的做法是保留会话需要的最低数据版本或写入复制位点，在切换前确认目标副本已应用到该位置，并在确认后使用足够新的查询快照。需要比较的是可比的复制历史中的位置；多个独立分片可能需要分别记录位置。

如果没有合适的副本，可以继续读主库、等待满足条件的副本，或按业务要求返回暂时不可用。只确认日志已经接收或落盘，也不一定代表查询已经能看到修改，应检查应用或重放进度。

相关机制参见 [PostgreSQL：复制与同步提交](https://www.postgresql.org/docs/18/warm-standby.html#SYNCHRONOUS-REPLICATION)。

### 9. 与单调读、粘性会话有什么区别？

| 策略或保证 | 关注点 |
| --- | --- |
| 读己之写 | 后续读取能反映当前会话此前成功写入的结果 |
| 单调读 | 后续读取的数据状态不早于当前会话之前已经看到的状态 |
| 粘性副本路由 | 会话持续读取同一个 DB 副本，减少因切换到落后副本而倒退的机会 |
| 写后暂时读主库 | 在写入后的窗口内从主库获取数据，窗口结束后仍需考虑副本进度 |

固定读取同一个落后的从库，并不保证立即读到刚写入主库的修改。写后读主库 60 秒，也不自动保证窗口结束后的单调读。

切换副本时，版本或复制位点检查可以判断新副本是否足够新。理论上的会话保证需要在找不到满足条件的副本时等待或报告无法满足，而不能直接返回旧结果。

参见 [Session Guarantees for Weakly Consistent Replicated Data](https://www.cs.cornell.edu/courses/cs734/2000FA/cached%20papers/SessionGuaranteesPDIS_1.html)。

## 故障场景

### 场景 1：复制延迟超过 60 秒

窗口结束后切换到落后的从库，用户重新看到旧数据。

固定时间窗口无法覆盖所有复制异常。严格要求应结合实际应用位点判断是否可以切换。

### 场景 2：每次读取都刷新 TTL

用户不断刷新页面，标记一直不失效，主库持续承担该用户的读取。

应将窗口绑定到新的成功写入，普通读取只检查有效期。

### 场景 3：状态只保存在一个应用实例的内存中

写入请求由实例 A 处理，后续读取被负载均衡到实例 B，B 不知道该用户刚刚写入过。

可以使用请求携带的 Cookie、共享 Session 或共享路由状态，让所有实例执行一致的判断。

### 场景 4：路由到了主库，却仍返回旧缓存

数据库路由正确，但响应直接使用尚未更新的缓存，用户仍然看不到修改。

需要把缓存更新、失效或读取版本校验纳入写后读取路径，不能仅依赖 DB 目标选择。

## 面试追问

- 为什么事务提交成功后，从库仍可能读不到修改？
- 最近写入时间、截止时间和带 TTL 的标记有什么区别？
- Max-Age=60 的 Cookie 与 Session Cookie 有什么区别？
- localStorage 是否会自动过期、自动随请求发送？
- 为什么不需要为这个窗口设置一个持续运行的倒计时？
- 为什么不能直接把整个登录 Session 的有效期改成 60 秒？
- 新的成功写入和普通读取分别是否重置窗口？
- 多个应用实例怎样共享路由状态？
- 为什么不需要为每个用户保持一条 DB 连接 60 秒？
- 复制延迟超过窗口后，怎样防止读到旧状态？
- 从库收到 WAL 与从库已经应用 WAL 有什么区别？
- 读己之写、单调读和粘性副本路由分别解决什么问题？

## 关键结论

- 路由状态在事务成功提交后设置，用于决定相关读取的数据库目标。
- Cookie 可使用 Max-Age；localStorage 需要手动检查时间；Session 可保存截止时间，Redis 可使用独立 TTL 标记。
- 新的成功写入重置窗口，普通读取不延长窗口。
- 路由标记的有效期与登录 Session、数据库连接的生命周期分别管理。
- 固定 60 秒只是经验策略，窗口结束不代表副本必定已追上。
- 严格读己之写需要检查相应写入位点已经应用，并使用合适的查询快照与缓存结果。
- 粘性副本可以减少读取倒退，但不能单独保证读到刚写入主库的数据。

## 参考资料

- [MDN：Set-Cookie、Max-Age 与 Session Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)
- [MDN：localStorage 生命周期](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)
- [Redis：SET 与 EX 过期参数](https://redis.io/docs/latest/commands/set/)
- [PostgreSQL：异步复制、同步复制与 remote_apply](https://www.postgresql.org/docs/18/warm-standby.html)
- [PostgreSQL：pg_last_wal_replay_lsn 等复制进度函数](https://www.postgresql.org/docs/18/functions-admin.html)
- [Terry 等：Session Guarantees for Weakly Consistent Replicated Data](https://www.cs.cornell.edu/courses/cs734/2000FA/cached%20papers/SessionGuaranteesPDIS_1.html)

# API-01 REST API 中的资源、URI、参数位置与请求体如何设计？

**标签**

`#APIDesign` `#REST` `#HTTP` `#Resource` `#URI` `#RouteParameter` `#QueryString` `#RequestBody`

## 问题

REST API 中的资源是什么？在 GET /users/123 中，users、123 与 URI 分别是什么关系？GET 参数应该放在路径还是查询字符串中？GET、POST、PUT、DELETE 是否都有请求体？

## 面试回答

REST API 围绕资源设计接口，用户集合和某个具体用户都可以是资源。`/users` 表示用户集合的路径，`/users/123` 定位 ID 为 123 的用户；`123` 是资源 ID，也是该路由中的参数值，不是完整的 URI。

参数通常按含义分配：标识具体资源或所属层级的参数放在路径中，筛选、搜索、分页、排序等条件放在 query string 中。是否必填不能单独决定参数的位置，查询参数也可以必填。

POST、PUT 通常携带请求体，GET、DELETE 通常不携带。HTTP 标准没有为 GET、DELETE 的请求体定义通用语义，普通接口应避免依赖它们的 body。POST、PUT 是否允许空请求体则取决于资源表示和接口契约。

## 详细解释

### 1. 资源、集合与资源表示

资源（Resource）是 API 对外暴露的业务对象或概念，可以是用户、订单，也可以是用户集合、订单集合。

| 路径 | 表示的资源 |
| --- | --- |
| `/users` | 用户集合 |
| `/users/123` | ID 为 123 的用户 |
| `/users/123/orders` | 用户 123 的订单集合 |
| `/orders/456` | ID 为 456 的订单 |

因此，集合和集合中的单个对象都可以是资源。`users` 是表示用户集合的路径名称，集合资源通过 `/users` 这样的 URI 路径被访问。

资源是业务抽象，不要求与数据库表一一对应。返回的 JSON 则是资源的一种表示（Representation），接口也可以约定其他表示格式。

### 2. URI、路径与资源 ID 有什么区别？

以完整请求目标为例：

```text
https://api.example.com/users/123?fields=id,name
```

| 部分 | 含义 |
| --- | --- |
| `https` | URI scheme |
| `api.example.com` | 主机名 |
| `/users/123` | 路径（Path） |
| `users` | 表示用户集合的路径段 |
| `123` | 标识具体用户的路径段；在该接口中是资源 ID |
| `fields=id,name` | 查询字符串（Query），示例中用于字段选择 |

上面整个 HTTPS 地址是一个 URI，也属于 URL。路径和查询字符串都是它的组成部分。

对于：

```http
GET /users/123
```

- `GET` 是 HTTP 方法，不属于 URI。
- `/users/123` 是请求目标中的路径；作为 URI 引用使用时，需要结合基础地址解释。
- 路由模板可以写成 `/users/{id}`，实际参数值为 `id = 123`。
- `123` 在这里承担资源 ID 的作用，不能把这个局部参数值等同于完整资源 URI。

HTTP 方法表达对资源的操作意图，URI 标识资源。例如 GET 和 DELETE 可以使用同一个 `/users/123` 路径，分别表达读取和删除。

### 3. Route parameter 与 query string 一般如何选择？

常见设计约定：

| 参数作用 | 常见位置 | 示例 |
| --- | --- | --- |
| 标识具体资源 | 路径参数 | `GET /users/123` |
| 指明资源所属关系 | 路径参数 | `GET /users/123/orders` |
| 筛选集合 | 查询参数 | `GET /users?status=active` |
| 搜索 | 查询参数 | `GET /users?name=Alice` |
| 分页 | 查询参数 | `GET /users?page=2&pageSize=20` |
| 排序或字段选择 | 查询参数 | `GET /users?sort=name&fields=id,name` |

这是一种常见分工，并非 HTTP 对业务参数位置的强制要求。URI 的 query 部分也参与资源标识，不能据此认为带查询参数的地址就不是资源 URI。

例如：

```http
GET /users/123/orders?status=paid&page=2&pageSize=20
```

- `/users/123/orders` 定位用户 123 的订单集合。
- `status=paid` 筛选已付款订单。
- `page=2&pageSize=20` 指定分页方式。

资源 ID 不一定是数字，也可以采用接口约定的 UUID、用户名或其他稳定标识。

### 4. 必填参数是否应该放在路径中？

判断依据应是业务含义，必填与否是接口契约的另一个维度。

例如接口可以要求：

```http
GET /search?q=redis
```

即使 q 是必填搜索条件，它仍然适合放在 query string 中。

对比：

| 请求 | 常见含义 |
| --- | --- |
| `GET /users/123` | 读取用户 123，通常返回单个用户表示 |
| `GET /users?id=123` | 在用户集合中按 ID 筛选，通常保持集合查询的响应结构 |

两种请求都能被实现，但应在契约中明确响应结构、未匹配时的行为，并保持一致。不能仅根据查询条件最多匹配一条记录，就自动认定它是单资源接口。

### 5. GET、POST、PUT、DELETE 都有 request body 吗？

需要区分协议如何承载内容、方法对内容的语义，以及具体接口如何校验内容。

| 方法 | 常见用途 | 请求体的常见用法 |
| --- | --- | --- |
| GET | 读取资源 | 通常不带；定位和查询条件放在 URI 中 |
| POST | 创建资源、提交数据或执行操作 | 通常带；内容按目标资源约定处理 |
| PUT | 创建或替换指定资源的状态 | 通常带；描述资源的期望状态 |
| DELETE | 删除指定资源 | 通常不带；URI 指明目标 |

GET、DELETE 的请求在消息格式上并非绝对无法承载内容，但 HTTP 标准没有为它们的请求体定义通用语义，并建议客户端通常不要发送。部分实现可能拒绝此类请求，因此普通 REST API 应避免依赖这种设计。

POST、PUT 通常携带请求体，也不能据此认为每个请求都必须有非空 body。例如某接口可以约定 `POST /jobs/123/cancel` 无需额外参数。普通 JSON 写接口是否允许空 body，应在接口契约中明确。

PUT 的含义是用提交的表示创建或替换目标资源状态；不要默认把“只提交部分字段并保留其余字段”当作 PUT 的通用语义。部分修改通常应考虑 PATCH，并约定补丁格式。

请求体也不限于 JSON；内容格式通过 Content-Type 等元数据约定。

### 6. 同一组资源的请求示例

以下是省略无关协议字段的请求示意。

查询用户：

```http
GET /users/123
```

筛选并分页查询用户集合：

```http
GET /users?status=active&page=1&pageSize=20
```

提交新用户数据：

```http
POST /users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

替换用户 123 的可写状态；此例假定 name 和 email 构成接口约定的完整可写表示：

```http
PUT /users/123
Content-Type: application/json

{
  "name": "Alice",
  "email": "new@example.com"
}
```

删除用户：

```http
DELETE /users/123
```

具体哪些字段必填、能否通过 PUT 创建新资源、删除是否执行软删除，都由服务端契约定义，不能仅凭路径推断。

## 故障场景

### 场景 1：服务端依赖 GET body 中的筛选条件

某客户端可以发出请求，但链路上的其他实现未按预期支持请求体，导致查询被拒绝或条件未被处理。

常规筛选条件应放在查询字符串中；复杂查询需要 body 时，可以设计明确的 POST 查询接口。

### 场景 2：把所有必填参数都塞进路径

例如为状态、页码和每页数量建立固定层级，使路径同时承担大量查询选项，接口难以扩展。

应按资源定位和查询条件的作用设计位置，并分别声明参数是否必填。

### 场景 3：把 123 当成完整 URI

只保存或传递用户 ID，却丢失服务地址、资源类型及所属上下文，会使调用者无法直接确定目标资源。

应明确区分业务 ID、路由参数、资源路径和完整 URI。

### 场景 4：PUT 的缺失字段含义不明确

客户端以为省略 email 表示保持原值，服务端却按替换语义清空它，造成数据丢失。

应明确完整表示与部分修改的契约，并为缺失字段定义一致的行为。

## 面试追问

- 用户集合和单个用户是否都可以是资源？
- Resource、Representation、资源 ID 与 URI 有什么区别？
- GET /users/123 中哪些部分属于方法、路径和路由参数？
- URI 与 URL 是什么关系？查询字符串是否属于 URI？
- 路径参数一定是数字 ID 吗？
- 必填查询参数应该改成路径参数吗？
- GET /users/123 与 GET /users?id=123 的常见契约有什么区别？
- 为什么通常不依赖 GET、DELETE 的请求体？
- POST 是否必须有非空请求体？
- PUT 与部分修改的语义有什么区别？
- 复杂搜索条件不适合放在 URI 中时，可以如何设计接口？

## 关键结论

- REST 面向资源设计接口，集合和单个对象都可以是资源。
- users 是路径名称，123 是该接口中的用户 ID，/users/123 是资源路径。
- URI 标识资源，HTTP 方法表达操作，返回的数据是资源表示。
- 路径参数通常用于资源定位与层级，查询参数通常用于筛选、搜索、分页和排序。
- 查询字符串是 URI 的组成部分；参数位置与是否必填应分别设计。
- POST、PUT 通常带请求体，GET、DELETE 通常不带。
- GET、DELETE 的请求体缺少通用语义，应避免依赖；空 body 的处理需要明确契约。
- PUT 应按创建或替换语义设计，部分修改需要明确其他契约。

## 参考资料

- [Microsoft：REST API 资源、路径与查询参数设计](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [RFC 3986：URI 的组成部分](https://www.rfc-editor.org/rfc/rfc3986.html#section-3)
- [RFC 9110：GET 与请求内容](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.1)
- [RFC 9110：POST 语义](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.3)
- [RFC 9110：PUT 语义](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.4)
- [RFC 9110：DELETE 与请求内容](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.5)

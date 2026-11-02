# SYS-02 gRPC 如何进行版本控制与 Protobuf 兼容演进？

**标签**

`#SystemDesign` `#gRPC` `#Protobuf` `#APIVersioning` `#BackwardCompatibility` `#DotNet` `#契约演进`

## 问题

gRPC 如何实现 API 版本控制？什么时候可以直接给 .proto 增加字段，什么时候需要发布 v2？如何让新旧客户端共存，并在 .NET 中同时提供多个版本？

## 面试回答

gRPC 通常结合两种方式演进：

- **兼容变更**：保持原有服务身份，在 Protobuf 消息中增加新编号的字段，或增加新的 RPC 方法。
- **破坏性变更**：通过版本化的 package，例如 users.v1 和 users.v2，发布并行的服务契约，保留旧版本供旧客户端使用。

在常见的 Protobuf 二进制传输中，消息字段通过编号识别。旧客户端可以解析包含新增字段的消息；新代码也必须合理处理旧消息没有提供的字段。已有编号不能随意更改、复用，字段类型和业务含义也应保持兼容。

**传输格式兼容不等于业务行为兼容。**例如新增请求字段后，如果服务端立刻要求旧客户端也必须提供它，仍然会破坏旧客户端。

版本号不是每次改字段都递增。优先在同一主版本中保持兼容，确实需要破坏旧契约时再增加新主版本，并安排客户端迁移。

## 详细解释

### 1. 先区分兼容演进与显式版本

添加字段是契约的兼容演进；发布 users.v2 是显式创建新接口版本。两者不是同一种操作。

gRPC 默认使用 Protobuf 定义服务、RPC 方法及请求响应消息。兼容性至少有三个层面：

| 层面 | 需要回答的问题 |
| --- | --- |
| 传输兼容 | 新旧两端能否识别请求路径并解析消息？ |
| 生成代码兼容 | 更新 .proto、重新生成代码或升级客户端程序集后，调用代码是否还能编译和运行？ |
| 业务行为兼容 | 即使能调用和解析，旧客户端原本依赖的含义和规则是否仍成立？ |

例如，重命名字段但保留编号，通常不影响二进制字段识别，却会改变生成的属性名称；如果采用 ProtoJSON，还可能影响使用字段名的 JSON 契约。

### 2. 新增字段通常如何保持兼容？

原始 User 消息：

```protobuf
message User {
  string id = 1;
  string name = 2;
}
```

兼容增加邮箱后的 User 消息：

```protobuf
message User {
  string id = 1;
  string name = 2;
  optional string email = 3;
}
```

这里使用新编号 3，没有修改原有字段编号和含义。

| 组合 | 在二进制格式下的典型行为 |
| --- | --- |
| 旧客户端读取新服务端的响应 | 未识别字段不供旧业务代码使用，通常保存在未知字段中 |
| 新客户端读取旧服务端的响应 | email 未提供；optional 允许代码检查其存在性 |
| 旧客户端调用新增了请求字段的服务端 | 新字段未提供，服务端必须有兼容处理 |
| 新客户端把新字段发送给旧服务端 | 旧服务端不理解该字段，也不会自动实现其新业务功能 |

对于 proto3 中没有显式存在性标记的普通标量，缺失字段通常表现为默认值，例如空字符串、0 或 false。optional 可以帮助区分“未提供”和“显式提供了默认值”；它不代表服务端可以忽略业务上的兼容要求。

增加字段也不需要同步更新所有客户端：只有需要使用新字段的客户端，才需要更新契约和生成代码。但客户端依赖的新行为必须先在服务端部署完成。

### 3. 字段编号和删除规则

```protobuf
string name = 2;
```

这里的 2 是二进制消息识别字段的重要标识，不是字段在文件中的行号，也不能当成可以随意重新排序的序号。

主要规则：

| 变更 | 处理原则 |
| --- | --- |
| 新增字段 | 使用在该消息中从未使用、也未被保留的编号 |
| 修改已有字段编号 | 避免；相当于删除旧字段并增加另一个字段 |
| 将已有编号用于另一种含义 | 避免；旧消息或旧客户端可能被错误解释 |
| 修改字段类型 | 不兼容类型会破坏解析；部分可解析变化也可能丢失信息，应单独评估 |
| 删除字段 | 不再复用旧编号，通常也保留旧名称；仍需检查旧客户端是否依赖该字段 |
| 增加枚举值 | 二进制格式通常可演进，但旧代码必须能处理未知数值 |
| 修改 oneof 结构 | 不能视为普通字段调整，应按具体兼容规则检查 |

字段不再使用且已经完成迁移后，可以通过 reserved 防止误用。例如下面是另一个演进阶段的消息片段：

```protobuf
message User {
  reserved 2;
  reserved "name";

  string id = 1;
  optional string email = 3;
}
```

**reserved 只防止未来误用，不会补回旧客户端需要的数据。**如果旧客户端仍依赖 name，应继续保留旧版本的字段与行为，或者把不兼容的新结构发布为 v2。

### 4. 破坏性变更：使用版本化 package

假设旧接口返回一个完整姓名 name，新契约要改为 given_name 和 family_name，可以将不同结构放在不同主版本下。

v1 的独立 .proto 文件示例：

```protobuf
syntax = "proto3";

package users.v1;
option csharp_namespace = "InterviewNotes.Users.V1";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
  string id = 1;
}

message User {
  string id = 1;
  string name = 2;
  optional string email = 3;
}
```

v2 的另一个独立 .proto 文件示例：

```protobuf
syntax = "proto3";

package users.v2;
option csharp_namespace = "InterviewNotes.Users.V2";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
  string id = 1;
}

message User {
  string id = 1;
  string given_name = 2;
  string family_name = 3;
  optional string email = 4;
}
```

v1.User 与 v2.User 是不同契约中的不同消息类型。上例在 v2 中定义的编号不会修改 v1.User；“不复用编号”要求保护的是同一消息契约的演进历史。

gRPC 请求路径包含完整服务名和方法名，两个版本分别为：

```text
/users.v1.UserService/GetUser
/users.v2.UserService/GetUser
```

它们可以在同一个主机和端口提供服务。旧客户端继续调用 v1，新客户端使用按 v2 生成的客户端。

只把原 package 改成 users.v2 并移除 v1，并不能保持兼容。旧客户端仍会调用原来的完整路径。

### 5. .NET 中如何同时注册 v1 和 v2？

分别根据两份契约生成服务基类，并实现对应服务：

- UserServiceV1 继承 InterviewNotes.Users.V1.UserService.UserServiceBase。
- UserServiceV2 继承 InterviewNotes.Users.V2.UserService.UserServiceBase。

完成服务实现后，Program.cs 的注册部分可以是：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddGrpc();

var app = builder.Build();

app.MapGrpcService<UserServiceV1>();
app.MapGrpcService<UserServiceV2>();

app.Run();
```

这是注册片段，假定项目已包含两份 proto、生成代码以及对应服务实现类，不是完整项目。

两个接口实现可以复用内部业务逻辑，通过各自适配层映射到不同的请求响应契约，避免为每个 API 版本复制整套业务。

注意：

- package 参与 gRPC 服务身份和请求路径。
- csharp_namespace 控制生成的 C# 类型命名空间；只改它不会创建新的线上 API 版本。
- 修改 proto 文件名、文件夹名或 NuGet 包版本，也不会自动创建 users.v2 服务。
- 为两个版本选择不同的 C# 命名空间，有助于避免同名生成类型冲突。

### 6. 发布与迁移顺序

兼容变更的常见顺序：

```text
部署兼容旧请求的新服务端
            ↓
确认需要的新能力已在目标实例可用
            ↓
更新需要使用新字段或新方法的客户端
```

新增 RPC 不会破坏仍调用原方法的旧客户端，但新客户端调用尚未部署该方法的旧服务端，可能得到 UNIMPLEMENTED。

发布新主版本时：

```text
继续维护 v1
      ↓
同时部署 v1 和 v2
      ↓
客户端逐步迁移到 v2
      ↓
观察 v1 的调用与依赖，按约定安排停用
      ↓
移除 v1
```

应同时检查网关路由、权限配置和负载均衡后的所有目标实例，而不只是修改 .proto。

接口兼容验证应关注新旧客户端与新旧服务端的组合，以及字段缺失、未知枚举值、已有业务含义等行为。只证明“新客户端能调用新服务端”不足以证明兼容。

## 故障场景

### 场景 1：新增字段，但立即要求必填

旧客户端没有发送新请求字段。服务端虽然可以解析请求，却在业务校验中拒绝它。这是业务行为破坏，不能仅因 Protobuf 新增字段通常安全就忽略。

### 场景 2：直接覆盖 package，旧版本消失

服务端只提供 users.v2.UserService，旧客户端还在调用 users.v1.UserService。原 RPC 路径不再存在，通常得到 UNIMPLEMENTED。

解决方式是提供并行版本，而不是要求所有客户端与服务端在同一时刻升级。

### 场景 3：复用旧字段编号

旧版中 name = 2，随后在同一消息契约里把 2 用作另一种数据。旧客户端或历史消息可能被错误解析，甚至产生看似有效却含义错误的值。应保留旧编号，使用新编号或新的主版本契约。

### 场景 4：新客户端先于服务端上线

新客户端调用 GetUserDetails，但部分服务端实例尚未提供它。即使增加方法是对旧客户端兼容的，新方法仍不是旧服务端已经具备的能力。

### 场景 5：字段类型和编号不变，但含义变了

例如某个金额字段以前单位为“分”，后来改成“元”。消息仍能正常解析，但旧客户端会错误计算。这类语义变更同样需要兼容设计或新版本。

### 场景 6：把二进制兼容规则直接用于 JSON 接口

启用 JSON 转码或使用 ProtoJSON 时，字段名、未知字段等规则与 Protobuf 二进制格式不同。二进制下可解析的变更，不能直接当作 JSON 契约也安全。

## 面试追问

- gRPC 的兼容演进与显式主版本分别解决什么问题？
- 为什么新增字段通常不需要发布 v2？
- 字段名称与字段编号分别影响什么？
- optional 如何帮助处理字段缺失？
- 删除字段后为什么要 reserved？
- reserved 是否代表删除字段一定不会影响业务？
- 新增 RPC 方法对旧客户端、旧服务端分别有什么影响？
- 改 package、改 csharp_namespace、改 NuGet 版本有什么区别？
- 一个服务能否同时提供 users.v1 和 users.v2？
- 为什么修改字段业务含义也可能是破坏性变更？
- 为什么 Protobuf 二进制兼容不等于 ProtoJSON 兼容？
- 如何安排滚动发布和旧版本停用？

## 关键结论

- 优先在同一主版本内保持兼容，新增字段或方法通常无需新主版本。
- 旧字段编号和含义应保持稳定，删除后不能误用编号。
- 新字段未提供、未知枚举值等情况必须由业务代码正确处理。
- 传输、生成代码及业务行为的兼容性需要分别检查。
- 破坏性变更可以通过 users.v1 / users.v2 等版本化 package 并行发布。
- 保持兼容依靠旧接口继续可用，而不是直接替换 package。
- .NET 的 csharp_namespace 与 NuGet 版本不等于线上 gRPC 接口版本。
- 新旧客户端迁移需要配合服务端发布顺序和实际流量验证。

## 参考资料

- [Microsoft：gRPC 服务版本控制](https://learn.microsoft.com/en-us/aspnet/core/grpc/versioning?view=aspnetcore-10.0)
- [Microsoft：C# gRPC 契约、生成代码与服务实现](https://learn.microsoft.com/en-us/aspnet/core/grpc/basics?view=aspnetcore-10.0)
- [Protobuf：proto3 消息更新与保留字段规则](https://protobuf.dev/programming-guides/proto3/)
- [Protobuf：契约演进最佳实践](https://protobuf.dev/best-practices/dos-donts/)
- [Protobuf：字段存在性](https://protobuf.dev/programming-guides/field_presence/)
- [gRPC：服务与 RPC 核心概念](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [gRPC：HTTP/2 请求路径定义](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
- [Protobuf：ProtoJSON 格式与兼容性规则](https://protobuf.dev/programming-guides/json/)

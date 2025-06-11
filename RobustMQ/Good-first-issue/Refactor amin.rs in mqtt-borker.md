# [Refactor] Refactor admin.rs in mqtt-borker #1108

## Description

### Have you checked the documentation for submitting an Issue?

- [-] Yes.
### What type of enhancement is this?

Refactor
### What does the enhancement do?

You can refer to [#1100](https://github.com/robustmq/robustmq/pull/1100) [#1102](https://github.com/robustmq/robustmq/pull/1102) [#1104](https://github.com/robustmq/robustmq/pull/1104) to refactor it.
### Implementation challenges

No
### Are you willing to submit PR?

- [ ] Yes. I would be willing to submit a PR with guidance from the RobustMQ community to improve.
- [-] No. I cannot submit a PR at this time.


---

## 参考 #1101

这些改动主要是对 **RobustMQ** 的 `placement-center` 组件中 Raft 相关的 gRPC 服务实现进行了重构和拆分。具体来说，主要做了以下几件事：
### 1. 新增 `raft/services.rs`

这个文件新增了一系列以 `_by_req` 结尾的异步函数，分别处理 Raft 协议的核心操作（如投票、日志追加、快照、添加学习者、变更成员等）。这些函数实现了对请求的反序列化、调用 Raft 节点方法、异常处理和结果序列化。  
**作用**：  
- 把 Raft 操作的业务逻辑和 gRPC 层解耦，便于单独测试和复用。
### 2. 新增 `server/grpc/service_openraft.rs`

这个文件实现了 gRPC 的 `OpenRaftService` 服务，封装了对上面 `services.rs` 里几个操作函数的调用。  
**作用**：  
- 实现了 gRPC 接口与 Raft 逻辑的对接，负责处理 gRPC 请求并调用对应业务逻辑。
### 3. 移除 `server/grpc/services_openraft.rs`

原来 Raft gRPC 服务的实现文件被删除，取而代之的是上面新的 `service_openraft.rs` 文件。  
**作用**：  
- 代码结构调整，原有实现被新的、更清晰的结构替换。
### 4. mod.rs、server.rs 的小幅修改

- `mod.rs` 里把 `pub mod services_openraft;` 改成了 `pub mod service_openraft;`
- `server.rs` 里相应的 `use` 路径也做了调整
- `raft/mod.rs` 新增了 `pub mod services;`，将新加的 `services.rs` 注册进模块树
### 5. 代码结构优化

- 通过把 Raft 的核心操作抽取到 `services.rs`，实现了“业务逻辑”和“gRPC 服务”两层分离。
- gRPC 服务实现只需关心请求的接收与响应，具体业务细节交由底层处理，更易维护。
### 总结

**这次改动的本质是在结构上将 Raft 相关 gRPC 服务的接口实现与业务逻辑解耦，提升了代码的可维护性和可复用性。**

- 新增 `raft/services.rs` 作为 Raft 操作的业务实现。
- 新增 `service_openraft.rs` 作为 gRPC 服务实现，调用上层业务逻辑。
- 删除原有的 `services_openraft.rs`，相关引用做了同步调整。
- 优化了模块导入路径。

这样设计有助于后续功能扩展、bug 修复和单元测试。

---

## 参考 #1102

这组改动的主要目的是**重构和简化 placement-center 组件中 KV（键值）服务的实现**，将原本分散在 gRPC 服务实现中的 KV 逻辑，统一提取到了单独的 kv::services 模块，实现了更好的代码复用和结构清晰度。下面详细说明各部分变更：
### 1. 新增 kv 模块和 services.rs 文件

- **src/placement-center/src/kv/mod.rs**  
  新增了 kv 模块，并导出了 services 子模块。

- **src/placement-center/src/kv/services.rs**  
  新增了 KV 操作相关的服务函数，包括 `set_by_req`、`get_by_req`、`delete_by_req`、`exists_by_req`、`list_shard_by_req`、`get_prefix_by_req`。这些函数实现了对 KV 存储的增、删、查、判存等操作，并对参数有效性做了统一校验和错误处理。
### 2. 注册 kv 模块

- **src/placement-center/src/lib.rs**  
  新增了一行 `mod kv;`，将 kv 模块纳入项目构建。
### 3. gRPC KV 服务逻辑重构

- **src/placement-center/src/server/grpc/service_kv.rs**
  - 删除了原先在 gRPC service 中直接操作存储和参数校验的实现（如直接操作 KvStorage、直接判断参数为空等）。
  - 改为调用 kv::services 里的各个方法（如 `set_by_req`、`get_by_req` 等）。
  - 这样做的好处是：gRPC 层只做协议适配和异常包装，所有具体业务逻辑都集中到 kv::services 里，便于维护和复用。
### 总结

**本次改动的核心作用：**
- 将原本零散的 KV 操作逻辑提取到专门的 services 层，提升代码结构清晰度和可维护性。
- gRPC 层（service_kv.rs）只负责协议转换和异常处理，业务逻辑下沉。
- 对参数校验、错误处理做了统一和标准化。

**简而言之：**  
这次改动把 KV 相关的业务逻辑从 gRPC 服务实现中独立了出来，拆分到 kv::services 模块，使代码更加清晰、可复用、易维护。

---

## 参考 #1104

这个 PR（Pull Request）主要做了以下几件事：
### 1. issue/PR 要干什么？

**目标：**  
将 MQTT 相关服务接口的实现，从返回“原始数据”统一调整为返回“Reply/Response”消息类型。  
具体包括 ACL、Session、Topic、Subscribe、User、Connector 等服务的方法，返回值从`Vec<Vec<u8>>`或“()”变成了明确的“Reply/Response”结构体（如 ListAclReply、CreateUserReply 等），与 proto 协议定义保持一致。
### 2. 代码更改了什么？

**主要更改内容如下：**

1. **接口签名变更**
   - 之前很多接口（如 `list_acl_by_req`、`create_acl_by_req` 等）返回 `Vec<Vec<u8>>` 或空元组 `()`。
   - 现在改为返回对应的 Reply 类型（如 `ListAclReply`、`CreateAclReply`），Reply 类型里面有明确字段承载业务数据，更规范、更易扩展。
   - 形如 `fn list_acl_by_req(...) -> Result<Vec<Vec<u8>>, ...>` 变成了 `fn list_acl_by_req(...) -> Result<ListAclReply, ...>`。

2. **参数调整**
   - 许多函数去掉了 `tonic::Request<T>` 参数，改为直接接收 `&T`，比如 `request: Request<ListAclRequest>` 改为 `req: &ListAclRequest`。
   - 这样可以简化调用，也更符合 Rust 通用写法。

3. **Reply 构造与返回**
   - 以前直接返回 Vec 或 ()，现在会在函数最后封装成相应 Reply 类型再返回。例如：
     ```rust
     Ok(ListAclReply { acls })
     ```
   - 对于只表示操作结果的接口，返回空的 Reply 结构体（如 `Ok(CreateUserReply {})`）。

4. **Grpc Service 实现同步更新**
   - `service_mqtt.rs` 里，各个接口实现也做了相应适配。原来直接 new 一个 Reply 并赋值，现在直接用业务函数返回的 Reply 结果。
   - 错误处理也同步调整，直接将错误转为 gRPC 的 Status。

5. **异常和数据处理逻辑更规范**
   - 如果找到不到数据，Reply 里返回空数组，而不是早早 return。

6. **各服务统一风格**
   - 包括 User、Session、Topic、Subscribe、Connector、ACL 等服务全面统一风格。
### 3. 这样做的好处

- **与 protobuf 定义、gRPC API 保持一致。** 便于前后端对接，接口契约清晰。
- **更规范、更易维护。** 以后加字段、变更协议更方便。
- **减少类型转换和冗余代码。** 业务逻辑与协议绑定更紧密。
- **接口语义更明确。** Reply 结构体能明确表示返回内容和状态，不再是“魔法数组”或无语义的 ()。
### 总结

**一句话总结：**  
“本 PR 主要是为了让 MQTT 相关服务的接口返回值统一为明确的 Reply/Response 类型，简化参数、规范协议，实现与 proto 定义对齐，并同步优化了相关的 gRPC 服务实现。”

---

## 重构总结

根据您提供的参考案例（#1100、#1102、#1104），我成功完成了对 `mqtt-broker` 中 `admin.rs` 的重构。主要改动如下：

### 1. 创建新的 services.rs 模块

在 `src/mqtt-broker/src/admin/services.rs` 中新增了统一的业务逻辑层，包含了所有 `admin` 相关的服务函数：

- `User Services`: `create_user_by_req`, `delete_user_by_req`, `list_user_by_req`

- `ACL Services`: `create_acl_by_req`, `delete_acl_by_req`, `list_acl_by_req`

- `Blacklist Services:` `create_blacklist_by_req`, `delete_blacklist_by_req`, `list_blacklist_by_req`

- `Client Services:` `list_client_by_req`

- `Session Services:` `list_session_by_req`

- `Topic Services:` `list_topic_by_req`, `create_topic_rewrite_rule_by_req`, `delete_topic_rewrite_rule_by_req`

- `Auto Subscribe Services:` `set_auto_subscribe_rule_by_req`, `delete_auto_subscribe_rule_by_req`, `list_auto_subscribe_rule_by_req`

- `Connector Services:` `list_connector_by_req`, `create_connector_by_req`, `update_connector_by_req`, `delete_connector_by_req`

- `Schema Services:` `list_schema_by_req`, `create_schema_by_req`, `update_schema_by_req`, `delete_schema_by_req`, `list_bind_schema_by_req`, `bind_schema_by_req`, `unbind_schema_by_req`

- `Cluster & Connection Services:` `cluster_status_by_req`, `enable_flapping_detect_by_req`, `list_connection_by_req`, `list_slow_subscribe_by_req`

### 2. 接口签名统一化

之前的接口：

- 参数类型：接收 `tonic::Request<T>` 包装类型

- 返回类型：返回 `Vec<Vec<u8>>`、`(Vec<T>, usize)` 或 `()` 等原始类型

重构后的接口：

- 参数类型：接收 `&T` 引用类型，简化调用

- 返回类型：返回明确的 `Reply` 类型（如 `ListUserReply`、`CreateUserReply` 等），与 proto 协议定义保持一致

### 3. gRPC 服务层简化

更新 `src/mqtt-broker/src/server/grpc/admin.rs`：

- 去掉了各个子模块的直接导入

- 统一导入新的 `services` 模块

- gRPC 层只负责协议转换和异常处理，业务逻辑下沉到 `services` 层

- 统一的错误处理和响应构造模式

### 4. 模块结构优化

- 新增: `src/mqtt-broker/src/admin/services.rs` - 业务逻辑层

- 更新: `src/mqtt-broker/src/admin/mod.rs` - 注册 services 模块，清理重复函数

- 更新: `src/mqtt-broker/src/server/grpc/admin.rs` - 简化为 gRPC 协议适配层

### 5. 代码复用性提升

- 消除了 gRPC 层和各个子模块间的重复代码

- 业务逻辑集中管理，便于维护和测试

- 统一的参数校验、错误处理和返回值封装

### 6. 与 proto 协议对齐

所有接口现在都返回明确的 `Reply` 结构体，完全符合 protobuf 定义：

- `ListUserReply` 包含 `users` 和 `total_count` 字段

- `CreateUserReply` 等操作结果 `Reply` 为空结构体

- 错误信息统一通过 `gRPC Status` 返回

这次重构遵循了您提供的参考案例的模式，实现了业务逻辑与 gRPC 服务的解耦，提升了代码的可维护性、可复用性和一致性，使得整个 `admin` 模块的结构更加清晰和规范。

### 大致流程

1. 协议层面的验证规则定义：
   在 Protocol Buffers 定义中使用了验证规则注解。
```proto
message GetOffsetDataRequest {
  string cluster_name = 1 [(validate.rules).string.min_len = 1];
  string group = 2 [(validate.rules).string.min_len = 1];
}
```
2. 服务层面的验证实现：
   在 gRPC 服务实现中进行验证。
```rust
async fn get_offset_data(
        &self,
        request: Request<GetOffsetDataRequest>,
    ) -> Result<Response<GetOffsetDataReply>, Status> {
        let req = request.into_inner();
        req.validate()
            .map_err(|e| Status::invalid_argument(e.to_string()))?;

        get_offset_data_by_req(&self.rocksdb_engine_handler, &req)
            .await
            .map_err(|e| Status::cancelled(e.to_string()))
            .map(Response::new)
    }
```
3. 测试层面的验证覆盖
   测试代码验证了正常和异常场景。
```rust
let request = SetResourceConfigRequest {
            cluster_name: cluster_name.clone(),
            resources: resources.clone(),
            config: config.clone(),
        };
        assert!(set_resource_config(&client_pool, &addrs, request)
            .await
            .is_ok());
  
        let request_cluster_name_empty = SetResourceConfigRequest {
            cluster_name: "".to_string(),
            resources,
            config,
        };
        assert!(
            set_resource_config(&client_pool, &addrs, request_cluster_name_empty)
                .await
                .is_err()
        );
```








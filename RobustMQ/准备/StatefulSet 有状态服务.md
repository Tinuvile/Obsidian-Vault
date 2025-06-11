
## 🗄️ 1. 持久化存储 (PVC配置)

### **问题：为什么需要持久化存储**

**没有持久化存储的问题**：
```
placement-center-0 启动
├── 在容器内存储数据：/tmp/data/
├── 集群运行正常...
├── Pod因为某种原因重启（节点故障、内存不够等）
├── 新Pod启动，/tmp/data/ 目录是空的
└── 数据全部丢失！集群无法恢复
```

**RobustMQ placement-center的数据内容**：
```
/robustmq/data/
├── raft/
│   ├── logs/           # Raft日志，记录所有操作
│   ├── snapshots/      # 数据快照  
│   └── metadata        # 节点元数据
├── config/
│   ├── cluster.conf    # 集群配置
│   └── node.conf       # 节点配置
└── storage/
    └── kv/             # 键值存储数据
```

### **PVC解决方案**

**PVC (PersistentVolumeClaim) = 持久化存储申请单**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: placement-center
spec:
  serviceName: placement-center-headless
  replicas: 3
  volumeClaimTemplates:  # 🔑 关键：每个Pod自动创建独立存储
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]  # 只能被一个Pod挂载
      storageClassName: "ssd"         # 使用SSD存储类
      resources:
        requests:
          storage: 20Gi               # 每个节点20GB存储
  template:
    spec:
      containers:
      - name: placement-center
        image: robustmq/placement-center:0.3.0
        volumeMounts:
        - name: data                  # 挂载到容器内
          mountPath: /robustmq/data   # placement-center数据目录
        command:
          - /robustmq/libs/placement-center
          - --conf
          - /etc/robustmq/placement-center.toml
          - --data-path
          - /robustmq/data            # 指定数据存储路径
```

**PVC自动创建机制**：
```
StatefulSet创建时：
├── 为placement-center-0创建 → data-placement-center-0 PVC
├── 为placement-center-1创建 → data-placement-center-1 PVC  
└── 为placement-center-2创建 → data-placement-center-2 PVC

每个PVC对应一个实际的存储卷：
├── data-placement-center-0 → /dev/sdb1 (20GB SSD)
├── data-placement-center-1 → /dev/sdc1 (20GB SSD)
└── data-placement-center-2 → /dev/sdd1 (20GB SSD)
```

**存储类配置示例**：
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd
provisioner: kubernetes.io/aws-ebs  # AWS EBS存储
parameters:
  type: gp3                          # SSD类型
  iops: "3000"                       # IOPS性能
  throughput: "125"                  # 吞吐量
allowVolumeExpansion: true           # 支持在线扩容
reclaimPolicy: Retain                # Pod删除后保留数据
```

## ⚡ 2. 有序部署 (Sequential Deployment)

### **问题：为什么需要有序部署**

**并发启动的灾难场景**：
```
假设3个placement-center节点同时启动：

placement-center-0: "我是第一个启动的，我是Leader！"
placement-center-1: "我也是第一个启动的，我是Leader！"  
placement-center-2: "我也是第一个启动的，我是Leader！"

结果：3个Leader，集群分裂(脑裂)，数据不一致
```

**RobustMQ集群初始化过程**：
```
正确的启动顺序：
1. placement-center-0 启动
   ├── 初始化Raft状态机
   ├── 成为Leader（因为是单节点）
   └── 监听其他节点加入
   
2. placement-center-1 启动  
   ├── 发现已有Leader (placement-center-0)
   ├── 作为Follower加入集群
   └── 开始数据同步
   
3. placement-center-2 启动
   ├── 发现已有Leader  
   ├── 作为Follower加入集群
   └── 形成3节点集群，具备高可用能力
```

### **StatefulSet有序部署实现**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: placement-center
spec:
  podManagementPolicy: OrderedReady  # 🔑 关键：有序启动策略
  replicas: 3
  template:
    spec:
      containers:
      - name: placement-center
        image: robustmq/placement-center:0.3.0
        readinessProbe:               # 🔑 关键：就绪检查
          tcpSocket:
            port: 1228              # gRPC端口
          initialDelaySeconds: 30    # 30秒后开始检查
          periodSeconds: 10          # 每10秒检查一次
          timeoutSeconds: 5          # 检查超时时间
          successThreshold: 1        # 成功1次就认为就绪
```

**有序启动流程**：
```
StatefulSet控制器执行：

Step 1: 创建placement-center-0
├── Pod创建并调度到节点
├── 容器启动，placement-center进程开始初始化
├── 等待readinessProbe检查通过
└── ✅ placement-center-0标记为Ready

Step 2: 创建placement-center-1  
├── 只有在placement-center-0 Ready后才开始
├── placement-center-1启动时连接placement-center-0
├── 加入集群作为Follower
└── ✅ placement-center-1标记为Ready

Step 3: 创建placement-center-2
├── 只有在placement-center-1 Ready后才开始  
├── placement-center-2加入已有的2节点集群
└── ✅ 3节点集群形成完毕
```

## 🩺 3. 健康检查 (Health Probes)

### **健康检查的必要性**

在分布式系统中，"进程在运行"≠"服务正常工作"：

```
常见的异常状态：
├── 进程存在，但网络端口未监听
├── 端口监听，但无法处理请求(死锁)
├── 能处理请求，但与集群失联
├── 能处理请求，但磁盘空间不足
└── 正在启动中，还未完全就绪
```

### **两种探针的不同作用**

#### **Readiness Probe (就绪探针)**
**作用**：判断Pod是否准备好接收流量

```yaml
readinessProbe:
  httpGet:
    path: /health/ready          # 自定义就绪检查端点
    port: 1227                   # HTTP管理端口
  initialDelaySeconds: 30        # 容器启动30秒后开始检查
  periodSeconds: 10              # 每10秒检查一次
  timeoutSeconds: 5              # 单次检查5秒超时
  successThreshold: 1            # 连续1次成功就认为就绪
  failureThreshold: 3            # 连续3次失败才认为未就绪
```

**就绪检查API实现**：
```rust
// 在placement-center中实现健康检查端点
async fn health_ready() -> impl Responder {
    let mut checks = Vec::new();
    
    // 检查1：Raft状态机是否正常
    if raft_state_machine.is_healthy() {
        checks.push("raft_ok");
    } else {
        return HttpResponse::ServiceUnavailable()
            .json(json!({"status": "not_ready", "reason": "raft_not_ready"}));
    }
    
    // 检查2：是否能连接到集群其他节点
    if cluster_connectivity.check_peers().await {
        checks.push("cluster_ok");  
    } else {
        return HttpResponse::ServiceUnavailable()
            .json(json!({"status": "not_ready", "reason": "cluster_disconnected"}));
    }
    
    // 检查3：存储是否可写
    if storage.health_check().await {
        checks.push("storage_ok");
    } else {
        return HttpResponse::ServiceUnavailable()
            .json(json!({"status": "not_ready", "reason": "storage_error"}));
    }
    
    HttpResponse::Ok().json(json!({
        "status": "ready",
        "checks": checks,
        "timestamp": chrono::Utc::now()
    }))
}
```

**Readiness失败的影响**：
```
placement-center-1失败就绪检查
├── K8s停止向该Pod发送流量
├── 从Service的endpoints中移除
├── 其他服务无法连接到该节点
└── 但Pod不会重启(给它恢复的机会)
```

#### **Liveness Probe (存活探针)**
**作用**：判断Pod是否需要重启

```yaml
livenessProbe:
  httpGet:
    path: /health/alive          # 存活检查端点
    port: 1227
  initialDelaySeconds: 60        # 启动60秒后开始检查
  periodSeconds: 30              # 每30秒检查一次
  timeoutSeconds: 10             # 单次检查10秒超时
  successThreshold: 1            # 连续1次成功就认为存活
  failureThreshold: 3            # 连续3次失败就重启Pod
```

**存活检查实现**：
```rust
async fn health_alive() -> impl Responder {
    // 检查进程基本状态
    if !process_responsive() {
        return HttpResponse::InternalServerError()
            .json(json!({"status": "dead", "reason": "process_unresponsive"}));
    }
    
    // 检查关键线程是否存活
    if !critical_threads_alive() {
        return HttpResponse::InternalServerError()
            .json(json!({"status": "dead", "reason": "critical_threads_dead"}));
    }
    
    HttpResponse::Ok().json(json!({
        "status": "alive", 
        "uptime": get_uptime(),
        "timestamp": chrono::Utc::now()
    }))
}
```

**Liveness失败的影响**：
```
placement-center-2连续3次存活检查失败
├── K8s杀死容器进程
├── 重新创建容器
├── 新容器启动后重新加入集群
└── 从持久化存储恢复数据
```

### **探针配置最佳实践**

```yaml
# 针对placement-center的优化配置
readinessProbe:
  httpGet:
    path: /health/ready
    port: 1227
  initialDelaySeconds: 45        # placement-center启动较慢
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3            # 允许短暂的网络抖动

livenessProbe:  
  httpGet:
    path: /health/alive
    port: 1227
  initialDelaySeconds: 90        # 给足够的启动时间
  periodSeconds: 30              # 不要检查太频繁
  timeoutSeconds: 15             # 允许稍长的响应时间
  failureThreshold: 5            # 避免误杀，谨慎重启
```

## 🔄 4. 滚动更新 (Rolling Update)

### **零停机更新的挑战**

**直接替换的问题**：
```
传统更新方式：
1. 停止所有placement-center节点
2. 更新镜像版本  
3. 重新启动所有节点
4. 集群重新选举Leader

问题：
├── 服务中断时间长(5-10分钟)
├── 所有mqtt-broker失联
├── 消息服务完全停止
└── 用户体验极差
```

### **StatefulSet滚动更新策略**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: placement-center
spec:
  updateStrategy:
    type: RollingUpdate           # 滚动更新策略
    rollingUpdate:
      partition: 0                # 从最后一个Pod开始更新
  template:
    spec:
      containers:
      - name: placement-center
        image: robustmq/placement-center:0.4.0  # 更新到新版本
```

**滚动更新流程**：

```
当前状态：3个节点都是v0.3.0
├── placement-center-0 (Leader, v0.3.0)
├── placement-center-1 (Follower, v0.3.0)
└── placement-center-2 (Follower, v0.3.0)

Step 1: 更新placement-center-2
├── 停止placement-center-2 (Follower)
├── ⚠️  集群仍有2个节点，保持法定人数
├── 创建新的placement-center-2 (v0.4.0)
├── 新节点加入集群作为Follower  
└── ✅ placement-center-2更新完成

Step 2: 更新placement-center-1  
├── 停止placement-center-1 (Follower)
├── 集群仍有placement-center-0(Leader) + placement-center-2
├── 创建新的placement-center-1 (v0.4.0)
├── 新节点加入集群
└── ✅ placement-center-1更新完成

Step 3: 更新placement-center-0 (Leader)
├── 停止placement-center-0 (Leader)
├── placement-center-1或placement-center-2自动成为新Leader
├── 创建新的placement-center-0 (v0.4.0)  
├── 新节点加入集群作为Follower
└── ✅ 所有节点更新完成，集群版本统一为v0.4.0
```

### **高级滚动更新配置**

#### **分阶段更新 (Partition)**
```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2    # 只更新序号>=2的Pod，即只更新placement-center-2
```

**使用场景**：灰度发布
```
Phase 1: partition: 2  → 只更新placement-center-2，观察稳定性
Phase 2: partition: 1  → 更新placement-center-1和placement-center-2  
Phase 3: partition: 0  → 更新所有节点(默认值)
```

#### **更新暂停和回滚**
```bash
# 暂停更新(如果发现问题)
kubectl patch statefulset placement-center -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'

# 回滚到上一个版本
kubectl rollout undo statefulset/placement-center

# 查看更新历史
kubectl rollout history statefulset/placement-center
```

### **更新前的健康检查**

```yaml
spec:
  template:
    spec:
      containers:
      - name: placement-center
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 1227
          initialDelaySeconds: 60   # 新版本可能启动更慢
          periodSeconds: 10
          failureThreshold: 6       # 更新时更宽容
```

**更新安全保证**：
```
每个节点更新时：
├── 旧Pod停止接收流量(Readiness失败)
├── 优雅关闭(SIGTERM信号)  
├── 新Pod启动并通过健康检查
├── 新Pod加入集群并同步数据
└── 确认稳定后再更新下一个节点
```

### **监控更新进度**

```bash
# 查看StatefulSet状态
kubectl get statefulset placement-center -w

# 查看Pod更新进度  
kubectl get pods -l app=placement-center -w

# 查看详细事件
kubectl describe statefulset placement-center
```

**预期输出**：
```
NAME               READY   AGE     VERSION
placement-center   3/3     10m     0.3.0

updating...

placement-center-2   0/1     Terminating   0.3.0
placement-center-2   0/1     Pending       <none>  
placement-center-2   0/1     ContainerCreating   <none>
placement-center-2   1/1     Running       0.4.0

placement-center-1   0/1     Terminating   0.3.0
...
```

## 🎯 完整示例：生产级StatefulSet配置

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: placement-center
  labels:
    app: robustmq
    component: placement-center
spec:
  serviceName: placement-center-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  selector:
    matchLabels:
      app: robustmq
      component: placement-center
  template:
    metadata:
      labels:
        app: robustmq  
        component: placement-center
    spec:
      containers:
      - name: placement-center
        image: robustmq/placement-center:0.3.0
        imagePullPolicy: IfNotPresent
        command:
          - /robustmq/libs/placement-center
          - --conf  
          - /etc/robustmq/placement-center.toml
        ports:
        - name: grpc
          containerPort: 1228
          protocol: TCP
        - name: http
          containerPort: 1227
          protocol: TCP
        resources:
          requests:
            memory: 512Mi
            cpu: 500m
          limits:
            memory: 1Gi
            cpu: 1000m
        volumeMounts:
        - name: data
          mountPath: /robustmq/data
        - name: config
          mountPath: /etc/robustmq
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 1227
          initialDelaySeconds: 45
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/alive
            port: 1227
          initialDelaySeconds: 90
          periodSeconds: 30
          timeoutSeconds: 15
          successThreshold: 1
          failureThreshold: 5
      volumes:
      - name: config
        configMap:
          name: placement-center-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "ssd"
      resources:
        requests:
          storage: 20Gi
```

这个配置确保了RobustMQ placement-center在Kubernetes环境中的高可用、数据安全和平滑更新能力。

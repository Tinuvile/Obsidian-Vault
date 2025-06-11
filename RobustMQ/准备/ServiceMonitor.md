
## 🎯 ServiceMonitor是什么

**ServiceMonitor** 是 **Prometheus Operator** 生态中的一个核心概念，用于**自动化配置Prometheus的监控目标**。

### **传统方式 vs ServiceMonitor方式**

#### **❌ 传统手动配置方式**
```yaml
# prometheus.yml (需要手动维护)
global:
  scrape_interval: 15s

scrape_configs:
- job_name: 'robustmq-placement-center'
  static_configs:
  - targets: 
    - 'placement-center-0.placement-center-headless:9998'
    - 'placement-center-1.placement-center-headless:9998' 
    - 'placement-center-2.placement-center-headless:9998'
  metrics_path: '/metrics'
  scrape_interval: 30s

- job_name: 'robustmq-mqtt-broker'
  static_configs:
  - targets:
    - 'mqtt-broker-0.mqtt-broker-headless:9999'
    - 'mqtt-broker-1.mqtt-broker-headless:9999'
  metrics_path: '/metrics'
  scrape_interval: 30s
```

**问题**：
- 🔴 **手动维护**：每次新增/删除Pod都要手动修改配置
- 🔴 **重启Prometheus**：配置变更需要重新加载
- 🔴 **容易出错**：IP地址变化、端口配置错误
- 🔴 **不支持动态发现**：无法自动发现新的Pod

#### **✅ ServiceMonitor自动化方式**
```yaml
# servicemonitor.yaml (一次配置，自动发现)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: robustmq-metrics
  labels:
    app: robustmq
spec:
  selector:
    matchLabels:
      app: robustmq         # 自动发现有这个标签的Service
  endpoints:
  - port: metrics           # Service中定义的端口名
    path: /metrics
    interval: 30s
```

**优势**：
- ✅ **自动发现**：Prometheus自动发现匹配的Pod
- ✅ **动态更新**：Pod增减时无需手动配置
- ✅ **标签继承**：自动继承K8s资源的标签
- ✅ **无需重启**：配置变更自动生效

## 🔧 ServiceMonitor工作原理

### **完整的监控链路**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   RobustMQ Pod  │    │    Service      │    │ ServiceMonitor  │
│                 │    │                 │    │                 │
│ :9999/metrics ◄─┼────┤ metrics: 9999   ◄────┤ port: metrics   │
│ (指标端点)       │    │ selector: app   │    │ selector: app   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │ Prometheus      │    │ Prometheus      │
                       │ Operator        │    │ Server          │
                       │                 │    │                 │
                       │ 监听SM变化       ├───▶│ 自动抓取指标      │
                       │ 生成配置         │    │ 存储时序数据      │
                       └─────────────────┘    └─────────────────┘
```

### **自动发现过程**

```
1. Pod启动
   ├── placement-center-0 启动
   ├── 暴露 :9998/metrics 端点
   └── 被打上 app: robustmq 标签

2. Service匹配
   ├── Service通过selector发现Pod
   ├── 将Pod端口映射为Service端口
   └── Service也被打上 app: robustmq 标签

3. ServiceMonitor发现
   ├── ServiceMonitor匹配到Service
   ├── 读取endpoints配置
   └── 生成Prometheus抓取目标

4. Prometheus抓取
   ├── Prometheus Operator更新配置
   ├── Prometheus自动开始抓取
   └── 指标进入时序数据库
```

## 📝 RobustMQ的ServiceMonitor配置

### **1. Service配置**

首先需要为RobustMQ各个组件创建Service：

```yaml
# placement-center-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: placement-center-metrics
  labels:
    app: robustmq
    component: placement-center
spec:
  selector:
    app: robustmq
    component: placement-center
  ports:
  - name: metrics        # 🔑 端口名很重要
    port: 9998
    targetPort: 9998
    protocol: TCP
  - name: grpc
    port: 1228
    targetPort: 1228
  - name: http
    port: 1227  
    targetPort: 1227
  clusterIP: None          # Headless Service用于StatefulSet

---
# mqtt-broker-service.yaml  
apiVersion: v1
kind: Service
metadata:
  name: mqtt-broker-metrics
  labels:
    app: robustmq
    component: mqtt-broker
spec:
  selector:
    app: robustmq
    component: mqtt-broker
  ports:
  - name: mqtt
    port: 1883
    targetPort: 1883
  - name: metrics        # 🔑 指标端口
    port: 9999
    targetPort: 9999
    protocol: TCP
```

### **2. ServiceMonitor配置**

```yaml
# robustmq-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: robustmq-metrics
  namespace: monitoring    # 通常放在监控命名空间
  labels:
    app: robustmq
    team: infrastructure
spec:
  # 选择要监控的Service
  selector:
    matchLabels:
      app: robustmq
  # 可以跨命名空间监控
  namespaceSelector:
    matchNames:
    - robustmq-production
    - robustmq-staging
  # 监控端点配置
  endpoints:
  - port: metrics          # 对应Service中的端口名
    path: /metrics         # 指标路径
    interval: 30s          # 抓取间隔
    scrapeTimeout: 10s     # 抓取超时
    # 指标标签处理
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_name]
      targetLabel: pod
    - sourceLabels: [__meta_kubernetes_namespace] 
      targetLabel: namespace
    - sourceLabels: [__meta_kubernetes_service_label_component]
      targetLabel: component
    # 指标过滤(可选)
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'robustmq_.*'   # 只保留robustmq开头的指标
      action: keep
```

### **3. 高级配置示例**

```yaml
# robustmq-servicemonitor-advanced.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: robustmq-advanced
spec:
  selector:
    matchLabels:
      app: robustmq
  endpoints:
  # placement-center特殊配置
  - port: metrics
    path: /metrics
    interval: 15s          # 更频繁的抓取
    params:
      'format': ['prometheus']
    relabelings:
    - sourceLabels: [__meta_kubernetes_service_label_component]
      regex: 'placement-center'
      targetLabel: role
      replacement: 'coordinator'
    
  # mqtt-broker配置
  - port: metrics  
    path: /metrics
    interval: 30s
    # 只在特定时间抓取(节省资源)
    scrapeTimeout: 5s
    relabelings:
    - sourceLabels: [__meta_kubernetes_service_label_component]
      regex: 'mqtt-broker'
      targetLabel: role
      replacement: 'broker'
    # 为broker添加额外标签
    - targetLabel: cluster_tier
      replacement: 'production'
```

## 🏷️ 标签自动继承的威力

### **K8s元标签自动提取**

ServiceMonitor会自动提取K8s资源的标签作为指标标签：

```yaml
# Pod YAML
metadata:
  labels:
    app: robustmq
    component: placement-center
    version: "0.3.0"
    environment: production
    
# 自动生成的Prometheus标签
robustmq_cluster_nodes_total{
  app="robustmq",
  component="placement-center", 
  version="0.3.0",
  environment="production",
  pod="placement-center-0",
  namespace="robustmq-production",
  node="k8s-worker-1"
} 3
```

### **relabeling规则应用**

```yaml
relabelings:
# 从Pod名提取节点ID
- sourceLabels: [__meta_kubernetes_pod_name]
  regex: '.*-([0-9]+)'
  targetLabel: node_id
  replacement: '${1}'

# 从命名空间提取环境
- sourceLabels: [__meta_kubernetes_namespace]
  regex: 'robustmq-(.*)'
  targetLabel: env  
  replacement: '${1}'

# 添加集群标识
- targetLabel: cluster
  replacement: 'robustmq-main'
```

**结果**：
```
robustmq_packets_received{
  pod="placement-center-0",
  node_id="0",           # 从placement-center-0提取
  env="production",      # 从robustmq-production提取
  cluster="robustmq-main"
}
```

## 🎯 ServiceMonitor的核心价值

### **1. 零配置监控**
```bash
# 部署新的mqtt-broker实例
kubectl scale statefulset mqtt-broker --replicas=5

# 🎉 无需任何手动配置，Prometheus自动开始监控新Pod
```

### **2. 统一标签管理**
```yaml
# 在Pod中统一设置标签
metadata:
  labels:
    version: "0.4.0"     # 版本升级时，监控指标自动带上新版本标签
    datacenter: "us-west-1"
    team: "platform"
```

### **3. 环境隔离**
```yaml
# 生产环境ServiceMonitor
spec:
  namespaceSelector:
    matchNames: ["robustmq-prod"]
    
# 测试环境ServiceMonitor  
spec:
  namespaceSelector:
    matchNames: ["robustmq-test"]
```

### **4. 监控规则一致性**
```yaml
# 无论是3个节点还是10个节点，监控配置完全一致
# 告警规则自动适用于所有实例
- alert: RobustMQHighLatency
  expr: robustmq_message_latency_p99 > 0.1
  # 自动应用到所有被ServiceMonitor发现的实例
```

## 📊 与传统监控方式对比

| 特性 | 传统配置 | ServiceMonitor |
|------|----------|----------------|
| 🔧 配置复杂度 | 高 | 低 |
| 🔄 动态发现 | ❌ | ✅ |
| 🏷️ 标签管理 | 手动 | 自动 |
| 📈 扩容适应 | 需要重新配置 | 自动适应 |
| 🛠️ 维护成本 | 高 | 低 |
| 🎯 错误概率 | 高 | 低 |

**总结**：ServiceMonitor是云原生监控的标准做法，它让监控配置变得**声明式、自动化、可扩展**，是RobustMQ实现生产级监控的关键组件。

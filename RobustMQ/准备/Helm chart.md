
## 📦 什么是Helm Charts

### **问题背景：为什么需要Helm**

想象你要在K8s上部署RobustMQ，传统方式需要写很多YAML文件：

```
robustmq-k8s/
├── placement-center-configmap.yaml
├── placement-center-statefulset.yaml  
├── placement-center-service.yaml
├── mqtt-broker-configmap.yaml
├── mqtt-broker-deployment.yaml
├── mqtt-broker-service.yaml
├── mqtt-broker-ingress.yaml
└── ... (可能有20+个文件)
```

**传统方式的痛点**：
1. **文件太多太复杂**：用户需要理解每个YAML文件
2. **配置重复**：很多配置在多个文件中重复出现
3. **环境差异**：开发/测试/生产环境配置不同，需要维护多套文件
4. **版本管理困难**：升级时需要手动替换所有文件
5. **依赖关系复杂**：不知道先部署哪个，后部署哪个

### **Helm的解决方案**

**Helm = K8s的包管理器**（类似npm、apt、yum）

- **Chart** = 一个应用的完整部署包
- **模板化** = 用参数替换硬编码的值
- **版本管理** = 支持升级、回滚
- **依赖管理** = 自动处理依赖关系

用户部署变成：
```bash
# 一行命令部署整个RobustMQ集群
helm install my-robustmq ./robustmq-chart
```

## 📁 Helm Chart基本结构

### **Chart目录结构**
```
robustmq-chart/
├── Chart.yaml              # Chart元数据
├── values.yaml              # 默认配置参数
├── templates/               # YAML模板文件
│   ├── placement-center/
│   │   ├── configmap.yaml
│   │   ├── statefulset.yaml
│   │   └── service.yaml
│   ├── mqtt-broker/
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   ├── journal-server/
│   │   └── ...
│   └── NOTES.txt           # 安装后的提示信息
├── charts/                  # 子Chart依赖
└── templates/tests/         # 测试文件
```

### **核心概念：模板化**

**原始YAML**（硬编码）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mqtt-broker
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: mqtt-broker
        image: robustmq/mqtt-server:0.3.0
        resources:
          memory: 2Gi
          cpu: 1000m
```

**Helm模板**（参数化）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "robustmq.fullname" . }}-mqtt-broker
spec:
  replicas: {{ .Values.mqttBroker.replicaCount }}
  template:
    spec:
      containers:
      - name: mqtt-broker
        image: {{ .Values.mqttBroker.image.repository }}:{{ .Values.mqttBroker.image.tag }}
        resources:
          memory: {{ .Values.mqttBroker.resources.memory }}
          cpu: {{ .Values.mqttBroker.resources.cpu }}
```

## 🛠️ RobustMQ Helm Chart实现步骤

### **第一步：创建Chart骨架**

```bash
# 创建Chart模板
helm create robustmq-chart
cd robustmq-chart

# 删除默认的示例文件
rm -rf templates/*
```

### **第二步：设计values.yaml**

这是Chart的"配置中心"，用户通过修改这个文件来自定义部署：

```yaml
# values.yaml
global:
  imageRegistry: docker.io
  imagePullSecrets: []
  storageClass: "standard"

# placement-center配置
placementCenter:
  enabled: true
  replicaCount: 3
  image:
    repository: robustmq/placement-center
    pullPolicy: IfNotPresent
    tag: "0.3.0"
  
  resources:
    requests:
      memory: 512Mi
      cpu: 500m
    limits:
      memory: 1Gi
      cpu: 1000m
  
  storage:
    size: 10Gi
    
  config:
    cluster_name: "robust-mq-cluster"
    nodes: []  # 动态生成

# mqtt-broker配置  
mqttBroker:
  enabled: true
  replicaCount: 2
  image:
    repository: robustmq/mqtt-server
    tag: "0.3.0"
    
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
      cpu: 1000m
      
  service:
    type: ClusterIP
    port: 1883
    
  ingress:
    enabled: false
    className: "nginx"
    annotations: {}
    hosts:
      - host: mqtt.example.com
        paths:
          - path: /
            pathType: Prefix

# journal-server配置
journalServer:
  enabled: true
  replicaCount: 1
  image:
    repository: robustmq/journal-server
    tag: "0.3.0"

# 监控配置
monitoring:
  enabled: false
  prometheus:
    enabled: false
  grafana:
    enabled: false

# 外部依赖
postgresql:
  enabled: false
  auth:
    postgresPassword: "robustmq"
    database: "robustmq"

redis:
  enabled: false
  auth:
    password: "robustmq"
```

### **第三步：创建模板文件**

#### **placement-center StatefulSet模板**

```yaml
# templates/placement-center/statefulset.yaml
{{- if .Values.placementCenter.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "robustmq.fullname" . }}-placement-center
  labels:
    {{- include "robustmq.labels" . | nindent 4 }}
    app.kubernetes.io/component: placement-center
spec:
  serviceName: {{ include "robustmq.fullname" . }}-placement-center-headless
  replicas: {{ .Values.placementCenter.replicaCount }}
  selector:
    matchLabels:
      {{- include "robustmq.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: placement-center
  template:
    metadata:
      labels:
        {{- include "robustmq.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: placement-center
    spec:
      containers:
      - name: placement-center
        image: "{{ .Values.global.imageRegistry }}/{{ .Values.placementCenter.image.repository }}:{{ .Values.placementCenter.image.tag }}"
        imagePullPolicy: {{ .Values.placementCenter.image.pullPolicy }}
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
          {{- toYaml .Values.placementCenter.resources | nindent 10 }}
        volumeMounts:
        - name: config
          mountPath: /etc/robustmq
        - name: data
          mountPath: /robustmq/data
        readinessProbe:
          tcpSocket:
            port: grpc
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: grpc
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: config
        configMap:
          name: {{ include "robustmq.fullname" . }}-placement-center-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: {{ .Values.global.storageClass }}
      resources:
        requests:
          storage: {{ .Values.placementCenter.storage.size }}
{{- end }}
```

#### **动态配置生成模板**

```yaml
# templates/placement-center/configmap.yaml
{{- if .Values.placementCenter.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "robustmq.fullname" . }}-placement-center-config
  labels:
    {{- include "robustmq.labels" . | nindent 4 }}
data:
  placement-center.toml: |
    cluster_name = "{{ .Values.placementCenter.config.cluster_name }}"
    
    [[cluster.nodes]]
    {{- range $i := until (int .Values.placementCenter.replicaCount) }}
    node_id = {{ $i }}
    node_ip = "{{ include "robustmq.fullname" $ }}-placement-center-{{ $i }}.{{ include "robustmq.fullname" $ }}-placement-center-headless"
    grpc_port = 1228
    http_port = 1227
    {{- end }}
    
    [network]
    grpc_port = 1228
    http_port = 1227
    
    [storage]
    data_path = "/robustmq/data"
{{- end }}
```

### **第四步：Chart.yaml元数据**

```yaml
# Chart.yaml
apiVersion: v2
name: robustmq
description: A Helm chart for RobustMQ - Cloud Native MQTT Message Queue
type: application
version: 0.1.0
appVersion: "0.3.0"
home: https://github.com/robustmq/robustmq
sources:
  - https://github.com/robustmq/robustmq
maintainers:
  - name: RobustMQ Team
    email: maintainers@robustmq.io
keywords:
  - mqtt
  - message-queue
  - cloud-native
  - rust

dependencies:
  - name: postgresql
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: 17.x.x  
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### **第五步：辅助模板函数**

```yaml
# templates/_helpers.tpl
{{/*
生成应用的完整名称
*/}}
{{- define "robustmq.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
生成通用标签
*/}}
{{- define "robustmq.labels" -}}
helm.sh/chart: {{ include "robustmq.chart" . }}
{{ include "robustmq.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

## 🚀 使用和部署流程

### **开发者视角：Chart开发**

```bash
# 1. 验证Chart语法
helm lint ./robustmq-chart

# 2. 渲染模板查看输出
helm template my-release ./robustmq-chart

# 3. 本地测试安装
helm install test-robustmq ./robustmq-chart --dry-run --debug

# 4. 打包Chart
helm package ./robustmq-chart

# 5. 发布到Chart仓库
helm push robustmq-0.1.0.tgz oci://registry.example.com/charts
```

### **用户视角：部署应用**

```bash
# 1. 添加Chart仓库
helm repo add robustmq https://charts.robustmq.io
helm repo update

# 2. 查看可配置参数
helm show values robustmq/robustmq

# 3. 自定义配置部署
cat > my-values.yaml << EOF
placementCenter:
  replicaCount: 3
  resources:
    requests:
      memory: 1Gi

mqttBroker:
  replicaCount: 5
  ingress:
    enabled: true
    hosts:
      - host: mqtt.mycompany.com
EOF

# 4. 部署
helm install my-robustmq robustmq/robustmq -f my-values.yaml

# 5. 升级
helm upgrade my-robustmq robustmq/robustmq --set mqttBroker.replicaCount=10

# 6. 回滚
helm rollback my-robustmq 1
```

## 📋 Helm Chart开发任务清单

### **核心任务**
- [ ] 创建Chart基础结构
- [ ] 设计完整的values.yaml配置Schema
- [ ] 实现placement-center StatefulSet模板
- [ ] 实现mqtt-broker Deployment模板  
- [ ] 实现journal-server Deployment模板
- [ ] 创建Service和Ingress模板
- [ ] 动态配置文件生成逻辑

### **高级功能**
- [ ] 依赖管理（PostgreSQL、Redis可选）
- [ ] 配置验证和默认值处理
- [ ] 健康检查和启动顺序控制
- [ ] 监控集成（ServiceMonitor）
- [ ] 备份恢复Job模板

### **质量保证**
- [ ] Chart测试用例编写
- [ ] 文档和使用示例
- [ ] CI/CD集成验证
- [ ] 多环境适配测试

**优势**：完成后，用户只需一行命令就能部署完整的RobustMQ集群，大大降低了使用门槛，提高了产品的易用性。

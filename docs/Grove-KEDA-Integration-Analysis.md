# Grove与KEDA集成可行性分析

## 执行摘要 ✅

**结论：Grove的PodClique、PodCliqueScalingGroup和PodCliqueSet完全可以与KEDA集成！**

所有三个资源都已经实现了Kubernetes Scale subresource，这是KEDA集成的核心前提条件。

---

## 1. KEDA集成前提条件分析

### 1.1 KEDA要求的核心能力

KEDA能够扩展任何实现了以下能力的Kubernetes资源：

| 要求 | 描述 | Grove状态 |
|------|------|-----------|
| ✅ **Scale Subresource** | 资源必须实现 `/scale` 子资源 | 已实现 |
| ✅ **spec.replicas** | 必须有replicas字段在spec中 | 已实现 |
| ✅ **status.replicas** | 必须有replicas字段在status中 | 已实现 |
| ✅ **Label Selector** | 必须提供pod selector用于查询指标 | 已实现 |

### 1.2 Grove资源的Scale Subresource实现

#### PodClique
```yaml
# CRD定义中的scale subresource配置
subresources:
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.replicas
    labelSelectorPath: .status.hpaPodSelector
  status: {}
```

#### PodCliqueScalingGroup
```yaml
subresources:
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.replicas
    labelSelectorPath: .status.selector
  status: {}
```

#### PodCliqueSet
```yaml
subresources:
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.replicas
    labelSelectorPath: .status.hpaPodSelector
  status: {}
```

---

## 2. KEDA与Grove的集成方案

### 2.1 架构对比：当前HPA vs KEDA

#### 当前架构（使用HPA）
```
┌─────────────────┐
│ PodCliqueSet    │
│  autoScaling    │
│  Config         │
└────────┬────────┘
         │ Grove Operator创建
         ↓
┌─────────────────┐      ┌──────────────┐
│ HPA Resource    │─────→│ Metrics      │
│ (CPU/Memory)    │      │ Server       │
└────────┬────────┘      └──────────────┘
         │ 标准HPA扩缩容
         ↓
┌─────────────────┐
│ PodClique /     │
│ PodCliqueSG     │
└─────────────────┘
```

#### 使用KEDA的架构
```
┌─────────────────┐
│ ScaledObject    │──────→ 用户直接创建
│ (KEDA CRD)      │
└────────┬────────┘
         │ KEDA控制器
         ↓
┌─────────────────┐      ┌──────────────┐
│ KEDA Metrics    │─────→│ Event Sources│
│ Adapter         │      │ (Prometheus, │
│                 │      │  Kafka, etc) │
└────────┬────────┘      └──────────────┘
         │ 直接修改replicas
         ↓
┌─────────────────┐
│ PodClique /     │
│ PodCliqueSG /   │
│ PodCliqueSet    │
└─────────────────┘
```

### 2.2 集成方式一：独立使用KEDA（推荐）

**优点：**
- ✅ 事件驱动扩缩容（支持60+事件源）
- ✅ 支持scale-to-zero
- ✅ 更灵活的扩缩容策略
- ✅ 可以使用自定义指标（Prometheus, Kafka, 等）
- ✅ 无需修改Grove代码

**配置示例：**

#### 示例1: 使用KEDA扩缩容单个PodClique（基于Prometheus指标）

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pca-scaler
  namespace: default
spec:
  # 目标资源：Grove的PodClique
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: simple1-0-pca
  
  # 扩缩容范围
  minReplicaCount: 1
  maxReplicaCount: 10
  
  # 轮询间隔和冷却时间
  pollingInterval: 15
  cooldownPeriod: 60
  
  # 高级行为配置
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
          - type: Percent
            value: 50
            periodSeconds: 60
          - type: Pods
            value: 1
            periodSeconds: 60
          selectPolicy: Min
        scaleUp:
          stabilizationWindowSeconds: 0
          policies:
          - type: Percent
            value: 100
            periodSeconds: 30
          - type: Pods
            value: 4
            periodSeconds: 15
          selectPolicy: Max
  
  # 扩缩容触发器
  triggers:
  # 1. 基于请求率
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: request_rate
      query: |
        sum(rate(http_requests_total{
          pod=~"simple1-0-pca-.*"
        }[2m]))
      threshold: "100"
      activationThreshold: "10"
      ignoreNullValues: "true"
  
  # 2. 基于自定义业务指标
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: queue_depth
      query: |
        avg(queue_depth{
          namespace="default",
          role="pca"
        })
      threshold: "50"
      activationThreshold: "5"
      ignoreNullValues: "true"
```

#### 示例2: 使用KEDA扩缩容PodCliqueScalingGroup（多个PodClique同时扩缩容）

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: scaling-group-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodCliqueScalingGroup
    name: simple1-0-sga
  
  minReplicaCount: 1
  maxReplicaCount: 6
  pollingInterval: 30
  cooldownPeriod: 120
  
  triggers:
  # 基于Kafka消费延迟
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka.svc:9092
      consumerGroup: inference-group
      topic: inference-requests
      lagThreshold: "100"
      activationLagThreshold: "10"
  
  # 基于GPU利用率（需要dcgm-exporter）
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: gpu_utilization
      query: |
        avg(DCGM_FI_DEV_GPU_UTIL{
          pod=~"simple1-0-sga-.*"
        })
      threshold: "80"
      activationThreshold: "30"
```

#### 示例3: 使用KEDA扩缩容整个PodCliqueSet

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: podcliqueset-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodCliqueSet
    name: simple1
  
  minReplicaCount: 1
  maxReplicaCount: 5
  pollingInterval: 60
  cooldownPeriod: 300
  
  triggers:
  # 基于时间表（定时扩缩容）
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: 0 8 * * *    # 每天8点扩容到3个副本
      end: 0 20 * * *      # 每天20点缩容到1个副本
      desiredReplicas: "3"
  
  # 基于整体负载
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: cluster_load
      query: |
        sum(rate(inference_requests_total[5m]))
      threshold: "1000"
```

#### 示例4: Scale-to-Zero支持（AI推理空闲时完全关闭）

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: inference-scale-to-zero
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: simple1-0-pca
  
  # Scale to zero配置
  minReplicaCount: 0        # 可以缩容到0
  maxReplicaCount: 10
  idleReplicaCount: 0       # 无活动时保持0副本
  cooldownPeriod: 300       # 5分钟无请求后缩容到0
  
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: active_requests
      query: |
        sum(active_inference_requests{
          pod=~"simple1-0-pca-.*"
        })
      threshold: "1"          # 有1个请求就扩容
      activationThreshold: "0.1"
```

### 2.3 集成方式二：HPA与KEDA共存

**场景：**
- 某些PodClique使用HPA（CPU/Memory）
- 某些PodClique使用KEDA（事件驱动）

**注意事项：**
⚠️ **同一个资源不能同时被HPA和KEDA管理！**

```yaml
# PodCliqueSet配置
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: mixed-scaling
spec:
  replicas: 1
  template:
    cliques:
    # 使用Grove HPA的PodClique（CPU扩缩容）
    - name: frontend
      spec:
        roleName: frontend
        replicas: 3
        autoScalingConfig:      # Grove会创建HPA
          maxReplicas: 10
          metrics:
          - type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                averageUtilization: 70
        podSpec:
          containers:
          - name: frontend
            image: frontend:latest
    
    # 不配置autoScalingConfig，使用KEDA的PodClique
    - name: backend
      spec:
        roleName: backend
        replicas: 2
        # 注意：不设置autoScalingConfig
        podSpec:
          containers:
          - name: backend
            image: backend:latest
---
# 为backend PodClique创建KEDA ScaledObject
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: backend-keda-scaler
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: mixed-scaling-0-backend
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.svc:9092
      topic: backend-tasks
      lagThreshold: "50"
```

---

## 3. KEDA相比HPA的优势

### 3.1 功能对比表

| 功能 | HPA | KEDA | Grove当前支持 |
|------|-----|------|---------------|
| CPU/Memory扩缩容 | ✅ | ✅ | ✅ |
| 自定义指标（Prometheus） | ❌（需要额外配置Prometheus Adapter） | ✅ | ❌ |
| 事件驱动（Kafka, RabbitMQ等） | ❌ | ✅ | ❌ |
| Scale-to-Zero | ❌ | ✅ | ❌ |
| 外部指标（AWS CloudWatch等） | ❌ | ✅ | ❌ |
| Cron定时扩缩容 | ❌ | ✅ | ❌ |
| 多触发器组合 | ❌ | ✅ | ❌ |
| 更细粒度的扩缩容行为控制 | ⚠️ 有限 | ✅ | ⚠️ 有限 |

### 3.2 KEDA支持的事件源（与AI推理相关）

1. **Prometheus** - 任何Prometheus查询结果
2. **Kafka** - 基于消息队列深度
3. **RabbitMQ** - 基于队列长度
4. **HTTP** - 基于HTTP端点返回的指标
5. **CPU/Memory** - 与HPA相同
6. **Cron** - 时间调度
7. **External** - 自定义外部指标
8. **AWS CloudWatch** - AWS指标
9. **Azure Monitor** - Azure指标
10. **GCP Stackdriver** - GCP指标

---

## 4. 实施步骤

### 4.1 环境准备

```bash
# 1. 安装KEDA（假设已有Kubernetes集群）
kubectl apply -f https://github.com/kedacore/keda/releases/latest/download/keda-2.17.0.yaml

# 或使用Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace

# 2. 验证KEDA安装
kubectl get pods -n keda
# 应该看到：
# - keda-operator
# - keda-operator-metrics-apiserver

# 3. 确认Grove已安装
kubectl get crd | grep grove.io
# 应该看到：
# - podcliques.grove.io
# - podcliquescalinggroups.grove.io
# - podcliquesets.grove.io
```

### 4.2 验证Scale Subresource

```bash
# 测试PodClique的scale subresource
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliques/simple1-0-pca/scale | jq

# 预期输出：
{
  "kind": "Scale",
  "apiVersion": "autoscaling/v1",
  "metadata": {
    "name": "simple1-0-pca",
    "namespace": "default",
    ...
  },
  "spec": {
    "replicas": 3
  },
  "status": {
    "replicas": 3,
    "selector": "..."
  }
}
```

### 4.3 部署第一个KEDA ScaledObject

```bash
# 1. 部署Grove PodCliqueSet（不使用autoScalingConfig）
cat <<EOF | kubectl apply -f -
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: keda-test
spec:
  replicas: 1
  template:
    cliques:
    - name: worker
      spec:
        roleName: worker
        replicas: 1
        # 注意：不配置autoScalingConfig
        podSpec:
          containers:
          - name: worker
            image: nginx:latest
            resources:
              requests:
                cpu: 100m
                memory: 128Mi
EOF

# 2. 创建KEDA ScaledObject
cat <<EOF | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: keda-test-scaler
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: keda-test-0-worker
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
  - type: cpu
    metricType: Utilization
    metadata:
      value: "50"
EOF

# 3. 验证KEDA是否接管扩缩容
kubectl get scaledobject keda-test-scaler
kubectl describe scaledobject keda-test-scaler

# 4. 查看KEDA创建的HPA
kubectl get hpa
# 应该看到KEDA自动创建的HPA：keda-hpa-keda-test-scaler
```

### 4.4 监控和调试

```bash
# 查看KEDA operator日志
kubectl logs -n keda deployment/keda-operator -f

# 查看ScaledObject状态
kubectl get scaledobject -A

# 查看KEDA metrics
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 | jq

# 查看扩缩容历史
kubectl describe scaledobject keda-test-scaler
```

---

## 5. 最佳实践

### 5.1 选择HPA还是KEDA？

**使用Grove原生HPA的场景：**
- ✅ 简单的CPU/Memory扩缩容
- ✅ 不需要scale-to-zero
- ✅ 不需要复杂的事件源
- ✅ 希望配置简单（在PodCliqueSet中配置）

**使用KEDA的场景：**
- ✅ 需要基于业务指标扩缩容（Prometheus查询）
- ✅ 需要scale-to-zero节省成本
- ✅ 需要基于消息队列（Kafka/RabbitMQ）扩缩容
- ✅ 需要定时扩缩容
- ✅ 需要多个触发器组合
- ✅ 需要更细粒度的扩缩容行为控制

### 5.2 避免的陷阱

1. ⚠️ **不要同时使用Grove autoScalingConfig和KEDA**
   ```yaml
   # ❌ 错误示例
   spec:
     autoScalingConfig:    # Grove会创建HPA
       maxReplicas: 10
   # 然后再创建KEDA ScaledObject → 冲突！
   ```

2. ⚠️ **注意minAvailable和minReplicas的关系**
   ```yaml
   # PodClique配置
   spec:
     replicas: 3
     minAvailable: 2    # Gang scheduling最小需求
   
   # KEDA配置
   spec:
     minReplicaCount: 2  # 必须 >= minAvailable
   ```

3. ⚠️ **scale-to-zero需要特殊考虑**
   - Grove的gang scheduling可能与scale-to-zero冲突
   - 建议只对非关键的PodClique使用scale-to-zero

### 5.3 监控指标推荐

```yaml
# 推荐的Prometheus指标用于AI推理扩缩容
triggers:
- type: prometheus
  metadata:
    # 1. 请求率
    query: sum(rate(inference_requests_total[2m]))
    threshold: "100"

- type: prometheus
  metadata:
    # 2. 队列深度
    query: avg(inference_queue_depth)
    threshold: "50"

- type: prometheus
  metadata:
    # 3. P95延迟
    query: histogram_quantile(0.95, rate(inference_latency_bucket[5m]))
    threshold: "1.0"  # 1秒

- type: prometheus
  metadata:
    # 4. GPU利用率
    query: avg(DCGM_FI_DEV_GPU_UTIL)
    threshold: "80"
```

---

## 6. 与Dynamo Planner的对比

Grove文档提到可以与Dynamo Planner集成。以下是对比：

| 特性 | KEDA | Dynamo Planner |
|------|------|----------------|
| 开源 | ✅ | ✅ |
| 成熟度 | ✅ 非常成熟 | ⚠️ 较新 |
| 社区支持 | ✅ 活跃 | ⚠️ 有限 |
| AI推理优化 | ⚠️ 通用 | ✅ 专门优化 |
| 与Grove集成 | ✅ 开箱即用 | ✅ 深度集成 |
| 学习曲线 | ✅ 简单 | ⚠️ 较陡 |

**建议：**
- 通用扩缩容场景：使用KEDA
- AI推理深度优化场景：考虑Dynamo Planner
- 可以在不同PodClique上混用两者

---

## 7. 示例场景：完整的AI推理部署

```yaml
# 场景：多节点disaggregated推理系统
# - Prefill组件：使用KEDA基于队列深度扩缩容
# - Decode组件：使用Grove HPA基于GPU利用率扩缩容
# - Router组件：使用KEDA支持scale-to-zero

apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference
spec:
  replicas: 1
  template:
    cliques:
    # Router - 使用KEDA，支持scale-to-zero
    - name: router
      spec:
        roleName: router
        replicas: 1
        # 不配置autoScalingConfig，使用KEDA
        podSpec:
          containers:
          - name: router
            image: inference-router:latest
    
    # Prefill - 使用KEDA，基于请求队列
    - name: prefill
      spec:
        roleName: prefill
        replicas: 2
        # 不配置autoScalingConfig
        podSpec:
          containers:
          - name: prefill
            image: prefill:latest
            resources:
              limits:
                nvidia.com/gpu: 1
    
    # Decode - 使用Grove HPA
    - name: decode
      spec:
        roleName: decode
        replicas: 2
        autoScalingConfig:    # 使用Grove原生HPA
          maxReplicas: 10
          metrics:
          - type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                averageUtilization: 70
        podSpec:
          containers:
          - name: decode
            image: decode:latest
            resources:
              limits:
                nvidia.com/gpu: 2
---
# KEDA ScaledObject for Router (scale-to-zero)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: router-scaler
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: llm-inference-0-router
  minReplicaCount: 0
  maxReplicaCount: 5
  cooldownPeriod: 300
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      query: sum(rate(router_requests_total[1m]))
      threshold: "0.1"
---
# KEDA ScaledObject for Prefill (queue-based)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prefill-scaler
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: llm-inference-0-prefill
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      query: avg(prefill_queue_depth)
      threshold: "20"
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      query: avg(DCGM_FI_DEV_GPU_UTIL{role="prefill"})
      threshold: "85"
```

---

## 8. 总结和建议

### ✅ 集成可行性结论

**Grove与KEDA完全兼容！** 所有三个扩缩容资源（PodClique、PodCliqueScalingGroup、PodCliqueSet）都可以直接使用KEDA进行扩缩容。

### 🎯 推荐集成策略

1. **短期（立即可用）：**
   - 对需要高级扩缩容能力的PodClique，不配置autoScalingConfig
   - 直接创建KEDA ScaledObject指向这些PodClique
   - 无需修改Grove代码

2. **中期（增强）：**
   - 在Grove文档中添加KEDA集成指南
   - 提供官方KEDA集成示例
   - 添加验证逻辑，防止HPA和KEDA同时管理同一资源

3. **长期（深度集成）：**
   - 考虑在PodCliqueSet中添加`kedaScalingConfig`字段
   - Grove Operator自动创建KEDA ScaledObject
   - 提供统一的扩缩容管理界面

### 📚 下一步行动

1. ✅ **验证集成** - 在测试环境部署KEDA和Grove，验证scale subresource工作正常
2. ✅ **创建示例** - 编写完整的KEDA + Grove集成示例
3. ✅ **文档更新** - 在Grove文档中添加KEDA集成章节
4. ✅ **最佳实践** - 总结AI推理场景下的KEDA扩缩容最佳实践
5. ⚠️ **测试边界情况** - 测试scale-to-zero与gang scheduling的兼容性

---

## 附录A：快速参考命令

```bash
# 安装KEDA
helm install keda kedacore/keda --namespace keda --create-namespace

# 验证Grove CRD scale subresource
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliques/<name>/scale

# 创建KEDA ScaledObject
kubectl apply -f scaledobject.yaml

# 查看KEDA状态
kubectl get scaledobject -A
kubectl describe scaledobject <name>

# 查看KEDA创建的HPA
kubectl get hpa | grep keda-hpa

# 查看KEDA日志
kubectl logs -n keda deployment/keda-operator -f

# 删除KEDA ScaledObject
kubectl delete scaledobject <name>
```

## 附录B：故障排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| ScaledObject创建失败 | Scale subresource未正确实现 | 验证CRD定义和Grove版本 |
| KEDA不扩缩容 | 指标获取失败 | 检查Prometheus连接和查询 |
| 与HPA冲突 | 同时配置了autoScalingConfig | 移除autoScalingConfig |
| Scale-to-zero不工作 | minAvailable限制 | 调整minAvailable配置 |
| 扩缩容过于频繁 | 稳定窗口太短 | 增加cooldownPeriod和stabilizationWindow |

---

**文档版本：** 1.0  
**最后更新：** 2025-01-15  
**作者：** KEDA & Grove Integration Expert


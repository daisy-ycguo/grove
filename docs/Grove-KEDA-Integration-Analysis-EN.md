# Grove and KEDA Integration Feasibility Analysis

## Executive Summary ✅

**Conclusion: Grove's PodClique, PodCliqueScalingGroup, and PodCliqueSet are fully compatible with KEDA integration!**

All three resources have already implemented the Kubernetes Scale subresource, which is the core prerequisite for KEDA integration.

---

## 1. KEDA Integration Prerequisites Analysis

### 1.1 Core Capabilities Required by KEDA

KEDA can scale any Kubernetes resource that implements the following capabilities:

| Requirement | Description | Grove Status |
|------------|-------------|--------------|
| ✅ **Scale Subresource** | Resource must implement the `/scale` subresource | Implemented |
| ✅ **spec.replicas** | Must have replicas field in spec | Implemented |
| ✅ **status.replicas** | Must have replicas field in status | Implemented |
| ✅ **Label Selector** | Must provide pod selector for querying metrics | Implemented |

### 1.2 Grove Resources' Scale Subresource Implementation

#### PodClique
```yaml
# Scale subresource configuration in CRD definition
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

## 2. KEDA Integration with Grove

### 2.1 Architecture Comparison: Current HPA vs KEDA

#### Current Architecture (Using HPA)
```
┌─────────────────┐
│ PodCliqueSet    │
│  autoScaling    │
│  Config         │
└────────┬────────┘
         │ Grove Operator creates
         ↓
┌─────────────────┐      ┌──────────────┐
│ HPA Resource    │─────→│ Metrics      │
│ (CPU/Memory)    │      │ Server       │
└────────┬────────┘      └──────────────┘
         │ Standard HPA scaling
         ↓
┌─────────────────┐
│ PodClique /     │
│ PodCliqueSG     │
└─────────────────┘
```

#### Architecture Using KEDA
```
┌─────────────────┐
│ ScaledObject    │──────→ User creates directly
│ (KEDA CRD)      │
└────────┬────────┘
         │ KEDA controller
         ↓
┌─────────────────┐      ┌──────────────┐
│ KEDA Metrics    │─────→│ Event Sources│
│ Adapter         │      │ (Prometheus, │
│                 │      │  Kafka, etc) │
└────────┬────────┘      └──────────────┘
         │ Directly modifies replicas
         ↓
┌─────────────────┐
│ PodClique /     │
│ PodCliqueSG /   │
│ PodCliqueSet    │
└─────────────────┘
```

### 2.2 Integration Method 1: Using KEDA Independently (Recommended)

**Advantages:**
- ✅ Event-driven scaling (supports 60+ event sources)
- ✅ Supports scale-to-zero
- ✅ More flexible scaling strategies
- ✅ Can use custom metrics (Prometheus, Kafka, etc.)
- ✅ No need to modify Grove code

**Configuration Examples:**

#### Example 1: Using KEDA to Scale a Single PodClique (Based on Prometheus Metrics)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pca-scaler
  namespace: default
spec:
  # Target resource: Grove's PodClique
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: simple1-0-pca
  
  # Scaling range
  minReplicaCount: 1
  maxReplicaCount: 10
  
  # Polling interval and cooldown period
  pollingInterval: 15
  cooldownPeriod: 60
  
  # Advanced behavior configuration
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
  
  # Scaling triggers
  triggers:
  # 1. Based on request rate
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
  
  # 2. Based on custom business metrics
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

#### Example 2: Using KEDA to Scale PodCliqueScalingGroup (Multiple PodCliques Scale Together)

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
  # Based on Kafka consumer lag
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka.svc:9092
      consumerGroup: inference-group
      topic: inference-requests
      lagThreshold: "100"
      activationLagThreshold: "10"
  
  # Based on GPU utilization (requires dcgm-exporter)
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

#### Example 3: Using KEDA to Scale Entire PodCliqueSet

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
  # Based on schedule (scheduled scaling)
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: 0 8 * * *    # Scale to 3 replicas at 8am daily
      end: 0 20 * * *      # Scale to 1 replica at 8pm daily
      desiredReplicas: "3"
  
  # Based on overall load
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: cluster_load
      query: |
        sum(rate(inference_requests_total[5m]))
      threshold: "1000"
```

#### Example 4: Scale-to-Zero Support (Completely Shut Down AI Inference During Idle)

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
  
  # Scale to zero configuration
  minReplicaCount: 0        # Can scale down to 0
  maxReplicaCount: 10
  idleReplicaCount: 0       # Maintain 0 replicas when no activity
  cooldownPeriod: 300       # Scale to 0 after 5 minutes without requests
  
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: active_requests
      query: |
        sum(active_inference_requests{
          pod=~"simple1-0-pca-.*"
        })
      threshold: "1"          # Scale up with 1 request
      activationThreshold: "0.1"
```

### 2.3 Integration Method 2: HPA and KEDA Coexistence

**Scenario:**
- Some PodCliques use HPA (CPU/Memory)
- Some PodCliques use KEDA (event-driven)

**Important Note:**
⚠️ **The same resource cannot be managed by both HPA and KEDA simultaneously!**

```yaml
# PodCliqueSet configuration
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: mixed-scaling
spec:
  replicas: 1
  template:
    cliques:
    # PodClique using Grove HPA (CPU scaling)
    - name: frontend
      spec:
        roleName: frontend
        replicas: 3
        autoScalingConfig:      # Grove will create HPA
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
    
    # PodClique not configuring autoScalingConfig, using KEDA
    - name: backend
      spec:
        roleName: backend
        replicas: 2
        # Note: Do not set autoScalingConfig
        podSpec:
          containers:
          - name: backend
            image: backend:latest
---
# Create KEDA ScaledObject for backend PodClique
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

## 3. KEDA Advantages Over HPA

### 3.1 Feature Comparison Table

| Feature | HPA | KEDA | Grove Current Support |
|---------|-----|------|----------------------|
| CPU/Memory scaling | ✅ | ✅ | ✅ |
| Custom metrics (Prometheus) | ❌ (Requires additional Prometheus Adapter configuration) | ✅ | ❌ |
| Event-driven (Kafka, RabbitMQ, etc.) | ❌ | ✅ | ❌ |
| Scale-to-Zero | ❌ | ✅ | ❌ |
| External metrics (AWS CloudWatch, etc.) | ❌ | ✅ | ❌ |
| Cron scheduled scaling | ❌ | ✅ | ❌ |
| Multiple trigger combinations | ❌ | ✅ | ❌ |
| Fine-grained scaling behavior control | ⚠️ Limited | ✅ | ⚠️ Limited |

### 3.2 KEDA Supported Event Sources (Related to AI Inference)

1. **Prometheus** - Any Prometheus query result
2. **Kafka** - Based on message queue depth
3. **RabbitMQ** - Based on queue length
4. **HTTP** - Based on metrics returned from HTTP endpoints
5. **CPU/Memory** - Same as HPA
6. **Cron** - Time scheduling
7. **External** - Custom external metrics
8. **AWS CloudWatch** - AWS metrics
9. **Azure Monitor** - Azure metrics
10. **GCP Stackdriver** - GCP metrics

---

## 4. Implementation Steps

### 4.1 Environment Preparation

```bash
# 1. Install KEDA (assuming you have a Kubernetes cluster)
kubectl apply -f https://github.com/kedacore/keda/releases/latest/download/keda-2.17.0.yaml

# Or use Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace

# 2. Verify KEDA installation
kubectl get pods -n keda
# You should see:
# - keda-operator
# - keda-operator-metrics-apiserver

# 3. Confirm Grove is installed
kubectl get crd | grep grove.io
# You should see:
# - podcliques.grove.io
# - podcliquescalinggroups.grove.io
# - podcliquesets.grove.io
```

### 4.2 Verify Scale Subresource

```bash
# Test PodClique's scale subresource
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliques/simple1-0-pca/scale | jq

# Expected output:
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

### 4.3 Deploy First KEDA ScaledObject

```bash
# 1. Deploy Grove PodCliqueSet (without using autoScalingConfig)
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
        # Note: Do not configure autoScalingConfig
        podSpec:
          containers:
          - name: worker
            image: nginx:latest
            resources:
              requests:
                cpu: 100m
                memory: 128Mi
EOF

# 2. Create KEDA ScaledObject
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

# 3. Verify KEDA has taken over scaling
kubectl get scaledobject keda-test-scaler
kubectl describe scaledobject keda-test-scaler

# 4. View HPA created by KEDA
kubectl get hpa
# You should see KEDA's automatically created HPA: keda-hpa-keda-test-scaler
```

### 4.4 Monitoring and Debugging

```bash
# View KEDA operator logs
kubectl logs -n keda deployment/keda-operator -f

# View ScaledObject status
kubectl get scaledobject -A

# View KEDA metrics
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 | jq

# View scaling history
kubectl describe scaledobject keda-test-scaler
```

---

## 5. Best Practices

### 5.1 Choose HPA or KEDA?

**Scenarios for Using Grove Native HPA:**
- ✅ Simple CPU/Memory scaling
- ✅ Don't need scale-to-zero
- ✅ Don't need complex event sources
- ✅ Want simple configuration (configured in PodCliqueSet)

**Scenarios for Using KEDA:**
- ✅ Need to scale based on business metrics (Prometheus queries)
- ✅ Need scale-to-zero to save costs
- ✅ Need to scale based on message queues (Kafka/RabbitMQ)
- ✅ Need scheduled scaling
- ✅ Need multiple trigger combinations
- ✅ Need fine-grained scaling behavior control

### 5.2 Pitfalls to Avoid

1. ⚠️ **Don't use Grove autoScalingConfig and KEDA simultaneously**
   ```yaml
   # ❌ Wrong example
   spec:
     autoScalingConfig:    # Grove will create HPA
       maxReplicas: 10
   # Then create KEDA ScaledObject → Conflict!
   ```

2. ⚠️ **Pay attention to the relationship between minAvailable and minReplicas**
   ```yaml
   # PodClique configuration
   spec:
     replicas: 3
     minAvailable: 2    # Gang scheduling minimum requirement
   
   # KEDA configuration
   spec:
     minReplicaCount: 2  # Must be >= minAvailable
   ```

3. ⚠️ **Scale-to-zero requires special consideration**
   - Grove's gang scheduling may conflict with scale-to-zero
   - Recommend only using scale-to-zero for non-critical PodCliques

### 5.3 Recommended Monitoring Metrics

```yaml
# Recommended Prometheus metrics for AI inference scaling
triggers:
- type: prometheus
  metadata:
    # 1. Request rate
    query: sum(rate(inference_requests_total[2m]))
    threshold: "100"

- type: prometheus
  metadata:
    # 2. Queue depth
    query: avg(inference_queue_depth)
    threshold: "50"

- type: prometheus
  metadata:
    # 3. P95 latency
    query: histogram_quantile(0.95, rate(inference_latency_bucket[5m]))
    threshold: "1.0"  # 1 second

- type: prometheus
  metadata:
    # 4. GPU utilization
    query: avg(DCGM_FI_DEV_GPU_UTIL)
    threshold: "80"
```

---

## 6. Comparison with Dynamo Planner

Grove documentation mentions integration with Dynamo Planner. Here's a comparison:

| Feature | KEDA | Dynamo Planner |
|---------|------|----------------|
| Open Source | ✅ | ✅ |
| Maturity | ✅ Very mature | ⚠️ Newer |
| Community Support | ✅ Active | ⚠️ Limited |
| AI Inference Optimization | ⚠️ General purpose | ✅ Specifically optimized |
| Grove Integration | ✅ Out of the box | ✅ Deep integration |
| Learning Curve | ✅ Simple | ⚠️ Steeper |

**Recommendations:**
- General scaling scenarios: Use KEDA
- Deep AI inference optimization scenarios: Consider Dynamo Planner
- Can mix both on different PodCliques

---

## 7. Example Scenario: Complete AI Inference Deployment

```yaml
# Scenario: Multi-node disaggregated inference system
# - Prefill component: Use KEDA based on queue depth scaling
# - Decode component: Use Grove HPA based on GPU utilization scaling
# - Router component: Use KEDA with scale-to-zero support

apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference
spec:
  replicas: 1
  template:
    cliques:
    # Router - Use KEDA, supports scale-to-zero
    - name: router
      spec:
        roleName: router
        replicas: 1
        # Do not configure autoScalingConfig, use KEDA
        podSpec:
          containers:
          - name: router
            image: inference-router:latest
    
    # Prefill - Use KEDA, based on request queue
    - name: prefill
      spec:
        roleName: prefill
        replicas: 2
        # Do not configure autoScalingConfig
        podSpec:
          containers:
          - name: prefill
            image: prefill:latest
            resources:
              limits:
                nvidia.com/gpu: 1
    
    # Decode - Use Grove HPA
    - name: decode
      spec:
        roleName: decode
        replicas: 2
        autoScalingConfig:    # Use Grove native HPA
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

## 8. Summary and Recommendations

### ✅ Integration Feasibility Conclusion

**Grove is fully compatible with KEDA!** All three scalable resources (PodClique, PodCliqueScalingGroup, PodCliqueSet) can directly use KEDA for scaling.

### 🎯 Recommended Integration Strategy

1. **Short-term (Immediately Available):**
   - For PodCliques needing advanced scaling capabilities, don't configure autoScalingConfig
   - Directly create KEDA ScaledObject pointing to these PodCliques
   - No need to modify Grove code

2. **Mid-term (Enhancement):**
   - Add KEDA integration guide to Grove documentation
   - Provide official KEDA integration examples
   - Add validation logic to prevent HPA and KEDA from managing the same resource simultaneously

3. **Long-term (Deep Integration):**
   - Consider adding `kedaScalingConfig` field in PodCliqueSet
   - Grove Operator automatically creates KEDA ScaledObjects
   - Provide unified scaling management interface

### 📚 Next Steps

1. ✅ **Verify Integration** - Deploy KEDA and Grove in test environment, verify scale subresource works correctly
2. ✅ **Create Examples** - Write complete KEDA + Grove integration examples
3. ✅ **Documentation Update** - Add KEDA integration chapter to Grove documentation
4. ✅ **Best Practices** - Summarize KEDA scaling best practices for AI inference scenarios
5. ⚠️ **Test Edge Cases** - Test compatibility of scale-to-zero with gang scheduling

---

## Appendix A: Quick Reference Commands

```bash
# Install KEDA
helm install keda kedacore/keda --namespace keda --create-namespace

# Verify Grove CRD scale subresource
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliques/<name>/scale

# Create KEDA ScaledObject
kubectl apply -f scaledobject.yaml

# View KEDA status
kubectl get scaledobject -A
kubectl describe scaledobject <name>

# View HPA created by KEDA
kubectl get hpa | grep keda-hpa

# View KEDA logs
kubectl logs -n keda deployment/keda-operator -f

# Delete KEDA ScaledObject
kubectl delete scaledobject <name>
```

## Appendix B: Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| ScaledObject creation fails | Scale subresource not properly implemented | Verify CRD definition and Grove version |
| KEDA doesn't scale | Metrics retrieval failure | Check Prometheus connection and query |
| Conflict with HPA | Both autoScalingConfig configured | Remove autoScalingConfig |
| Scale-to-zero doesn't work | minAvailable limitation | Adjust minAvailable configuration |
| Scaling too frequent | Stabilization window too short | Increase cooldownPeriod and stabilizationWindow |

---

**Document Version:** 1.0  
**Last Updated:** December 16, 2025  
**Author:** KEDA & Grove Integration Expert


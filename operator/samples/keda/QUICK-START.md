# Grove + KEDA Quick Start Guide

## 🎯 Core Concepts

**Grove + KEDA = Powerful Event-Driven Autoscaling**

- **Grove**: Provides PodClique, PodCliqueScalingGroup, and PodCliqueSet resources
- **KEDA**: Provides event-driven autoscaling capabilities (60+ event sources)
- **Integration**: KEDA directly uses Grove resources' Scale subresource

## ✅ Prerequisites Check

```bash
# 1. Verify Grove is installed
kubectl get crd | grep grove.io
# ✅ Should see: podcliques.grove.io, podcliquescalinggroups.grove.io, podcliquesets.grove.io

# 2. Install KEDA (if not already installed)
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace

# 3. Verify KEDA is installed
kubectl get pods -n keda
# ✅ Should see: keda-operator, keda-operator-metrics-apiserver
```

## 🚀 5-Minute Quick Example

### Step 1: Deploy Grove PodCliqueSet (without autoScalingConfig)

```yaml
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: my-app
spec:
  replicas: 1
  template:
    cliques:
    - name: worker
      spec:
        roleName: worker
        replicas: 1
        # ⚠️ Key: Do NOT configure autoScalingConfig
        podSpec:
          containers:
          - name: worker
            image: nginx:latest
            resources:
              requests:
                cpu: 100m
```

### Step 2: Create KEDA ScaledObject

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaler
spec:
  scaleTargetRef:
    apiVersion: grove.io/v1alpha1
    kind: PodClique
    name: my-app-0-worker        # Format: {pcs-name}-{replica}-{clique-name}
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
  - type: cpu
    metricType: Utilization
    metadata:
      value: "70"
```

### Step 3: Verify

```bash
# Deploy
kubectl apply -f podcliqueset.yaml
kubectl apply -f scaledobject.yaml

# Check status
kubectl get pclq
kubectl get scaledobject
kubectl get hpa | grep keda-hpa
```

## 📋 Common Scenarios Quick Reference

| Scenario | Resource Used | KEDA Trigger Type | Example File |
|----------|--------------|-------------------|--------------|
| CPU-based scaling | PodClique | cpu | 01-basic-cpu-scaling.yaml |
| Business metrics | PodClique | prometheus | 02-prometheus-scaling.yaml |
| Scale-to-Zero | PodClique | prometheus | 03-scale-to-zero.yaml |
| Message queue based | PodClique | kafka | 04-kafka-scaling.yaml |
| Scheduled scaling | PodClique | cron | 05-cron-scaling.yaml |
| Coordinated scaling | PodCliqueScalingGroup | prometheus | 07-scaling-group.yaml |
| Full app scaling | PodCliqueSet | prometheus | 08-podcliqueset-scaling.yaml |

## 🎓 Scaling Comparison: Three Resource Types

### 1. PodClique Scaling
**Effect**: Scales the replica count of a single PodClique

```yaml
scaleTargetRef:
  kind: PodClique
  name: my-app-0-worker
# Result: Worker pod count increases from 3 to 5
```

### 2. PodCliqueScalingGroup Scaling
**Effect**: Scales all PodCliques in the group simultaneously

```yaml
scaleTargetRef:
  kind: PodCliqueScalingGroup
  name: my-app-0-prefill-group
# Result: Both prefill-leader and prefill-worker scale together
# Scaling from 1 to 2 creates a second complete set of leader+workers
```

### 3. PodCliqueSet Scaling
**Effect**: Creates/deletes entire PodCliqueSet replicas

```yaml
scaleTargetRef:
  kind: PodCliqueSet
  name: my-app
# Result: Creates a complete second set of all PodCliques
```

## ⚠️ Important Considerations

### ❌ Don't Do This

```yaml
# Wrong: Using both autoScalingConfig and KEDA
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
spec:
  template:
    cliques:
    - name: worker
      spec:
        autoScalingConfig:      # ❌ Grove will create HPA
          maxReplicas: 10
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  scaleTargetRef:
    kind: PodClique
    name: xxx-worker            # ❌ Conflict!
```

### ✅ Correct Approach

**Option A: Use only Grove autoScalingConfig**
```yaml
spec:
  cliques:
  - name: worker
    spec:
      autoScalingConfig:        # ✅ Use Grove HPA
        maxReplicas: 10
# Do not create KEDA ScaledObject
```

**Option B: Use only KEDA**
```yaml
spec:
  cliques:
  - name: worker
    spec:
      replicas: 3
      # ✅ Do not configure autoScalingConfig
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject              # ✅ Use KEDA
```

## 🔍 Debugging Commands

```bash
# View Scale subresource
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliques/my-app-0-worker/scale | jq

# View ScaledObject status
kubectl describe scaledobject my-app-scaler

# View KEDA logs
kubectl logs -n keda deployment/keda-operator -f

# View KEDA-created HPA
kubectl get hpa | grep keda-hpa
kubectl describe hpa keda-hpa-my-app-scaler

# View scaling history
kubectl describe pclq my-app-0-worker
```

## 📊 Recommended AI Inference Scaling Metrics

### 1. Request Rate
```yaml
query: sum(rate(inference_requests_total[2m]))
threshold: "100"
```

### 2. Queue Depth
```yaml
query: avg(inference_queue_depth)
threshold: "50"
```

### 3. P95 Latency
```yaml
query: histogram_quantile(0.95, rate(inference_latency_bucket[5m]))
threshold: "1.0"
```

### 4. GPU Utilization
```yaml
query: avg(DCGM_FI_DEV_GPU_UTIL)
threshold: "85"
```

### 5. Active Connections
```yaml
query: sum(active_connections)
threshold: "1000"
```

## 🎯 Selection Guide

| Requirement | Recommended Solution |
|-------------|---------------------|
| Simple CPU/Memory scaling | Grove autoScalingConfig |
| Business metrics scaling | KEDA + Prometheus |
| Need scale-to-zero | KEDA |
| Message queue based scaling | KEDA + Kafka/RabbitMQ |
| Scheduled scaling | KEDA + Cron |
| Multi-trigger combination | KEDA |
| Simple configuration | Grove autoScalingConfig |
| Maximum flexibility | KEDA |

## 📚 More Resources

- [Complete Integration Analysis](../../docs/Grove-KEDA-Integration-Analysis.md)
- [All Example Files](./README.md)
- [KEDA Official Documentation](https://keda.sh/)
- [Grove Official Documentation](../../docs/)

## 💡 Next Steps

1. Choose an example to start with (recommended: 01-basic-cpu-scaling.yaml)
2. Modify configuration based on your actual needs
3. Deploy to test environment for validation
4. Monitor scaling behavior and tune parameters
5. Gradually roll out to production

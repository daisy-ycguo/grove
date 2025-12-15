# Grove KEDA Integration Examples

This directory contains examples for event-driven autoscaling of Grove resources using KEDA.

## Prerequisites

1. **Install Grove Operator**
   ```bash
   cd operator
   make deploy
   ```

2. **Install KEDA**
   ```bash
   helm repo add kedacore https://kedacore.github.io/charts
   helm repo update
   helm install keda kedacore/keda --namespace keda --create-namespace
   ```

3. **Install Prometheus (optional, for custom metrics)**
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm install prometheus prometheus-community/kube-prometheus-stack
   ```

## Example List

| File | Description | Use Case |
|------|-------------|----------|
| `01-basic-cpu-scaling.yaml` | KEDA CPU-based scaling | Getting started |
| `02-prometheus-scaling.yaml` | Prometheus metrics scaling | Custom business metrics |
| `03-scale-to-zero.yaml` | Scale-to-zero example | Dev/test cost savings |
| `04-kafka-scaling.yaml` | Kafka queue-based scaling | Message-driven workloads |
| `05-cron-scaling.yaml` | Scheduled scaling | Predictable load patterns |
| `06-multi-trigger.yaml` | Multi-trigger combination | Complex scaling strategies |
| `07-scaling-group.yaml` | PodCliqueScalingGroup scaling | Coordinated multi-PodClique scaling |
| `08-podcliqueset-scaling.yaml` | PodCliqueSet-level scaling | Full application stack scaling |

## Quick Start

### 1. Deploy Basic Example

```bash
# Deploy PodCliqueSet (without autoScalingConfig)
kubectl apply -f 01-basic-cpu-scaling.yaml

# Verify PodClique is created
kubectl get pclq

# Verify KEDA ScaledObject is created
kubectl get scaledobject

# View KEDA-created HPA
kubectl get hpa
```

### 2. Generate Load to Test Scaling

```bash
# Get Pod name
POD_NAME=$(kubectl get pod -l grove.io/app-name=keda-basic-0-worker -o jsonpath='{.items[0].metadata.name}')

# Generate CPU load in Pod
kubectl exec -it $POD_NAME -- sh -c "dd if=/dev/zero of=/dev/null &"

# Watch scaling
kubectl get pclq keda-basic-0-worker -w
```

### 3. Cleanup

```bash
kubectl delete -f 01-basic-cpu-scaling.yaml
```

## Important Notes

⚠️ **Important: Do NOT use Grove autoScalingConfig and KEDA ScaledObject simultaneously**

```yaml
# ❌ Wrong Example
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
spec:
  template:
    cliques:
    - name: worker
      spec:
        autoScalingConfig:    # This creates an HPA
          maxReplicas: 10
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
spec:
  scaleTargetRef:
    kind: PodClique
    name: xxx-worker      # ❌ Conflict! Same resource cannot be managed by two autoscalers
```

✅ **Correct Approach:**
- Choose either Grove's autoScalingConfig **OR** KEDA's ScaledObject
- Do not configure both on the same resource

## Verify Scale Subresource

All Grove scaling resources implement the Scale subresource, which can be verified with:

```bash
# Verify PodClique
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliques/<name>/scale | jq

# Verify PodCliqueScalingGroup
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliquescalinggroups/<name>/scale | jq

# Verify PodCliqueSet
kubectl get --raw /apis/grove.io/v1alpha1/namespaces/default/podcliquesets/<name>/scale | jq
```

## Monitoring and Debugging

```bash
# View ScaledObject status
kubectl describe scaledobject <name>

# View KEDA operator logs
kubectl logs -n keda deployment/keda-operator -f

# View KEDA metrics server logs
kubectl logs -n keda deployment/keda-operator-metrics-apiserver -f

# View KEDA-created HPA
kubectl get hpa | grep keda-hpa
kubectl describe hpa keda-hpa-<scaledobject-name>
```

## More Information

- [KEDA Official Documentation](https://keda.sh/)
- [Grove Documentation](../../docs/)
- [Grove-KEDA Integration Analysis](../../docs/Grove-KEDA-Integration-Analysis.md)


# Hybrid Architecture: Knative Serving + Grove Integration Guide

## Executive Summary

This document describes a **production-ready hybrid architecture** that combines Knative Serving's serverless capabilities with Grove's advanced gang scheduling and GPU orchestration. This approach leverages the strengths of both platforms while avoiding architectural conflicts.

**Key Benefits:**
- ✅ Scale-to-zero for frontend services (cost savings)
- ✅ Advanced traffic management and blue-green deployments (Knative)
- ✅ Gang scheduling and topology-aware GPU placement (Grove)
- ✅ Clean separation of concerns
- ✅ No modifications required to either platform

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Component Responsibilities](#component-responsibilities)
3. [Reference Architecture](#reference-architecture)
4. [Implementation Guide](#implementation-guide)
5. [Traffic Flow](#traffic-flow)
6. [Scaling Strategies](#scaling-strategies)
7. [Deployment Patterns](#deployment-patterns)
8. [Configuration Examples](#configuration-examples)
9. [Monitoring and Observability](#monitoring-and-observability)
10. [Best Practices](#best-practices)
11. [Real-World Use Cases](#real-world-use-cases)
12. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

### High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                      External Traffic                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Knative Serving Layer                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Gateway    │  │   Router     │  │   API        │      │
│  │   Service    │  │   Service    │  │   Adapter    │      │
│  │              │  │              │  │              │      │
│  │ • Scale-to-0 │  │ • Traffic    │  │ • Protocol   │      │
│  │ • Ingress    │  │   Splitting  │  │   Conversion │      │
│  │ • TLS        │  │ • Canary     │  │ • Auth       │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
└─────────┼──────────────────┼──────────────────┼─────────────┘
          │                  │                  │
          │   HTTP/gRPC      │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Service Mesh (Optional)              │
│                    (Istio / Linkerd)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      Grove Layer                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │           PodCliqueSet: llm-inference                  │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  Replica 0 (Complete Inference System)           │ │ │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │ │ │
│  │  │  │  Prefill    │─▶│   Decode    │◀─│  Cache   │ │ │ │
│  │  │  │  Clique     │  │   Clique    │  │  Clique  │ │ │ │
│  │  │  │  (2 pods)   │  │   (4 pods)  │  │  (1 pod) │ │ │ │
│  │  │  │  16 GPUs    │  │   16 GPUs   │  │  Memory  │ │ │ │
│  │  │  └─────────────┘  └─────────────┘  └──────────┘ │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  Replica 1 (Complete Inference System)           │ │ │
│  │  │  [Same structure as Replica 0]                   │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Features:                                                   │
│  • Gang scheduling (all-or-nothing)                         │
│  • Topology-aware placement (NVLink domains)                │
│  • Multi-level autoscaling                                  │
│  • Startup ordering (cache → prefill → decode)              │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

### Knative Serving Responsibilities

| Capability | Description | Why Knative? |
|------------|-------------|--------------|
| **Ingress & Gateway** | HTTP/HTTPS entry point, TLS termination | Built-in integration with Istio/Contour |
| **Scale-to-Zero** | Frontend services can scale to 0 when idle | Core Knative feature, saves costs |
| **Traffic Splitting** | Blue-green, canary deployments | Declarative traffic management |
| **Request Routing** | Route to appropriate Grove backend | Service mesh integration |
| **Protocol Adaptation** | Convert HTTP → gRPC for Grove | Stateless transformation |
| **Authentication** | OAuth, JWT validation | Frontend security layer |
| **Rate Limiting** | Protect backend from overload | Before requests reach GPU workloads |
| **Request Queuing** | Buffer requests during scaling | Knative queue-proxy |

### Grove Responsibilities

| Capability | Description | Why Grove? |
|------------|-------------|------------|
| **Gang Scheduling** | Schedule multi-pod inference systems atomically | Core Grove feature |
| **GPU Orchestration** | Manage GPU allocation and topology | GPU-aware scheduling |
| **Multi-Node Coordination** | Coordinate disaggregated inference (prefill/decode) | Complex pod dependencies |
| **Topology-Aware Placement** | Place pods in optimal NVLink domains | Performance optimization |
| **Startup Ordering** | Ensure components start in correct sequence | Initialization dependencies |
| **Model Serving** | Run actual AI inference workloads | GPU-intensive computation |
| **Multi-Level Autoscaling** | Scale individual components and complete systems | Hierarchical scaling |
| **Resource Management** | Manage expensive GPU resources | Cost optimization |

---

## Reference Architecture

### Layered Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: External Access                                        │
├─────────────────────────────────────────────────────────────────┤
│ • DNS (example.com/api/inference)                               │
│ • External Load Balancer                                        │
│ • CDN (optional, for static content)                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 2: Knative Serving (Stateless Services)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ Knative Service: inference-gateway                         ││
│  │ ┌────────────┬────────────┬────────────┐                  ││
│  │ │  Revision  │  Revision  │  Revision  │                  ││
│  │ │   v1       │   v2       │   v3       │                  ││
│  │ │  (canary)  │  (stable)  │  (preview) │                  ││
│  │ │   5%       │    90%     │     5%     │                  ││
│  │ └────────────┴────────────┴────────────┘                  ││
│  │                                                             ││
│  │ Responsibilities:                                           ││
│  │ • Validate API requests                                     ││
│  │ • Route to appropriate Grove backend                        ││
│  │ • Convert HTTP → gRPC                                       ││
│  │ • Return responses to client                                ││
│  │ • Scale 0 → N based on traffic                             ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 3: Service Mesh (Optional but Recommended)               │
├─────────────────────────────────────────────────────────────────┤
│ • Traffic observability (Jaeger, Zipkin)                        │
│ • Service-to-service authentication (mTLS)                      │
│ • Circuit breaking and retry policies                           │
│ • Request shadowing for testing                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 4: Kubernetes Services (Abstraction)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  llm-inference-router.default.svc.cluster.local:8080           │
│  llm-inference-prefill.default.svc.cluster.local:8081          │
│  llm-inference-decode.default.svc.cluster.local:8082           │
│                                                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 5: Grove Orchestration (Stateful GPU Workloads)         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PodCliqueSet: llm-inference                                    │
│  ├─ Replica 0                                                   │
│  │  ├─ PodClique: router (1 pod)                               │
│  │  ├─ PodCliqueScalingGroup: inference-pipeline               │
│  │  │  ├─ PodClique: prefill (2 pods, 8 GPUs each)            │
│  │  │  └─ PodClique: decode (4 pods, 4 GPUs each)             │
│  │  └─ PodClique: cache (1 pod, memory-only)                   │
│  │                                                              │
│  └─ Replica 1 (same structure)                                  │
│                                                                  │
│  Grove Scheduler ensures:                                       │
│  • All pods in a gang are scheduled together                    │
│  • Topology-aware placement (NVLink, PCIe, NUMA)               │
│  • Startup ordering: cache → prefill → decode → router         │
│  • Resource quotas and GPU allocation                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Guide

### Prerequisites

```bash
# 1. Kubernetes cluster (v1.28+)
kubectl version

# 2. Install Knative Serving
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-core.yaml

# 3. Install Knative networking layer (choose one)
# Option A: Istio
kubectl apply -f https://github.com/knative/net-istio/releases/download/knative-v1.12.0/net-istio.yaml

# Option B: Contour
kubectl apply -f https://github.com/knative/net-contour/releases/download/knative-v1.12.0/net-contour.yaml

# 4. Install Grove
cd operator && make deploy

# 5. Verify installations
kubectl get pods -n knative-serving
kubectl get pods -n grove-system
```

### Step 1: Deploy Grove Backend

```yaml
# grove-backend.yaml
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference
  namespace: default
spec:
  replicas: 2  # 2 complete inference systems
  
  template:
    cliques:
    # Component 1: Router (lightweight, no GPU)
    - name: router
      spec:
        roleName: router
        replicas: 1
        podSpec:
          containers:
          - name: router
            image: myorg/llm-router:v1.0
            ports:
            - name: grpc
              containerPort: 8080
              protocol: TCP
            env:
            - name: PREFILL_ENDPOINT
              value: "llm-inference-prefill:8081"
            - name: DECODE_ENDPOINT
              value: "llm-inference-decode:8082"
            resources:
              requests:
                cpu: "2"
                memory: 4Gi
              limits:
                cpu: "4"
                memory: 8Gi
            readinessProbe:
              grpc:
                port: 8080
              initialDelaySeconds: 10
              periodSeconds: 5
    
    # Component 2: Prefill (GPU-intensive)
    - name: prefill
      spec:
        roleName: prefill
        replicas: 2
        minAvailable: 2  # Gang scheduling: need both pods
        
        # Grove autoscaling for prefill component
        autoScalingConfig:
          maxReplicas: 6
          metrics:
          - type: Pods
            pods:
              metric:
                name: prefill_queue_depth
              target:
                type: AverageValue
                averageValue: "20"
        
        podSpec:
          containers:
          - name: prefill
            image: myorg/llm-prefill:v1.0
            ports:
            - name: grpc
              containerPort: 8081
            env:
            - name: MODEL_PATH
              value: "/models/llama-70b"
            - name: TENSOR_PARALLEL_SIZE
              value: "8"
            resources:
              requests:
                nvidia.com/gpu: 8
                cpu: "16"
                memory: 128Gi
              limits:
                nvidia.com/gpu: 8
                cpu: "32"
                memory: 256Gi
            volumeMounts:
            - name: model-cache
              mountPath: /models
            readinessProbe:
              grpc:
                port: 8081
              initialDelaySeconds: 120  # Model loading takes time
              periodSeconds: 10
          
          volumes:
          - name: model-cache
            persistentVolumeClaim:
              claimName: model-cache-pvc
          
          # Topology hints for GPU placement
          nodeSelector:
            nvidia.com/gpu.present: "true"
          
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
    
    # Component 3: Decode (GPU-intensive)
    - name: decode
      spec:
        roleName: decode
        replicas: 4
        minAvailable: 4  # Gang scheduling: need all decode pods
        
        # Grove autoscaling for decode component
        autoScalingConfig:
          maxReplicas: 12
          metrics:
          - type: Pods
            pods:
              metric:
                name: decode_queue_depth
              target:
                type: AverageValue
                averageValue: "30"
        
        podSpec:
          containers:
          - name: decode
            image: myorg/llm-decode:v1.0
            ports:
            - name: grpc
              containerPort: 8082
            env:
            - name: MODEL_PATH
              value: "/models/llama-70b"
            - name: TENSOR_PARALLEL_SIZE
              value: "4"
            resources:
              requests:
                nvidia.com/gpu: 4
                cpu: "8"
                memory: 64Gi
              limits:
                nvidia.com/gpu: 4
                cpu: "16"
                memory: 128Gi
            volumeMounts:
            - name: model-cache
              mountPath: /models
            readinessProbe:
              grpc:
                port: 8082
              initialDelaySeconds: 120
              periodSeconds: 10
          
          volumes:
          - name: model-cache
            persistentVolumeClaim:
              claimName: model-cache-pvc
          
          nodeSelector:
            nvidia.com/gpu.present: "true"
          
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
    
    # Startup ordering: router must wait for prefill and decode
    startupOrder:
    - cliques: [prefill, decode]  # Start these first
    - cliques: [router]            # Then start router
  
  # PodCliqueSet-level autoscaling (scale complete inference systems)
  autoScalingConfig:
    maxReplicas: 5
    metrics:
    - type: External
      external:
        metric:
          name: total_inference_requests_per_second
          selector:
            matchLabels:
              service: llm-inference
        target:
          type: AverageValue
          averageValue: "1000"

---
# Expose Grove components as Kubernetes Services
apiVersion: v1
kind: Service
metadata:
  name: llm-inference-router
  namespace: default
spec:
  selector:
    grove.io/podcliqueset: llm-inference
    grove.io/clique-role: router
  ports:
  - name: grpc
    port: 8080
    targetPort: 8080
    protocol: TCP
  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: llm-inference-prefill
  namespace: default
spec:
  selector:
    grove.io/podcliqueset: llm-inference
    grove.io/clique-role: prefill
  ports:
  - name: grpc
    port: 8081
    targetPort: 8081
    protocol: TCP
  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: llm-inference-decode
  namespace: default
spec:
  selector:
    grove.io/podcliqueset: llm-inference
    grove.io/clique-role: decode
  ports:
  - name: grpc
    port: 8082
    targetPort: 8082
    protocol: TCP
  type: ClusterIP
```

### Step 2: Deploy Knative Frontend

```yaml
# knative-gateway.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inference-gateway
  namespace: default
spec:
  template:
    metadata:
      annotations:
        # Knative autoscaling configuration
        autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"
        autoscaling.knative.dev/target: "100"  # Target 100 concurrent requests per pod
        autoscaling.knative.dev/min-scale: "0"  # Scale to zero
        autoscaling.knative.dev/max-scale: "20"
        autoscaling.knative.dev/scale-down-delay: "5m"
        
        # Timeout configuration
        autoscaling.knative.dev/target-utilization-percentage: "70"
    
    spec:
      containerConcurrency: 100  # Max concurrent requests per pod
      timeoutSeconds: 300  # 5-minute timeout for long inference
      
      containers:
      - name: gateway
        image: myorg/inference-gateway:v1.0
        ports:
        - name: http1
          containerPort: 8080
        
        env:
        # Grove backend endpoints
        - name: GROVE_ROUTER_ENDPOINT
          value: "llm-inference-router.default.svc.cluster.local:8080"
        
        # Gateway configuration
        - name: MAX_TOKENS
          value: "2048"
        - name: ENABLE_STREAMING
          value: "true"
        
        resources:
          requests:
            cpu: "500m"
            memory: 512Mi
          limits:
            cpu: "2"
            memory: 2Gi
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
  
  # Traffic management (blue-green deployment example)
  traffic:
  - percent: 90
    latestRevision: false
    revisionName: inference-gateway-v1
    tag: stable
  - percent: 10
    latestRevision: true
    tag: canary

---
# Optional: Custom domain mapping
apiVersion: serving.knative.dev/v1beta1
kind: DomainMapping
metadata:
  name: api.example.com
  namespace: default
spec:
  ref:
    name: inference-gateway
    kind: Service
    apiVersion: serving.knative.dev/v1
```

### Step 3: Configure Monitoring

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: grove-inference-metrics
  namespace: default
spec:
  selector:
    matchLabels:
      grove.io/podcliqueset: llm-inference
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: knative-gateway-metrics
  namespace: default
spec:
  selector:
    matchLabels:
      serving.knative.dev/service: inference-gateway
  endpoints:
  - port: http-metrics
    interval: 10s
    path: /metrics
```

---

## Traffic Flow

### Request Flow Diagram

```
1. External Request
   │
   │ HTTP POST /v1/completions
   │ Host: api.example.com
   │ Body: {"model": "llama-70b", "prompt": "..."}
   │
   ▼
┌─────────────────────────────────────────┐
│  Knative Ingress (Istio/Contour)       │
│  • TLS termination                      │
│  • Route to Knative Service             │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  Knative Activator (if scaled to zero) │
│  • Detect service is at 0 replicas      │
│  • Buffer request                       │
│  • Trigger scale-up                     │
│  • Wait for pod ready                   │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  Knative Service: inference-gateway     │
│  Revision: v1 (90%) or v2 (10%)         │
│  • Validate request                     │
│  • Check authentication/authorization   │
│  • Rate limiting                        │
│  • Convert HTTP → gRPC                  │
└───────────────┬─────────────────────────┘
                │
                │ gRPC: InferenceRequest
                │
                ▼
┌─────────────────────────────────────────┐
│  Kubernetes Service:                    │
│  llm-inference-router                   │
│  ClusterIP: 10.96.123.45:8080          │
└───────────────┬─────────────────────────┘
                │
                │ Load balanced to one of:
                │
                ▼
┌─────────────────────────────────────────┐
│  Grove PodClique: router                │
│  Pod: llm-inference-0-router-abc123     │
│  • Receive gRPC request                 │
│  • Determine routing strategy           │
│  • Select prefill pod (round-robin)     │
└───────────────┬─────────────────────────┘
                │
                │ gRPC: PrefillRequest
                │
                ▼
┌─────────────────────────────────────────┐
│  Grove PodClique: prefill               │
│  Pods: llm-inference-0-prefill-0,1      │
│  • Load model (if not cached)           │
│  • Process prompt                       │
│  • Generate KV cache                    │
│  • Return first token + cache key       │
└───────────────┬─────────────────────────┘
                │
                │ gRPC: DecodeRequest
                │ (streaming)
                ▼
┌─────────────────────────────────────────┐
│  Grove PodClique: decode                │
│  Pods: llm-inference-0-decode-0,1,2,3   │
│  • Load KV cache                        │
│  • Generate tokens iteratively          │
│  • Stream back to router                │
└───────────────┬─────────────────────────┘
                │
                │ gRPC: DecodeResponse stream
                │
                ▼
┌─────────────────────────────────────────┐
│  Grove Router (aggregation)             │
│  • Collect streamed tokens              │
│  • Format response                      │
│  • Send back to gateway                 │
└───────────────┬─────────────────────────┘
                │
                │ gRPC: InferenceResponse
                │
                ▼
┌─────────────────────────────────────────┐
│  Knative Gateway (conversion)           │
│  • Convert gRPC → HTTP SSE              │
│  • Stream tokens to client              │
│  • Track metrics                        │
└───────────────┬─────────────────────────┘
                │
                │ HTTP/1.1 200 OK
                │ Content-Type: text/event-stream
                │ data: {"choices": [{"text": "token"}]}
                │
                ▼
          Client Receives
          Streaming Response
```

### Latency Budget

```
Component                         Latency        Cumulative
────────────────────────────────────────────────────────────
Knative Ingress                   1-5ms          5ms
Knative Gateway (validation)      2-10ms         15ms
K8s Service routing               1-2ms          17ms
Grove Router (routing logic)      5-10ms         27ms
Grove Prefill (prompt processing) 100-500ms      527ms
Grove Decode (token generation)   20-50ms/token  variable
Grove Router (aggregation)        5-10ms         ---
Knative Gateway (formatting)      2-5ms          ---
Knative Ingress (egress)          1-2ms          ---
────────────────────────────────────────────────────────────
Total (first token)               ~550ms
Total (100 tokens)                ~5.5s
```

---

## Scaling Strategies

### Three-Tier Scaling Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Tier 1: Knative Autoscaling (Gateway Layer)            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Metric: Concurrent requests per pod                    │
│  Target: 100 concurrent requests                        │
│  Scale: 0 → 20 pods                                     │
│  Speed: Fast (seconds)                                  │
│                                                          │
│  When to scale UP:                                      │
│  • Incoming request rate > 100 * current pods           │
│  • Average pod utilization > 70%                        │
│                                                          │
│  When to scale DOWN:                                    │
│  • No requests for 5 minutes → scale to 0              │
│  • Average utilization < 30% for 2 minutes             │
│                                                          │
└─────────────────────────────────────────────────────────┘
                         │
                         │ Forwards requests to
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Tier 2: Grove Component-Level Autoscaling              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Prefill Clique:                                        │
│  ├─ Metric: prefill_queue_depth                        │
│  ├─ Target: 20 requests per pod                        │
│  ├─ Scale: 2 → 6 pods (in pairs, gang scheduled)      │
│  └─ Speed: Medium (30-60 seconds, GPU init)           │
│                                                          │
│  Decode Clique:                                         │
│  ├─ Metric: decode_queue_depth                         │
│  ├─ Target: 30 requests per pod                        │
│  ├─ Scale: 4 → 12 pods (in groups of 4)               │
│  └─ Speed: Medium (30-60 seconds, GPU init)           │
│                                                          │
│  Router Clique:                                         │
│  ├─ Metric: CPU utilization                            │
│  ├─ Target: 70%                                        │
│  ├─ Scale: 1 → 5 pods                                 │
│  └─ Speed: Fast (10-20 seconds, no GPU)               │
│                                                          │
└─────────────────────────────────────────────────────────┘
                         │
                         │ When component maxes out
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Tier 3: Grove System-Level Autoscaling                 │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  PodCliqueSet:                                          │
│  ├─ Metric: total_inference_requests_per_second        │
│  ├─ Target: 1000 RPS per complete system               │
│  ├─ Scale: 1 → 5 complete inference systems            │
│  ├─ Speed: Slow (2-5 minutes, full gang)              │
│  │                                                      │
│  └─ Each replica adds:                                 │
│      ├─ 1 router pod                                   │
│      ├─ 2 prefill pods (16 GPUs)                       │
│      └─ 4 decode pods (16 GPUs)                        │
│      Total: 7 pods, 32 GPUs per replica                │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Scaling Decision Matrix

| Scenario | Tier 1 Action | Tier 2 Action | Tier 3 Action |
|----------|---------------|---------------|---------------|
| **Sudden traffic spike** | Scale gateway 0→10 (immediate) | No action (queue absorbs) | No action |
| **Sustained high load** | Gateway at max | Scale prefill 2→4 | No action |
| **All components maxed** | Gateway at max | All cliques at max | Add new replica |
| **Traffic drops** | Scale gateway 10→5 | No action (stable) | No action |
| **Long idle period** | Scale gateway to 0 | No action (keep 1) | No action |
| **Gradual growth** | Gateway scales gradually | Components scale as needed | Add replicas when Tier 2 maxed |

---

## Deployment Patterns

### Pattern 1: Blue-Green Deployment (Knative Only)

Deploy new version of gateway without touching Grove backend:

```yaml
# Update Knative Service with new revision
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inference-gateway
spec:
  template:
    metadata:
      name: inference-gateway-v2  # New revision
    spec:
      containers:
      - name: gateway
        image: myorg/inference-gateway:v2.0  # New version
        # ... rest of config
  
  traffic:
  - percent: 100
    revisionName: inference-gateway-v1  # Old version gets all traffic
    tag: blue
  - percent: 0
    revisionName: inference-gateway-v2  # New version gets 0%
    tag: green

---
# Step 1: Deploy and verify green
kubectl apply -f knative-gateway-v2.yaml

# Step 2: Test green endpoint
curl -H "Host: green-inference-gateway.default.example.com" \
     https://api.example.com/v1/completions

# Step 3: Shift traffic gradually
kubectl patch ksvc inference-gateway --type json \
  -p '[{"op": "replace", "path": "/spec/traffic/0/percent", "value": 50},
       {"op": "replace", "path": "/spec/traffic/1/percent", "value": 50}]'

# Step 4: Complete cutover
kubectl patch ksvc inference-gateway --type json \
  -p '[{"op": "replace", "path": "/spec/traffic/0/percent", "value": 0},
       {"op": "replace", "path": "/spec/traffic/1/percent", "value": 100}]'
```

### Pattern 2: Canary Deployment with Grove Backend Update

Update both Knative gateway and Grove backend:

```yaml
# Step 1: Deploy new Grove PodCliqueSet
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-v2  # New backend
spec:
  replicas: 1  # Start with 1 replica
  template:
    cliques:
    - name: prefill
      spec:
        podSpec:
          containers:
          - image: myorg/llm-prefill:v2.0  # New model version
    # ... rest of config

---
# Step 2: Create Kubernetes Service for v2
apiVersion: v1
kind: Service
metadata:
  name: llm-inference-router-v2
spec:
  selector:
    grove.io/podcliqueset: llm-inference-v2
    grove.io/clique-role: router
  ports:
  - port: 8080

---
# Step 3: Update Knative Service to route to both
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inference-gateway
spec:
  template:
    spec:
      containers:
      - name: gateway
        env:
        - name: GROVE_ROUTER_V1
          value: "llm-inference-router.default.svc:8080"
        - name: GROVE_ROUTER_V2
          value: "llm-inference-router-v2.default.svc:8080"
        - name: V2_TRAFFIC_PERCENT
          value: "10"  # Route 10% to v2

---
# Step 4: Gradually increase v2 traffic in gateway logic
# (Application-level routing based on V2_TRAFFIC_PERCENT)

# Step 5: Monitor metrics
kubectl exec -it <gateway-pod> -- curl localhost:8080/metrics | grep v2_success_rate

# Step 6: Complete cutover
kubectl set env -n default ksvc/inference-gateway V2_TRAFFIC_PERCENT=100

# Step 7: Decommission v1
kubectl delete podcliqu eset llm-inference
kubectl delete service llm-inference-router
```

### Pattern 3: Shadow Traffic for Testing

Send duplicate traffic to new version for testing:

```yaml
# Using Istio VirtualService
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inference-gateway-shadow
spec:
  hosts:
  - inference-gateway.default.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: /v1/completions
    route:
    - destination:
        host: inference-gateway.default.svc.cluster.local
        subset: stable
      weight: 100
    mirror:
      host: inference-gateway.default.svc.cluster.local
      subset: canary
    mirrorPercentage:
      value: 10.0  # Shadow 10% of traffic
```

---

## Configuration Examples

### Example 1: Development Environment

Small-scale setup for development:

```yaml
# Knative: 1 gateway pod, no scale-to-zero
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inference-gateway
  annotations:
    autoscaling.knative.dev/min-scale: "1"  # Always keep 1 pod
    autoscaling.knative.dev/max-scale: "3"

# Grove: Minimal GPU usage
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-dev
spec:
  replicas: 1
  template:
    cliques:
    - name: prefill
      spec:
        replicas: 1  # Single pod
        podSpec:
          containers:
          - name: prefill
            resources:
              requests:
                nvidia.com/gpu: 1  # Just 1 GPU
    - name: decode
      spec:
        replicas: 1
        podSpec:
          containers:
          - name: decode
            resources:
              requests:
                nvidia.com/gpu: 1
```

### Example 2: Production Environment

High-availability setup:

```yaml
# Knative: Scale-to-zero enabled, high limits
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inference-gateway
  annotations:
    autoscaling.knative.dev/min-scale: "0"  # Scale to zero
    autoscaling.knative.dev/max-scale: "50"
    autoscaling.knative.dev/target: "100"

# Grove: Multiple replicas with autoscaling
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-prod
spec:
  replicas: 3  # 3 complete inference systems
  
  autoScalingConfig:
    maxReplicas: 10
    metrics:
    - type: External
      external:
        metric:
          name: inference_qps
        target:
          type: AverageValue
          averageValue: "500"
  
  template:
    cliques:
    - name: prefill
      spec:
        replicas: 4  # More pods for higher throughput
        autoScalingConfig:
          maxReplicas: 8
        podSpec:
          containers:
          - name: prefill
            resources:
              requests:
                nvidia.com/gpu: 8
    - name: decode
      spec:
        replicas: 8
        autoScalingConfig:
          maxReplicas: 16
        podSpec:
          containers:
          - name: decode
            resources:
              requests:
                nvidia.com/gpu: 4
```

### Example 3: Cost-Optimized Setup

Aggressive scale-to-zero with spot instances:

```yaml
# Knative: Aggressive scale-down
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inference-gateway
  annotations:
    autoscaling.knative.dev/min-scale: "0"
    autoscaling.knative.dev/max-scale: "10"
    autoscaling.knative.dev/scale-down-delay: "1m"  # Fast scale-down
    autoscaling.knative.dev/scale-to-zero-pod-retention-period: "1m"

# Grove: Use spot instances
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-spot
spec:
  replicas: 1
  template:
    cliques:
    - name: prefill
      spec:
        replicas: 2
        podSpec:
          # Use spot/preemptible nodes
          nodeSelector:
            cloud.google.com/gke-preemptible: "true"
          tolerations:
          - key: cloud.google.com/gke-preemptible
            operator: Equal
            value: "true"
            effect: NoSchedule
          # Handle interruptions gracefully
          terminationGracePeriodSeconds: 30
```

---

## Monitoring and Observability

### Key Metrics to Track

#### Knative Gateway Metrics

```prometheus
# Request rate
sum(rate(revision_request_count{service_name="inference-gateway"}[1m]))

# Latency (P50, P95, P99)
histogram_quantile(0.95, 
  sum(rate(revision_request_latencies_bucket{service_name="inference-gateway"}[1m])) by (le)
)

# Active pods
revision_actual_pod_count{service_name="inference-gateway"}

# Desired pods
revision_desired_pod_count{service_name="inference-gateway"}

# Scale-from-zero events
activator_go_requests_total{service="inference-gateway"}

# Error rate
sum(rate(revision_request_count{service_name="inference-gateway",response_code!~"2.."}[1m]))
  / sum(rate(revision_request_count{service_name="inference-gateway"}[1m]))
```

#### Grove Backend Metrics

```prometheus
# GPU utilization
avg(DCGM_FI_DEV_GPU_UTIL{pod=~"llm-inference-.*-prefill-.*"})

# Queue depth
avg(grove_clique_queue_depth{clique="prefill"})

# Pod ready count
sum(kube_pod_status_ready{pod=~"llm-inference-.*"} == 1)

# Gang scheduling success rate
rate(grove_gang_scheduling_success_total[5m])
  / rate(grove_gang_scheduling_attempts_total[5m])

# Component-specific throughput
sum(rate(inference_requests_total{component="prefill"}[1m]))
sum(rate(inference_requests_total{component="decode"}[1m]))

# Inter-component latency
histogram_quantile(0.95,
  sum(rate(grove_component_latency_seconds_bucket{from="prefill",to="decode"}[1m])) by (le)
)
```

### Grafana Dashboard Example

```json
{
  "dashboard": {
    "title": "Knative + Grove Hybrid Architecture",
    "panels": [
      {
        "title": "Request Flow",
        "targets": [
          {
            "expr": "sum(rate(revision_request_count{service_name=\"inference-gateway\"}[1m]))",
            "legendFormat": "Knative Gateway"
          },
          {
            "expr": "sum(rate(inference_requests_total{component=\"router\"}[1m]))",
            "legendFormat": "Grove Router"
          },
          {
            "expr": "sum(rate(inference_requests_total{component=\"prefill\"}[1m]))",
            "legendFormat": "Grove Prefill"
          },
          {
            "expr": "sum(rate(inference_requests_total{component=\"decode\"}[1m]))",
            "legendFormat": "Grove Decode"
          }
        ]
      },
      {
        "title": "Scaling Status",
        "targets": [
          {
            "expr": "revision_actual_pod_count{service_name=\"inference-gateway\"}",
            "legendFormat": "Gateway Pods"
          },
          {
            "expr": "sum(kube_pod_info{pod=~\"llm-inference-.*-prefill-.*\"})",
            "legendFormat": "Prefill Pods"
          },
          {
            "expr": "sum(kube_pod_info{pod=~\"llm-inference-.*-decode-.*\"})",
            "legendFormat": "Decode Pods"
          }
        ]
      },
      {
        "title": "GPU Utilization",
        "targets": [
          {
            "expr": "avg(DCGM_FI_DEV_GPU_UTIL{pod=~\"llm-inference-.*\"}) by (pod)",
            "legendFormat": "{{pod}}"
          }
        ]
      },
      {
        "title": "End-to-End Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(revision_request_latencies_bucket[1m])) by (le))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(revision_request_latencies_bucket[1m])) by (le))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(revision_request_latencies_bucket[1m])) by (le))",
            "legendFormat": "P99"
          }
        ]
      }
    ]
  }
}
```

### Alerting Rules

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: hybrid-architecture-alerts
spec:
  groups:
  - name: knative-alerts
    interval: 30s
    rules:
    - alert: KnativeGatewayHighErrorRate
      expr: |
        sum(rate(revision_request_count{service_name="inference-gateway",response_code!~"2.."}[5m]))
        / sum(rate(revision_request_count{service_name="inference-gateway"}[5m])) > 0.05
      for: 2m
      annotations:
        summary: "High error rate in Knative gateway ({{ $value | humanizePercentage }})"
    
    - alert: KnativeGatewayHighLatency
      expr: |
        histogram_quantile(0.95,
          sum(rate(revision_request_latencies_bucket{service_name="inference-gateway"}[5m])) by (le)
        ) > 10
      for: 5m
      annotations:
        summary: "P95 latency above 10s in Knative gateway"
  
  - name: grove-alerts
    interval: 30s
    rules:
    - alert: GroveGangSchedulingFailure
      expr: |
        rate(grove_gang_scheduling_failures_total[5m]) > 0.1
      for: 5m
      annotations:
        summary: "Grove gang scheduling failures detected"
    
    - alert: GroveHighGPUUtilization
      expr: |
        avg(DCGM_FI_DEV_GPU_UTIL{pod=~"llm-inference-.*"}) > 90
      for: 10m
      annotations:
        summary: "GPU utilization above 90% for 10 minutes - consider scaling"
    
    - alert: GroveComponentQueueBacklog
      expr: |
        grove_clique_queue_depth{clique=~"prefill|decode"} > 100
      for: 3m
      annotations:
        summary: "High queue backlog in {{ $labels.clique }} component"
```

---

## Best Practices

### 1. Resource Management

**Knative Gateway:**
```yaml
resources:
  requests:
    cpu: "500m"      # Low CPU for stateless gateway
    memory: 512Mi
  limits:
    cpu: "2"         # Allow bursting
    memory: 2Gi      # Prevent OOM
```

**Grove GPU Workloads:**
```yaml
resources:
  requests:
    nvidia.com/gpu: 8
    cpu: "16"        # Sufficient CPU for data preprocessing
    memory: 128Gi    # Enough for model + KV cache
  limits:
    nvidia.com/gpu: 8  # Match requests (no GPU overcommit)
    cpu: "32"          # Allow some burst
    memory: 256Gi      # 2x for safety
```

### 2. Scaling Configuration

**Knative - Favor fast scaling:**
```yaml
autoscaling.knative.dev/target: "100"  # Lower = more aggressive
autoscaling.knative.dev/scale-down-delay: "5m"  # Quick scale-down
```

**Grove - Favor stability:**
```yaml
autoScalingConfig:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 5-minute window
      policies:
      - type: Percent
        value: 50  # Max 50% reduction per interval
        periodSeconds: 60
```

### 3. Health Checks

**Knative Gateway - Fast checks:**
```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5      # Check every 5s
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 3
```

**Grove GPU Pods - Slow checks:**
```yaml
readinessProbe:
  grpc:
    port: 8081
  initialDelaySeconds: 120  # Allow model loading
  periodSeconds: 30         # Check every 30s
  timeoutSeconds: 10        # GPU operations can be slow
  successThreshold: 1
  failureThreshold: 2
```

### 4. Network Policies

```yaml
# Allow Knative gateway to access Grove services only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: grove-ingress
spec:
  podSelector:
    matchLabels:
      grove.io/podcliqueset: llm-inference
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          serving.knative.dev/service: inference-gateway
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 8081
    - protocol: TCP
      port: 8082
```

### 5. Cost Optimization

| Strategy | Knative Gateway | Grove Backend |
|----------|-----------------|---------------|
| **Scale-to-zero** | ✅ Yes (for dev/staging) | ❌ No (long cold start) |
| **Spot/Preemptible** | ⚠️ Risky (latency spikes) | ✅ Yes (with proper handling) |
| **Resource limits** | ✅ Tight limits | ⚠️ Generous limits (avoid OOM) |
| **Autoscaling** | Aggressive | Conservative |
| **Idle time** | Scale to 0 after 5 min | Keep minReplicas=1 |

---

## Real-World Use Cases

### Use Case 1: SaaS LLM API Platform

**Requirements:**
- Multi-tenant inference service
- Pay-per-use billing
- 99.9% SLA uptime
- Scale from 0 to 1000 RPS

**Architecture:**
```yaml
# Tenant-specific Knative Services
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: tenant-acme-gateway
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"  # Scale to zero per tenant
        autoscaling.knative.dev/target: "50"
    spec:
      containers:
      - name: gateway
        env:
        - name: TENANT_ID
          value: "acme"
        - name: GROVE_BACKEND
          value: "llm-inference-shared:8080"  # Shared Grove backend
        - name: BILLING_ENABLED
          value: "true"

# Shared Grove backend for all tenants
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-shared
spec:
  replicas: 5  # Always keep 5 systems running
  # ... GPU configuration
```

**Benefits:**
- Each tenant gets dedicated Knative Service (isolation, billing)
- All tenants share Grove GPU backend (cost efficiency)
- Knative scales tenant gateways independently
- Grove handles GPU resource pooling

### Use Case 2: Research Lab with Batch + Interactive Workloads

**Requirements:**
- Interactive inference for researchers (low latency)
- Batch inference for experiments (high throughput)
- Efficient GPU sharing

**Architecture:**
```yaml
# Interactive Knative Service (low latency)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: interactive-inference
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "10"  # Low concurrency for latency
    spec:
      containers:
      - name: gateway
        env:
        - name: PRIORITY
          value: "high"
        - name: TIMEOUT
          value: "30s"

---
# Batch Knative Service (high throughput)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: batch-inference
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "100"  # High concurrency for throughput
    spec:
      containers:
      - name: gateway
        env:
        - name: PRIORITY
          value: "low"
        - name: TIMEOUT
          value: "5m"

---
# Grove backend with priority queues
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: research-inference
spec:
  replicas: 3
  template:
    cliques:
    - name: prefill
      spec:
        replicas: 2
        podSpec:
          containers:
          - name: prefill
            env:
            - name: ENABLE_PRIORITY_QUEUE
              value: "true"
```

### Use Case 3: Edge Inference with Regional Deployments

**Requirements:**
- Deploy in multiple regions (US, EU, APAC)
- Route users to nearest region
- Failover between regions

**Architecture:**
```yaml
# Global Traffic Manager (Knative)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: global-inference-gateway
  annotations:
    networking.knative.dev/ingress.class: "istio"
spec:
  template:
    spec:
      containers:
      - name: gateway
        env:
        - name: US_ENDPOINT
          value: "llm-inference-us:8080"
        - name: EU_ENDPOINT
          value: "llm-inference-eu:8080"
        - name: APAC_ENDPOINT
          value: "llm-inference-apac:8080"
        # Gateway implements geolocation-based routing

---
# Regional Grove deployments
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-us
  namespace: us-east-1
spec:
  replicas: 10
  # ... US-specific GPU configuration

---
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-eu
  namespace: eu-west-1
spec:
  replicas: 5
  # ... EU-specific GPU configuration

---
apiVersion: grove.io/v1alpha1
kind: PodCliqueSet
metadata:
  name: llm-inference-apac
  namespace: asia-southeast-1
spec:
  replicas: 3
  # ... APAC-specific GPU configuration
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: Knative Gateway Can't Reach Grove Backend

**Symptoms:**
```
Error: connection refused to llm-inference-router:8080
```

**Debug steps:**
```bash
# 1. Verify Grove pods are running
kubectl get pods -l grove.io/podcliqueset=llm-inference

# 2. Check service endpoints
kubectl get endpoints llm-inference-router

# 3. Test connectivity from gateway pod
kubectl exec -it <gateway-pod> -- curl llm-inference-router:8080/healthz

# 4. Check network policies
kubectl get networkpolicies
```

**Solution:**
Ensure Kubernetes Service selector matches Grove pod labels:
```yaml
spec:
  selector:
    grove.io/podcliqueset: llm-inference  # Must match!
    grove.io/clique-role: router
```

#### Issue 2: Slow Cold Start from Scale-to-Zero

**Symptoms:**
```
First request after idle period times out or takes >60 seconds
```

**Debug steps:**
```bash
# Check Knative activator logs
kubectl logs -n knative-serving -l app=activator

# Check Grove gang scheduling time
kubectl get events --sort-by='.lastTimestamp' | grep PodGang
```

**Solution:**
```yaml
# Option A: Keep min replicas > 0
autoscaling.knative.dev/min-scale: "1"

# Option B: Increase activation timeout
autoscaling.knative.dev/scale-to-zero-pod-retention-period: "10m"

# Option C: Use pre-warmed Grove backend
# Grove PodCliqueSet never scales to zero (always ready)
```

#### Issue 3: Gang Scheduling Failures

**Symptoms:**
```
Grove pods stuck in Pending state
Event: Cannot schedule gang - insufficient GPUs
```

**Debug steps:**
```bash
# Check GPU availability
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUs:.status.capacity.'nvidia\.com/gpu'

# Check gang requirements
kubectl describe podgang <gang-name>

# Check Grove scheduler logs
kubectl logs -n grove-system -l app=grove-scheduler
```

**Solution:**
```yaml
# Reduce gang size or increase cluster capacity
spec:
  cliques:
  - name: prefill
    spec:
      replicas: 2  # Reduce from 4 to 2
      minAvailable: 1  # Allow partial gang (if acceptable)
```

#### Issue 4: High Latency Between Knative and Grove

**Symptoms:**
```
P95 latency is 2x higher than expected
Slow inter-pod communication
```

**Debug steps:**
```bash
# Check if pods are co-located
kubectl get pods -o wide | grep llm-inference

# Check network latency
kubectl exec <gateway-pod> -- ping llm-inference-router

# Check service mesh overhead
kubectl exec <gateway-pod> -- curl -w "@curl-format.txt" llm-inference-router:8080
```

**Solution:**
```yaml
# Use pod affinity to co-locate Knative and Grove
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inference-gateway
spec:
  template:
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  grove.io/podcliqueset: llm-inference
              topologyKey: kubernetes.io/hostname
```

#### Issue 5: Knative and Grove Autoscalers Fighting

**Symptoms:**
```
Pods constantly scaling up and down
Metrics oscillating
Resource thrashing
```

**Debug steps:**
```bash
# Check HPA status
kubectl get hpa

# Check Knative PodAutoscaler
kubectl get kpa

# Check metrics
kubectl top pods
```

**Solution:**
```yaml
# Separate scaling concerns with different metrics

# Knative: Scale on request count
autoscaling.knative.dev/target: "100"  # Requests
autoscaling.knative.dev/metric: "concurrency"

# Grove: Scale on queue depth (different metric!)
autoScalingConfig:
  metrics:
  - type: Pods
    pods:
      metric:
        name: queue_depth  # Different from concurrency!
      target:
        type: AverageValue
        averageValue: "20"
```

---

## Summary

### Advantages of Hybrid Architecture

| Benefit | Description |
|---------|-------------|
| **🚀 Best of Both Worlds** | Knative's serverless features + Grove's GPU orchestration |
| **💰 Cost Efficient** | Scale-to-zero frontend, optimized GPU backend |
| **🔧 Modular** | Update gateway without touching GPU workloads |
| **📈 Independent Scaling** | Each layer scales based on appropriate metrics |
| **🛡️ Production Ready** | Both platforms are mature and battle-tested |
| **🔌 Clean Integration** | Standard Kubernetes Services, no code changes |

### When to Use This Architecture

✅ **Perfect fit:**
- AI inference with expensive GPU resources
- Variable traffic patterns (need scale-to-zero)
- Multi-component inference pipelines
- Need for advanced traffic management

❌ **Not recommended:**
- Simple single-pod inference (use Knative only)
- No GPU requirements (use Knative only)
- No traffic variability (use Grove only)

### Next Steps

1. **Prototype** - Deploy minimal setup in dev environment
2. **Benchmark** - Test latency and throughput
3. **Tune** - Adjust autoscaling parameters
4. **Monitor** - Set up observability stack
5. **Optimize** - Fine-tune based on production metrics
6. **Scale** - Gradually increase to production load

---

**Document Version:** 1.0  
**Last Updated:** December 23, 2025  
**Maintained By:** Grove + Knative Integration Team

**Feedback and Questions:**
- GitHub Discussions: [Grove Repository](https://github.com/ai-dynamo/grove/discussions)
- Knative Slack: #serving channel
- Grove Discord: [NVIDIA Dynamo Server](https://discord.gg/UxcbxEYqS4)


Hybrid Architecture: Knative Serving + Grove Integration
═══════════════════════════════════════════════════════════════════════════════════════════════════════════

┌──────────┐      ┌────────────────────────────────────────────┐      ┌────────────────────────────────────────────────────────────┐
│          │      │    Knative Serving Layer                   │      │              Grove Orchestration Layer                      │
│ External │──────▶  (Stateless Services)                      │──────▶            (Stateful GPU Workloads)                       │
│ Traffic  │      │                                            │      │                                                            │
│          │      │  ┌──────────────────────────────────┐     │      │  ┌───────────────────────────────────────────────────────┐ │
│  HTTP/   │      │  │  Knative Service: Gateway        │     │      │  │  PodCliqueSet: llm-inference (Replica 0)           │ │
│  HTTPS   │      │  │  ┌────────────┐  ┌────────────┐ │     │      │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │ │
│          │      │  │  │ Revision   │  │ Revision   │ │     │      │  │  │ Router   │  │ Prefill  │  │  Decode  │        │ │
└──────────┘      │  │  │  v1 (90%)  │  │  v2 (10%)  │ │     │      │  │  │ Clique   │  │ Clique   │  │  Clique  │        │ │
                  │  │  │            │  │  (Canary)  │ │     │      │  │  │ 1 pod    │  │ 2 pods   │  │  4 pods  │        │ │
                  │  │  └────────────┘  └────────────┘ │     │      │  │  │ No GPU   │  │ 16 GPUs  │  │  16 GPUs │        │ │
                  │  │                                  │     │      │  │  └────┬─────┘  └────┬─────┘  └────┬─────┘        │ │
                  │  │  Features:                       │     │      │  │       │             │             │               │ │
                  │  │  • Scale-to-Zero (0 → 20 pods)  │     │      │  │       └─────────────┼─────────────┘               │ │
                  │  │  • Traffic Splitting            │     │      │  │                     │                             │ │
                  │  │  • Blue-Green Deployment        │     │      │  │          Coordinated via Gang Scheduling         │ │
                  │  │  • Canary Releases              │     │      │  └───────────────────────────────────────────────────────┘ │
                  │  │  • Auto TLS / Custom Domains    │     │      │                                                            │
                  │  │  • Request Authentication       │     │      │  ┌───────────────────────────────────────────────────────┐ │
                  │  │  • HTTP ⟷ gRPC Conversion       │     │      │  │  PodCliqueSet: llm-inference (Replica 1)           │ │
                  │  └──────────────┬───────────────────┘     │      │  │  [Same structure as Replica 0]                     │ │
                  │                 │                         │      │  └───────────────────────────────────────────────────────┘ │
                  │                 │                         │      │                                                            │
                  │    Fast scaling │ (seconds)               │      │  Features:                                                │
                  │                 │                         │      │  • Gang Scheduling (all-or-nothing)                       │
                  └─────────────────┼─────────────────────────┘      │  • Topology-Aware GPU Placement (NVLink)                  │
                                    │                                │  • Startup Ordering (prefill → decode → router)           │
                                    │                                │  • Multi-Level Autoscaling                                │
                                    ▼                                │  • Resource Quotas & GPU Allocation                       │
                  ┌─────────────────────────────────────────┐       │                                                            │
                  │   Kubernetes Service Abstraction        │       │  Slow scaling (minutes, GPU initialization)               │
                  │                                         │       └────────────────────────────────────────────────────────────┘
                  │   llm-inference-router.default.svc      │
                  │   ClusterIP: 10.96.x.x:8080            │
                  │                                         │
                  │   • Load Balancing                      │
                  │   • Service Discovery                   │
                  │   • Network Abstraction                 │
                  └─────────────────┬───────────────────────┘
                                    │
                                    │ gRPC
                                    ▼


Request Flow (Left to Right):
──────────────────────────────────────────────────────────────────────────────────────────────────────────

   Client          Ingress        Gateway         K8s Svc        Router         Prefill         Decode
     │                │              │               │              │              │              │
     │  HTTP POST     │              │               │              │              │              │
     ├───────────────▶│              │               │              │              │              │
     │                │  Route       │               │              │              │              │
     │                ├─────────────▶│               │              │              │              │
     │                │              │  Validate &   │              │              │              │
     │                │              │  Convert      │              │              │              │
     │                │              ├──────────────▶│              │              │              │
     │                │              │               │  gRPC Call   │              │              │
     │                │              │               ├─────────────▶│              │              │
     │                │              │               │              │  Process     │              │
     │                │              │               │              │  Prompt      │              │
     │                │              │               │              ├─────────────▶│              │
     │                │              │               │              │              │  Generate    │
     │                │              │               │              │              │  Tokens      │
     │                │              │               │              │              ├─────────────▶│
     │                │              │               │              │              │◀─────────────┤
     │                │              │               │              │◀─────────────┤   Stream     │
     │                │              │               │◀─────────────┤              │   Response   │
     │                │              │◀──────────────┤              │              │              │
     │◀───────────────┴──────────────┤  Convert &    │              │              │              │
     │   HTTP Response (Streaming)   │  Return       │              │              │              │
     │                                                                                             


Key Benefits:
──────────────────────────────────────────────────────────────────────────────────────────────────────────

  Knative Layer Benefits              │  Grove Layer Benefits              │  Integration Benefits
  ────────────────────────────────────│────────────────────────────────────│──────────────────────────────
  ✓ Cost savings (scale-to-zero)      │  ✓ Efficient GPU utilization       │  ✓ Clean separation of concerns
  ✓ Fast response to traffic spikes   │  ✓ Complex multi-pod coordination  │  ✓ Independent scaling policies
  ✓ Easy A/B testing & rollouts       │  ✓ Optimized for AI workloads      │  ✓ No platform modifications needed
  ✓ Built-in traffic management       │  ✓ Topology-aware scheduling       │  ✓ Standard Kubernetes integration
  ✓ Simple stateless services         │  ✓ Gang scheduling guarantees      │  ✓ Best of both worlds
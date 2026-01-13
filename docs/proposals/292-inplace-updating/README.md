# GREP-292: Rolling Update with In-Place Pod Image Update

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: Image-Only Rollout for Production Services](#story-1-image-only-rollout-for-production-services)
    - [Story 2: Rapid Iteration During Development](#story-2-rapid-iteration-during-development)
    - [Story 3: Large-Scale ML Model Updates](#story-3-large-scale-ml-model-updates)
    - [Story 4: Coordinated Distributed System Updates](#story-4-coordinated-distributed-system-updates)
  - [Limitations/Risks &amp; Mitigations](#limitationsrisks--mitigations)
    - [In-Place Update Failure Scenarios](#in-place-update-failure-scenarios)
    - [MinAvailable Constraint Handling](#minavailable-constraint-handling)
    - [Kubernetes Version Dependency](#kubernetes-version-dependency)
    - [Observability During Updates](#observability-during-updates)
    - [Serial Update Limitations for Distributed Systems](#serial-update-limitations-for-distributed-systems)
    - [Batch Mode and MinAvailable Conflict](#batch-mode-and-minavailable-conflict)
- [Design Details](#design-details)
  - [High-Level Implementation Logic](#high-level-implementation-logic)
    - [Decision Flow](#decision-flow)
    - [Update Strategy Selection](#update-strategy-selection)
  - [Component Architecture](#component-architecture)
    - [Change Detection Logic](#change-detection-logic)
    - [In-Place Update Execution](#in-place-update-execution)
    - [Update Completion Detection](#update-completion-detection)
  - [Implementation Details](#implementation-details)
    - [Modified computeUpdateWork()](#modified-computeupdatework)
    - [New Function: canUpdateInPlace()](#new-function-canupdateinplace)
    - [Modified processPendingUpdates()](#modified-processpdateupdates)
    - [New Function: updatePodInPlace()](#new-function-updatepodinplace)
    - [Enhanced isCurrentPodUpdateComplete()](#enhanced-iscurrentpodupdatecomplete)
  - [API Design](#api-design)
    - [RollingUpdateStrategy API](#rollingupdatestrategy-api)
    - [Default Behavior (Backward Compatible)](#default-behavior-backward-compatible)
    - [Strategy Selection Logic](#strategy-selection-logic)
  - [Update Strategy Type Implementation](#update-strategy-type-implementation)
    - [Strategy Selection in processPendingUpdates()](#strategy-selection-in-processpdateupdates)
  - [Batch Update Implementation](#batch-update-implementation)
  - [Status Tracking for Batch Updates](#status-tracking-for-batch-updates)
  - [Configuration Examples](#configuration-examples)
  - [Update Flow for Batch Mode](#update-flow-for-batch-mode)
  - [Common Questions about Batch Mode and MinAvailable](#common-questions-about-batch-mode-and-minavailable)
  - [Benefits of Batch Mode](#benefits-of-batch-mode)
  - [Monitoring](#monitoring)
  - [Dependencies](#dependencies)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
- [Appendix](#appendix)
<!-- /toc -->

## Summary

This proposal introduces in-place pod image update capability for Grove's rolling update mechanism. When a `PodClique` spec change involves only container image updates, the controller can patch the pod's container images in-place, allowing Kubelet to restart containers without deleting the entire pod. This optimization significantly reduces rolling update overhead by eliminating pod rescheduling delays.

The proposal adds a new optional `RollingUpdateStrategy` field to PodClique spec, providing users with explicit control over:
1. **Update Strategy Type**: Whether to use in-place updates, recreate (delete/create), or automatic detection
2. **Update Concurrency Mode**: Whether to update pods serially (one-by-one) or in batch (all simultaneously)

This design gives users fine-grained control over rolling update behavior while maintaining backward compatibility through optional fields with sensible defaults.

## Motivation

Currently, Grove's rolling update process follows a delete-and-recreate strategy for all pod updates. When a PodClique's template changes, the controller deletes old pods and creates new ones, triggering the full Kubernetes scheduling pipeline. This approach incurs significant overhead:

- **Scheduling Delays**: Deleted pods must be rescheduled, competing with other workloads for scheduler attention
- **Resource Reallocation**: PodGroup/gang scheduling constraints require coordination across multiple pods
- **Network Disruption**: Pod deletion causes IP address changes and service endpoint churn
- **Volume Re-attachment**: Persistent volumes must be detached and re-attached
- **Startup Overhead**: New pods go through full initialization (image pull, volume mounts, probes)

For image-only updates, Kubernetes 1.27+ supports in-place container restart without pod deletion. Kubelet detects container image changes in the pod spec, pulls new images, and restarts containers while preserving the pod's identity, IP address, volumes, and scheduling placement. This capability can reduce typical update times from 30-120 seconds down to 5-30 seconds.

In busy GPU clusters where scheduling resources are scarce and pod placement is topology-constrained, avoiding unnecessary rescheduling becomes critical for operational efficiency and workload availability.

### Goals

- Implement in-place pod image update capability that patches container images without pod deletion
- Add API field to allow users to explicitly control update strategy (in-place, recreate, or auto-detect)
- Support serial (one-by-one) and batch (all-at-once) update concurrency modes
- Ensure proper handling of `MinAvailable` constraints during in-place updates
- Provide clear observability through events and status updates
- Maintain backward compatibility through optional fields with sensible defaults
- Enable coordinated updates for distributed systems that require version consistency

### Non-Goals

- Supporting in-place updates for changes beyond container images (e.g., environment variables, resource limits, volumes)
- Modifying Kubernetes core behavior or requiring custom Kubelet patches
- Providing rollback mechanisms beyond standard PodClique template reversion
- Optimizing image pull performance (caching, pre-pulling, etc.)
- Supporting in-place updates for init containers or ephemeral containers in initial implementation
- Handling in-place updates for StatefulSet-like ordered rollouts

## Proposal

The proposal implements in-place pod image updates within Grove's existing rolling update framework by extending the PodClique API with a new `RollingUpdateStrategy` field. This field provides users with explicit control over update behavior through three dimensions:

### Key Design Principles

1. **Explicit Control**: Users can specify exactly how updates should be executed
2. **Flexible Strategy**: Support multiple update strategies (in-place, recreate, auto-detect)
3. **Concurrency Options**: Choose between serial (safe) or batch (fast) update modes
4. **Backward Compatible**: Optional field with sensible defaults maintains existing behavior

### API Extension

The proposal adds an optional `RollingUpdateStrategy` field to `PodCliqueSpec`:

```go
type RollingUpdateStrategy struct {
    // Type controls whether to use in-place updates or recreate
    Type UpdateStrategyType
    
    // UpdateMode controls update concurrency (serial vs batch)
    UpdateMode UpdateMode
    
    // BatchUpdateTimeout for batch mode
    BatchUpdateTimeout *metav1.Duration
}
```

### Implementation Components

The implementation augments the `rollingupdate.go` component within the PodClique controller:

1. **Change Detection Logic**: Identifies image-only updates vs other changes
2. **Strategy Selection**: Determines update strategy based on user configuration and change type
3. **In-Place Patch Execution**: Uses Kubernetes client PATCH operations to update container images
4. **Batch Coordination**: Manages simultaneous updates across multiple pods
5. **Completion Detection**: Tracks container image IDs and readiness status

### User Stories

#### Story 1: Image-Only Rollout for Production Services

As a platform engineer managing large-scale AI inference workloads, I need to deploy new model versions (packaged as container images) across hundreds of pods. With the current recreate strategy, rolling out an image update takes 45+ minutes due to scheduling delays and gang coordination overhead. With in-place updates, the same rollout completes in under 10 minutes since pods remain scheduled and only containers restart.

#### Story 2: Rapid Iteration During Development

As an ML researcher developing and testing model improvements, I frequently update model container images (several times per hour). Each update currently requires waiting for pod deletion, rescheduling (which may fail due to resource constraints), and full pod startup. In-place updates allow me to iterate 5-10x faster by eliminating rescheduling overhead.

#### Story 3: Large-Scale ML Model Updates

As an MLOps engineer running disaggregated inference with topology-constrained placement (GREP-244), I need to update model images across prefill and decode workers. The current recreate strategy risks violating topology constraints during updates as deleted pods may not be rescheduled to topology-optimal nodes. In-place updates preserve the original topology-aware placement, maintaining inference performance during rollouts.

#### Story 4: Coordinated Distributed System Updates

As a distributed systems engineer running a consensus-based workload (e.g., distributed training, parameter server), I need all worker pods to update their model container images simultaneously and reach a Ready state together before resuming coordinated operations. Serial one-by-one updates cause version skew where some workers run old code while others run new code, breaking protocol compatibility. 

Using batch in-place update mode with a low `MinAvailable` setting allows all pods to update in parallel and wait for all to be Ready before the system resumes. While this means temporary unavailability during the update (5-30 seconds), this is acceptable for my workload because:
- Training jobs pause during updates anyway
- Version synchronization is more critical than continuous availability
- The update completes much faster than serial mode (seconds vs minutes)
- All workers resume with identical versions, maintaining protocol compatibility

### Limitations/Risks & Mitigations

#### In-Place Update Failure Scenarios

**Risk**: In-place updates can fail in several ways:
- Image pull failures (wrong tag, registry unavailable, authentication issues)
- Container crashes in crash-loop-backoff after restart
- Readiness probe failures preventing pod from becoming Ready

**Impact**: Failed in-place updates leave pods in NotReady state with old images still running or containers in crash loops.

**Mitigation**:
- Controller detects failed in-place updates by monitoring pod readiness and container status
- On detection of persistent failure (pod NotReady beyond timeout), controller marks pod for deletion
- Next reconciliation loop recreates the failed pod using standard delete-recreate strategy
- Events are recorded for each failure mode to aid debugging
- PodClique status reflects the failure and retry strategy

#### MinAvailable Constraint Handling

**Risk**: During in-place updates, pods temporarily become NotReady while containers restart. If multiple in-place updates happen concurrently, the number of Ready pods may drop below `MinAvailable`.

**Impact**: This could violate availability guarantees and impact workload functionality.

**Mitigation**:
- Controller tracks the number of pods currently undergoing in-place updates
- Before initiating a new in-place update, controller verifies: `ReadyPods - InProgressInPlaceUpdates > MinAvailable`
- In-place updates are serialized to ensure only one pod per clique updates at a time (initial implementation)
- Future optimization: Allow controlled concurrency based on available headroom above MinAvailable

#### Kubernetes Version Dependency

**Risk**: In-place container restart by patching pod.spec.containers[*].image requires Kubernetes 1.27+ where Kubelet properly handles this scenario.

**Impact**: Deployments on older Kubernetes versions may experience undefined behavior.

**Mitigation**:
- Document minimum Kubernetes version requirement (1.27+)
- Controller could optionally detect Kubernetes version and disable in-place updates on older versions (future enhancement)
- Extensive testing on supported Kubernetes versions (1.27, 1.28, 1.29, 1.30)

#### Observability During Updates

**Risk**: In-place updates are less visible than pod deletions/creations in kubectl output and events.

**Impact**: Users may not understand why pods are restarting or how updates are progressing.

**Mitigation**:
- Emit Kubernetes events for each in-place update attempt, success, and failure
- Update PodClique status with current update strategy ("in-place" vs "recreate")
- Track and expose metrics for in-place update success/failure rates
- Document the expected pod behavior during in-place updates

#### Serial Update Limitations for Distributed Systems

**Risk**: When using serial update mode, pods update one-by-one. For distributed systems requiring coordinated state (e.g., consensus protocols, distributed training), this causes version skew where some pods run old code while others run new code.

**Impact**: 
- Protocol incompatibility between old and new versions during rollout
- Distributed training jobs may fail when workers have mismatched model code
- Consensus systems may experience split-brain or quorum issues

**Mitigation**:
- Use batch update mode (`UpdateMode: Batch`) where all pods update simultaneously
- Batch mode ensures all pods reach Ready state together, eliminating version skew
- Configure appropriate `BatchUpdateTimeout` to handle slow updates
- **Note**: Batch mode requires sufficient headroom above MinAvailable (see next section)

#### Batch Mode and MinAvailable Are Mutually Exclusive

**Design Decision**: Batch mode and MinAvailable are fundamentally incompatible goals and are therefore **mutually exclusive**.

**Why They Conflict**:

1. **Batch Mode Goal**: All pods update simultaneously to maintain API/version consistency
   - During in-place update, ALL pods become NotReady while containers restart (5-30 seconds)
   - ReadyReplicas temporarily drops to 0

2. **MinAvailable Goal**: Maintain minimum number of Ready pods at all times
   - Requires `ReadyReplicas >= MinAvailable` always
   - Cannot tolerate all pods being NotReady simultaneously

**These goals are contradictory**: You cannot have all pods NotReady (batch update) while maintaining minimum Ready pods (MinAvailable).

**Simple Rule**: 

```
Batch Mode + MinAvailable > 0  =  ERROR (Configuration Rejected)
```

**Validation Behavior**:

The controller enforces mutual exclusivity at update time:

```yaml
# ❌ INVALID - Will be rejected
spec:
  replicas: 16
  minAvailable: 16  # MinAvailable is set
  rollingUpdateStrategy:
    updateMode: Batch  # ❌ Conflict! Rejected with error
```

```yaml
# ❌ INVALID - Any MinAvailable with Batch is rejected
spec:
  replicas: 32
  minAvailable: 1  # Even minAvailable=1 conflicts with Batch
  rollingUpdateStrategy:
    updateMode: Batch  # ❌ Rejected
```

```yaml
# ✅ VALID - Batch without MinAvailable
spec:
  replicas: 16
  # No minAvailable specified (or minAvailable: 0)
  rollingUpdateStrategy:
    updateMode: Batch  # ✅ Allowed
```

```yaml
# ✅ VALID - MinAvailable with Serial
spec:
  replicas: 16
  minAvailable: 16
  rollingUpdateStrategy:
    updateMode: Serial  # ✅ Allowed
```

**Error Handling**:

When both are configured, controller will:
1. Detect the configuration conflict during rolling update
2. Emit `BatchModeMinAvailableConflict` error event
3. Reject the rolling update entirely
4. Require user to fix configuration before proceeding

**Decision Guide**:

| Your Priority | Configuration |
|--------------|---------------|
| **Version Consistency** (all pods same version) | Use `Batch` mode, omit `minAvailable` or set to 0 |
| **High Availability** (maintain minimum ready pods) | Use `Serial` mode with `minAvailable` |
| **Both** (not possible) | Choose one - they are mutually exclusive |

**Impact**:

Users must make an explicit choice:
- **Batch mode**: Accept temporary full unavailability (5-30s) for version consistency
- **Serial mode**: Accept version skew during rollout for continuous availability
- **No middle ground**: Cannot have both simultaneously

## Design Details

### High-Level Implementation Logic

The in-place update mechanism integrates into Grove's existing rolling update flow in `rollingupdate.go`. The controller analyzes pod specification changes and routes each pod to the appropriate update path.

#### Decision Flow

```
Pod needs update (PodTemplateHash mismatch)
    ↓
    ├─→ Is pod Pending or Unhealthy?
    │       ↓ YES
    │       └─→ DELETE pod immediately (existing behavior)
    │
    ↓ NO (Pod is Ready)
    │
    ├─→ Analyze Pod Spec Changes
    │   │
    │   ├─→ Only container images changed?
    │   │       ↓ YES
    │   │       └─→ [IN-PLACE UPDATE PATH]
    │   │           ├─ Patch pod.spec.containers[*].image
    │   │           ├─ Kubelet pulls new image
    │   │           ├─ Kubelet restarts containers
    │   │           ├─ Wait for containers to be Ready
    │   │           └─ Update PodTemplateHash label
    │   │
    │   └─→ Other changes detected?
    │           ↓ YES (volumes, env vars, resources, etc.)
    │           └─→ [RECREATE PATH]
    │               ├─ Check MinAvailable constraint
    │               ├─ DELETE pod
    │               ├─ Wait for deletion
    │               ├─ CREATE new pod
    │               └─ Wait for new pod Ready
    │
    ↓
Proceed to next pod
```

#### Update Strategy Selection

The controller performs deep comparison of pod specifications to determine update eligibility:

**Scenario 1: Image-Only Change → In-Place Update**
```yaml
Old Pod Spec:                    New Pod Spec:
  containers:                      containers:
  - name: app                      - name: app
    image: nginx:1.19                image: nginx:1.20  ← ONLY CHANGE
    ports: [80]                      ports: [80]
    env: [...]                       env: [...]
    
Result: ✅ IN-PLACE UPDATE
```

**Scenario 2: Image + Other Changes → Recreate**
```yaml
Old Pod Spec:                    New Pod Spec:
  containers:                      containers:
  - name: app                      - name: app
    image: nginx:1.19                image: nginx:1.20
    ports: [80]                      ports: [80, 443]   ← EXTRA CHANGE
    env: [...]                       env: [...]
    
Result: ❌ RECREATE (port added)
```

**Scenario 3: Multiple Container Images → In-Place Update**
```yaml
Old Pod Spec:                    New Pod Spec:
  containers:                      containers:
  - name: app                      - name: app
    image: app:v1                    image: app:v2      ← CHANGED
  - name: sidecar                  - name: sidecar
    image: sidecar:1.0               image: sidecar:1.1 ← CHANGED
    
Result: ✅ IN-PLACE UPDATE (all images changed, nothing else)
```

### Component Architecture

#### Change Detection Logic

The controller examines pod specifications at a granular level to determine update strategy:

**Fields that trigger RECREATE if changed:**
- Container fields (except `image`): command, args, ports, env, volumeMounts, resources, probes, lifecycle, securityContext
- Volumes: any volume addition, removal, or modification
- Pod-level settings: serviceAccountName, securityContext, affinity, tolerations, nodeSelector
- Network settings: hostNetwork, dnsPolicy, dnsConfig
- Other immutable fields: initContainers, imagePullSecrets, schedulerName

**Fields that allow IN-PLACE update:**
- Container images only (all containers can have image changes)

#### In-Place Update Execution

When an in-place update is executed:

1. **Patch Request**: Controller sends a PATCH request to update `pod.spec.containers[*].image`
2. **Kubelet Detection**: Kubelet watches pod spec and detects image change
3. **Image Pull**: Kubelet pulls new container images
4. **Container Restart**: Kubelet stops old containers and starts new ones
5. **Pod Persistence**: Pod object persists with same name, IP, volumes, node placement
6. **Status Update**: Container statuses reflect new ImageID and RestartCount increments
7. **Readiness**: Containers pass readiness probes and pod becomes Ready again

**Timing:**
- Typical in-place update: 5-30 seconds
- Typical recreate update: 30-120 seconds

#### Update Completion Detection

The controller verifies update completion differently based on strategy:

**For In-Place Updates:**
```go
// Check if pod is running expected images AND is Ready
isPodRunningExpectedImages(pod) {
    for each container:
        ✓ containerStatus.ImageID matches expected image digest
        ✓ containerStatus.Ready == true
    return all_match
}
```

**For Recreate Updates (existing behavior):**
- Pod with old PodTemplateHash is fully deleted
- New pod with new PodTemplateHash is created and becomes Ready

### Implementation Details

#### Modified computeUpdateWork()

**Location**: `operator/internal/controller/podclique/components/pod/rollingupdate.go` (lines ~136-159)

Add capability detection to identify pods eligible for in-place updates:

```go
type updateWork struct {
    oldTemplateHashPendingPods   []*corev1.Pod
    oldTemplateHashUnhealthyPods []*corev1.Pod
    oldTemplateHashReadyPods     []*corev1.Pod
    newTemplateHashReadyPods     []*corev1.Pod
    
    // NEW: Track which pods can be updated in-place
    inPlaceUpdatablePods         []*corev1.Pod
    mustRecreatePods             []*corev1.Pod
}

func (r _resource) computeUpdateWork(logger logr.Logger, sc *syncContext) *updateWork {
    work := &updateWork{}
    
    // Get expected pod spec from template
    expectedPodSpec := sc.pclq.Spec.PodSpec
    
    for _, pod := range sc.existingPCLQPods {
        if pod.Labels[common.LabelPodTemplateHash] != sc.expectedPodTemplateHash {
            if r.hasPodDeletionBeenTriggered(sc, pod) {
                continue
            }
            
            // NEW: Check if pod can be updated in-place
            if k8sutils.IsPodReady(pod) && canUpdateInPlace(pod, expectedPodSpec) {
                work.inPlaceUpdatablePods = append(work.inPlaceUpdatablePods, pod)
            } else {
                // Categorize for recreation based on health status
                if k8sutils.IsPodPending(pod) {
                    work.oldTemplateHashPendingPods = append(work.oldTemplateHashPendingPods, pod)
                } else if k8sutils.HasAnyContainerError(pod) {
                    work.oldTemplateHashUnhealthyPods = append(work.oldTemplateHashUnhealthyPods, pod)
                } else {
                    work.mustRecreatePods = append(work.mustRecreatePods, pod)
                }
            }
        } else if k8sutils.IsPodReady(pod) {
            work.newTemplateHashReadyPods = append(work.newTemplateHashReadyPods, pod)
        }
    }
    return work
}
```

#### New Function: canUpdateInPlace()

**Location**: `operator/internal/controller/podclique/components/pod/rollingupdate.go`

```go
// canUpdateInPlace determines if a pod can be updated in-place without recreation
func canUpdateInPlace(existingPod *corev1.Pod, expectedPodSpec corev1.PodSpec) bool {
    // Compare existing pod spec with expected spec
    changes := computePodSpecDiff(existingPod.Spec, expectedPodSpec)
    
    // Only allow in-place update if ONLY container images changed
    if len(changes) == 1 && changes[0] == "container_images" {
        return true
    }
    
    // Any other changes require pod recreation
    return false
}

// computePodSpecDiff identifies what fields have changed
func computePodSpecDiff(existing, expected corev1.PodSpec) []string {
    var diffs []string
    
    // Check containers
    if !reflect.DeepEqual(existing.Containers, expected.Containers) {
        if onlyImagesChanged(existing.Containers, expected.Containers) {
            diffs = append(diffs, "container_images")
        } else {
            diffs = append(diffs, "containers_other")
        }
    }
    
    // Check other immutable fields
    if !reflect.DeepEqual(existing.Volumes, expected.Volumes) {
        diffs = append(diffs, "volumes")
    }
    if !reflect.DeepEqual(existing.SecurityContext, expected.SecurityContext) {
        diffs = append(diffs, "security_context")
    }
    if !reflect.DeepEqual(existing.ServiceAccountName, expected.ServiceAccountName) {
        diffs = append(diffs, "service_account")
    }
    if !reflect.DeepEqual(existing.Affinity, expected.Affinity) {
        diffs = append(diffs, "affinity")
    }
    if !reflect.DeepEqual(existing.Tolerations, expected.Tolerations) {
        diffs = append(diffs, "tolerations")
    }
    if !reflect.DeepEqual(existing.NodeSelector, expected.NodeSelector) {
        diffs = append(diffs, "node_selector")
    }
    
    return diffs
}

// onlyImagesChanged verifies only image fields differ between container specs
func onlyImagesChanged(existing, expected []corev1.Container) bool {
    if len(existing) != len(expected) {
        return false
    }
    
    for i := range existing {
        existingCopy := existing[i].DeepCopy()
        expectedCopy := expected[i].DeepCopy()
        
        // Check container names match
        if existingCopy.Name != expectedCopy.Name {
            return false
        }
        
        // Normalize images for comparison
        existingCopy.Image = ""
        expectedCopy.Image = ""
        
        // If anything besides image differs, return false
        if !reflect.DeepEqual(existingCopy, expectedCopy) {
            return false
        }
    }
    
    return true
}
```

#### Modified processPendingUpdates()

**Location**: `operator/internal/controller/podclique/components/pod/rollingupdate.go` (lines ~72-133)

```go
func (r _resource) processPendingUpdates(logger logr.Logger, sc *syncContext) error {
    work := r.computeUpdateWork(logger, sc)
    pclq := sc.pclq
    
    // Step 1: Delete old pending/unhealthy pods (existing behavior)
    if err := r.deleteOldPendingAndUnhealthyPods(logger, sc, work); err != nil {
        return err
    }
    
    // Step 2: Check if current update is complete
    if isAnyPodSelectedForUpdate(pclq) && !isCurrentPodUpdateComplete(sc, work) {
        return groveerr.New(
            groveerr.ErrCodeContinueReconcileAndRequeue,
            component.OperationSync,
            "waiting for current pod update to complete",
        )
    }
    
    // Step 3: NEW - Select next pod and strategy
    var nextPodToUpdate *corev1.Pod
    var updateStrategy string
    
    if len(work.inPlaceUpdatablePods) > 0 {
        // Prioritize in-place updates (faster, less disruptive)
        nextPodToUpdate = work.inPlaceUpdatablePods[0]
        updateStrategy = "in-place"
    } else if len(work.mustRecreatePods) > 0 {
        // Check MinAvailable constraint for recreate strategy
        if pclq.Status.ReadyReplicas < *pclq.Spec.MinAvailable {
            return groveerr.New(
                groveerr.ErrCodeContinueReconcileAndRequeue,
                component.OperationSync,
                fmt.Sprintf("ready replicas %d less than minAvailable %d", 
                    pclq.Status.ReadyReplicas, *pclq.Spec.MinAvailable),
            )
        }
        nextPodToUpdate = work.mustRecreatePods[0]
        updateStrategy = "recreate"
    }
    
    // Step 4: Execute the update
    if nextPodToUpdate != nil {
        logger.Info("Selected pod for update", 
            "pod", client.ObjectKeyFromObject(nextPodToUpdate),
            "strategy", updateStrategy)
        
        // Update PodClique status
        if err := r.updatePCLQStatusWithNextPodToUpdate(sc.ctx, logger, pclq, 
            nextPodToUpdate.Name, updateStrategy); err != nil {
            return err
        }
        
        // Execute based on strategy
        if updateStrategy == "in-place" {
            if err := r.updatePodInPlace(sc.ctx, logger, pclq, nextPodToUpdate, sc); err != nil {
                logger.Error(err, "in-place update failed, will recreate on next reconcile")
                // Mark pod for recreation on next reconcile
                r.recordInPlaceUpdateFailure(sc, nextPodToUpdate)
                // Don't return error - handle in next reconciliation
            }
        } else {
            // Existing delete logic
            deletionTask := r.createPodDeletionTask(logger, pclq, nextPodToUpdate, 
                sc.pclqExpectationsStoreKey)
            if err := deletionTask.Fn(sc.ctx); err != nil {
                return err
            }
        }
        
        return groveerr.New(
            groveerr.ErrCodeContinueReconcileAndRequeue,
            component.OperationSync,
            fmt.Sprintf("updated pod %s using %s strategy", 
                nextPodToUpdate.Name, updateStrategy),
        )
    }
    
    // Step 5: Mark update completion
    return r.markRollingUpdateEnd(sc.ctx, logger, pclq)
}
```

#### New Function: updatePodInPlace()

**Location**: `operator/internal/controller/podclique/components/pod/rollingupdate.go`

```go
// updatePodInPlace patches the pod spec to trigger kubelet container restart
func (r _resource) updatePodInPlace(ctx context.Context, logger logr.Logger, 
    pclq *grovecorev1alpha1.PodClique, pod *corev1.Pod, sc *syncContext) error {
    
    podObjKey := client.ObjectKeyFromObject(pod)
    logger.Info("Attempting in-place pod update", "pod", podObjKey)
    
    // Get expected pod spec
    expectedPodSpec := sc.pclq.Spec.PodSpec
    
    // Create patch to update container images
    patch := client.MergeFrom(pod.DeepCopy())
    
    // Update container images in pod spec
    for i := range pod.Spec.Containers {
        containerName := pod.Spec.Containers[i].Name
        
        // Find matching container in expected spec
        for _, expectedContainer := range expectedPodSpec.Containers {
            if expectedContainer.Name == containerName {
                oldImage := pod.Spec.Containers[i].Image
                newImage := expectedContainer.Image
                
                if oldImage != newImage {
                    pod.Spec.Containers[i].Image = newImage
                    logger.Info("Updating container image in-place",
                        "pod", podObjKey,
                        "container", containerName,
                        "oldImage", oldImage,
                        "newImage", newImage)
                }
                break
            }
        }
    }
    
    // Apply the patch
    if err := r.client.Patch(ctx, pod, patch); err != nil {
        r.eventRecorder.Eventf(pclq, corev1.EventTypeWarning, 
            constants.ReasonPodUpdateFailed, 
            "Failed to update pod %s in-place: %v", pod.Name, err)
        return groveerr.WrapError(err,
            errCodeUpdatePod,
            component.OperationSync,
            fmt.Sprintf("failed to patch pod %v for in-place update", podObjKey),
        )
    }
    
    logger.Info("Successfully patched pod for in-place update", "pod", podObjKey)
    r.eventRecorder.Eventf(pclq, corev1.EventTypeNormal, 
        constants.ReasonPodUpdatedInPlace, 
        "Updated pod %s in-place (image only)", pod.Name)
    
    return nil
}
```

#### Enhanced isCurrentPodUpdateComplete()

**Location**: `operator/internal/controller/podclique/components/pod/rollingupdate.go`

```go
func isCurrentPodUpdateComplete(sc *syncContext, work *updateWork) bool {
    currentlyUpdatingPodName := sc.pclq.Status.RollingUpdateProgress.ReadyPodsSelectedToUpdate.Current
    pod, ok := lo.Find(sc.existingPCLQPods, func(pod *corev1.Pod) bool {
        return currentlyUpdatingPodName == pod.Name
    })
    
    if !ok {
        // Pod doesn't exist - recreate strategy completed
        podsSelectedToUpdate := len(sc.pclq.Status.RollingUpdateProgress.ReadyPodsSelectedToUpdate.Completed) + 1
        return len(work.newTemplateHashReadyPods) >= podsSelectedToUpdate
    }
    
    // Pod exists - check if it has new template hash
    if pod.Labels[common.LabelPodTemplateHash] == sc.expectedPodTemplateHash {
        // Pod has new template hash and is ready
        return k8sutils.IsPodReady(pod)
    }
    
    // Pod still has old template hash
    if k8sutils.IsResourceTerminating(pod.ObjectMeta) {
        // Recreate strategy in progress
        return false
    }
    
    // Check if in-place update completed by comparing actual vs expected images
    if isPodRunningExpectedImages(pod, sc) && k8sutils.IsPodReady(pod) {
        // In-place update complete - update PodTemplateHash label
        return r.updatePodTemplateHashLabel(sc.ctx, pod, sc.expectedPodTemplateHash)
    }
    
    return false
}

// isPodRunningExpectedImages checks if pod containers are running expected images
func isPodRunningExpectedImages(pod *corev1.Pod, sc *syncContext) bool {
    expectedSpec := sc.pclq.Spec.PodSpec
    
    // Check all containers have expected images
    for _, containerStatus := range pod.Status.ContainerStatuses {
        // Find expected image for this container
        var expectedImage string
        for _, expectedContainer := range expectedSpec.Containers {
            if expectedContainer.Name == containerStatus.Name {
                expectedImage = expectedContainer.Image
                break
            }
        }
        
        if expectedImage == "" {
            return false
        }
        
        // Extract image name without tag/digest for comparison
        expectedImageName := getImageNameWithoutTag(expectedImage)
        actualImageName := getImageNameFromImageID(containerStatus.ImageID)
        
        if expectedImageName != actualImageName {
            return false
        }
        
        // Container must be ready
        if !containerStatus.Ready {
            return false
        }
    }
    
    return true
}

// updatePodTemplateHashLabel updates the pod's template hash label after successful in-place update
func (r _resource) updatePodTemplateHashLabel(ctx context.Context, pod *corev1.Pod, newHash string) error {
    patch := client.MergeFrom(pod.DeepCopy())
    pod.Labels[common.LabelPodTemplateHash] = newHash
    return r.client.Patch(ctx, pod, patch)
}
```

### API Design

#### RollingUpdateStrategy API

Extend PodClique spec with a new `RollingUpdateStrategy` field:

```go
// UpdateStrategyType defines how pods should be updated
type UpdateStrategyType string

const (
    // UpdateStrategyInPlaceIfPossible automatically uses in-place update for image-only
    // changes, falls back to recreate for other changes (default)
    UpdateStrategyInPlaceIfPossible UpdateStrategyType = "InPlaceIfPossible"
    
    // UpdateStrategyInPlace forces in-place updates for all changes.
    // Update will fail if changes are not compatible with in-place updates
    UpdateStrategyInPlace UpdateStrategyType = "InPlace"
    
    // UpdateStrategyRecreate always deletes and recreates pods (traditional behavior)
    UpdateStrategyRecreate UpdateStrategyType = "Recreate"
)

// UpdateMode defines update concurrency behavior
type UpdateMode string

const (
    // UpdateModeSerial updates pods one-by-one (default, safest)
    UpdateModeSerial UpdateMode = "Serial"
    
    // UpdateModeBatch updates all pods simultaneously
    UpdateModeBatch UpdateMode = "Batch"
)

// RollingUpdateStrategy defines the strategy for rolling updates
type RollingUpdateStrategy struct {
    // Type controls the update strategy type
    // - InPlaceIfPossible: Auto-detect, use in-place for image-only changes (default)
    // - InPlace: Force in-place updates, fail if not possible
    // - Recreate: Always delete and recreate pods
    // +optional
    // +kubebuilder:default=InPlaceIfPossible
    // +kubebuilder:validation:Enum=InPlaceIfPossible;InPlace;Recreate
    Type UpdateStrategyType `json:"type,omitempty"`
    
    // UpdateMode controls update concurrency
    // - Serial: Update pods one-by-one (default)
    // - Batch: Update all pods simultaneously
    // +optional
    // +kubebuilder:default=Serial
    // +kubebuilder:validation:Enum=Serial;Batch
    UpdateMode UpdateMode `json:"updateMode,omitempty"`
    
    // BatchUpdateTimeout specifies maximum time to wait for batch update to complete
    // Only applies when UpdateMode is "Batch"
    // Defaults to 300 seconds (5 minutes)
    // +optional
    BatchUpdateTimeout *metav1.Duration `json:"batchUpdateTimeout,omitempty"`
}
```

**PodClique API Update**:

```go
type PodCliqueSpec struct {
    // ... existing fields ...
    
    // RollingUpdateStrategy configures the rolling update behavior
    // If not specified, defaults to InPlaceIfPossible with Serial mode
    // +optional
    RollingUpdateStrategy *RollingUpdateStrategy `json:"rollingUpdateStrategy,omitempty"`
}
```

#### Default Behavior (Backward Compatible)

When `RollingUpdateStrategy` is not specified (nil), the controller uses:
- **Type**: `InPlaceIfPossible` - automatically use in-place updates when possible
- **UpdateMode**: `Serial` - update pods one-by-one

This maintains safe, predictable behavior for existing PodCliques.

#### Strategy Selection Logic

```
Pod needs update
    ↓
Check RollingUpdateStrategy.Type
    ↓
    ├─→ "InPlaceIfPossible" (default)
    │   ├─→ Only images changed? → IN-PLACE UPDATE
    │   └─→ Other changes? → RECREATE
    │
    ├─→ "InPlace"
    │   ├─→ Only images changed? → IN-PLACE UPDATE
    │   └─→ Other changes? → FAIL UPDATE (emit error event)
    │
    └─→ "Recreate"
        └─→ Always → RECREATE (delete + create)
```

### Update Strategy Type Implementation

#### Strategy Selection in processPendingUpdates()

The controller determines update strategy based on `RollingUpdateStrategy.Type`:

```go
func (r _resource) determineUpdateStrategy(pod *corev1.Pod, expectedPodSpec corev1.PodSpec, 
    strategyType UpdateStrategyType) (string, error) {
    
    // Check what changed
    changes := computePodSpecDiff(pod.Spec, expectedPodSpec)
    onlyImagesChanged := len(changes) == 1 && changes[0] == "container_images"
    
    switch strategyType {
    case UpdateStrategyInPlaceIfPossible:
        // Auto-detect: use in-place if only images changed
        if onlyImagesChanged && k8sutils.IsPodReady(pod) {
            return "in-place", nil
        }
        return "recreate", nil
        
    case UpdateStrategyInPlace:
        // Force in-place: fail if not image-only change
        if !onlyImagesChanged {
            return "", fmt.Errorf("in-place update not possible: non-image changes detected: %v", changes)
        }
        if !k8sutils.IsPodReady(pod) {
            return "", fmt.Errorf("in-place update not possible: pod is not ready")
        }
        return "in-place", nil
        
    case UpdateStrategyRecreate:
        // Always recreate
        return "recreate", nil
        
    default:
        // Fallback to auto-detect
        if onlyImagesChanged && k8sutils.IsPodReady(pod) {
            return "in-place", nil
        }
        return "recreate", nil
    }
}
```

### Batch Update Implementation

**Modified processPendingUpdates() for Batch Mode**:

```go
func (r _resource) processPendingUpdates(logger logr.Logger, sc *syncContext) error {
    work := r.computeUpdateWork(logger, sc)
    pclq := sc.pclq
    
    // Step 1: Delete old pending/unhealthy pods (existing behavior)
    if err := r.deleteOldPendingAndUnhealthyPods(logger, sc, work); err != nil {
        return err
    }
    
    // Determine update mode (default to Serial)
    updateMode := UpdateModeSerial
    if pclq.Spec.RollingUpdateStrategy != nil {
        updateMode = pclq.Spec.RollingUpdateStrategy.UpdateMode
    }
    
    // Step 2: Batch mode - update all pods simultaneously
    if updateMode == UpdateModeBatch && len(work.inPlaceUpdatablePods) > 0 {
        return r.processBatchInPlaceUpdate(logger, sc, work)
    }
    
    // Step 3: Serial mode - existing one-by-one logic
    return r.processSerialUpdate(logger, sc, work)
}

// processBatchInPlaceUpdate handles batch in-place updates for all pods
func (r _resource) processBatchInPlaceUpdate(logger logr.Logger, sc *syncContext, work *updateWork) error {
    pclq := sc.pclq
    
    // CRITICAL: Batch mode and MinAvailable are mutually exclusive
    // Batch mode requires all pods to update simultaneously (all become NotReady)
    // MinAvailable requires maintaining minimum ready pods
    // These are contradictory requirements
    
    if pclq.Spec.MinAvailable != nil && *pclq.Spec.MinAvailable > 0 {
        errMsg := fmt.Sprintf(
            "Batch update rejected: Batch mode and MinAvailable are mutually exclusive. "+
            "Batch mode requires all pods to update simultaneously (temporarily NotReady), "+
            "while MinAvailable=%d requires maintaining ready pods. "+
            "Choose one: (1) Use Batch mode with minAvailable=0 or unset for version consistency, "+
            "OR (2) Use Serial mode with minAvailable=%d for high availability.",
            *pclq.Spec.MinAvailable, *pclq.Spec.MinAvailable)
        
        logger.Error(nil, "Batch mode configuration conflict", 
            "minAvailable", *pclq.Spec.MinAvailable,
            "updateMode", "Batch")
        
        r.eventRecorder.Eventf(pclq, corev1.EventTypeWarning,
            constants.ReasonBatchModeMinAvailableConflict,
            "Batch mode and MinAvailable=%d are mutually exclusive. "+
            "Set minAvailable=0 to use Batch mode, or use Serial mode to maintain availability.",
            *pclq.Spec.MinAvailable)
        
        return groveerr.New(
            groveerr.ErrCodeInvalidConfiguration,
            component.OperationSync,
            errMsg,
        )
    }
    
    logger.Info("Starting batch in-place update (no MinAvailable constraint)",
        "totalPods", len(work.inPlaceUpdatablePods),
        "updateMode", "Batch")
    
    // Check if batch update is already in progress
    if isBatchUpdateInProgress(pclq) {
        // Wait for all pods to complete update
        if areAllPodsUpdatedAndReady(work.inPlaceUpdatablePods, sc) {
            logger.Info("Batch in-place update completed successfully",
                "updatedPods", len(work.inPlaceUpdatablePods))
            
            // Update all pod labels with new template hash
            for _, pod := range work.inPlaceUpdatablePods {
                if err := r.updatePodTemplateHashLabel(sc.ctx, pod, sc.expectedPodTemplateHash); err != nil {
                    logger.Error(err, "failed to update pod template hash", "pod", pod.Name)
                }
            }
            
            // Mark batch update as completed
            if err := r.markBatchUpdateCompleted(sc.ctx, logger, pclq); err != nil {
                return err
            }
            
            r.eventRecorder.Eventf(pclq, corev1.EventTypeNormal,
                constants.ReasonBatchUpdateCompleted,
                "Batch in-place update completed for %d pods", len(work.inPlaceUpdatablePods))
            
            return nil
        }
        
        // Check for timeout
        if isBatchUpdateTimedOut(pclq) {
            logger.Error(nil, "Batch in-place update timed out, will recreate failed pods")
            r.eventRecorder.Eventf(pclq, corev1.EventTypeWarning,
                constants.ReasonBatchUpdateTimeout,
                "Batch in-place update timed out after %v", getBatchTimeout(pclq))
            
            // Mark failed pods for recreation
            for _, pod := range work.inPlaceUpdatablePods {
                if !isPodRunningExpectedImages(pod, sc) || !k8sutils.IsPodReady(pod) {
                    r.recordInPlaceUpdateFailure(sc, pod)
                }
            }
            
            // Reset batch update state
            r.markBatchUpdateCompleted(sc.ctx, logger, pclq)
            return groveerr.New(
                groveerr.ErrCodeContinueReconcileAndRequeue,
                component.OperationSync,
                "batch update timed out, falling back to recreate for failed pods",
            )
        }
        
        // Still waiting for pods to complete update
        logger.Info("Waiting for batch in-place update to complete",
            "totalPods", len(work.inPlaceUpdatablePods),
            "readyPods", countReadyPodsWithExpectedImages(work.inPlaceUpdatablePods, sc))
        
        return groveerr.New(
            groveerr.ErrCodeContinueReconcileAndRequeue,
            component.OperationSync,
            "waiting for batch in-place update to complete",
        )
    }
    
    // Start batch update - update all pods simultaneously
    logger.Info("Starting batch in-place update",
        "totalPods", len(work.inPlaceUpdatablePods),
        "updateMode", "Batch")
    
    // Update all pods in parallel
    var updateErrors []error
    for _, pod := range work.inPlaceUpdatablePods {
        if err := r.updatePodInPlace(sc.ctx, logger, pclq, pod, sc); err != nil {
            logger.Error(err, "failed to update pod in batch", "pod", pod.Name)
            updateErrors = append(updateErrors, err)
        }
    }
    
    // Mark batch update as started
    if err := r.markBatchUpdateStarted(sc.ctx, logger, pclq, len(work.inPlaceUpdatablePods)); err != nil {
        return err
    }
    
    r.eventRecorder.Eventf(pclq, corev1.EventTypeNormal,
        constants.ReasonBatchUpdateStarted,
        "Started batch in-place update for %d pods", len(work.inPlaceUpdatablePods))
    
    if len(updateErrors) > 0 {
        logger.Info("Some pods failed to update, will retry", "failures", len(updateErrors))
    }
    
    return groveerr.New(
        groveerr.ErrCodeContinueReconcileAndRequeue,
        component.OperationSync,
        fmt.Sprintf("batch in-place update started for %d pods", len(work.inPlaceUpdatablePods)),
    )
}

// Helper functions for batch update tracking

func isBatchUpdateInProgress(pclq *grovecorev1alpha1.PodClique) bool {
    return pclq.Status.RollingUpdateProgress != nil &&
           pclq.Status.RollingUpdateProgress.BatchUpdate != nil &&
           pclq.Status.RollingUpdateProgress.BatchUpdate.InProgress
}

func isBatchUpdateTimedOut(pclq *grovecorev1alpha1.PodClique) bool {
    if !isBatchUpdateInProgress(pclq) {
        return false
    }
    
    startTime := pclq.Status.RollingUpdateProgress.BatchUpdate.StartTime
    timeout := getBatchTimeout(pclq)
    
    return time.Since(startTime.Time) > timeout
}

func getBatchTimeout(pclq *grovecorev1alpha1.PodClique) time.Duration {
    if pclq.Spec.RollingUpdateStrategy != nil && 
       pclq.Spec.RollingUpdateStrategy.BatchUpdateTimeout != nil {
        return pclq.Spec.RollingUpdateStrategy.BatchUpdateTimeout.Duration
    }
    return 5 * time.Minute // default timeout
}

func areAllPodsUpdatedAndReady(pods []*corev1.Pod, sc *syncContext) bool {
    for _, pod := range pods {
        if !isPodRunningExpectedImages(pod, sc) || !k8sutils.IsPodReady(pod) {
            return false
        }
    }
    return true
}

func countReadyPodsWithExpectedImages(pods []*corev1.Pod, sc *syncContext) int {
    count := 0
    for _, pod := range pods {
        if isPodRunningExpectedImages(pod, sc) && k8sutils.IsPodReady(pod) {
            count++
        }
    }
    return count
}
```

#### Status Tracking for Batch Updates

Extend PodClique status to track batch update progress:

```go
type RollingUpdateProgress struct {
    // ... existing fields ...
    
    // BatchUpdate tracks the progress of batch in-place updates
    // Only set when UpdateMode is "Batch"
    // +optional
    BatchUpdate *BatchUpdateProgress `json:"batchUpdate,omitempty"`
}

type BatchUpdateProgress struct {
    // InProgress indicates if batch update is currently in progress
    InProgress bool `json:"inProgress"`
    
    // StartTime is when the batch update started
    StartTime metav1.Time `json:"startTime"`
    
    // TotalPods is the total number of pods in the batch
    TotalPods int `json:"totalPods"`
    
    // UpdatedPods is the number of pods that completed the update
    UpdatedPods int `json:"updatedPods"`
    
    // ReadyPods is the number of pods that are Ready with new images
    ReadyPods int `json:"readyPods"`
}
```

#### Configuration Examples

**Example 1: Default Behavior (Auto-detect + Serial)**

```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: inference-workers
spec:
  replicas: 10
  # No rollingUpdateStrategy specified - uses defaults:
  # - Type: InPlaceIfPossible (auto-detect)
  # - UpdateMode: Serial (one-by-one)
  podSpec:
    containers:
    - name: inference
      image: model:v1.0
```

**Example 2: Force In-Place + Serial (Fail if not image-only)**

```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: stateful-app
spec:
  replicas: 5
  rollingUpdateStrategy:
    type: InPlace  # Force in-place, fail if other changes detected
    updateMode: Serial
  podSpec:
    containers:
    - name: app
      image: app:v2.0
```

**Example 3: Always Recreate + Serial (Traditional behavior)**

```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: legacy-app
spec:
  replicas: 3
  rollingUpdateStrategy:
    type: Recreate  # Always delete and recreate pods
    updateMode: Serial
  podSpec:
    containers:
    - name: app
      image: legacy:v1.0
```

**Example 4: Batch Mode for Version Consistency (No MinAvailable)**

```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: distributed-training
spec:
  replicas: 16
  # No minAvailable - batch mode requires omitting MinAvailable
  rollingUpdateStrategy:
    type: InPlaceIfPossible  # Auto-detect (can be omitted, it's default)
    updateMode: Batch  # Update all pods simultaneously
    batchUpdateTimeout: 10m
  podSpec:
    containers:
    - name: trainer
      image: training:v1.0
  # Configuration rationale:
  # - Distributed training requires all workers at same version
  # - Batch mode ensures version consistency
  # - MinAvailable is omitted/0 to allow batch coordination
  # - Accept temporary unavailability (5-30s) during update
  # - All workers resume synchronized
```

**Example 5: High Availability with Serial Updates**

```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: gpu-workers
spec:
  replicas: 32
  minAvailable: 30  # High availability requirement
  rollingUpdateStrategy:
    type: InPlace  # Only allow in-place updates
    updateMode: Serial  # Must use Serial with MinAvailable
    # Cannot use Batch - mutually exclusive with MinAvailable
  podSpec:
    containers:
    - name: worker
      image: gpu-workload:v3.0
  # Configuration rationale:
  # - High availability required (minAvailable=30)
  # - Must use Serial mode (Batch conflicts with MinAvailable)
  # - In-place updates minimize disruption per pod
  # - Pods update one-by-one, maintaining 30+ ready pods
```

#### Update Flow for Batch Mode

When updating with `UpdateMode: Batch`:

1. **Validation Phase**:
   - Controller checks: `MinAvailable == nil or MinAvailable == 0`
   - If validation fails (MinAvailable > 0):
     - Emit `BatchModeMinAvailableConflict` error event
     - Reject rolling update entirely
     - Log error: "Batch mode and MinAvailable are mutually exclusive"
   - If validation passes, proceed to next step

2. **Strategy Evaluation**:
   - Controller evaluates update strategy for all pods
   - Determines which pods can be updated in-place vs need recreate
   - Only in-place eligible pods participate in batch update

3. **Batch Update Execution** (for in-place updates):
   - Controller patches **all** eligible pods simultaneously with new images
   - Kubelet on each node detects spec change
   - Kubelet pulls new images and restarts containers
   - During restart: **All** pods become NotReady (typically 5-30 seconds)
   - ⚠️ **Complete unavailability during this window**

4. **Monitoring Phase**:
   - Controller monitors all pods for completion
   - Waits for all pods to become Ready with new images
   - Tracks progress in status: `ReadyPods / TotalPods`

5. **Completion**:
   - Once all pods Ready with expected images, mark update complete
   - Update PodTemplateHash labels on all pods
   - Emit `BatchUpdateCompleted` event
   - All pods now synchronized at same version

6. **Timeout Handling**:
   - If timeout occurs, identify failed pods
   - Mark failed pods for recreate on next reconcile
   - Emit `BatchUpdateTimeout` warning

**Key Point**: In batch mode, there is **no MinAvailable protection**. All pods are updated simultaneously, accepting complete unavailability for the duration of container restarts.

#### Benefits of Batch Mode

1. **Eliminates Version Skew**: All pods run the same version, no mixed old/new state during rollout
2. **Much Faster**: Parallel updates complete in seconds vs minutes for serial (5-30s vs minutes)
3. **Coordinated State**: Distributed systems can assume all peers are at same version after update
4. **Simpler Coordination**: No complex logic to handle mixed versions during rollout
5. **Predictable Downtime**: Brief, predictable unavailability window (5-30s) vs prolonged gradual rollout

#### When to Use Batch Mode

Use Batch mode when:
- ✅ Version consistency is critical (distributed protocols, consensus, training)
- ✅ Workload naturally pauses during updates (batch jobs, training)
- ✅ You can tolerate brief complete unavailability (5-30 seconds)
- ✅ Fast rollout is more important than continuous availability

Do NOT use Batch mode when:
- ❌ High availability is required (use Serial + MinAvailable instead)
- ❌ Service must remain available during updates
- ❌ Cannot tolerate any downtime window

### Monitoring

**Kubernetes Events**

New event reasons are added to track rolling updates:

| Event Reason | Type | Description |
|-------------|------|-------------|
| `PodUpdatedInPlace` | Normal | Pod successfully patched for in-place image update |
| `PodUpdateFailed` | Warning | In-place update patch operation failed |
| `PodImagePullFailed` | Warning | New image pull failed during in-place update |
| `PodRestartFailed` | Warning | Container restart failed after in-place update |
| `InPlaceUpdateNotPossible` | Warning | In-place update forced but non-image changes detected (Type=InPlace) |
| `BatchUpdateStarted` | Normal | Batch update initiated for N pods |
| `BatchUpdateCompleted` | Normal | Batch update completed successfully |
| `BatchUpdateTimeout` | Warning | Batch update timed out, falling back to recreate |
| `BatchModeMinAvailableConflict` | Warning | Batch mode rejected - mutually exclusive with MinAvailable > 0 |
| `RecreateStrategyUsed` | Normal | Pod will be deleted and recreated (Type=Recreate or fallback) |

**Metrics**

Proposed Prometheus metrics:

```
# Total rolling updates by strategy type
grove_rolling_updates_total{
    podclique="...",
    namespace="...",
    type="inplace-if-possible|inplace|recreate",
    mode="serial|batch",
    result="success|failure"
}

# Update duration (seconds)
grove_rolling_update_duration_seconds{
    podclique="...",
    namespace="...",
    type="inplace-if-possible|inplace|recreate",
    mode="serial|batch"
}

# Batch update specific metrics
grove_batch_update_pods_total{podclique="...",namespace="..."}
grove_batch_update_ready_pods{podclique="...",namespace="..."}
grove_batch_update_timeout_total{podclique="...",namespace="..."}

# Strategy selection outcomes
grove_update_strategy_selected_total{
    podclique="...",
    namespace="...",
    requested_type="inplace-if-possible|inplace|recreate",
    actual_strategy="inplace|recreate",
    reason="image-only|other-changes|forced|failed"
}
```

**PodClique Status Fields**

Enhance `RollingUpdateProgress` status to include update strategy:

```go
type RollingUpdateProgress struct {
    // ... existing fields ...
    
    // CurrentUpdateStrategy indicates the strategy being used for current update
    // +optional
    CurrentUpdateStrategy *UpdateStrategyStatus `json:"currentUpdateStrategy,omitempty"`
    
    // BatchUpdate tracks the progress of batch updates
    // Only set when UpdateMode is "Batch"
    // +optional
    BatchUpdate *BatchUpdateProgress `json:"batchUpdate,omitempty"`
}

type UpdateStrategyStatus struct {
    // Type is the configured strategy type
    Type UpdateStrategyType `json:"type"`
    
    // Mode is the configured update mode (serial or batch)
    Mode UpdateMode `json:"mode"`
    
    // ActualStrategy is what strategy is actually being used
    // "in-place" or "recreate"
    ActualStrategy string `json:"actualStrategy"`
    
    // Reason explains why this strategy was chosen
    Reason string `json:"reason,omitempty"`
}

type BatchUpdateProgress struct {
    // InProgress indicates if batch update is currently in progress
    InProgress bool `json:"inProgress"`
    
    // StartTime is when the batch update started
    StartTime metav1.Time `json:"startTime"`
    
    // TotalPods is the total number of pods in the batch
    TotalPods int `json:"totalPods"`
    
    // UpdatedPods is the number of pods that completed the update
    UpdatedPods int `json:"updatedPods"`
    
    // ReadyPods is the number of pods that are Ready with new images
    ReadyPods int `json:"readyPods"`
}
```

### Dependencies

**Kubernetes Version**: 1.27+
- Kubelet support for in-place container restart via pod spec patching
- Testing conducted on Kubernetes 1.27, 1.28, 1.29, 1.30

**No external dependencies required** - implementation uses standard Kubernetes client-go APIs

### Test Plan

**Unit Tests**

- `canUpdateInPlace()` logic with various pod spec differences
  - Image-only changes (single and multiple containers)
  - Image + env var changes
  - Image + volume changes
  - Image + resource limit changes
- `computePodSpecDiff()` for all pod spec fields
- `onlyImagesChanged()` with edge cases (container order, missing containers)
- `isPodRunningExpectedImages()` with various image formats (tag, digest, registry)

**Integration Tests**

Create issue tracking integration test scenarios:

**Strategy Type Tests**:
1. **InPlaceIfPossible - Image Only**: Image-only change triggers in-place update
2. **InPlaceIfPossible - With Other Changes**: Non-image change falls back to recreate
3. **InPlace - Image Only**: Forced in-place succeeds for image-only change
4. **InPlace - With Other Changes**: Forced in-place fails with error event
5. **Recreate - Always**: Always uses recreate regardless of changes
6. **Default Strategy**: Unspecified strategy defaults to InPlaceIfPossible

**Serial Mode Tests**:
7. **Serial In-Place Update**: Pods update one-by-one with in-place strategy
8. **Serial Recreate Update**: Pods recreate one-by-one
9. **Serial Mixed Strategy**: Some pods in-place, some recreate in same update
10. **Serial with MinAvailable**: Respects MinAvailable constraints
11. **Multi-Container Serial**: All container images updated correctly

**Batch Mode Tests**:
12. **Basic Batch Update**: All pods updated simultaneously without MinAvailable, all reach Ready together
13. **Batch Update Timeout**: Timeout triggers fallback to recreate for failed pods
14. **Batch with MinAvailable Conflict**: Batch mode rejected when `MinAvailable > 0` (any value)
15. **Batch without MinAvailable**: Batch mode succeeds when MinAvailable is omitted or 0
16. **Batch Mode MinAvailable=1 Rejected**: Verify even MinAvailable=1 conflicts with Batch
17. **Large Scale Batch**: 50+ pods updated in batch mode without MinAvailable
18. **Batch Failure Recovery**: Failed batch update correctly identifies and handles failed pods
19. **Distributed Training Scenario**: Simulated training workload without MinAvailable for version sync

**General Tests**:
18. **Update Failure Recovery**: In-place failures fall back to recreate
19. **Rollback Scenario**: Update followed by rollback to previous image
20. **Concurrent Updates**: Multiple PodCliques updating simultaneously
21. **Status Tracking**: Verify status fields correctly reflect update progress

**E2E Tests**

Create dedicated issue for E2E test plan covering:

- End-to-end rolling update with in-place strategy
- Performance comparison: in-place vs recreate update duration
- PodCliqueSet rolling updates with mixed update strategies
- Failure recovery and automatic fallback scenarios
- Update progress observability through events and status

**Test Issue**: [Create GitHub issue for comprehensive test plan]

### Graduation Criteria

**Alpha** (Initial Implementation)

- API extension: `RollingUpdateStrategy` field added to PodClique spec
- Core in-place update logic implemented in `rollingupdate.go`
- Strategy type support: InPlaceIfPossible, InPlace, Recreate
- Update mode support: Serial and Batch
- Automatic detection of image-only changes for InPlaceIfPossible
- Fallback mechanisms for failed in-place updates
- Basic unit test coverage (>80%)
- Documentation: GREP proposal, user guide with examples
- Known limitations documented

**Beta** (Stabilization)

- Integration test suite covering all strategy combinations
- E2E test coverage for common use cases
- Enhanced observability: comprehensive metrics, detailed events
- Production validation on 3+ clusters
- Performance benchmarks documenting update time improvements across strategies
- Bug fixes and performance improvements based on alpha feedback
- Comprehensive testing for batch updates with distributed workloads
- Documentation improvements: decision guide, troubleshooting
- Support for init container image updates (if feasible)

**GA** (Production Ready)

- Proven stability in production environments (6+ months beta usage)
- Comprehensive documentation with troubleshooting guides
- Performance optimizations for large-scale deployments (100+ pods)
- Full E2E test coverage (>90%)
- API stability guarantees
- Clear best practices for each strategy type and mode
- Support for complex update scenarios (multi-container, sidecars)

## Implementation History

- **2026-02-03**: GREP-292 proposal created with full API design
- **TBD**: Alpha release - Core implementation with all strategy types and modes
- **TBD**: Beta release - Production validation and performance optimization
- **TBD**: GA release - Production ready with comprehensive documentation

## Alternatives

### Alternative 1: Explicit API Field for Update Strategy

**Approach**: Add an explicit `updateStrategy` field to PodClique spec allowing users to choose "InPlace", "Recreate", or "Auto".

**Pros**:
- Explicit user control over update behavior
- Clear declarative intent in API

**Cons**:
- Requires API changes (violates design goal)
- Adds cognitive overhead for users to understand when to use each strategy
- Risk of users choosing suboptimal strategies
- Complicates validation and rollback logic

**Decision**: Rejected in favor of automatic detection which requires zero API changes and provides optimal behavior without user intervention.

### Alternative 2: Kubernetes Native InPlaceUpdatePolicy

**Approach**: Wait for Kubernetes core to implement native in-place update policies (KEP under discussion).

**Pros**:
- Standardized approach across ecosystem
- Potential for better Kubelet integration

**Cons**:
- Timeline uncertain (KEP not yet approved)
- May not address Grove-specific requirements (gang scheduling, MinAvailable)
- Delays delivering value to users

**Decision**: Rejected. Implement now using existing Kubernetes capabilities (pod spec patching) which is well-supported and tested. Future migration to native KEP possible if it materializes.

### Alternative 3: Image-Only PodSpec Subset

**Approach**: Define a separate "ImageUpdateSpec" that can be updated independently from full PodSpec.

**Pros**:
- Clear separation of updateable vs immutable fields
- Potential for faster reconciliation

**Cons**:
- Significant API redesign required
- Breaks existing PodClique/PodCliqueSet patterns
- Complicates spec validation and defaulting

**Decision**: Rejected. Violates design goal of zero API changes and introduces unnecessary complexity.

### Alternative 4: CRD-Based Update Strategy Resources

**Approach**: Create a separate CRD (e.g., `PodUpdatePolicy`) that defines update strategies per PodClique.

**Pros**:
- Separation of concerns
- Advanced users can customize behavior

**Cons**:
- Increases operational complexity
- Additional CRD to manage
- Violates simplicity and automatic detection goals

**Decision**: Rejected. Adds unnecessary abstraction when automatic detection provides optimal behavior.

### Alternative 5: Container Lifecycle Hooks for Coordination

**Approach**: Use Kubernetes [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) (PostStart, PreStop) to coordinate batch updates across pods.

**How it would work**:
```yaml
lifecycle:
  postStart:
    exec:
      command:
      - /bin/sh
      - -c
      - |
        # Wait for container to be ready
        while ! curl -f http://localhost:8080/health; do sleep 1; done
        # Register with coordination service
        curl -X POST http://coordination-svc/register-updated
        # Wait for all pods to complete update
        while ! curl -f http://coordination-svc/all-ready; do sleep 5; done
```

**Pros**:
- Uses native Kubernetes mechanisms
- Application-level awareness of update coordination
- Can implement custom coordination logic

**Cons**:
- **Container-level scope only**: Hooks run in individual containers and cannot directly coordinate across pods
- **Requires external coordination service**: Need to deploy and manage a separate coordination service
- **Application dependency**: Applications must be modified to participate in coordination
- **Complexity**: Hook scripts can fail, timeout, or hang, complicating debugging
- **No controller visibility**: Grove controller cannot track or manage hook-based coordination
- **Tight coupling**: Applications become tightly coupled to deployment infrastructure
- **Failure handling**: If coordination service fails, all updates block indefinitely

**Decision**: Rejected. According to the [Kubernetes documentation](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/), lifecycle hooks are designed for container-level events, not cluster-wide coordination. Implementing batch update coordination at the controller level (Alternative/Approach 1) provides:
- Centralized orchestration with full visibility
- No application code changes required
- Robust timeout and failure handling
- Clean separation between application logic and deployment strategy
- Consistent behavior across all workloads

## Appendix

### Related GitHub Issues

- [Issue #292: Rolling Update support inplace update pod image](https://github.com/ai-dynamo/grove/issues/292)

### Related GREPs

- [GREP-244: Topology Aware Scheduling](../244-topology-aware-scheduling/README.md) - In-place updates preserve topology-aware pod placement

### Kubernetes References

- [KEP-1287: In-place Update of Pod Resources](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1287-in-place-update-pod-resources) - Background on Kubernetes in-place update capabilities
- [Pod Spec Patching](https://kubernetes.io/docs/reference/using-api/api-concepts/#patch) - Kubernetes PATCH operation documentation

### Performance Benchmarks (Estimated)

Based on typical Grove deployments:

| Metric | Recreate Strategy | In-Place Serial | In-Place Batch | Improvement |
|--------|------------------|-----------------|----------------|-------------|
| Single Pod Update | 45-120 seconds | 5-30 seconds | 5-30 seconds | **6-10x faster** |
| 10-Pod Rolling Update | 8-20 minutes | 1-5 minutes | 30-120 seconds | **5-10x faster** |
| 100-Pod Rolling Update | 80-200 minutes | 15-50 minutes | 5-15 minutes | **8-16x faster** |

*Note: Actual performance depends on image size, cluster load, and network conditions. Batch mode provides additional speedup by parallelizing updates.*

### Decision Guide: Choosing Update Strategy

#### Strategy Type Selection

| Strategy Type | When to Use | Pros | Cons |
|--------------|-------------|------|------|
| **InPlaceIfPossible** (Default) | Most use cases | Automatic optimization, safe fallback | None |
| **InPlace** | Strict requirement for in-place only | Fails fast if incompatible, prevents accidental recreate | Update fails on non-image changes |
| **Recreate** | Legacy behavior, complex state | Predictable behavior, clean slate | Slower, higher disruption |

**Recommendations**:
- 🟢 **Use InPlaceIfPossible** for most workloads - provides best balance
- 🟡 **Use InPlace** when you need guarantee of in-place behavior (e.g., IP must not change)
- 🔴 **Use Recreate** only if you have issues with in-place updates

#### Update Mode Selection

| Update Mode | When to Use | Pros | Cons | MinAvailable Compatibility |
|------------|-------------|------|------|---------------------------|
| **Serial** (Default) | Independent pods, gradual rollout, high availability required | Safe, early failure detection, maintains availability | Slower, version skew during rollout | ✅ Compatible with any MinAvailable |
| **Batch** | Distributed systems requiring version consistency | Fast, eliminates version skew, coordinated | All pods unavailable during update (5-30s) | ❌ **Incompatible with MinAvailable > 0** |

**Critical Rule**: Batch mode and MinAvailable are **mutually exclusive**

```
Batch Mode  →  MinAvailable must be 0 or unset
MinAvailable > 0  →  Must use Serial mode
```

**Recommendations**:

| Workload Type | Recommended Mode | MinAvailable | Reasoning |
|--------------|------------------|--------------|-----------|
| **Stateless API Services** | Serial | High (e.g., N-2) | Maintain availability, tolerate version skew |
| **Distributed Training** | Batch | 0 or unset | Version consistency required, training pauses anyway |
| **Database Clusters** | Serial | Quorum | High availability critical |
| **Consensus Protocols** | Batch | 0 or unset | Protocol requires all nodes at same version |
| **Microservices** | Serial | Moderate | Independent pods, gradual rollout |

**Decision Guide**:

```
Do you need strict availability (maintain minimum ready pods)?
├─ YES → Use Serial mode with MinAvailable
│         Accept version skew during rollout
│
└─ NO → Can tolerate brief full unavailability?
         ├─ YES → Use Batch mode without MinAvailable
         │         All pods update simultaneously
         │
         └─ NO → Use Serial mode with MinAvailable=0
                  Gradual rollout without availability guarantee
```

**Why Mutually Exclusive?**

The goals are fundamentally contradictory:
- **Batch**: All pods NotReady simultaneously (for version consistency)
- **MinAvailable**: Maintain N ready pods always (for availability)
- **Cannot achieve both**: Choose availability OR version consistency, not both

#### Example Decision Matrix

| Workload Type | Strategy Type | Update Mode | MinAvailable | Reasoning |
|--------------|---------------|-------------|--------------|-----------|
| Stateless API Service | InPlaceIfPossible | Serial | N-2 | Independent pods, gradual rollout, maintain availability |
| Distributed Training | InPlaceIfPossible | Batch | 0 (unset) | Version consistency required, training pauses anyway |
| Database Cluster | Recreate | Serial | Quorum (N/2+1) | Complex state, prefer clean slate, maintain quorum |
| Real-time Inference | InPlace | Serial | N-5 | IP stability + maintain availability |
| Consensus Protocol | InPlaceIfPossible | Batch | 0 (unset) | All nodes must be at same version |
| Legacy Application | Recreate | Serial | N-1 | Traditional approach, proven behavior |

#### Configuration Examples

**Scenario 1: Stateless Microservice (Recommended Default)**
```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: api-service
spec:
  replicas: 20
  minAvailable: 15
  # No rollingUpdateStrategy - uses defaults:
  # type: InPlaceIfPossible, updateMode: Serial
  podSpec:
    containers:
    - name: api
      image: api:v2.0
```

**Scenario 2: Distributed Training (Batch without MinAvailable)**
```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: pytorch-training
spec:
  replicas: 16
  # No minAvailable - required for Batch mode
  rollingUpdateStrategy:
    type: InPlaceIfPossible  # Can omit (default)
    updateMode: Batch  # All workers update together
    batchUpdateTimeout: 10m
  podSpec:
    containers:
    - name: trainer
      image: pytorch:v2.0
  # Configuration rationale:
  # - Distributed training requires all workers at same version
  # - Training pauses during update anyway (no availability needed)
  # - MinAvailable omitted to allow Batch mode
  # - All workers synchronize to new version (5-30s downtime)
  # - Training resumes with all workers at same version
```

**Scenario 3: IP-Sensitive Workload (Force In-Place)**
```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: ip-registered-service
spec:
  replicas: 5
  rollingUpdateStrategy:
    type: InPlace  # Must keep pod IP, fail if not possible
    updateMode: Serial
  podSpec:
    containers:
    - name: service
      image: service:v1.5
```

**Scenario 4: Conservative Legacy App (Always Recreate)**
```yaml
apiVersion: grove.io/v1alpha1
kind: PodClique
metadata:
  name: legacy-database
spec:
  replicas: 3
  rollingUpdateStrategy:
    type: Recreate  # Always delete and recreate
    updateMode: Serial
  podSpec:
    containers:
    - name: db
      image: db:v5.0
```

### Common Questions about Batch Mode and MinAvailable

**Q: Why are Batch mode and MinAvailable mutually exclusive?**

A: They have contradictory goals:
- **Batch mode**: All pods update simultaneously → All pods NotReady for 5-30 seconds
- **MinAvailable**: Maintain N ready pods always → Cannot tolerate all pods NotReady

You cannot have all pods NotReady while maintaining minimum ready pods. Choose one:
- Need availability? → Use Serial mode with MinAvailable
- Need version consistency? → Use Batch mode without MinAvailable

**Q: What if I set `minAvailable: 1` with Batch mode?**

A: **Still rejected**. Any `minAvailable > 0` conflicts with Batch mode. Even with `minAvailable: 1`, during batch update:
- All pods become NotReady → ReadyReplicas = 0
- Violates `minAvailable: 1` constraint

**Simple rule**: Batch mode requires `minAvailable = 0` or unset.

**Q: Can I have high availability AND version synchronization?**

A: **No - these are fundamentally incompatible**. You must choose:

| Priority | Configuration |
|----------|---------------|
| **High Availability** | `Serial mode + minAvailable: N` |
| **Version Consistency** | `Batch mode + no minAvailable` |

**Q: What if my distributed training needs both?**

A: For distributed training:
- Training **pauses during updates anyway** → no real availability need
- Use **Batch mode without MinAvailable**
- Accept brief unavailability (5-30s) for version sync
- Training resumes with all workers synchronized

**Q: What happens if I accidentally configure both?**

A: The controller will:
1. Detect the conflict during rolling update
2. Emit `BatchModeMinAvailableConflict` error event
3. **Reject the rolling update entirely**
4. Log clear error message explaining the conflict
5. Require you to fix configuration before proceeding

**Q: How do I fix the conflict?**

A: Choose one approach:

**Option 1: Use Batch (for version consistency)**
```yaml
spec:
  replicas: 16
  # Remove or set minAvailable: 0
  rollingUpdateStrategy:
    updateMode: Batch
```

**Option 2: Use Serial (for availability)**
```yaml
spec:
  replicas: 16
  minAvailable: 14  # Maintain availability
  rollingUpdateStrategy:
    updateMode: Serial  # Change to Serial
```

**Q: Why such a strict rule? Can't you make exceptions?**

A: No. The design is intentionally simple and clear:
- **Avoids complexity**: No headroom calculations, no partial updates, no edge cases
- **Explicit trade-offs**: Users must consciously choose availability OR consistency
- **Predictable behavior**: No surprises about which pods update when
- **Clear semantics**: Batch = all at once, Serial = one by one

If you think you need both, reconsider whether you truly need strict MinAvailable during updates.

> NOTE: This GREP template has been inspired by [KEP Template](https://github.com/kubernetes/enhancements/blob/f90055d254c356b2c038a1bdf4610bf4acd8d7be/keps/NNNN-kep-template/README.md).

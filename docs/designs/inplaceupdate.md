# Rolling Update with In-Place Pod Image Update Support

## Solution Approach

### Overview

Implement in-place pod image update capability in the rolling update logic without modifying the PodClique/PodCliqueSet API definitions. The controller will automatically detect when in-place updates are possible and apply them, falling back to the existing recreate strategy when necessary.

### Design Principles

1. **Automatic Detection**: Controller automatically determines if in-place update is feasible
2. **Safe Fallback**: When in-place update isn't possible, fall back to delete-recreate
3. **Backwards Compatible**: Existing behavior remains unchanged for non-image updates
4. **Zero API Changes**: No modifications to CRD schemas or API types

---

## High-Level Implementation Logic

### Overview

The in-place update mechanism integrates seamlessly into Grove's existing rolling update flow. The controller analyzes pod specification changes and intelligently routes each pod to either an in-place update path or the traditional recreate path.

### Decision Flow

The controller follows this decision tree for each pod requiring an update:

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

### Rolling Update State Machine

The rolling update process maintains state for each pod being updated:

```
┌─────────────────────────────────────────────────────────────────┐
│                     ROLLING UPDATE STATES                        │
└─────────────────────────────────────────────────────────────────┘

  [Pod Identified for Update]
            ↓
    ┌───────────────┐
    │  ANALYZING    │ ← Determine update strategy
    └───────────────┘
            ↓
         ╱     ╲
        ╱       ╲
   [In-Place]  [Recreate]
       ↓           ↓
  ┌─────────┐  ┌──────────────┐
  │ PATCHING│  │ DELETING_POD │
  └─────────┘  └──────────────┘
       ↓              ↓
  ┌─────────────┐  ┌────────────┐
  │ RESTARTING  │  │ WAITING_   │
  │ (Kubelet)   │  │ DELETION   │
  └─────────────┘  └────────────┘
       ↓              ↓
  ┌─────────────┐  ┌────────────┐
  │ WAITING_    │  │ CREATING_  │
  │ READY       │  │ NEW_POD    │
  └─────────────┘  └────────────┘
       ↓              ↓
       └──────┬───────┘
              ↓
      ┌───────────────┐
      │   COMPLETED   │
      └───────────────┘
              ↓
      [Next Pod Update]
```

### Component Interaction Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                    PodClique Controller                         │
└────────────────────────────────────────────────────────────────┘
                           │
                           │ Reconcile Loop
                           ↓
    ┌──────────────────────────────────────────────────────┐
    │         rollingupdate.processPendingUpdates()        │
    │                                                       │
    │  1. computeUpdateWork()                              │
    │     ├─ List existing pods                            │
    │     ├─ Compare with expected template                │
    │     └─ Categorize: InPlace vs Recreate              │
    │                                                       │
    │  2. canUpdateInPlace()                               │
    │     ├─ computePodSpecDiff()                          │
    │     │  └─ Compare all pod spec fields               │
    │     └─ onlyImagesChanged()                           │
    │        └─ Deep compare containers                    │
    │                                                       │
    │  3. Route to update path                             │
    │     ├─ IN-PLACE: updatePodInPlace()                  │
    │     │            ├─ Patch pod.spec.containers.image  │
    │     │            └─ Record event                     │
    │     │                                                 │
    │     └─ RECREATE: createPodDeletionTask()            │
    │                  └─ client.Delete(pod)               │
    │                                                       │
    │  4. Monitor completion                               │
    │     ├─ isPodRunningExpectedImages()                  │
    │     └─ isCurrentPodUpdateComplete()                  │
    └──────────────────────────────────────────────────────┘
                           │
                           ↓
    ┌──────────────────────────────────────────────────────┐
    │              Kubernetes API Server                    │
    └──────────────────────────────────────────────────────┘
                           │
                           ↓
    ┌──────────────────────────────────────────────────────┐
    │                    Kubelet                            │
    │  (Handles container restart for in-place updates)    │
    └──────────────────────────────────────────────────────┘
```

### Update Strategy Selection Logic

The controller examines pod specification changes at a granular level:

**Scenario 1: Image-Only Change (In-Place Update)**
```
Old Pod Spec:                    New Pod Spec:
  containers:                      containers:
  - name: app                      - name: app
    image: nginx:1.19        →       image: nginx:1.20  ← ONLY CHANGE
    ports: [80]                      ports: [80]
    env: [...]                       env: [...]
    
Result: ✅ IN-PLACE UPDATE
```

**Scenario 2: Image + Other Changes (Recreate)**
```
Old Pod Spec:                    New Pod Spec:
  containers:                      containers:
  - name: app                      - name: app
    image: nginx:1.19        →       image: nginx:1.20
    ports: [80]                      ports: [80, 443]   ← EXTRA CHANGE
    env: [...]                       env: [...]
    
Result: ❌ RECREATE (port added)
```

**Scenario 3: Multiple Container Images (In-Place Update)**
```
Old Pod Spec:                    New Pod Spec:
  containers:                      containers:
  - name: app                      - name: app
    image: app:v1          →         image: app:v2      ← CHANGED
  - name: sidecar                  - name: sidecar
    image: sidecar:1.0     →         image: sidecar:1.1 ← CHANGED
    
Result: ✅ IN-PLACE UPDATE (all images changed, nothing else)
```

### Key Implementation Details

#### 1. **Change Detection**

The controller performs a deep comparison of pod specifications:

- **Containers**: Compare each field except `image`
- **Volumes**: Any volume change triggers recreate
- **Security Context**: Any security change triggers recreate
- **Network**: Any network config change triggers recreate
- **Scheduling**: Node selectors, affinity, tolerations changes trigger recreate

#### 2. **In-Place Update Mechanics**

When an in-place update is executed:

1. **Patch Request**: Controller sends a PATCH request to update `pod.spec.containers[*].image`
2. **Kubelet Detection**: Kubelet watches pod spec and detects image change
3. **Image Pull**: Kubelet pulls new container images
4. **Container Restart**: Kubelet stops old containers and starts new ones
5. **Pod Remains**: Pod object persists with same name, IP, volumes, etc.
6. **Status Update**: Container statuses reflect new ImageID and RestartCount increments

#### 3. **MinAvailable Handling**

Critical difference between update strategies:

**In-Place Updates:**
- Pod temporarily becomes NotReady during container restart
- Controller must track in-flight in-place updates
- Limit concurrent in-place updates to maintain MinAvailable
- Typical restart time: 5-30 seconds

**Recreate Updates:**
- Pod is fully deleted before new one is created
- Requires strict MinAvailable enforcement before deletion
- Typical replacement time: 30-120 seconds (includes scheduling)

#### 4. **Completion Detection**

The controller verifies update completion differently:

**For In-Place Updates:**
```go
// Check if pod is running expected images AND is Ready
isPodRunningExpectedImages(pod) {
    for each container:
        ✓ containerStatus.ImageID matches expected image
        ✓ containerStatus.Ready == true
    return all_match
}
```

#### 5. **Failure Handling**

Each update path has distinct failure modes:

**In-Place Update Failures:**
- Image pull failure → Pod stays running old image
- Container crash loop → Pod becomes NotReady
- **Recovery**: Controller detects failure, marks pod for recreate on next reconcile

---

### Implementation Strategy

#### 1. Detection Logic in `computeUpdateWork()`

**Location**: [rollingupdate.go](../../operator/internal/controller/podclique/components/pod/rollingupdate.go) lines 136-159

Add capability detection to identify pods that can be updated in-place:

type updateWork struct {
    oldTemplateHashPendingPods   []*corev1.Pod
    oldTemplateHashUnhealthyPods []*corev1.Pod
    oldTemplateHashReadyPods     []*corev1.Pod
    newTemplateHashReadyPods     []*corev1.Pod
    
    // New: Track which pods can be updated in-place
    inPlaceUpdatablePods         []*corev1.Pod
    mustRecreateP pods            []*corev1.Pod
}

func (r _resource) computeUpdateWork(logger logr.Logger, sc *syncContext) *updateWork {
    work := &updateWork{}
    
    // Get expected pod spec from template
    expectedPodSpec := getExpectedPodSpec(sc)
    
    for _, pod := range sc.existingPCLQPods {
        if pod.Labels[common.LabelPodTemplateHash] != sc.expectedPodTemplateHash {
            if r.hasPodDeletionBeenTriggered(sc, pod) {
                continue
            }
            
            // NEW: Check if pod can be updated in-place
            if canUpdateInPlace(pod, expectedPodSpec) {
                work.inPlaceUpdatablePods = append(work.inPlaceUpdatablePods, pod)
            } else {
                // Categorize based on health status for recreation
                if k8sutils.IsPodPending(pod) {
                    work.oldTemplateHashPendingPods = append(...)
                } else if k8sutils.HasAnyContainerError(pod) {
                    work.oldTemplateHashUnhealthyPods = append(...)
                } else if k8sutils.IsPodReady(pod) {
                    work.mustRecreatePods = append(...)
                }
            }
        }
    }
    return work
}

#### 2. In-Place Update Detection Function

**New function** in `rollingupdate.go`:

// canUpdateInPlace determines if a pod can be updated in-place without recreation
func canUpdateInPlace(existingPod *corev1.Pod, expectedPodSpec corev1.PodSpec) bool {
    // Check Kubernetes version supports in-place updates (1.27+)
    // This could be a one-time check cached at controller startup
    
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
    
    // Check if only images changed
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
    // ... check other fields
    
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

#### 3. Modified `processPendingUpdates()` Flow

**Location**: `rollingupdate.go` lines 72-133

Update the main rolling update logic:

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
    
    // Step 3: NEW - Prioritize in-place updates
    var nextPodToUpdate *corev1.Pod
    var updateStrategy string
    
    if len(work.inPlaceUpdatablePods) > 0 {
        // In-place updates can happen without MinAvailable check
        // because pod remains running during container restart
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
        
        // Update status
        if err := r.updatePCLQStatusWithNextPodToUpdate(sc.ctx, logger, pclq, 
            nextPodToUpdate.Name, updateStrategy); err != nil {
            return err
        }
        
        // Execute based on strategy
        if updateStrategy == "in-place" {
            if err := r.updatePodInPlace(sc.ctx, logger, pclq, nextPodToUpdate, sc); err != nil {
                logger.Error(err, "in-place update failed, will recreate on next reconcile")
                // Don't return error - let it be handled in next reconciliation
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
            fmt.Sprintf("updated pod %s using %s strategy, requeuing", 
                nextPodToUpdate.Name, updateStrategy),
        )
    }
    
    // Step 5: Mark update completion
    return r.markRollingUpdateEnd(sc.ctx, logger, pclq)
}

#### 4. In-Place Update Execution

**New function** in `rollingupdate.go`:

// updatePodInPlace patches the pod spec to trigger kubelet container restart
func (r _resource) updatePodInPlace(ctx context.Context, logger logr.Logger, 
    pclq *grovecorev1alpha1.PodClique, pod *corev1.Pod, sc *syncContext) error {
    
    podObjKey := client.ObjectKeyFromObject(pod)
    logger.Info("Attempting in-place pod update", "pod", podObjKey)
    
    // Get expected pod spec
    expectedPodSpec := getExpectedPodSpec(sc)
    
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
        constants.ReasonPodUpdateSuccessful, 
        "Updated pod %s in-place (image only)", pod.Name)
    
    return nil
}

// getExpectedPodSpec retrieves the expected pod spec from the PodClique template
func getExpectedPodSpec(sc *syncContext) corev1.PodSpec {
    // This is the pod spec that should exist based on current PodClique definition
    return sc.pclq.Spec.PodSpec
}

#### 5. Update Completion Detection

Modify `isCurrentPodUpdateComplete()` to handle in-place updates:

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
    
    // Pod exists - check if it's using new template
    if pod.Labels[common.LabelPodTemplateHash] == sc.expectedPodTemplateHash {
        // Pod has new template hash and is ready - update complete
        return k8sutils.IsPodReady(pod)
    }
    
    // Pod still has old template hash
    if k8sutils.IsResourceTerminating(pod.ObjectMeta) {
        // Recreate strategy in progress
        return false
    }
    
    // Check if in-place update completed by comparing actual vs expected images
    return isPodRunningExpectedImages(pod, sc)
}

// isPodRunningExpectedImages checks if pod containers are running expected images
func isPodRunningExpectedImages(pod *corev1.Pod, sc *syncContext) bool {
    expectedSpec := sc.pclq.Spec.PodSpec
    
    for _, containerStatus := range pod.Status.ContainerStatuses {
        // Find expected image for this container
        var expectedImage string
        for _, expectedContainer := range expectedSpec.Containers {
            if expectedContainer.Name == containerStatus.Name {
                expectedImage = expectedContainer.Image
                break
            }
        }
        
        // Check if container is running expected image
        if !strings.Contains(containerStatus.ImageID, getImageDigest(expectedImage)) {
            return false
        }
        
        // Check if container is ready
        if !containerStatus.Ready {
            return false
        }
    }
    
    return true
}

### Conclusion

This approach provides significant performance improvements for image-only rolling updates while maintaining full backwards compatibility and requiring zero API changes. The implementation is self-contained within the rolling update logic and automatically optimizes based on what changed in the pod specification.
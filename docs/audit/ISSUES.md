# ISSUES — Karmada Repository Audit

> **Audit Date**: 2026-06-15  
> **Auditor**: Automated Deep Audit  
> **Total Issues Found**: 25

---

## ISSUE-001: Race Condition in Scheduler Assumption Cache Memory Leak

| Field | Value |
|---|---|
| **Severity** | Critical |
| **Category** | Bug |
| **Location** | `pkg/scheduler/scheduler.go` |
| **Code Reference** | `patchScheduleResultForResourceBinding()`, Lines 704-708 |

**Problem Description**: The scheduler's `AssigningResourceBindings().Add(result)` call at line 707 is performed *after* the patch to the API server succeeds. There is a documented TODO at line 705-706 acknowledging a race condition: if the Informer's event for the patched result arrives *before* the `Add` call, the assumption cache entry will never be cleaned up, causing a memory leak. The GC goroutine (line 336-341) only runs every minute and relies on TTL, but if the informer event arrives quickly (common in low-latency environments), the entry is added after the informer already processed it, making the TTL-based GC the only cleanup path.

**Reproduction Steps**: Under high scheduling throughput with low API server latency, schedule many ResourceBindings rapidly. The informer event for the patched binding arrives before `Add()` is called, leaving stale entries in the assumption cache.

**Expected Behavior**: Assumption cache entries should always be cleaned up when the corresponding binding event is processed.

**Actual Behavior**: Stale assumption cache entries accumulate, consuming memory and potentially causing incorrect overcommit protection decisions.

**Root Cause Analysis**: The `Add()` call occurs after the API server patch, creating a window where the informer event is processed before the cache entry exists. This is a classic TOCTOU (time-of-check-time-of-use) race.

**Business Impact**: Memory leak in production scheduler; incorrect capacity estimation may cause workload scheduling failures or overcommit.

**Recommended Fix**: Move the `Add()` call to *before* the API server patch, using a "assume then confirm" pattern. If the patch fails, remove the assumption. This is the pattern used by the Kubernetes scheduler.

**Estimated Complexity**: Medium

**GitHub Match**: NONE FOUND (acknowledged as TODO in code)

---

## ISSUE-002: Unbounded Error Retry in Legacy Scheduler Queue

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Performance |
| **Location** | `pkg/scheduler/scheduler.go` |
| **Code Reference** | `legacyHandleErr()`, Lines 948-955 |

**Problem Description**: The `legacyHandleErr()` function calls `s.queue.AddRateLimited(key)` on any non-nil error, with no maximum retry count. This means a permanently failing binding will be retried indefinitely, consuming CPU cycles and filling the rate-limited queue.

**Reproduction Steps**: Create a ResourceBinding that consistently fails scheduling (e.g., references a non-existent cluster affinity with no fallback). Observe the queue growing unboundedly.

**Expected Behavior**: Items should be discarded after a maximum number of retries with appropriate logging.

**Actual Behavior**: Items are retried indefinitely, consuming resources.

**Root Cause Analysis**: No maximum retry limit is enforced. The `handleErr()` method for priority queue also lacks a max retry check.

**Business Impact**: Under error conditions, scheduler CPU usage increases linearly; queue memory grows unboundedly.

**Recommended Fix**: Add a maximum retry counter (e.g., 15 retries as in the `asyncWorker`) before dropping the item with an error log.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-003: `ItemPriorityIfInInitialList` Returns Pointer to Constant

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Bug |
| **Location** | `pkg/util/worker.go` |
| **Code Reference** | `ItemPriorityIfInInitialList()`, Lines 247-252 |

**Problem Description**: The function `ItemPriorityIfInInitialList` calls `new(LowPriority)` where `LowPriority` is a constant (`-100`). In Go, `new(T)` allocates a zero-value `T`, not a value initialized to `LowPriority`. This means the returned pointer points to `0`, not `-100`.

**Reproduction Steps**: Call `ItemPriorityIfInInitialList(true)` and dereference the returned pointer — it will be `0`, not `-100`.

**Expected Behavior**: Returns a pointer to `-100`.

**Actual Behavior**: Returns a pointer to `0` (zero value of `int`).

**Root Cause Analysis**: `new(LowPriority)` is semantically incorrect. Go's `new()` takes a type, not a value. Since `LowPriority` is an untyped constant with value `-100`, `new(LowPriority)` is interpreted as `new(int)` which allocates a zero-value int.

**Business Impact**: Priority queue items from initial list sync are assigned priority 0 instead of -100, breaking the intended low-priority behavior during initial list processing.

**Recommended Fix**: Replace `return new(LowPriority)` with `p := LowPriority; return &p` or use `ptr.To(LowPriority)` from `k8s.io/utils/ptr`.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-004: `context.TODO()` Used Throughout Production Code

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Architecture |
| **Location** | Multiple files (scheduler, detector, override manager, dependencies distributor) |
| **Code Reference** | ~50+ usages across `pkg/scheduler/scheduler.go`, `pkg/detector/detector.go`, `pkg/util/overridemanager/overridemanager.go`, `pkg/dependenciesdistributor/dependencies_distributor.go` |

**Problem Description**: `context.TODO()` is used extensively in production code paths for API calls, including scheduling operations, policy listing, cluster lookups, and patch operations. This prevents proper context cancellation propagation, making graceful shutdown unreliable and preventing timeout enforcement.

**Reproduction Steps**: Initiate a graceful shutdown of the controller-manager while API calls are in-flight. The `context.TODO()` calls will not be cancelled, causing shutdown to hang until the calls complete or time out at the HTTP level.

**Expected Behavior**: All API calls should use the reconciliation context or a derived context with appropriate timeout.

**Actual Behavior**: API calls use `context.TODO()` and ignore cancellation signals.

**Root Cause Analysis**: Historical pattern carried forward from early development.

**Business Impact**: Graceful shutdown takes longer than expected; resource leaks during shutdown; inability to enforce operation timeouts.

**Recommended Fix**: Replace `context.TODO()` with the reconciliation context (`ctx` parameter) in all controller reconcile methods. For methods not receiving a context, thread it through from the caller.

**Estimated Complexity**: Large

**GitHub Match**: NONE FOUND

---

## ISSUE-005: Security — ClusterRole Grants Wildcard Access to All Resources

| Field | Value |
|---|---|
| **Severity** | Critical |
| **Category** | Security |
| **Location** | `pkg/util/credential.go` |
| **Code Reference** | `ClusterPolicyRules`, Lines 45-51; `namespacedPolicyRules`, Lines 37-43 |

**Problem Description**: The RBAC policy rules grant `*` (all verbs) on `*` (all API groups) for `*` (all resources). This gives the Karmada service account in each member cluster **cluster-admin equivalent** privileges, violating the principle of least privilege.

**Reproduction Steps**: Register a member cluster and inspect the ClusterRole created. It has full admin access to all resources.

**Expected Behavior**: Service accounts should have only the permissions needed to manage propagated resources.

**Actual Behavior**: Full cluster-admin access is granted, allowing Karmada to read/modify/delete any resource including secrets, RBAC, and node configurations.

**Root Cause Analysis**: Convenience-driven design decision; complex to determine minimum required permissions dynamically.

**Business Impact**: Compromised Karmada control plane can fully control all member clusters. Any CVE in Karmada becomes a cluster-admin exploit across all member clusters.

**Recommended Fix**: Implement scoped RBAC based on the resources actually being propagated. At minimum, document the risk and provide guidance for narrowing permissions.

**Estimated Complexity**: Large

**GitHub Match**: NONE FOUND

---

## ISSUE-006: Security — gRPC Insecure Skip Verify Option

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Security |
| **Location** | `pkg/scheduler/scheduler.go` |
| **Code Reference** | `WithSchedulerEstimatorConnection()`, Lines 161-171 |

**Problem Description**: The `InsecureSkipServerVerify` option for the scheduler estimator gRPC connection allows disabling TLS certificate verification. When enabled, this makes the connection vulnerable to man-in-the-middle attacks. The scheduler estimator influences scheduling decisions, so a MITM attacker could manipulate cluster capacity estimates to cause mis-scheduling.

**Reproduction Steps**: Start the scheduler with `--scheduler-estimator-insecure-skip-verify=true`. All gRPC connections to estimators skip certificate verification.

**Expected Behavior**: TLS verification should always be enforced in production.

**Actual Behavior**: TLS can be completely bypassed.

**Root Cause Analysis**: Flag provided for development convenience without adequate production safeguards.

**Business Impact**: Potential man-in-the-middle attacks on scheduler estimator connections; attacker could manipulate scheduling decisions.

**Recommended Fix**: Add a startup warning when `insecureSkipVerify` is true. Consider removing this option in favor of providing proper CA certificates. At minimum, log a WARNING-level message.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-007: Resource Detector Lists All Policies Per Resource Event

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Performance |
| **Location** | `pkg/detector/detector.go` |
| **Code Reference** | `LookForMatchedPolicy()`, Lines 381-410; `LookForMatchedClusterPolicy()`, Lines 413-438 |

**Problem Description**: Every time a resource event is processed, the detector calls `Client.List()` to fetch ALL PropagationPolicies in the namespace and ALL ClusterPropagationPolicies. With thousands of policies and resources, this creates O(n*m) API calls.

**Reproduction Steps**: Deploy 1000 PropagationPolicies and 1000 Deployments. Observe the API server load from the detector listing all policies for each resource event.

**Expected Behavior**: An index or cache-based lookup should be used to efficiently match resources to policies.

**Actual Behavior**: Full list operations are performed for every resource reconciliation.

**Root Cause Analysis**: Uses `UnsafeDisableDeepCopy: new(true)` for performance, but the fundamental O(n) list per event is still expensive.

**Business Impact**: Control plane API server overload at scale; increased scheduling latency; potential timeout failures.

**Recommended Fix**: Use informer-backed caches with field selectors or build an in-memory index of policies by resource GVK for O(1) lookup.

**Estimated Complexity**: Large

**GitHub Match**: NONE FOUND

---

## ISSUE-008: Cluster Status Controller Lists All Pods and Nodes

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Performance |
| **Location** | `pkg/controllers/status/cluster_status_controller.go` |
| **Code Reference** | `listPods()`, Lines 512-524; `listNodes()`, Lines 527-539 |

**Problem Description**: The cluster status controller lists ALL pods and ALL nodes from each member cluster to compute resource summaries. In large clusters (10K+ nodes, 100K+ pods), this consumes significant memory in the Karmada control plane.

**Reproduction Steps**: Register a large member cluster with 10,000 nodes and 100,000 pods. Observe memory consumption of the controller-manager.

**Expected Behavior**: Resource summaries should be computed incrementally or by the member cluster itself.

**Actual Behavior**: Full list of all pods and nodes is loaded into memory for every status update cycle.

**Root Cause Analysis**: The resource modeling feature requires full node/pod lists to compute allocatable resources per node.

**Business Impact**: Memory pressure on controller-manager; potential OOM kills with large clusters; increased GC pauses.

**Recommended Fix**: Consider using aggregated metrics (e.g., from metrics-server) or have the member cluster agent compute and report resource summaries. Alternatively, use server-side filtering with field selectors.

**Estimated Complexity**: Large

**GitHub Match**: NONE FOUND

---

## ISSUE-009: Missing Error Propagation in Cluster Status Node Summary

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Bug |
| **Location** | `pkg/controllers/status/cluster_status_controller.go` |
| **Code Reference** | `setCurrentClusterStatus()`, Lines 272-280 |

**Problem Description**: When `listNodes()` or `listPods()` returns an error, the error is only logged but processing continues with potentially nil or partial data. The `getNodeSummary(nodes)` and `getResourceSummary(nodes, pods)` functions are called with the possibly-nil results, which could produce misleading zero-value summaries.

**Reproduction Steps**: Temporarily make the node lister unavailable during a status sync. Observe that the node summary shows 0 nodes even though the cluster has running nodes.

**Expected Behavior**: If node/pod listing fails, the resource summary should retain the previous values or the error should be propagated.

**Actual Behavior**: Zero-value summaries are written, making the cluster appear to have no capacity.

**Root Cause Analysis**: Error from `listNodes`/`listPods` is logged but not returned, allowing stale/zero data to be written.

**Business Impact**: Scheduler may avoid scheduling to clusters that appear to have zero capacity; workload starvation.

**Recommended Fix**: Return the error from `listNodes`/`listPods` to prevent writing zero-value summaries. If partial data is acceptable, document it explicitly.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-010: `getAllocatableModelings` Breaks on `nil` Node Available

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Bug |
| **Location** | `pkg/controllers/status/cluster_status_controller.go` |
| **Code Reference** | `getAllocatableModelings()`, Lines 664-670 |

**Problem Description**: The loop at line 664-670 uses `break` when `nodeAvailable` is `nil`, which exits the entire node iteration loop early. This means if ANY node has exceeded its pod limit, ALL subsequent nodes are skipped from the resource model, producing an inaccurate allocatable modeling.

**Reproduction Steps**: Have a cluster where one node has exceeded its max pod count. All nodes listed after it in the slice will be excluded from the resource model.

**Expected Behavior**: Individual nodes with nil availability should be skipped (`continue`), not cause the entire loop to abort.

**Actual Behavior**: A `break` exits the loop, skipping all remaining nodes.

**Root Cause Analysis**: Incorrect control flow — `break` should be `continue`.

**Business Impact**: Cluster resource models underreport available capacity, causing the scheduler to under-utilize clusters.

**Recommended Fix**: Change `break` to `continue` at line 667.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-011: Execution Controller Does Not Handle Partial Failure Idempotently

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Bug |
| **Location** | `pkg/controllers/execution/execution_controller.go` |
| **Code Reference** | `syncToClusters()`, Lines 266-309 |

**Problem Description**: When syncing multiple manifests in a Work object, if some manifests succeed and some fail, the next retry will re-apply all manifests (including already-succeeded ones). While `tryCreateOrUpdateWorkload` handles existing resources, the repeated updates generate unnecessary API calls and events.

More critically, the `updateAppliedCondition` is called with `ConditionFalse` for partial failures, but the error message includes aggregated errors that may change between retries, causing unnecessary condition transitions.

**Reproduction Steps**: Create a Work object with 5 manifests. Make one manifest invalid. Observe repeated updates to the 4 valid resources on each retry.

**Expected Behavior**: Successfully applied manifests should be tracked and skipped on retry.

**Actual Behavior**: All manifests are re-processed on every retry.

**Root Cause Analysis**: No per-manifest tracking of applied status; entire Work is retried as a unit.

**Business Impact**: Unnecessary API load on member clusters; misleading events; delayed convergence.

**Recommended Fix**: Track per-manifest applied status in the Work status to enable incremental sync.

**Estimated Complexity**: Medium

**GitHub Match**: NONE FOUND

---

## ISSUE-012: Override Manager Does Not Validate JSON Patch Paths

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Security |
| **Location** | `pkg/util/overridemanager/overridemanager.go` |
| **Code Reference** | `parseJSONPatchesByPlaintext()`, Lines 453-463 |

**Problem Description**: `PlaintextOverrider` allows users to specify arbitrary JSON patch operations with any path. There is no validation that the path targets expected fields. An attacker with policy-creation permissions could inject patches targeting sensitive fields like `.metadata.labels`, `.metadata.annotations`, or even `.spec.serviceAccountName`, potentially escalating privileges.

**Reproduction Steps**: Create an OverridePolicy with a PlaintextOverrider that sets `.spec.containers[0].securityContext.privileged` to `true`.

**Expected Behavior**: Override policies should be validated against a set of allowed paths or at least log warnings for sensitive path modifications.

**Actual Behavior**: Any JSON path is accepted without validation.

**Root Cause Analysis**: The override mechanism is intentionally flexible but lacks security guardrails.

**Business Impact**: Privilege escalation via malicious override policies; security-sensitive fields can be modified across all member clusters.

**Recommended Fix**: Add webhook validation for override policies that warns or blocks modifications to security-sensitive paths (`.spec.serviceAccountName`, `.spec.containers[*].securityContext`, `.metadata.namespace`).

**Estimated Complexity**: Medium

**GitHub Match**: NONE FOUND

---

## ISSUE-013: Dependencies Distributor Channel Not Buffered

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance |
| **Location** | `pkg/dependenciesdistributor/dependencies_distributor.go` |
| **Code Reference** | `SetupWithManager()`, Line 695; `reconcileResourceTemplate()`, Line 204 |

**Problem Description**: The `genericEvent` channel is created without a buffer (`make(chan ...)`). When `reconcileResourceTemplate` sends events to this channel (line 204), it will block if the controller's reconcile loop is not consuming events fast enough. This can cause the resource processor worker to stall.

**Reproduction Steps**: Create a large number of dependent resources that match many ResourceBindings. The resource processor workers will block on channel sends.

**Expected Behavior**: The channel should be buffered to prevent blocking.

**Actual Behavior**: Unbuffered channel causes blocking under load.

**Root Cause Analysis**: Default `make(chan)` creates an unbuffered channel.

**Business Impact**: Degraded dependency distribution performance under load; worker starvation.

**Recommended Fix**: Add a reasonable buffer size (e.g., 1024): `make(chan event.TypedGenericEvent[*workv1alpha2.ResourceBinding], 1024)`.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-014: Webhook Mutating Handler Does Not Validate Placement

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Bug |
| **Location** | `pkg/webhook/propagationpolicy/mutating.go` |
| **Code Reference** | `Handle()`, Line 72 |

**Problem Description**: The mutating webhook calls `helper.SetDefaultSpreadConstraints(policy.Spec.Placement.SpreadConstraints)` without nil-checking `policy.Spec.Placement`. If `Placement` is nil, this will panic.

**Reproduction Steps**: Submit a PropagationPolicy with `spec.placement` unset (nil). The webhook will panic.

**Expected Behavior**: Nil placement should be handled gracefully (either set defaults or skip).

**Actual Behavior**: Nil pointer dereference panic, crashing the webhook server.

**Root Cause Analysis**: Missing nil check before accessing `Placement` field.

**Business Impact**: Webhook crash causes all admission requests to fail (fail-open or fail-closed depending on webhook configuration).

**Recommended Fix**: Add a nil check: `if policy.Spec.Placement != nil { ... }`.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-015: Scheduler `HasTerminatingTargetClusters` Lists All Clusters

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance |
| **Location** | `pkg/scheduler/scheduler.go` |
| **Code Reference** | `HasTerminatingTargetClusters()`, Lines 552-569 |

**Problem Description**: For every binding that reaches the "should we reschedule?" check, `HasTerminatingTargetClusters` lists ALL clusters (`labels.Everything()`) and iterates through them to find terminating ones. With many clusters and bindings, this is O(clusters * bindings).

**Reproduction Steps**: With 100 clusters and 10,000 bindings, each binding reconciliation triggers a full cluster list scan.

**Expected Behavior**: Use an indexed lookup or cache for terminating clusters.

**Actual Behavior**: Full list scan for every binding reconciliation.

**Root Cause Analysis**: Convenience pattern; listing all clusters is simpler than maintaining an index.

**Business Impact**: Increased scheduling latency; higher CPU and memory usage in the scheduler.

**Recommended Fix**: Maintain a set of terminating cluster names in the scheduler cache, updated via cluster informer events.

**Estimated Complexity**: Medium

**GitHub Match**: NONE FOUND

---

## ISSUE-016: `reflect.DeepEqual` Used for Status Comparison in Hot Path

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance |
| **Location** | `pkg/scheduler/scheduler.go` |
| **Code Reference** | `patchBindingStatusCondition()`, Line 1018; `patchClusterBindingStatusCondition()`, Line 1069 |

**Problem Description**: `reflect.DeepEqual` is used to compare binding status objects on every scheduling cycle. `reflect.DeepEqual` is known to be slow due to reflection, especially on complex structs with many fields like `ResourceBindingStatus`.

**Reproduction Steps**: Profile the scheduler under high load. `reflect.DeepEqual` will show up as a significant contributor to CPU time.

**Expected Behavior**: Use `equality.Semantic.DeepEqual` (which is already used elsewhere in the codebase) or compare specific fields.

**Actual Behavior**: Expensive reflection-based comparison on every scheduling cycle.

**Root Cause Analysis**: Convenience of `reflect.DeepEqual` over field-by-field comparison.

**Business Impact**: Increased CPU usage in scheduler; reduced scheduling throughput.

**Recommended Fix**: Replace `reflect.DeepEqual` with `equality.Semantic.DeepEqual` or targeted field comparisons.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-017: Integer Overflow in Node Summary Conversion

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | Bug |
| **Location** | `pkg/controllers/status/cluster_status_controller.go` |
| **Code Reference** | `getNodeSummary()`, Lines 552-553 |

**Problem Description**: The code converts `int` to `int32` with a `#nosec` directive suppressing the gosec warning. While extremely unlikely in practice (>2 billion nodes), this represents an unhandled edge case. The comment acknowledges the risk but doesn't handle it.

**Reproduction Steps**: Theoretical: a cluster with more than 2^31 nodes would cause integer overflow.

**Expected Behavior**: Validation or capping of the value before conversion.

**Actual Behavior**: Silent overflow with `#nosec` suppression.

**Root Cause Analysis**: Accepted technical debt with nosec annotation.

**Business Impact**: Negligible in practice; theoretical data corruption.

**Recommended Fix**: Add a cap check: `if totalNum > math.MaxInt32 { totalNum = math.MaxInt32 }`.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-018: Detector `OnUpdate` Swallows `isClaimedByLazyPolicy` Error

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Bug |
| **Location** | `pkg/detector/detector.go` |
| **Code Reference** | `OnUpdate()`, Lines 345-349 |

**Problem Description**: When `isClaimedByLazyPolicy` returns an error at line 346, it is only logged at Error level. The error is not propagated, and the code proceeds with `isLazyActivation` being `false` (the zero value). This means on error, the system falls through to the non-lazy path, potentially triggering unnecessary reconciliation.

**Reproduction Steps**: Cause `isClaimedByLazyPolicy` to fail (e.g., binding lookup failure). The resource will be processed as non-lazy even if it should be lazy.

**Expected Behavior**: The error should cause a retry (requeue) rather than silent fallthrough.

**Actual Behavior**: Error is swallowed; resource processed incorrectly.

**Root Cause Analysis**: Error handling in event handler callbacks is limited because they don't support returning errors.

**Business Impact**: Resources bound by lazy policies may be processed prematurely, causing unnecessary work distribution.

**Recommended Fix**: Enqueue the item for retry on error rather than proceeding with incorrect state.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-019: Binding Controller `newOverridePolicyFunc` Fetches All Bindings

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Performance |
| **Location** | `pkg/controllers/binding/binding_controller.go` |
| **Code Reference** | `newOverridePolicyFunc()`, Lines 197-254 |

**Problem Description**: When an OverridePolicy or ClusterOverridePolicy changes, the handler lists ALL ResourceBindings (potentially thousands) and for each binding with matching resource selectors, fetches the resource template. This is O(bindings * resource_selectors) API calls triggered by a single policy change.

**Reproduction Steps**: With 5000 ResourceBindings, update a ClusterOverridePolicy. The handler will list all 5000 bindings and potentially fetch 5000 resource templates.

**Expected Behavior**: Use indexers or targeted watches to efficiently identify affected bindings.

**Actual Behavior**: Full scan of all bindings for every policy change.

**Root Cause Analysis**: The handler uses a brute-force approach for matching.

**Business Impact**: Severe API server load spikes on policy changes; potential timeouts.

**Recommended Fix**: Build an index mapping policy resource selectors to binding names, updated incrementally.

**Estimated Complexity**: Large

**GitHub Match**: NONE FOUND

---

## ISSUE-020: Graceful Eviction Uses `reflect.DeepEqual` on Complex Structs

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | Performance |
| **Location** | `pkg/controllers/gracefuleviction/rb_graceful_eviction_controller.go` |
| **Code Reference** | `syncBinding()`, Line 87 |

**Problem Description**: `reflect.DeepEqual` is used to compare `GracefulEvictionTasks` slices. For complex task structures with nested objects, this is unnecessarily expensive.

**Reproduction Steps**: Profile the graceful eviction controller under high churn.

**Expected Behavior**: Use `equality.Semantic.DeepEqual` or length + hash comparison.

**Actual Behavior**: Reflection-based deep comparison on every reconciliation.

**Root Cause Analysis**: Convenience pattern.

**Business Impact**: Minor CPU overhead per reconciliation.

**Recommended Fix**: Use `equality.Semantic.DeepEqual` for consistency with the rest of the codebase.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-021: Missing Makefile Windows Support for Release Builds

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | CI/CD |
| **Location** | `Makefile` |
| **Code Reference** | `release` target, Lines 163-171 |

**Problem Description**: The release target only builds for `linux/amd64`, `linux/arm64`, `darwin/amd64`, and `darwin/arm64`. Windows builds are not included despite `karmadactl` and `kubectl-karmada` being CLI tools that users may want to run on Windows.

**Reproduction Steps**: Run `make release`. No Windows binaries are produced.

**Expected Behavior**: Windows AMD64 and ARM64 binaries should be produced.

**Actual Behavior**: Only Linux and macOS binaries are built.

**Root Cause Analysis**: Windows support not prioritized in the build system.

**Business Impact**: Windows users cannot use release CLI tools directly; limits adoption.

**Recommended Fix**: Add Windows build targets: `@make release-karmadactl GOOS=windows GOARCH=amd64`.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-022: `excludeClusterPolicy` Called Inconsistently in Detector

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Bug |
| **Location** | `pkg/detector/detector.go` |
| **Code Reference** | `ApplyPolicy()`, Line 503; `ApplyClusterPolicy()`, Line 596 |

**Problem Description**: In `ApplyPolicy()` (for PropagationPolicy), `excludeClusterPolicy(bindingCopy)` is called at line 503 to clean up any leftover ClusterPropagationPolicy claims when a namespaced policy takes precedence. However, in `ApplyClusterPolicy()` when creating a ResourceBinding for namespace-scoped resources (line 596), this cleanup is NOT performed. This means if a resource switches from PP to CPP, stale PP claim metadata may persist.

**Reproduction Steps**: 1) Bind a resource to a PropagationPolicy. 2) Delete the PP. 3) Create a ClusterPropagationPolicy matching the same resource. The resource binding may retain stale PP metadata.

**Expected Behavior**: When CPP claims a resource, any stale PP metadata should be cleaned.

**Actual Behavior**: Stale PP claim labels/annotations persist.

**Root Cause Analysis**: Asymmetric cleanup logic between PP and CPP code paths.

**Business Impact**: Resources may appear to be bound by multiple policies; confusing operator experience; potential conflicts.

**Recommended Fix**: Add an `excludeNamespacedPolicy(bindingCopy)` call in the CPP binding update path at line 596.

**Estimated Complexity**: Small

**GitHub Match**: NONE FOUND

---

## ISSUE-023: Scheduler Single Worker Thread for Scheduling

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance |
| **Location** | `pkg/scheduler/scheduler.go` |
| **Code Reference** | `Run()`, Line 327 |

**Problem Description**: The scheduler runs only a single worker thread (`go wait.Until(s.worker, time.Second, ctx.Done())`). All scheduling decisions are serialized through one goroutine. This is a bottleneck when many bindings need to be scheduled simultaneously.

**Reproduction Steps**: Create 1000 bindings simultaneously. Observe that scheduling is serialized, taking O(n * scheduling_time) total.

**Expected Behavior**: Multiple worker goroutines should process the scheduling queue in parallel.

**Actual Behavior**: Single worker processes one binding at a time.

**Root Cause Analysis**: Simplicity choice; single worker avoids concurrency issues in the scheduler cache.

**Business Impact**: Scheduling latency increases linearly with queue depth; delay in workload placement during cluster events.

**Recommended Fix**: Increase worker count to configurable value (e.g., 3-5). Ensure scheduler cache operations are thread-safe (they already use informer-based snapshots).

**Estimated Complexity**: Medium

**GitHub Match**: NONE FOUND

---

## ISSUE-024: Missing Test Coverage for Scheduler Error Paths

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Testing |
| **Location** | `pkg/scheduler/scheduler.go` |
| **Code Reference** | `scheduleResourceBindingWithClusterAffinities()`, Lines 618-681 |

**Problem Description**: The `scheduleResourceBindingWithClusterAffinities()` function has complex error handling with multiple fallback paths (affinity fallback loop, firstErr tracking, fitErr detection, null schedule result patching). The test file `scheduler_test.go` is large (83K) but lacks specific tests for:
- All affinities failing with non-FitError
- Partial affinity failures followed by success
- Patch failures during affinity fallback

**Reproduction Steps**: Review `scheduler_test.go` for coverage of affinity fallback error paths.

**Expected Behavior**: Each error path should have dedicated test cases.

**Actual Behavior**: Complex error paths lack targeted test coverage.

**Root Cause Analysis**: Complex function with many branches; tests focus on happy paths.

**Business Impact**: Regressions in error handling may go undetected; production scheduling failures.

**Recommended Fix**: Add table-driven tests covering all error paths in `scheduleResourceBindingWithClusterAffinities`.

**Estimated Complexity**: Medium

**GitHub Match**: NONE FOUND

---

## ISSUE-025: Documentation — No Architecture Decision Records

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | Documentation |
| **Location** | `docs/` |
| **Code Reference** | N/A |

**Problem Description**: The repository lacks Architecture Decision Records (ADRs). Key design decisions (e.g., why Pull vs Push mode, why single scheduler worker, why wildcard RBAC) are not documented, making it difficult for new contributors and operators to understand trade-offs.

**Reproduction Steps**: Search the docs/ directory for architecture decision documents — none exist.

**Expected Behavior**: Key architectural decisions should be documented with context, decision, and consequences.

**Actual Behavior**: Design rationale is scattered across comments and PRs.

**Root Cause Analysis**: Documentation was not prioritized during rapid feature development.

**Business Impact**: Increased onboarding time for contributors; risk of reversing intentional design decisions; difficulty in security audits.

**Recommended Fix**: Create an `docs/adr/` directory and document key architectural decisions.

**Estimated Complexity**: Medium

**GitHub Match**: NONE FOUND

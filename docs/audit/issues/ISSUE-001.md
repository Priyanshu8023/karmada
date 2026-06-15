# ISSUE-001: Race Condition in Scheduler Assumption Cache Memory Leak

## Summary
The scheduler's assumption cache has a race condition where informer events can arrive before the `Add()` call, causing cache entries to never be cleaned up and leading to a memory leak.

## Technical Explanation

### Data Flow
1. Scheduler patches binding with schedule result → API Server accepts → returns `result`
2. Scheduler calls `s.schedulerCache.AssigningResourceBindings().Add(result)` at line 707
3. Meanwhile, informer receives the update event for the same binding

### Execution Flow (Normal)
```
Scheduler.patch() → API Success → Add(result) → Informer Event → Cache Cleanup ✓
```

### Failure Flow (Race)
```
Scheduler.patch() → API Success → Informer Event (no cache entry to clean) → Add(result) → LEAK
```
The informer event arrives and tries to clean up an entry that doesn't yet exist. When `Add()` is subsequently called, the entry is created but never cleaned. The only cleanup is the TTL-based GC running every 60 seconds.

## Root Cause
The `Add()` call at [scheduler.go:707](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go#L707) occurs **after** the API server patch succeeds, creating a window for the informer to process the event before the cache entry exists.

## Fix Strategy

### Minimal Fix
Move `Add()` before the patch call. On patch failure, call `Remove()`.

### Proper Fix
```go
// Before patch
s.schedulerCache.AssigningResourceBindings().Add(result)

result, err := s.KarmadaClient.WorkV1alpha2().ResourceBindings(newBinding.Namespace).Patch(...)
if err != nil {
    // Rollback assumption
    s.schedulerCache.AssigningResourceBindings().Remove(result)
    return err
}
```

### Enterprise-Grade Fix
Implement a proper assumption lifecycle with states: `Assumed → Confirmed → Released`. Use a reconciliation loop to detect stale assumptions regardless of ordering.

## Code Changes Required
- **File**: [scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go)
- **Lines**: 699-714

## Suggested Patch
```diff
 func (s *Scheduler) patchScheduleResultForResourceBinding(...) error {
     // ... existing code ...
 
+    if features.FeatureGate.Enabled(features.WorkloadAffinity) {
+        s.schedulerCache.AssigningResourceBindings().Add(newBinding)
+    }
+
     result, err := s.KarmadaClient.WorkV1alpha2().ResourceBindings(newBinding.Namespace).Patch(...)
     if err != nil {
+        if features.FeatureGate.Enabled(features.WorkloadAffinity) {
+            s.schedulerCache.AssigningResourceBindings().Remove(newBinding)
+        }
         klog.Errorf("Failed to patch...")
         return err
     }
-    if features.FeatureGate.Enabled(features.WorkloadAffinity) {
-        s.schedulerCache.AssigningResourceBindings().Add(result)
-    }
     // ... rest ...
 }
```

## Testing Strategy

### Unit Tests
- Test that `Add()` is called before patch
- Test that `Remove()` is called on patch failure
- Test that informer events correctly clean up pre-added entries

### Integration Tests
- High-throughput scheduling test with race detector enabled
- Verify assumption cache size remains bounded after sustained load

### Edge Case Tests
- Patch timeout followed by eventual success (server processed but client timed out)
- Concurrent scheduling of bindings to the same cluster

## Risk Assessment
**Low risk**: The change reorders existing operations. The rollback on failure prevents false assumptions. The Kubernetes scheduler uses this same pattern.

## Rollback Plan
Revert the commit. The GC goroutine will still clean up stale entries with TTL.

## Validation Checklist
- [ ] Bug reproduced with race detector
- [ ] Fix implemented
- [ ] Unit tests added for assume-before-patch
- [ ] Integration test with concurrent scheduling
- [ ] CI passed
- [ ] Memory profiling shows stable assumption cache size

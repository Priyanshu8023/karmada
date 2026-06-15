# ISSUE-002: Unbounded Error Retry in Legacy Scheduler Queue

## Summary
The legacy scheduler queue retries failed items indefinitely without a maximum retry count, potentially causing CPU and memory exhaustion.

## Technical Explanation
In [scheduler.go:948-955](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go#L948), `legacyHandleErr` calls `s.queue.AddRateLimited(key)` on any error. Rate limiting slows retries but never stops them. A permanently unschedulable binding will be retried forever.

## Suggested Patch
```diff
+const maxSchedulerRetries = 15
+
 func (s *Scheduler) legacyHandleErr(err error, key any) {
     if err == nil || apierrors.HasStatusCause(err, corev1.NamespaceTerminatingCause) {
         s.queue.Forget(key)
         return
     }
+    if s.queue.NumRequeues(key) >= maxSchedulerRetries {
+        klog.Warningf("Dropping key %v out of the queue after %d retries: %v", key, maxSchedulerRetries, err)
+        s.queue.Forget(key)
+        metrics.CountSchedulerBindings(metrics.ScheduleAttemptFailure)
+        return
+    }
     s.queue.AddRateLimited(key)
     metrics.CountSchedulerBindings(metrics.ScheduleAttemptFailure)
 }
```

## Validation Checklist
- [ ] Bug reproduced with permanently unschedulable binding
- [ ] Fix implemented with max retry limit
- [ ] Test added
- [ ] CI passed

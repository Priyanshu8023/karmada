# ISSUE-023: Scheduler Single Worker Thread

## Summary
The scheduler runs only one worker goroutine, serializing all scheduling decisions and creating a bottleneck.

## Suggested Patch
```diff
-go wait.Until(s.worker, time.Second, ctx.Done())
+const schedulerWorkerCount = 3  // Make configurable via flag
+for i := 0; i < schedulerWorkerCount; i++ {
+    go wait.Until(s.worker, time.Second, ctx.Done())
+}
```

## Risk Assessment
The scheduler cache uses snapshots which are read-only, so concurrent workers are safe for the scheduling algorithm. Patch operations are atomic. The main concern is duplicate scheduling of the same binding, which is prevented by the queue's deduplication.

## Validation Checklist
- [ ] Fix implemented with configurable worker count
- [ ] Race detector testing
- [ ] Benchmarked
- [ ] CI passed

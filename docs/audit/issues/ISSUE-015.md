# ISSUE-015: Scheduler `HasTerminatingTargetClusters` Lists All Clusters

## Summary
Every binding reconciliation calls `clusterLister.List(labels.Everything())` to check for terminating clusters — O(clusters) per binding.

## Suggested Patch
Maintain a set of terminating cluster names in the scheduler, updated via cluster informer events:
```go
// In Scheduler struct
terminatingClusters sync.Map // map[string]struct{}

// Updated via cluster watch
func (s *Scheduler) onClusterUpdate(old, new) {
    if !new.DeletionTimestamp.IsZero() {
        s.terminatingClusters.Store(new.Name, struct{}{})
    }
}
```

## Validation Checklist
- [ ] Fix implemented
- [ ] Benchmark with 100+ clusters
- [ ] CI passed

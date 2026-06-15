# ISSUE-007: Resource Detector Lists All Policies Per Resource Event

## Summary
Every resource reconciliation lists ALL PropagationPolicies in the namespace and ALL ClusterPropagationPolicies, creating O(n*m) API calls at scale.

## Technical Explanation
[detector.go:388](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go#L388) calls `Client.List()` with `UnsafeDisableDeepCopy` but still fetches the full list. With 1000 policies and 1000 resources, this is 1000 list calls per reconciliation cycle.

## Fix Strategy
Build an in-memory index of policies keyed by resource GVK for O(1) lookup:
```go
type policyIndex struct {
    mu      sync.RWMutex
    byGVK   map[schema.GroupVersionKind][]*policyv1alpha1.PropagationPolicy
}
```

Update the index via informer events instead of listing on each reconciliation.

## Validation Checklist
- [ ] Index implemented
- [ ] Benchmarked against list approach
- [ ] Integration tests pass
- [ ] CI passed

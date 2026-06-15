# ISSUE-022: `excludeClusterPolicy` Called Inconsistently in Detector

## Summary
When a PropagationPolicy claims a resource, stale ClusterPropagationPolicy metadata is cleaned up. But when a ClusterPropagationPolicy claims a namespace-scoped resource, stale PropagationPolicy metadata is NOT cleaned up.

## Suggested Patch
Add PP cleanup in the CPP binding update path at [detector.go:596](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go#L596):
```diff
 bindingCopy.Spec.WorkloadAffinityGroups = binding.Spec.WorkloadAffinityGroups
+excludeNamespacedPolicy(bindingCopy)
 return nil
```

## Validation Checklist
- [ ] Fix implemented
- [ ] Test for PP→CPP transition metadata cleanup
- [ ] CI passed

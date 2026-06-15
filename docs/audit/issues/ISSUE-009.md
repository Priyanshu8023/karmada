# ISSUE-009: Missing Error Propagation in Cluster Status Node Summary

## Summary
Errors from `listNodes()`/`listPods()` are logged but not returned, causing zero-value resource summaries to be written.

## Suggested Patch
```diff
 nodes, err := listNodes(clusterInformerManager)
 if err != nil {
     klog.ErrorS(err, "Failed to list nodes for Cluster", "cluster", cluster.GetName())
+    return nil, fmt.Errorf("failed to list nodes: %w", err)
 }
```

## Validation Checklist
- [ ] Fix implemented
- [ ] Test for error propagation added
- [ ] CI passed

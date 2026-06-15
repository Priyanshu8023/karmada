# ISSUE-010: `getAllocatableModelings` Breaks on nil Node Available

## Summary
A `break` statement at line 667 of `cluster_status_controller.go` exits the entire node loop when one node has nil availability, skipping all subsequent nodes.

## Technical Explanation
The `getNodeAvailable()` function returns `nil` when a node has exceeded its pod limit. The calling loop uses `break` instead of `continue`:
```go
for _, node := range nodes {
    nodeAvailable := getNodeAvailable(...)
    if nodeAvailable == nil {
        break  // BUG: should be continue
    }
    modelingSummary.AddToResourceSummary(...)
}
```

## Suggested Patch
```diff
 for _, node := range nodes {
     nodeAvailable := getNodeAvailable(node.Status.Allocatable.DeepCopy(), nodePodResourcesMap[node.Name])
     if nodeAvailable == nil {
-        break
+        continue
     }
     modelingSummary.AddToResourceSummary(modeling.NewClusterResourceNode(nodeAvailable))
 }
```

## Testing Strategy
### Unit Tests
```go
func TestGetAllocatableModelings_SaturatedNodeDoesNotBreakLoop(t *testing.T) {
    // Create nodes where the first node has exceeded pod limit
    // but subsequent nodes have available resources
    // Assert that subsequent nodes are still counted
}
```

## Validation Checklist
- [ ] Bug reproduced with saturated node + available nodes
- [ ] Fix implemented
- [ ] Unit test added
- [ ] CI passed

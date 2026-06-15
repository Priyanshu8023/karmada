# ISSUE-014: Webhook Mutating Handler Nil Placement Panic

## Summary
The mutating webhook accesses `policy.Spec.Placement.SpreadConstraints` without nil-checking `Placement`, causing a nil pointer dereference panic.

## Technical Explanation
At [mutating.go:72](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/webhook/propagationpolicy/mutating.go#L72):
```go
helper.SetDefaultSpreadConstraints(policy.Spec.Placement.SpreadConstraints)
```
If `policy.Spec.Placement` is nil, this panics. The webhook server crashes, causing all admission requests to fail.

## Suggested Patch
```diff
-    helper.SetDefaultSpreadConstraints(policy.Spec.Placement.SpreadConstraints)
+    if policy.Spec.Placement != nil {
+        helper.SetDefaultSpreadConstraints(policy.Spec.Placement.SpreadConstraints)
+    }

-    // When ReplicaSchedulingType is Divided...
-    helper.SetReplicaDivisionPreferenceWeighted(&policy.Spec.Placement)
+    if policy.Spec.Placement != nil {
+        helper.SetReplicaDivisionPreferenceWeighted(&policy.Spec.Placement)
+    }
```

## Testing Strategy
```go
func TestHandle_NilPlacement(t *testing.T) {
    policy := &policyv1alpha1.PropagationPolicy{
        Spec: policyv1alpha1.PropagationSpec{
            // Placement intentionally nil
        },
    }
    // Assert no panic
}
```

## Risk Assessment
**No risk**: Simple nil guard.

## Validation Checklist
- [ ] Panic reproduced with nil placement
- [ ] Fix implemented
- [ ] Test added
- [ ] CI passed

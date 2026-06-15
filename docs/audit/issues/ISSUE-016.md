# ISSUE-016: `reflect.DeepEqual` Used for Status Comparison in Hot Path

## Summary
`reflect.DeepEqual` at scheduler.go:1018 and 1069 is expensive on complex `ResourceBindingStatus` structs.

## Suggested Patch
```diff
-if reflect.DeepEqual(rb.Status, updateRB.Status) {
+if equality.Semantic.DeepEqual(rb.Status, updateRB.Status) {
```

## Validation Checklist
- [ ] Fix implemented
- [ ] Benchmarked
- [ ] CI passed

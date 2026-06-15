# ISSUE-020: Graceful Eviction Uses `reflect.DeepEqual` on Complex Structs

## Summary
`reflect.DeepEqual` used for comparing `GracefulEvictionTasks` slices. Minor performance overhead.

## Suggested Patch
Replace with `equality.Semantic.DeepEqual` for consistency.

## Validation Checklist
- [ ] Fix implemented
- [ ] CI passed

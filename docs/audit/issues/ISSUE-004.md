# ISSUE-004: `context.TODO()` Used Throughout Production Code

## Summary
~50+ usages of `context.TODO()` in production code paths prevent proper context cancellation and timeout enforcement.

## Technical Explanation
Production API calls (patch, list, get) use `context.TODO()` instead of the reconciliation context, making graceful shutdown unreliable and preventing timeout enforcement.

## Suggested Patch
Thread context from reconcile methods through to all API calls. Example:
```diff
-scheduleResult, err := s.Algorithm.Schedule(context.TODO(), &rb.Spec, ...)
+scheduleResult, err := s.Algorithm.Schedule(ctx, &rb.Spec, ...)
```

## Risk Assessment
**Low risk per change**, but **Large scope**: requires touching many files systematically.

## Validation Checklist
- [ ] All `context.TODO()` in pkg/ identified
- [ ] Replaced with appropriate contexts
- [ ] Graceful shutdown tested
- [ ] CI passed

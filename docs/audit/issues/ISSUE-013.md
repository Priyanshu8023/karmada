# ISSUE-013: Dependencies Distributor Channel Not Buffered

## Summary
Unbuffered `genericEvent` channel causes worker blocking under load.

## Suggested Patch
```diff
-d.genericEvent = make(chan event.TypedGenericEvent[*workv1alpha2.ResourceBinding])
+d.genericEvent = make(chan event.TypedGenericEvent[*workv1alpha2.ResourceBinding], 1024)
```

## Validation Checklist
- [ ] Fix implemented
- [ ] Load test with many dependent resources
- [ ] CI passed

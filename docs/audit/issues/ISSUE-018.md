# ISSUE-018: Detector `OnUpdate` Swallows `isClaimedByLazyPolicy` Error

## Summary
Error from `isClaimedByLazyPolicy` is logged but not propagated — falls through to non-lazy path silently.

## Suggested Patch
```diff
 isLazyActivation, err := d.isClaimedByLazyPolicy(unstructuredNewObj)
 if err != nil {
     klog.Errorf("...")
+    // On error, treat as non-lazy but enqueue for retry
+    d.Processor.Enqueue(ResourceItem{Obj: newRuntimeObj})
+    return
 }
```

## Validation Checklist
- [ ] Fix implemented
- [ ] Test for error retry path added
- [ ] CI passed

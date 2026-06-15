# ISSUE-003: `ItemPriorityIfInInitialList` Returns Pointer to Zero

## Summary
`new(LowPriority)` allocates a zero-value `int`, not a value initialized to the `LowPriority` constant (-100).

## Technical Explanation
In Go, `new(T)` allocates memory for type `T` and returns a pointer to its zero value. The constant `LowPriority = -100` is an untyped constant. `new(LowPriority)` is equivalent to `new(int)`, which returns `*int` pointing to `0`.

## Root Cause
Misunderstanding of Go's `new()` semantics. `new()` takes a type, not a value.

## Suggested Patch
```diff
 func ItemPriorityIfInInitialList(isInInitialList bool) *int {
     if !isInInitialList {
         return nil
     }
-    return new(LowPriority)
+    p := LowPriority
+    return &p
 }
```

## Testing Strategy
### Unit Tests
```go
func TestItemPriorityIfInInitialList(t *testing.T) {
    p := ItemPriorityIfInInitialList(true)
    if p == nil || *p != LowPriority {
        t.Errorf("expected %d, got %v", LowPriority, p)
    }
}
```

## Risk Assessment
**No risk**: Simple value correction.

## Validation Checklist
- [ ] Bug reproduced
- [ ] Fix implemented
- [ ] Unit test added
- [ ] CI passed

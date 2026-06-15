# ISSUE-017: Integer Overflow in Node Summary Conversion

## Summary
`int` to `int32` conversion in `getNodeSummary()` could overflow with >2^31 nodes (theoretical).

## Suggested Patch
```diff
+if totalNum > math.MaxInt32 { totalNum = math.MaxInt32 }
 nodeSummary.TotalNum = int32(totalNum)
```

## Validation Checklist
- [ ] Fix implemented
- [ ] CI passed

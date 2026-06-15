# ISSUE-012: Override Manager Does Not Validate JSON Patch Paths

## Summary
PlaintextOverrider allows arbitrary JSON patch paths, enabling modification of security-sensitive fields like `serviceAccountName` and `securityContext`.

## Fix Strategy
Add webhook validation that warns or blocks modifications to a configurable deny-list of sensitive paths:
```go
var sensitivePathPrefixes = []string{
    "/spec/serviceAccountName",
    "/spec/containers/*/securityContext",
    "/metadata/namespace",
    "/spec/nodeName",
}
```

## Validation Checklist
- [ ] Path validation webhook implemented
- [ ] Test for blocked sensitive paths added
- [ ] CI passed

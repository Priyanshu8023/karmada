# ISSUE-021: Missing Makefile Windows Support for Release Builds

## Summary
The `release` target only builds for Linux and macOS. Windows binaries are not produced.

## Suggested Patch
```diff
 release:
+    @make release-karmadactl GOOS=windows GOARCH=amd64
+    @make release-kubectl-karmada GOOS=windows GOARCH=amd64
     @make release-karmadactl GOOS=linux GOARCH=amd64
     ...
```

## Validation Checklist
- [ ] Fix implemented
- [ ] Windows build tested
- [ ] CI passed

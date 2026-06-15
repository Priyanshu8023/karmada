# ISSUE-011: Execution Controller Does Not Handle Partial Failure Idempotently

## Summary
When syncing multiple manifests in a Work object, partial failures cause all manifests to be re-processed on retry.

## Fix Strategy
Track per-manifest applied status in `Work.Status.ManifestStatuses`. On retry, skip manifests that are already successfully applied and unchanged.

## Validation Checklist
- [ ] Per-manifest tracking implemented
- [ ] Test for partial failure retry added
- [ ] CI passed

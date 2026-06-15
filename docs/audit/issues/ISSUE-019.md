# ISSUE-019: Binding Controller `newOverridePolicyFunc` Fetches All Bindings

## Summary
When an override policy changes, the handler lists ALL ResourceBindings and fetches resource templates, causing O(n) API calls.

## Fix Strategy
Build an index mapping resource GVK to binding names. On policy change, look up only affected bindings via the index instead of listing all.

## Validation Checklist
- [ ] Index implemented
- [ ] Benchmarked
- [ ] CI passed

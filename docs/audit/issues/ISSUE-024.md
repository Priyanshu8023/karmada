# ISSUE-024: Missing Test Coverage for Scheduler Error Paths

## Summary
Complex error handling in `scheduleResourceBindingWithClusterAffinities()` lacks targeted test coverage for edge cases.

## Fix Strategy
Add table-driven tests covering:
- All affinities failing with non-FitError
- Partial affinity failures followed by success on subsequent affinity
- Patch failures during affinity fallback
- `firstErr` tracking across multiple affinity attempts

## Validation Checklist
- [ ] Error path tests added
- [ ] Coverage report shows improvement
- [ ] CI passed

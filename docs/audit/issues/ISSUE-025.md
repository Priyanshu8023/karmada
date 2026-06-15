# ISSUE-025: Documentation — No Architecture Decision Records

## Summary
The repository lacks Architecture Decision Records (ADRs) for key design decisions. This makes onboarding difficult and increases the risk of reversing intentional decisions.

## Fix Strategy
Create `docs/adr/` directory with ADRs for:
1. Push vs Pull cluster registration modes
2. Wildcard RBAC in member clusters
3. Single scheduler worker design
4. Resource detector policy matching strategy
5. Override policy JSON patch flexibility

Use the [MADR](https://adr.github.io/madr/) template for consistency.

## Validation Checklist
- [ ] ADR template created
- [ ] At least 3 ADRs documented
- [ ] Linked from CONTRIBUTING.md

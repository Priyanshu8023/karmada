# FEATURE-024: Natural Language CLI (LLM-Powered)

## Feature Overview
Add `karmadactl ask` command that translates natural language queries to Karmada operations.

## CLI Usage Examples
```bash
$ karmadactl ask "show me all deployments in production clusters"
# Translates to: karmadactl get deployments --clusters=prod-us,prod-eu

$ karmadactl ask "propagate the redis deployment to all EU clusters"
# Generates PropagationPolicy and prompts for confirmation

$ karmadactl ask "why was nginx scheduled to member1?"
# Queries SchedulingDecision and presents human-readable explanation
```

## Code Locations
| File | Change |
|:---|:---|
| `pkg/karmadactl/ask/` | **NEW** package |
| `pkg/karmadactl/ask/llm.go` | LLM integration (configurable backend) |
| `pkg/karmadactl/ask/parser.go` | Intent parsing |

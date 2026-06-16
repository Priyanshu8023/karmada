# FEATURE-014: Webhook Admission Retry with Backoff

## Feature Overview
Add configurable retry logic with exponential backoff for webhook-based resource interpreter calls, preventing transient network failures from causing hard rejections.

## Code Locations
| File | Change |
|:---|:---|
| `pkg/resourceinterpreter/customized/webhook/request.go` | Add retry wrapper |
| `pkg/webhook/interpreter/` | Add retry configuration |
| `pkg/util/retry/` | **NEW** — Generic retry helper |

## Acceptance Criteria
- [ ] Configurable max retries (default: 3)
- [ ] Exponential backoff (100ms, 200ms, 400ms)
- [ ] Metrics for retry counts
- [ ] Unit tests

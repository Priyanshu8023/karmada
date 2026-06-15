# ISSUE PRIORITY MATRIX — Karmada Audit

> **P0** → Immediate (fix before next release)  
> **P1** → High (fix within current development cycle)  
> **P2** → Medium (plan for next quarter)  
> **P3** → Low (backlog)

## Priority Table

| Issue | Title | Severity | Business Impact | Effort | Priority |
|---|---|---|---|---|---|
| ISSUE-001 | Race Condition in Scheduler Assumption Cache Memory Leak | Critical | High — Memory leak & incorrect scheduling | Medium | **P0** |
| ISSUE-003 | `ItemPriorityIfInInitialList` Returns Pointer to Zero | Medium | Medium — Broken priority queue ordering | Small | **P0** |
| ISSUE-005 | ClusterRole Grants Wildcard Access | Critical | Critical — Full cluster-admin in all members | Large | **P0** |
| ISSUE-010 | `getAllocatableModelings` Breaks on nil Node Available | Medium | Medium — Inaccurate resource models | Small | **P0** |
| ISSUE-014 | Webhook Mutating Handler Nil Placement Panic | Medium | High — Webhook crash blocks all admissions | Small | **P0** |
| ISSUE-002 | Unbounded Error Retry in Legacy Scheduler Queue | High | High — CPU/memory exhaustion | Small | **P1** |
| ISSUE-006 | gRPC Insecure Skip Verify Option | High | High — MITM attack on scheduling | Small | **P1** |
| ISSUE-009 | Missing Error Propagation in Cluster Status | Medium | Medium — Zero-capacity cluster reporting | Small | **P1** |
| ISSUE-011 | Execution Controller Partial Failure Not Idempotent | High | Medium — Unnecessary API load | Medium | **P1** |
| ISSUE-012 | Override Manager No JSON Patch Path Validation | High | High — Privilege escalation vector | Medium | **P1** |
| ISSUE-013 | Dependencies Distributor Unbuffered Channel | Medium | Medium — Worker starvation | Small | **P1** |
| ISSUE-018 | Detector OnUpdate Swallows Lazy Policy Error | Medium | Medium — Premature resource processing | Small | **P1** |
| ISSUE-022 | excludeClusterPolicy Called Inconsistently | Medium | Medium — Stale policy metadata | Small | **P1** |
| ISSUE-004 | `context.TODO()` Used in Production Code | Medium | Medium — Unreliable graceful shutdown | Large | **P2** |
| ISSUE-007 | Detector Lists All Policies Per Event | High | High — API server overload at scale | Large | **P2** |
| ISSUE-008 | Cluster Status Lists All Pods and Nodes | High | High — Memory pressure at scale | Large | **P2** |
| ISSUE-015 | Scheduler Lists All Clusters Per Binding | Medium | Medium — Increased scheduling latency | Medium | **P2** |
| ISSUE-016 | `reflect.DeepEqual` in Scheduler Hot Path | Medium | Low — CPU overhead | Small | **P2** |
| ISSUE-019 | Binding Controller Lists All Bindings on Policy Change | High | High — API server load spikes | Large | **P2** |
| ISSUE-023 | Scheduler Single Worker Thread | Medium | Medium — Scheduling bottleneck | Medium | **P2** |
| ISSUE-024 | Missing Tests for Scheduler Error Paths | Medium | Medium — Regression risk | Medium | **P2** |
| ISSUE-017 | Integer Overflow in Node Summary | Low | Negligible | Small | **P3** |
| ISSUE-020 | Graceful Eviction reflect.DeepEqual | Low | Low — Minor CPU overhead | Small | **P3** |
| ISSUE-021 | Missing Windows Release Builds | Low | Low — Limited adoption | Small | **P3** |
| ISSUE-025 | No Architecture Decision Records | Low | Low — Contributor onboarding | Medium | **P3** |

## Remediation Roadmap

### Sprint 1 — P0 Critical (Week 1-2)
- [ ] ISSUE-003: Fix `new(LowPriority)` → `ptr.To(LowPriority)` (5 min fix)
- [ ] ISSUE-010: Change `break` to `continue` in allocatable modelings loop (5 min fix)
- [ ] ISSUE-014: Add nil check for `policy.Spec.Placement` in webhook (10 min fix)
- [ ] ISSUE-001: Refactor assumption cache to assume-before-patch pattern (1-2 days)
- [ ] ISSUE-005: Document wildcard RBAC risk; provide scoped-down alternative (2-3 days)

### Sprint 2 — P1 High (Week 3-4)
- [ ] ISSUE-002: Add max retry limit to legacy scheduler queue (30 min)
- [ ] ISSUE-006: Add startup warning for insecure gRPC (30 min)
- [ ] ISSUE-009: Propagate errors from listNodes/listPods (1 hour)
- [ ] ISSUE-013: Buffer the genericEvent channel (10 min)
- [ ] ISSUE-018: Requeue on lazy policy error (30 min)
- [ ] ISSUE-022: Add PP cleanup in CPP path (30 min)
- [ ] ISSUE-011: Track per-manifest applied status (2-3 days)
- [ ] ISSUE-012: Add webhook validation for override paths (2-3 days)

### Sprint 3 — P2 Medium (Next Quarter)
- [ ] ISSUE-004: Replace `context.TODO()` with proper contexts
- [ ] ISSUE-007: Build policy index in detector
- [ ] ISSUE-008: Incremental resource summaries
- [ ] ISSUE-015: Maintain terminating cluster set in cache
- [ ] ISSUE-016: Replace `reflect.DeepEqual` with targeted comparison
- [ ] ISSUE-019: Build override policy → binding index
- [ ] ISSUE-023: Add configurable scheduler worker count
- [ ] ISSUE-024: Add scheduler error path tests

### Backlog — P3 Low
- [ ] ISSUE-017: Cap int32 conversion
- [ ] ISSUE-020: Replace reflect.DeepEqual in graceful eviction
- [ ] ISSUE-021: Add Windows release targets
- [ ] ISSUE-025: Create ADR documentation

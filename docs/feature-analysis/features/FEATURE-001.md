# FEATURE-001: Scheduling Decision Explainability

## Feature Overview
Add a structured explanation mechanism to the Karmada scheduler that records *why* each scheduling decision was made — which plugins ran, which clusters were filtered out and why, how clusters were scored, and which final clusters were selected. Expose this information via API status fields and a CLI command.

## Why This Feature Matters

### User Impact
- **Debugging**: Operators can quickly understand why a workload landed on cluster-X instead of cluster-Y
- **Confidence**: Production teams can validate scheduling behavior before deploying
- **Education**: New users learn how scheduling works through real examples

### Developer Impact
- Plugin authors can see how their custom plugins affect scheduling decisions
- Integration tests can assert on scheduling reasons, not just outcomes

### Project Impact
- Most-requested debugging improvement in multi-cluster orchestration
- Differentiator vs. competitors (OCM has PlacementDecision; Karmada has nothing)

## Functional Requirements

1. **FR-1**: After each scheduling cycle, record a `SchedulingDecision` object containing:
   - List of candidate clusters before filtering
   - Per-plugin filter results (cluster → pass/fail with reason)
   - Per-plugin score results (cluster → score)
   - Final cluster selection with combined scores
   - Timestamp and scheduling duration

2. **FR-2**: Store the scheduling decision in `ResourceBinding.Status.SchedulingDecision` (and `ClusterResourceBinding.Status`)

3. **FR-3**: Provide a CLI command: `karmadactl explain scheduling <binding-name>` that renders the decision in a human-readable table

4. **FR-4**: Optionally, emit scheduling decision as a Kubernetes Event on the binding object

5. **FR-5**: Gated behind a feature flag `SchedulingExplainability` (Alpha, default false)

## Non-Functional Requirements

- **Performance**: Decision recording must add < 5ms overhead per scheduling cycle
- **Storage**: Decisions should be compact; limit to last decision per binding (not historical)
- **Security**: No sensitive data in decisions (no cluster credentials)
- **Scalability**: Must handle 10,000+ bindings without performance degradation

## Technical Design

### Components Affected
1. `pkg/scheduler/core/` — Generic scheduler algorithm
2. `pkg/scheduler/framework/` — Plugin interface and runtime
3. `pkg/apis/work/v1alpha2/` — ResourceBinding/ClusterResourceBinding types
4. `pkg/karmadactl/` — New `explain` subcommand
5. `pkg/features/features.go` — New feature gate

### API Changes

```go
// New type in pkg/apis/work/v1alpha2/types.go
type SchedulingDecision struct {
    // Timestamp when the scheduling decision was made
    Timestamp metav1.Time `json:"timestamp"`
    
    // Duration of the scheduling cycle
    Duration metav1.Duration `json:"duration"`
    
    // CandidateClusters lists all clusters considered
    CandidateClusters []string `json:"candidateClusters"`
    
    // FilterResults shows per-plugin filter outcomes
    FilterResults []PluginFilterResult `json:"filterResults,omitempty"`
    
    // ScoreResults shows per-plugin score outcomes
    ScoreResults []PluginScoreResult `json:"scoreResults,omitempty"`
    
    // SelectedClusters is the final placement
    SelectedClusters []TargetCluster `json:"selectedClusters"`
}

type PluginFilterResult struct {
    PluginName string                     `json:"pluginName"`
    Results    map[string]FilterOutcome   `json:"results"` // cluster -> outcome
}

type FilterOutcome struct {
    Passed bool   `json:"passed"`
    Reason string `json:"reason,omitempty"`
}

type PluginScoreResult struct {
    PluginName string           `json:"pluginName"`
    Scores     map[string]int64 `json:"scores"` // cluster -> score
}
```

### Architecture Flow

```
PropagationPolicy → ResourceBinding → Scheduler
                                         │
                                    ┌────▼────┐
                                    │ Filter  │ ← Record per-plugin results
                                    │ Phase   │
                                    └────┬────┘
                                         │
                                    ┌────▼────┐
                                    │ Score   │ ← Record per-plugin scores
                                    │ Phase   │
                                    └────┬────┘
                                         │
                                    ┌────▼────┐
                                    │ Select  │ ← Record final selection
                                    │ Phase   │
                                    └────┬────┘
                                         │
                              SchedulingDecision stored
                              in binding.Status
```

## Implementation Plan

### Step 1: Add Feature Gate (1 hour)
- Add `SchedulingExplainability` to `pkg/features/features.go`

### Step 2: Define API Types (2 hours)
- Add `SchedulingDecision` and related types to `pkg/apis/work/v1alpha2/types.go`
- Add `SchedulingDecision` field to `ResourceBindingStatus` and `ClusterResourceBindingStatus`
- Regenerate CRDs and deep-copy

### Step 3: Instrument Scheduler Framework (2–3 days)
- Modify `pkg/scheduler/framework/runtime/` to capture filter/score results
- Modify `pkg/scheduler/core/generic_scheduler.go` to aggregate results
- Gate all collection behind the feature flag

### Step 4: Persist to Binding Status (1 day)
- In `pkg/scheduler/scheduler.go`, patch the binding status with the decision after scheduling
- Ensure compact representation (marshal/unmarshal efficient)

### Step 5: CLI Command (1–2 days)
- Create `pkg/karmadactl/explain/scheduling.go`
- Parse `SchedulingDecision` from binding status
- Render as formatted table with color-coded pass/fail

## Code Locations

| File | Change Type |
|:---|:---|
| `pkg/features/features.go` | Add feature gate |
| `pkg/apis/work/v1alpha2/types.go` | Add SchedulingDecision types |
| `pkg/apis/work/v1alpha2/zz_generated.deepcopy.go` | Regenerate |
| `pkg/scheduler/framework/interface.go` | Extend Framework interface |
| `pkg/scheduler/framework/runtime/framework.go` | Capture filter/score results |
| `pkg/scheduler/core/generic_scheduler.go` | Aggregate and return decision |
| `pkg/scheduler/scheduler.go` | Patch binding with decision |
| `pkg/karmadactl/explain/scheduling.go` | **NEW** — CLI command |
| `pkg/karmadactl/karmadactl.go` | Register new command |

## Testing Plan

### Unit Tests
- Test SchedulingDecision serialization/deserialization
- Test filter result collection with mock plugins
- Test score result aggregation
- Test CLI output formatting

### Integration Tests
- Deploy workload with PropagationPolicy → verify SchedulingDecision is populated
- Test with all plugin types (affinity, taint, spread)
- Test decision updates on rescheduling

### Performance Tests
- Benchmark scheduling with/without explainability enabled
- Verify < 5ms overhead per cycle

## Risks

| Risk | Mitigation |
|:---|:---|
| Status field size explosion for large clusters | Limit to last decision; compress reasons |
| Performance overhead from result collection | Feature gate + lazy evaluation |
| API stability for alpha types | Version in API group, mark as experimental |

## Rollback Plan
Disable the `SchedulingExplainability` feature gate. The scheduler reverts to not collecting decisions. Existing decisions in binding status are harmless and will be overwritten on next schedule.

## Acceptance Criteria
- [ ] Feature gate `SchedulingExplainability` added
- [ ] `SchedulingDecision` type defined in Work API
- [ ] Scheduler populates decision for ResourceBinding
- [ ] Scheduler populates decision for ClusterResourceBinding
- [ ] `karmadactl explain scheduling` command works
- [ ] Unit tests added (>80% coverage)
- [ ] Integration tests added
- [ ] Performance benchmark shows < 5ms overhead
- [ ] Documentation updated
- [ ] CI passed

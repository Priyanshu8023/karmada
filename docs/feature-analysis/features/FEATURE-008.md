# FEATURE-008: Dry-Run Scheduling Simulation

## Feature Overview
Allow users to simulate scheduling decisions without actually creating or modifying any resources. The scheduler runs the full scheduling pipeline (filter → score → select) and returns the result as a preview.

## Why This Feature Matters
- CI/CD pipelines need to validate placement before deploying
- Operators want to test policy changes safely
- Debugging scheduling issues requires trial-and-error today

## Functional Requirements
1. **FR-1**: New API endpoint: `POST /apis/scheduling.karmada.io/v1alpha1/schedulingreviews`
2. **FR-2**: Accepts a ResourceBinding spec as input, returns scheduling decision
3. **FR-3**: CLI command: `karmadactl schedule --dry-run -f binding.yaml`
4. **FR-4**: No side effects — no writes to etcd, no events emitted
5. **FR-5**: Returns full explainability data (uses FEATURE-001 types)

## Technical Design

### API
```go
// SchedulingReview is a review request for dry-run scheduling
type SchedulingReview struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   SchedulingReviewSpec   `json:"spec"`
    Status SchedulingReviewStatus `json:"status,omitempty"`
}

type SchedulingReviewSpec struct {
    // Binding is the ResourceBinding spec to evaluate
    Binding workv1alpha2.ResourceBindingSpec `json:"binding"`
}

type SchedulingReviewStatus struct {
    // Decision contains the simulated scheduling result
    Decision SchedulingDecision `json:"decision,omitempty"`
    // Feasible indicates if the workload can be scheduled
    Feasible bool `json:"feasible"`
}
```

### Architecture
```
karmadactl schedule --dry-run
       │
       ▼
POST /schedulingreviews
       │
       ▼
Aggregated API Server
       │
       ▼
Scheduler (read-only mode)
  ├── Filter plugins (no state mutation)
  ├── Score plugins (no state mutation)
  └── Return SchedulingDecision
```

### Code Locations
| File | Change |
|:---|:---|
| `pkg/apis/scheduling/` | **NEW** API group with SchedulingReview |
| `cmd/aggregated-apiserver/` | Register scheduling API |
| `pkg/aggregatedapiserver/` | Add scheduling review handler |
| `pkg/scheduler/core/` | Expose `DryRunSchedule()` method |
| `pkg/karmadactl/schedule/` | **NEW** dry-run command |

## Implementation Plan
1. **Step 1** (2 days): Define SchedulingReview API types
2. **Step 2** (3 days): Implement read-only scheduler execution path
3. **Step 3** (2 days): Add API handler in aggregated-apiserver
4. **Step 4** (2 days): Build `karmadactl schedule --dry-run` CLI
5. **Step 5** (1 day): Tests and documentation

## Acceptance Criteria
- [ ] SchedulingReview API registered and accessible
- [ ] Dry-run returns accurate scheduling decisions
- [ ] Zero side effects confirmed (no writes to etcd)
- [ ] CLI command renders results correctly
- [ ] Integration with FEATURE-001 explainability
- [ ] Tests pass
- [ ] Documentation updated

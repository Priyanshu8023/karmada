# FEATURE-019: CLI Diagnostics (karmadactl doctor)

## Feature Overview
A comprehensive diagnostic command that runs health checks and reports issues.

## Diagnostic Checks
1. Control plane connectivity (API server, scheduler, controller-manager)
2. Cluster health (all registered clusters ready?)
3. Certificate expiry (< 30 days = warning, < 7 days = critical)
4. Version skew (Karmada vs. K8s version compatibility)
5. Pending ResourceBindings (stuck in scheduling?)
6. Work sync failures (any Work objects in error state?)
7. Webhook health (all webhooks responding?)
8. Resource quota utilization (any namespaces near limits?)
9. Feature gate conflicts (incompatible features enabled?)
10. API enablement gaps (CRDs missing on member clusters?)

## Implementation
```go
// pkg/karmadactl/doctor/doctor.go
type HealthCheck struct {
    Name     string
    Run      func(ctx context.Context) CheckResult
    Category string // "control-plane", "cluster", "security", "scheduling"
}

type CheckResult struct {
    Status  Status // Healthy, Warning, Critical, Error
    Message string
    Details string
}
```

## Code Locations
| File | Change |
|:---|:---|
| `pkg/karmadactl/doctor/` | **NEW** package |
| `pkg/karmadactl/doctor/doctor.go` | Check runner |
| `pkg/karmadactl/doctor/checks/` | Individual check implementations |
| `pkg/karmadactl/karmadactl.go` | Register `doctor` command |

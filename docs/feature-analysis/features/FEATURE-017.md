# FEATURE-017: Configuration Drift Detection

## Feature Overview
Detect when resources in member clusters deviate from their intended state in the Karmada control plane. Report drift and optionally auto-remediate.

## Technical Design
- Compare Work object manifests with actual resources in member clusters
- New field `Work.Status.DriftDetected` (bool) + `DriftDetails` (diff)
- Controller periodically polls or uses informers to detect changes
- Configurable actions: `Report`, `AutoFix`, `Ignore`

## Code Locations
| File | Change |
|:---|:---|
| `pkg/controllers/execution/drift_detector.go` | **NEW** |
| `pkg/apis/work/v1alpha2/types.go` | Add DriftStatus fields |
| `pkg/karmadactl/drift/` | **NEW** `karmadactl drift` command |

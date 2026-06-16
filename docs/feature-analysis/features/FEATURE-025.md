# FEATURE-025: Multi-Cluster Compliance Scanning

## Feature Overview
Automated compliance scanning across all member clusters using CIS Kubernetes Benchmark and Pod Security Standards.

## Design
- New `ComplianceScan` CRD triggers fleet-wide scans
- Results aggregated into `ComplianceReport` with per-cluster findings
- Remediation suggestions linked to FederatedPolicy (FEATURE-004)

## Code Locations
| File | Change |
|:---|:---|
| `pkg/apis/compliance/v1alpha1/` | **NEW** API group |
| `pkg/controllers/compliance/` | **NEW** scanner controller |
| `pkg/karmadactl/compliance/` | **NEW** CLI commands |

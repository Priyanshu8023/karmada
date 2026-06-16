# FEATURE-002: GitOps-Native Distribution (ArgoCD/Flux Integration)

## Feature Overview
Provide first-class integration between Karmada and GitOps tools (ArgoCD, Flux CD). This includes an ApplicationSet generator for ArgoCD, a Flux provider for Karmada, and a `karmadactl gitops` subcommand for bootstrapping.

## Why This Feature Matters
- GitOps is the #1 deployment pattern for Kubernetes (>70% adoption in production)
- No native Karmada integration means users spend days on custom glue code
- Rancher Fleet and OCM+ArgoCD have this — Karmada is losing users to competitors

## Functional Requirements
1. **FR-1**: ArgoCD ApplicationSet Generator plugin that reads Karmada clusters and generates per-cluster Applications
2. **FR-2**: Custom ArgoCD Resource Health check for Karmada CRDs (PropagationPolicy, ResourceBinding, Work)
3. **FR-3**: Flux Kustomization source that targets Karmada API server
4. **FR-4**: `karmadactl gitops bootstrap --provider argocd` command
5. **FR-5**: Drift detection webhook that creates Karmada Events when ArgoCD detects sync drift

## Technical Design

### ArgoCD Integration Architecture
```
Git Repository
     │
     ▼
ArgoCD (Hub Cluster)
     │
     ├── ApplicationSet Generator (Karmada Plugin)
     │     │
     │     ▼
     │   Karmada API ──► List Clusters
     │     │
     │     ▼
     │   Generate Application per cluster
     │
     ▼
Karmada Control Plane
     │
     ▼
PropagationPolicy + Resources
     │
     ▼
Member Clusters
```

### New Files
- `pkg/karmadactl/gitops/` — **NEW** package
- `pkg/karmadactl/gitops/bootstrap.go` — Bootstrap command
- `pkg/karmadactl/gitops/argocd.go` — ArgoCD-specific logic
- `examples/gitops/argocd/` — Example ApplicationSet configs
- `examples/gitops/flux/` — Example Flux configs
- `hack/deploy-argocd-integration.sh` — **NEW** deployment script

### Code Locations
| File | Change |
|:---|:---|
| `pkg/karmadactl/karmadactl.go` | Register `gitops` command group |
| `pkg/karmadactl/gitops/bootstrap.go` | **NEW** |
| `pkg/karmadactl/gitops/argocd.go` | **NEW** |
| `examples/gitops/` | **NEW** directory |
| `charts/karmada/templates/` | Optional ArgoCD resource health ConfigMap |

## Implementation Plan
1. **Week 1**: Develop ArgoCD ApplicationSet generator plugin
2. **Week 2**: Build custom health checks for Karmada CRDs in ArgoCD
3. **Week 3**: Create `karmadactl gitops bootstrap` command
4. **Week 4**: Build Flux provider and example configurations
5. **Week 5**: Documentation, examples, and E2E tests
6. **Week 6**: Community review and hardening

## Acceptance Criteria
- [ ] ArgoCD ApplicationSet generator generates apps from Karmada clusters
- [ ] ArgoCD shows health status for Karmada CRDs
- [ ] `karmadactl gitops bootstrap --provider argocd` works end-to-end
- [ ] Flux Kustomization can target Karmada API server
- [ ] Example configurations provided
- [ ] E2E tests pass
- [ ] Documentation published

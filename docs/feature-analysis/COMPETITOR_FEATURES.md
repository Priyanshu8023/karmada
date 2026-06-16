# Karmada Competitor Features Analysis

> Comparing Karmada against OCM, Liqo, Clusternet, KubeAdmiral, Rancher/Fleet, and Admiralty.

## Competitor Matrix

| Feature Area | Karmada | OCM (RHACM) | Liqo | Clusternet | KubeAdmiral | Rancher/Fleet |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| Multi-Cluster Orchestration | ✅ Strong | ✅ Strong | ⚠️ Limited | ✅ Good | ✅ Good | ✅ Good |
| Web Dashboard | ⚠️ Separate | ✅ Built-in | ❌ None | ❌ None | ❌ None | ✅ Built-in |
| GitOps-Native | ❌ None | ✅ ArgoCD | ❌ None | ❌ None | ❌ None | ✅ Fleet |
| Cluster Lifecycle Mgmt | ❌ None | ✅ Strong | ❌ None | ❌ None | ❌ None | ✅ RKE2 |
| Virtual Node Offloading | ❌ None | ❌ None | ✅ Core | ❌ None | ❌ None | ❌ None |
| Policy-as-Code | ❌ None | ✅ Strong | ❌ None | ❌ None | ❌ None | ⚠️ Basic |
| Audit Logging | ❌ None | ✅ Strong | ❌ None | ❌ None | ❌ None | ✅ Strong |
| Cost Awareness | ❌ None | ⚠️ Plugin | ❌ None | ❌ None | ❌ None | ❌ None |
| Observability Stack | ⚠️ Basic | ✅ Integrated | ❌ None | ❌ None | ❌ None | ✅ Integrated |

## Critical Gaps Identified

| # | Feature | Available In | Impact |
|:---|:---|:---|:---|
| 1 | Integrated Web Dashboard | OCM, Rancher | Very High |
| 2 | GitOps-Native Distribution | Fleet, ArgoCD+OCM | Very High |
| 3 | Policy-as-Code Governance | OCM | High |
| 4 | Scheduling Decision Explainability | OCM | High |
| 5 | Cluster Lifecycle Management | Rancher, OCM | High |
| 6 | Audit Logging | OCM, Rancher | High |
| 7 | Cost-Aware Scheduling | Admiralty, OpenCost | Medium |
| 8 | Cross-Cluster Service Mesh | Liqo | Medium |
| 9 | ArgoCD/FluxCD Integration | OCM, Fleet | High |
| 10 | Cluster Grouping/Sets API | OCM | Medium |

See [FEATURE_ANALYSIS.md](FEATURE_ANALYSIS.md) for full details.

# Architectural Impact Analysis: Karmada Proposed Features

This analysis evaluates all 25 proposed features from the roadmap to determine their architectural footprint. Implementing features with low/no core architectural changes allows rapid delivery with minimal regression risks.

---

## Summary Matrix

| Impact Tier | Count | Description | Key Examples |
| :--- | :---: | :--- | :--- |
| **Zero Core Impact** | 8 | External integrations, static configuration assets, or client-side CLI tools. | `karmadactl doctor`, Terraform Provider, Grafana Dashboards |
| **Low/Minimal Impact** | 10 | Localized helper controllers, client-side wrappers, standard scheduler plugins (using existing hooks), or telemetry. | Cost-Aware Scheduler Plugin, Webhook Retry, Distributed Tracing |
| **Moderate/High Impact** | 7 | Modifications to core sync engines, scheduling lifecycles, API schema boundaries, or control-plane agents. | Federation Rolling Updates, Dry-Run Simulation, Multi-Tenant RBAC |

---

## 1. Zero Core Architectural Impact (No Control Plane Changes)
These features require **no changes** to Karmada's control plane running binaries (`karmada-apiserver`, `karmada-scheduler`, `karmada-controller-manager`). They are implemented entirely as client-side tools, external integrations, or static configuration files.

| Feature ID | Feature Name | Footprint Location | Rationale |
| :--- | :--- | :--- | :--- |
| **[FEATURE-002](features/FEATURE-002.md)** | GitOps-Native Distribution | `examples/`, `charts/` | ArgoCD/Flux manifests and config templates. Done entirely via existing APIs. |
| **[FEATURE-006](features/FEATURE-006.md)** | Cluster Health Dashboard | `charts/` | Addition of static Grafana Dashboard JSON templates into the Helm chart. |
| **[FEATURE-007](features/FEATURE-007.md)** | CLI Interactive Wizard | `pkg/karmadactl/` | Purely interactive CLI wrapper (`karmadactl wizard`) using terminal prompts. |
| **[FEATURE-010](features/FEATURE-010.md)** | Grafana Dashboard Pack | `charts/` | Static dashboard JSON templates using existing Prometheus metrics. |
| **[FEATURE-019](features/FEATURE-019.md)** | CLI Diagnostics (`karmadactl doctor`) | `pkg/karmadactl/` | CLI-only tool that queries existing APIs (healthz, list clusters, list works) and runs checks. |
| **[FEATURE-020](features/FEATURE-020.md)** | Terraform Provider for Karmada | External Repo | External Terraform provider codebase interacting with existing Karmada APIs. |
| **[FEATURE-022](features/FEATURE-022.md)** | Multi-Cluster Chaos Engineering | `pkg/karmadactl/` | CLI wrappers that trigger cluster cordons or simulate network disruptions using standard API commands. |
| **[FEATURE-024](features/FEATURE-024.md)** | Natural Language CLI (LLM-Powered) | `pkg/karmadactl/` | CLI-only command (`karmadactl ask`) translating speech/text to standard CLI calls. |

---

## 2. Low/Minimal Architectural Impact
These features run inside the control plane but rely **strictly on existing extension points** or operate as **fully isolated standalone controllers**. They do not modify the core scheduling loop, synchronization loop, or storage model of Karmada.

| Feature ID | Feature Name | Footprint Location | Rationale |
| :--- | :--- | :--- | :--- |
| **[FEATURE-003](features/FEATURE-003.md)** | Cost-Aware Multi-Cluster Scheduling | `pkg/scheduler/framework/plugins/` | Uses the scheduler framework's existing `Score` phase extension point. The scheduler engine itself is unmodified. |
| **[FEATURE-009](features/FEATURE-009.md)** | Cross-Cluster Event Aggregation | `pkg/controllers/eventcollector/` | A self-contained controller that watches member cluster events via standard informers and copies them to the hub. |
| **[FEATURE-012](features/FEATURE-012.md)** | Federated Secret Management & Rotation | `pkg/controllers/secretrotation/` | Independent controller that updates control-plane Secrets (which are then synced by standard policies). |
| **[FEATURE-013](features/FEATURE-013.md)** | OpenTelemetry Distributed Tracing | Cross-cutting | Code-level instrumentation using OTel SDK. Enriches observability without changing execution flow. |
| **[FEATURE-014](features/FEATURE-014.md)** | Webhook Admission Retry with Backoff | `pkg/resourceinterpreter/` | Localized client-side HTTP wrapper inside webhook interpreter calls to handle transient failures. |
| **[FEATURE-017](features/FEATURE-017.md)** | Configuration Drift Detection | `pkg/controllers/execution/` | Scanning controller that periodically runs deep diffs between cached Work objects and cluster resources. |
| **[FEATURE-018](features/FEATURE-018.md)** | Resource Inventory & Dependency Graph | `pkg/controllers/inventory/` | Read-only aggregator controller that processes ResourceBindings and outputs to a new API resource. |
| **[FEATURE-021](features/FEATURE-021.md)** | AI-Powered Scheduling Advisor | Sidecar Daemon | An external daemon (`karmada-advisor`) that queries metrics and writes advice custom objects. |
| **[FEATURE-025](features/FEATURE-025.md)** | Multi-Cluster Compliance Scanning | `pkg/controllers/compliance/` | Independent worker controller that schedules benchmark scans on member clusters. |

---

## 3. Moderate to High Architectural Impact
These features require modifying the **core engine components** of Karmada. They either alter scheduling pipelines, inject custom handlers into the aggregated API server, change the state machines of ResourceBindings/Work, or adjust system security/tenant scopes.

| Feature ID | Feature Name | Footprint Location | Rationale |
| :--- | :--- | :--- | :--- |
| **[FEATURE-001](features/FEATURE-001.md)** | Scheduling Decision Explainability | `pkg/scheduler/core/`, `pkg/apis/` | Requires extending the scheduling framework runtime interfaces to collect metrics during Filter/Score phases, and patching binding status fields. |
| **[FEATURE-004](features/FEATURE-004.md)** | Policy-as-Code Governance | `pkg/controllers/federatedpolicy/`, Webhooks | Requires custom validating admission webhooks at the hub level and policy translation engines. |
| **[FEATURE-005](features/FEATURE-005.md)** | Comprehensive Audit Logging | `cmd/karmada-apiserver` | Requires configuring K8s audit policy filters and routing webhooks for the aggregated API server. |
| **[FEATURE-008](features/FEATURE-008.md)** | Dry-Run Scheduling Simulation | `pkg/scheduler/core/`, API | Requires exposing the scheduler's internal pipeline as a read-only endpoint in the aggregated API server. |
| **[FEATURE-011](features/FEATURE-011.md)** | Multi-Tenant RBAC Propagation | `pkg/controllers/unifiedauth/` | Deep modification to how credentials and namespaces are mapped and partitioned across clusters. |
| **[FEATURE-015](features/FEATURE-015.md)** | Federation-Level Rolling Updates | `pkg/controllers/`, API | High footprint. Requires a new rollout controller orchestrating the progressive deployment of OverridePolicies. |
| **[FEATURE-016](features/FEATURE-016.md)** | Cluster Grouping & Sets API | `pkg/scheduler/`, `pkg/apis/` | Integrates with the scheduler's cluster affinity evaluation logic to resolve groups dynamically. |
| **[FEATURE-023](features/FEATURE-023.md)** | Predictive Cross-Cluster Autoscaling | `pkg/controllers/federatedhpa/` | Requires refactoring the `FederatedHPA` reconcile loop to invoke predictive metrics engines instead of simple threshold checks. |

---

## Recommended First Implementation Path (Quick-Wins & Zero Core Risk)
To establish quick momentum and high visual impact with **zero core risk**:
1. **[FEATURE-019](features/FEATURE-019.md) - `karmadactl doctor`**: High value command for operators; self-contained CLI change.
2. **[FEATURE-010](features/FEATURE-010.md) & [FEATURE-006](features/FEATURE-006.md) - Grafana Dashboard Packs**: High-impact visual telemetry changes built purely via Helm chart configs.
3. **[FEATURE-007](features/FEATURE-007.md) - `karmadactl wizard`**: Terminal-based user onboarding command.
4. **[FEATURE-014](features/FEATURE-014.md) - Webhook Retry**: Localized reliability improvement with zero side-effects.

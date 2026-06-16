# FEATURE-006: Cluster Health Dashboard & Grafana Integration

## Feature Overview
Ship pre-built Grafana dashboards as JSON files and a ConfigMap in the Helm chart. Dashboards cover: control plane health, scheduler performance, cluster fleet overview, resource utilization, and failover events.

## Technical Design

### Dashboards
1. **Karmada Control Plane Overview** — API server latency, controller-manager queue depth, scheduler throughput
2. **Fleet Health** — Cluster readiness, node counts, resource utilization per cluster
3. **Scheduling Analytics** — Scheduling duration histogram, filter/score plugin latency, bindings per second
4. **Failover Monitor** — Failover events, graceful evictions, migration latency
5. **Resource Distribution** — Resources per cluster, work sync latency, error rates

### Code Locations
| File | Change |
|:---|:---|
| `charts/karmada/dashboards/` | **NEW** — 5 JSON dashboard files |
| `charts/karmada/templates/grafana-dashboards-configmap.yaml` | **NEW** |
| `charts/karmada/values.yaml` | Add `grafana.dashboards.enabled` |

## Acceptance Criteria
- [ ] 5 Grafana dashboards created
- [ ] ConfigMap template in Helm chart
- [ ] Dashboards work with existing metrics
- [ ] Screenshots in documentation

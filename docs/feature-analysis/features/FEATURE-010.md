# FEATURE-010: Pre-Built Grafana Dashboard Pack

## Feature Overview
A collection of Grafana dashboard JSON files that visualize all Karmada Prometheus metrics out-of-the-box. Distributed as part of the Helm chart.

*See FEATURE-006 for full dashboard context — this feature is the subset focused on dashboard JSON creation.*

### Metrics Covered
From `pkg/metrics/`:
- `resource_match_policy_duration_seconds`
- `resource_apply_policy_duration_seconds`
- `policy_apply_attempts_total`
- `binding_sync_work_duration_seconds`
- `work_sync_workload_duration_seconds`
- `create_resource_to_cluster`
- `update_resource_to_cluster`
- `delete_resource_from_cluster`

From `pkg/scheduler/metrics/`:
- `karmada_scheduler_binding_schedule_duration_seconds`
- `karmada_scheduler_queue_duration_seconds`

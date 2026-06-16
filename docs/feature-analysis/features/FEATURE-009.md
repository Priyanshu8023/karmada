# FEATURE-009: Cross-Cluster Event Aggregation

## Feature Overview
Add a controller that watches Kubernetes Events in member clusters and aggregates them into the Karmada control plane, enriched with cluster name and binding context.

## Technical Design
- Leverages existing `genericmanager.MultiClusterInformerManager` to watch Events across clusters
- Filters for relevant events (Pods, Deployments related to Karmada Work objects)
- Creates aggregated Events in the Karmada control plane namespace

### Code Locations
| File | Change |
|:---|:---|
| `pkg/controllers/eventcollector/` | **NEW** controller |
| `pkg/controllers/eventcollector/event_collector_controller.go` | **NEW** |

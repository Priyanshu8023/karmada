# FEATURE-018: Resource Inventory & Dependency Graph

## Feature Overview
Maintain a centralized inventory of all resources propagated by Karmada, their locations, dependencies, and health status.

## Architecture & API
- New `ResourceInventory` type computed from ResourceBindings and Work objects
- Dependency tracking via owner references and `PropagateDeps` annotations
- API endpoint: `GET /apis/inventory.karmada.io/v1alpha1/resources`
- CLI: `karmadactl inventory list` — tabular view of all propagated resources

## Code Locations
| File | Change |
|:---|:---|
| `pkg/apis/inventory/v1alpha1/` | **NEW** API group |
| `pkg/controllers/inventory/` | **NEW** controller |
| `pkg/karmadactl/inventory/` | **NEW** CLI command |

# FEATURE-016: Cluster Grouping & Sets API

## Feature Overview
Introduce a `ClusterSet` CRD for logical grouping of clusters. PropagationPolicy can target ClusterSets instead of enumerating clusters.

### New CRD
```yaml
apiVersion: cluster.karmada.io/v1alpha1
kind: ClusterSet
metadata:
  name: production-us
spec:
  clusterSelector:
    labelSelector:
      matchLabels:
        env: production
        region: us
  maxClusters: 10
status:
  matchedClusters: [us-east-1, us-west-2, us-central-1]
  readyClusters: 3
```

### Usage in PropagationPolicy
```yaml
spec:
  placement:
    clusterSets:
      - production-us
      - production-eu
```

## Code Locations
| File | Change |
|:---|:---|
| `pkg/apis/cluster/v1alpha1/types_clusterset.go` | **NEW** |
| `pkg/controllers/clusterset/` | **NEW** controller |
| `pkg/scheduler/framework/plugins/clusteraffinity/` | Support ClusterSet |

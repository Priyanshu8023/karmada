# FEATURE-015: Federation-Level Rolling Updates

## Feature Overview
Introduce a `FederatedRollout` CRD that orchestrates staged rollouts across cluster groups (canary → staging → production) with automatic pause on failure.

## Technical Design

### New CRD
```yaml
apiVersion: apps.karmada.io/v1alpha1
kind: FederatedRollout
metadata:
  name: nginx-rollout
spec:
  resourceRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  strategy:
    stages:
      - name: canary
        clusters:
          clusterNames: [canary-cluster]
        pause:
          duration: 300s  # Wait 5 min, check health
      - name: staging
        clusters:
          labelSelector:
            matchLabels:
              env: staging
        pause:
          manual: true  # Manual approval required
      - name: production
        clusters:
          labelSelector:
            matchLabels:
              env: production
    healthCheck:
      successThreshold: 3
      failureThreshold: 1
      interval: 30s
```

### Controller Flow
1. Watch FederatedRollout objects
2. For each stage, update the OverridePolicy to apply the new version to stage clusters
3. Monitor health checks via member cluster status
4. On success: proceed to next stage. On failure: pause and alert
5. Support rollback: revert OverridePolicy for all stages

## Code Locations
| File | Change |
|:---|:---|
| `pkg/apis/apps/v1alpha1/types_rollout.go` | **NEW** FederatedRollout CRD |
| `pkg/controllers/federatedrollout/` | **NEW** controller |
| `pkg/webhook/federatedrollout/` | **NEW** validation webhook |

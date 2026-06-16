# FEATURE-012: Federated Secret Management & Rotation

## Feature Overview
Add automated secret rotation for secrets propagated via Karmada. Integrates with external secret stores (Vault, AWS Secrets Manager) and ensures zero-downtime rotation across clusters.

## Technical Design

### Annotation-Based Rotation
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  annotations:
    secret.karmada.io/rotation-interval: "720h"  # 30 days
    secret.karmada.io/rotation-strategy: "rolling" # rolling | blue-green
    secret.karmada.io/external-source: "vault://secret/data/db"
```

### Controller Flow
1. Watch Secrets with rotation annotations
2. On rotation interval, fetch new secret from external source
3. Update secret in Karmada control plane
4. PropagationPolicy distributes to member clusters
5. Rolling strategy: update clusters one at a time with health checks

### Code Locations
| File | Change |
|:---|:---|
| `pkg/controllers/secretrotation/` | **NEW** controller |
| `pkg/util/externalsecret/` | **NEW** external secret providers |
| `pkg/util/externalsecret/vault.go` | **NEW** Vault integration |

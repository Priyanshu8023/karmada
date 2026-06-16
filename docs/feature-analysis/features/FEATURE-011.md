# FEATURE-011: Multi-Tenant RBAC Propagation Enhancement

## Feature Overview
Extend the `unifiedauth` controller with multi-tenancy primitives: tenant-scoped namespaces, hierarchical RBAC propagation, and cluster-scoped access control.

## Technical Design

### New CRD
```yaml
apiVersion: auth.karmada.io/v1alpha1
kind: FederatedTenant
metadata:
  name: team-alpha
spec:
  namespaces:
    - team-alpha-prod
    - team-alpha-staging
  clusterAccess:
    clusterAffinity:
      labelSelector:
        matchLabels:
          team: alpha
  roles:
    - name: developer
      rules:
        - apiGroups: ["apps"]
          resources: ["deployments"]
          verbs: ["get", "list", "create", "update"]
    - name: viewer
      rules:
        - apiGroups: [""]
          resources: ["pods", "services"]
          verbs: ["get", "list"]
```

### Code Locations
| File | Change |
|:---|:---|
| `pkg/apis/auth/v1alpha1/` | **NEW** API group |
| `pkg/controllers/tenant/` | **NEW** controller |
| `pkg/controllers/unifiedauth/` | Extend existing controller |

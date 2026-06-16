# FEATURE-004: Policy-as-Code Governance Framework

## Feature Overview
Introduce a `FederatedPolicy` CRD that defines compliance rules enforced across all member clusters. Rules can use CEL expressions and target specific resource types, namespaces, or cluster groups.

## Technical Design

### New CRD
```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: FederatedPolicy
metadata:
  name: require-resource-limits
spec:
  targetClusters:
    clusterAffinity:
      clusterNames: ["*"]
  rules:
    - name: containers-must-have-limits
      match:
        resources:
          - apiVersion: apps/v1
            kind: Deployment
      validate:
        cel:
          expressions:
            - "object.spec.template.spec.containers.all(c, has(c.resources.limits))"
          message: "All containers must define resource limits"
      enforcement: Deny  # Deny | Warn | Audit
```

### Architecture
- New controller: `pkg/controllers/federatedpolicy/`
- Generates per-cluster ValidatingAdmissionPolicy or Kyverno ClusterPolicy
- Reports compliance status back to `FederatedPolicy.Status`

### Code Locations
| File | Change |
|:---|:---|
| `pkg/apis/policy/v1alpha1/types_federatedpolicy.go` | **NEW** CRD types |
| `pkg/controllers/federatedpolicy/` | **NEW** controller |
| `pkg/webhook/federatedpolicy/` | **NEW** webhook |

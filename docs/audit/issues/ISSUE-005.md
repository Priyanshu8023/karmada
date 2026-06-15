# ISSUE-005: Security — ClusterRole Grants Wildcard Access to All Resources

## Summary
The Karmada service account in member clusters receives cluster-admin equivalent privileges through wildcard RBAC rules, violating the principle of least privilege.

## Technical Explanation
The `ClusterPolicyRules` in [credential.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/util/credential.go) grants:
- `Verbs: ["*"]` — all operations including delete
- `APIGroups: ["*"]` — all API groups including RBAC, secrets
- `Resources: ["*"]` — all resources including nodes, PVs

This means if the Karmada control plane is compromised, the attacker has full admin access to **every** member cluster.

## Root Cause
Convenience design: Karmada needs to manage arbitrary resource types (user-defined CRDs), making it difficult to predefine required permissions.

## Fix Strategy

### Minimal Fix
Document the security risk prominently in installation guides. Provide a "hardened" ClusterRole template for security-conscious deployments.

### Proper Fix
Implement dynamic RBAC: when a PropagationPolicy references specific GVKs, update the member cluster ClusterRole to include only those GVKs.

### Enterprise-Grade Fix
Implement a permission proxy in the Karmada agent that validates operations against a policy before executing them on the member cluster.

## Risk Assessment
**Medium risk** for the proper fix: changing RBAC rules may break existing deployments that propagate resources not covered by initial policies.

## Validation Checklist
- [ ] Security risk documented
- [ ] Hardened RBAC template provided
- [ ] Tested with common propagation scenarios
- [ ] CI passed

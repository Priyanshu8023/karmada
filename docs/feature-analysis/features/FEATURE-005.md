# FEATURE-005: Comprehensive Audit Logging

## Feature Overview
Add a structured audit logging framework to the Karmada control plane that records all significant operations with contextual metadata, enabling compliance, forensics, and operational analytics.

## Why This Feature Matters
- **Compliance**: SOC2, ISO 27001, HIPAA all require audit trails
- **Security**: Detect unauthorized changes to propagation policies
- **Operations**: Understand who changed what, when, and with what effect

## Functional Requirements
1. **FR-1**: Audit events for all Karmada CRD CRUD operations
2. **FR-2**: Enriched events including: user identity, operation, resource, cluster(s) affected, old/new values (configurable)
3. **FR-3**: Configurable audit policy (log levels: None, Metadata, Request, RequestResponse)
4. **FR-4**: Multiple sinks: file, webhook, stdout
5. **FR-5**: `karmadactl audit` CLI to query local audit logs

## Technical Design

### Audit Event Structure
```go
type AuditEvent struct {
    Timestamp       time.Time              `json:"timestamp"`
    AuditID         string                 `json:"auditID"`
    Level           AuditLevel             `json:"level"`
    Stage           AuditStage             `json:"stage"` // RequestReceived, ResponseComplete
    Verb            string                 `json:"verb"`  // create, update, delete, patch
    User            UserInfo               `json:"user"`
    Resource        ResourceRef            `json:"resource"`
    AffectedClusters []string             `json:"affectedClusters,omitempty"`
    RequestObject   *runtime.RawExtension  `json:"requestObject,omitempty"`
    ResponseStatus  int                    `json:"responseStatus"`
    Annotations     map[string]string      `json:"annotations,omitempty"`
}
```

### Architecture
```
karmada-apiserver
     │ (K8s audit policy)
     ▼
Audit Webhook ──► karmada-audit-sink (new component)
                       │
                  ┌────┴────┐
                  │ File    │ Structured JSON
                  │ Webhook │ Forward to SIEM
                  │ Stdout  │ Container logs
                  └─────────┘
```

### Code Locations
| File | Change |
|:---|:---|
| `cmd/karmada-audit/` | **NEW** audit sink component |
| `pkg/audit/` | **NEW** audit framework |
| `pkg/audit/policy.go` | Audit policy configuration |
| `pkg/audit/sink/` | File, webhook, stdout sinks |
| `pkg/karmadactl/audit/` | **NEW** CLI query command |
| `charts/karmada/templates/audit/` | **NEW** Helm templates |
| `hack/deploy-karmada.sh` | Add audit webhook config |

## Implementation Plan
1. **Step 1** (2 days): Define audit event schema and policy config
2. **Step 2** (3 days): Build audit sink component with file/stdout backends
3. **Step 3** (2 days): Configure K8s audit policy for karmada-apiserver
4. **Step 4** (2 days): Add webhook sink for SIEM integration
5. **Step 5** (1 day): Build `karmadactl audit` CLI
6. **Step 6** (2 days): Helm chart integration and documentation

## Acceptance Criteria
- [ ] Audit events generated for all CRD operations
- [ ] Configurable audit levels
- [ ] File sink produces valid structured JSON
- [ ] Webhook sink forwards events
- [ ] `karmadactl audit` displays recent events
- [ ] Helm chart supports audit configuration
- [ ] Unit and integration tests pass
- [ ] Documentation published

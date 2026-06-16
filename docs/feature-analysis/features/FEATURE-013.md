# FEATURE-013: OpenTelemetry Distributed Tracing

## Feature Overview
Instrument Karmada control plane components with OpenTelemetry spans for end-to-end tracing of the resource propagation pipeline.

## Trace Spans Flow
```
[PropagationPolicy Created]
  └─ [Detector: Match Policy to Resource] (pkg/detector/)
      └─ [Detector: Create ResourceBinding]
          └─ [Scheduler: Schedule Binding] (pkg/scheduler/)
              ├─ [Scheduler: Filter Phase]
              │   ├─ [Plugin: ClusterAffinity]
              │   ├─ [Plugin: TaintToleration]
              │   └─ [Plugin: SpreadConstraint]
              ├─ [Scheduler: Score Phase]
              │   ├─ [Plugin: ClusterLocality]
              │   └─ [Plugin: TaintToleration]
              └─ [Scheduler: Patch Binding]
                  └─ [Binding Controller: Sync Work] (pkg/controllers/binding/)
                      └─ [Execution Controller: Apply to Cluster] (pkg/controllers/execution/)
```

## Code Locations
| File | Change |
|:---|:---|
| `pkg/util/tracing/` | **NEW** — OTel initialization helpers |
| `pkg/scheduler/scheduler.go` | Add span around `doSchedule` |
| `pkg/scheduler/core/generic_scheduler.go` | Add spans for filter/score |
| `pkg/controllers/binding/binding_controller.go` | Add span for sync work |
| `pkg/controllers/execution/execution_controller.go` | Add span for member cluster apply |
| `pkg/detector/` | Add span for policy matching |
| `cmd/*/app.go` | Initialize OTel provider |
| `go.mod` | Add `go.opentelemetry.io/otel` |

## Acceptance Criteria
- [ ] End-to-end traces visible in Jaeger/Zipkin
- [ ] Context propagation via binding annotations
- [ ] Configurable exporter (OTLP, Jaeger, stdout)
- [ ] < 2% performance overhead

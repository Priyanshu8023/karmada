# FEATURE-003: Cost-Aware Multi-Cluster Scheduling

## Feature Overview
Add a cost-awareness dimension to the Karmada scheduler. Clusters are annotated with cost profiles ($/vCPU-hour, $/GiB-hour), and a new `CostOptimization` scoring plugin factors cost into placement decisions.

## Technical Design

### New Scheduler Plugin
```go
// pkg/scheduler/framework/plugins/costoptimization/cost_optimization.go
type CostOptimization struct{}

func (c *CostOptimization) Name() string { return "CostOptimization" }

func (c *CostOptimization) Score(ctx context.Context, spec *workv1alpha2.ResourceBindingSpec, cluster *clusterv1alpha1.Cluster) (int64, *framework.Result) {
    costPerCPU := getCostAnnotation(cluster, "scheduling.karmada.io/cost-per-vcpu-hour")
    costPerMem := getCostAnnotation(cluster, "scheduling.karmada.io/cost-per-gib-hour")
    // Lower cost = higher score
    totalCost := computeWorkloadCost(spec, costPerCPU, costPerMem)
    score := inverseCostScore(totalCost)
    return score, framework.NewResult(framework.Success)
}
```

### Cluster Annotations
```yaml
apiVersion: cluster.karmada.io/v1alpha1
kind: Cluster
metadata:
  name: aws-us-east-1
  annotations:
    scheduling.karmada.io/cost-per-vcpu-hour: "0.0416"
    scheduling.karmada.io/cost-per-gib-hour: "0.0052"
    scheduling.karmada.io/spot-available: "true"
```

### Code Locations
| File | Change |
|:---|:---|
| `pkg/scheduler/framework/plugins/costoptimization/` | **NEW** plugin |
| `pkg/scheduler/framework/plugins/registry.go` | Register plugin |
| `pkg/util/constants.go` | Add cost annotation keys |

## Acceptance Criteria
- [ ] Cost scoring plugin implemented
- [ ] Clusters with lower cost get higher scores
- [ ] Annotation-based cost configuration works
- [ ] Unit tests with mock cluster costs
- [ ] Documentation with examples

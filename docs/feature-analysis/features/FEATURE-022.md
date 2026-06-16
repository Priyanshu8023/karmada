# FEATURE-022: Multi-Cluster Chaos Engineering

## Feature Overview
Built-in chaos testing for multi-cluster scenarios, integrated with `karmadactl`.

## CLI Command Interface
```bash
# Simulate cluster unreachable
karmadactl chaos inject cluster-failure --cluster member1 --duration 5m

# Simulate network latency between control plane and member cluster
karmadactl chaos inject network-delay --cluster member2 --latency 500ms

# Simulate resource pressure on member cluster
karmadactl chaos inject resource-pressure --cluster member1 --cpu 90%

# Verify failover behavior
karmadactl chaos verify failover --deployment nginx --timeout 2m
```

## Code Locations
| File | Change |
|:---|:---|
| `pkg/karmadactl/chaos/` | **NEW** package |
| `pkg/karmadactl/chaos/inject.go` | Injection commands |
| `pkg/karmadactl/chaos/verify.go` | Verification commands |

# ISSUE-008: Cluster Status Controller Lists All Pods and Nodes

## Summary
Full pod/node lists from member clusters are loaded into Karmada control plane memory for resource modeling, causing memory pressure with large clusters.

## Fix Strategy
- **Short-term**: Use node/pod informer caches (already in place via `typedmanager`). Ensure caches are not re-listed unnecessarily.
- **Long-term**: Have the scheduler estimator (which runs per-cluster) compute and report resource summaries, avoiding control plane memory pressure.

## Validation Checklist
- [ ] Memory profiling with large cluster simulation
- [ ] Fix implemented
- [ ] CI passed

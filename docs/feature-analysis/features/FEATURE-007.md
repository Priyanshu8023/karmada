# FEATURE-007: CLI Interactive Mode (karmadactl wizard)

## Feature Overview
Add interactive wizards to `karmadactl` for common operations that are complex to express via flags. Uses the `survey` library for terminal prompts.

## Implementation Details
```go
// pkg/karmadactl/wizard/propagation.go
func RunPropagationWizard(f util.Factory, streams genericclioptions.IOStreams) error {
    // Step 1: Select resource type
    resourceType := promptSelect("Select resource type", []string{"Deployment", "Service", "ConfigMap", "Custom..."})
    
    // Step 2: Select resource
    resourceName := promptInput("Resource name")
    
    // Step 3: Select target clusters
    clusters := promptMultiSelect("Select target clusters", listClusters())
    
    // Step 4: Choose scheduling type
    schedType := promptSelect("Scheduling type", []string{"Duplicated", "Divided"})
    
    // Step 5: Generate and apply
    policy := generatePropagationPolicy(resourceType, resourceName, clusters, schedType)
    return applyPolicy(f, policy)
}
```

### Code Locations
| File | Change |
|:---|:---|
| `pkg/karmadactl/wizard/` | **NEW** package |
| `pkg/karmadactl/wizard/propagation.go` | **NEW** |
| `pkg/karmadactl/wizard/join.go` | **NEW** |
| `pkg/karmadactl/wizard/override.go` | **NEW** |
| `pkg/karmadactl/karmadactl.go` | Register `wizard` command |
| `go.mod` | Add `github.com/AlecAivazis/survey/v2` |

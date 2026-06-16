# FEATURE-020: Terraform Provider for Karmada

## Feature Overview
A Terraform provider for managing Karmada resources (PropagationPolicy, OverridePolicy, ClusterSet, Cluster) declaratively.

## Usage Example
```hcl
resource "karmada_propagation_policy" "nginx" {
  metadata {
    name      = "nginx-propagation"
    namespace = "default"
  }
  spec {
    resource_selectors {
      api_version = "apps/v1"
      kind        = "Deployment"
      name        = "nginx"
    }
    placement {
      cluster_affinity {
        cluster_names = ["member1", "member2"]
      }
    }
  }
}
```

## Repository & Dependency
- Separate repository: `github.com/karmada-io/terraform-provider-karmada`
- Uses Karmada Go client library

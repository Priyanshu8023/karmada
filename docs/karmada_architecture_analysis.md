# Karmada — Complete Architecture & Reverse-Engineering Analysis

---

## Phase 1: High-Level Understanding

### What Problem Does Karmada Solve?

Karmada (**K**ubernetes **Armada**) solves the **multi-cluster Kubernetes management** problem. In production, organizations run dozens of Kubernetes clusters across multiple clouds (AWS, GCP, Azure), regions, and on-premises data centers. Managing workloads, policies, and configurations across all of these clusters individually is unsustainable. Karmada provides a **single unified control plane** that distributes and manages Kubernetes resources across multiple member clusters, abstracting away the complexity of multi-cluster operations.

### Target Users

| User Type | Use Case |
|-----------|----------|
| Platform Engineers | Build internal multi-cloud / multi-cluster platforms |
| SREs / DevOps | Manage workloads across clusters for HA / DR |
| Cloud Architects | Design hybrid-cloud / multi-cloud infrastructure |
| Enterprise IT | Run compliant workloads across geographic regions |
| Open-Source Contributors | Extend multi-cluster Kubernetes capabilities |

### Main Features

1. **Multi-cluster resource propagation** — Deploy Kubernetes resources across clusters declaratively
2. **Policy-based scheduling** — Control which clusters receive workloads via placement rules
3. **Override policies** — Customize resources per-cluster (image registries, replicas, etc.)
4. **Federated HPA** — Horizontal Pod Autoscaler across multiple clusters
5. **Multi-cluster service discovery** — Service mesh across clusters (MCS API)
6. **Cluster failover** — Automatically migrate workloads from failed clusters
7. **Graceful eviction** — Controlled workload migration with zero downtime
8. **Federated resource quotas** — Enforce resource limits across the federation
9. **Resource interpreter framework** — Understand custom resource structures (CRDs)
10. **CLI tools** (`karmadactl`, `kubectl-karmada`) — Manage the federation from command line
11. **Operator-based installation** — Deploy Karmada itself via a Kubernetes operator

### Business Purpose

Enable organizations to operate **multi-cloud / multi-cluster Kubernetes** with a single control plane, achieving high availability, disaster recovery, geographic compliance, and vendor independence.

### Type of Application

**Distributed Systems Infrastructure Platform** — A collection of Kubernetes controllers, custom API servers, schedulers, webhooks, CLI tools, and agents that together form a multi-cluster management control plane. Written in **Go**, following Kubernetes controller patterns.

### Executive Summary

> Karmada is a CNCF incubating project that provides a Kubernetes-native multi-cluster management system. It extends the familiar Kubernetes API with custom resources (PropagationPolicy, OverridePolicy, ResourceBinding, etc.) to enable users to deploy and manage workloads across multiple Kubernetes clusters from a single control plane. The system follows a hub-and-spoke architecture where a central Karmada control plane (built on its own API server, controller manager, scheduler, and webhook server) detects user-created resources, matches them to propagation policies, schedules them to target clusters, and ensures their lifecycle is managed through a rich set of controllers. Member clusters are connected via either a "Push" mode (control plane pushes directly) or a "Pull" mode (agent running in member cluster pulls Work objects), making it suitable for diverse network topologies. The project is battle-tested by enterprises including Huawei, VIPKID, and others running production workloads at scale.

### Simple Language Explanation (For a College Student)

Imagine you have 5 different Kubernetes clusters — one in AWS US-East, one in GCP Europe, one in Azure Asia, and two on-premises. Right now, if you want to deploy a web app to all 5, you'd need to run `kubectl apply` five separate times, manage configurations separately, handle failures manually, and have no central view.

**Karmada is like a "meta-Kubernetes"** that sits on top of all your clusters. You interact with Karmada just like you would with a normal Kubernetes cluster — you create Deployments, Services, ConfigMaps, etc. But you also create a `PropagationPolicy` that says *"put this Deployment on clusters with label `region=europe` and `region=asia`"*. Karmada's **detector** sees your Deployment, its **scheduler** picks the right clusters, and its **controller** creates the actual resources in those member clusters. If a cluster goes down, Karmada can automatically move your workloads elsewhere. It's like a traffic controller for your entire multi-cloud Kubernetes fleet.

---

## Phase 2: Repository Structure Analysis

### Top-Level Directory Map

| Path | Purpose | Important Files | Notes |
|------|---------|-----------------|-------|
| `cmd/` | Binary entry points for all components | `controller-manager/`, `scheduler/`, `webhook/`, `agent/`, `karmadactl/` | Each subdirectory produces one executable |
| `pkg/` | Core business logic packages | `detector/`, `scheduler/`, `controllers/`, `webhook/`, `resourceinterpreter/` | Heart of the system |
| `pkg/apis/` | Custom Resource Definitions (API types) | `policy/`, `work/`, `cluster/`, `config/`, `search/` | Define Karmada's CRD schema |
| `pkg/controllers/` | Kubernetes controllers | 21 controller packages | React to CRD changes |
| `pkg/scheduler/` | Multi-cluster scheduler | `scheduler.go`, `core/`, `framework/`, `cache/` | Decides which clusters get workloads |
| `pkg/detector/` | Resource detection engine | `detector.go`, `policy.go`, `preemption.go` | Watches resources, matches policies |
| `pkg/webhook/` | Admission webhooks | 17 webhook packages | Validates/mutates CRD objects |
| `pkg/resourceinterpreter/` | Resource structure interpreter | `interpreter.go`, `default/`, `customized/` | Understands CRD structures |
| `pkg/karmadactl/` | CLI tool logic | 30+ command packages | `join`, `unjoin`, `promote`, `init`, etc. |
| `pkg/util/` | Shared utilities | `helper/`, `fedinformer/`, `objectwatcher/`, `overridemanager/` | Cross-cutting concerns |
| `pkg/generated/` | Auto-generated client code | `clientset/`, `informers/`, `listers/` | Generated via `code-generator` |
| `api/` | OpenAPI specifications | `openapi-spec/` | API documentation |
| `artifacts/` | Deployment manifests | `deploy/`, `agent/`, `example/` | YAML for installing Karmada |
| `charts/` | Helm charts | `karmada/`, `karmada-operator/` | Helm-based installation |
| `operator/` | Karmada Operator | `cmd/`, `pkg/`, `config/` | Install Karmada via operator pattern |
| `hack/` | Build/dev scripts | Various shell scripts | CI/CD, code generation |
| `test/` | Integration & E2E tests | `e2e/`, `helper/` | Ginkgo-based test suite |
| `examples/` | Example resources | Sample YAML files | Usage examples |
| `samples/` | Sample applications | Example workloads | Demo applications |
| `third_party/` | Vendored third-party code | Forked/copied packages | Modified upstream code |
| `vendor/` | Go module vendor directory | All dependencies | `go mod vendor` output |
| `docs/` | Documentation | Feature analysis, guides | Project documentation |

### Folder Interactions & Dependency Explanation

```
cmd/* ──imports──> pkg/*
                    │
    ┌───────────────┼───────────────────────┐
    │               │                       │
pkg/detector   pkg/scheduler         pkg/controllers/*
    │               │                       │
    └───────────────┼───────────────────────┘
                    │
              pkg/apis/* (CRD types)
                    │
              pkg/util/* (shared utilities)
                    │
              pkg/resourceinterpreter (resource understanding)
```

**What breaks if removed:**

| Folder Removed | Impact |
|---------------|--------|
| `pkg/detector/` | ❌ No resource detection — policies never matched to resources |
| `pkg/scheduler/` | ❌ No scheduling — resources never assigned to clusters |
| `pkg/controllers/execution/` | ❌ Work objects never dispatched to member clusters |
| `pkg/controllers/status/` | ❌ No status collection from member clusters |
| `pkg/apis/` | ❌ No CRD types — entire system collapses |
| `pkg/webhook/` | ⚠️ No validation — invalid resources accepted |
| `pkg/resourceinterpreter/` | ⚠️ Can't understand CRD replica counts, dependencies |
| `cmd/` | ❌ No executables — nothing runs |

---

## Phase 3: Entry Point Discovery

### Component Entry Points

Karmada consists of **11 separate binaries**, each with its own `main()`:

| Component | Entry Point | Purpose |
|-----------|-------------|---------|
| **karmada-controller-manager** | [controller-manager.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/controller-manager/controller-manager.go) | Core controller hub |
| **karmada-scheduler** | [main.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/scheduler/main.go) | Multi-cluster scheduler |
| **karmada-webhook** | [main.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/webhook/main.go) | Admission webhooks |
| **karmada-agent** | [main.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/agent/main.go) | Member cluster agent (Pull mode) |
| **karmada-aggregated-apiserver** | `cmd/aggregated-apiserver/` | Extended API server |
| **karmada-search** | `cmd/karmada-search/` | Cross-cluster resource search |
| **karmada-descheduler** | `cmd/descheduler/` | Workload descheduling |
| **karmada-scheduler-estimator** | `cmd/scheduler-estimator/` | Replica estimation per cluster |
| **karmada-metrics-adapter** | `cmd/metrics-adapter/` | Custom metrics aggregation |
| **karmadactl** | `cmd/karmadactl/` | CLI tool |
| **kubectl-karmada** | `cmd/kubectl-karmada/` | kubectl plugin |

### Startup Sequence (Controller Manager — Primary Component)

```
User Starts Binary
        │
        ▼
main() in controller-manager.go
        │
        ├── controllerruntime.SetupSignalHandler() ─── Create context with OS signal handling
        │
        ├── app.NewControllerManagerCommand(ctx) ─── Build cobra CLI command
        │       │
        │       ├── Parse flags (--kubeconfig, --controllers, etc.)
        │       ├── Validate options
        │       └── Call Run(ctx, opts)
        │
        ├── Run(ctx, opts)
        │       │
        │       ├── Get REST config from kubeconfig
        │       ├── Create controller-runtime Manager
        │       │       ├── Set leader election
        │       │       ├── Configure cache sync period
        │       │       ├── Set health probes
        │       │       └── Register metrics
        │       │
        │       ├── setupControllers(ctx, mgr, opts)
        │       │       │
        │       │       ├── Create dynamic/discovery/kube client sets
        │       │       ├── Create OverrideManager
        │       │       ├── Create SkippedResourceConfig
        │       │       ├── Create ControlPlaneInformerManager
        │       │       ├── Create ResourceInterpreter ← START
        │       │       ├── Create ResourceDetector ← ADD to manager
        │       │       ├── Create DependenciesDistributor (if PropagateDeps enabled)
        │       │       ├── Create ObjectWatcher
        │       │       │
        │       │       └── controllers.StartControllers()
        │       │               ├── startClusterController
        │       │               ├── startClusterStatusController
        │       │               ├── startBindingController
        │       │               ├── startExecutionController
        │       │               ├── startWorkStatusController
        │       │               ├── startNamespaceController
        │       │               ├── ... (27 controllers total)
        │       │               └── startClusterTaintPolicyController
        │       │
        │       └── controllerManager.Start(ctx) ─── Block until context cancelled
        │
        └── logs.FlushLogs() + os.Exit()
```

### Environment & Configuration Loading

1. **Kubeconfig**: Loaded via `controller-runtime`'s `GetConfig()` (reads `--kubeconfig` flag or `KUBECONFIG` env var)
2. **Feature Gates**: Parsed from `--feature-gates` flag, registered in [features.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/features/features.go)
3. **Rate Limiters**: Configured via `--rate-limiter-*` flags
4. **Leader Election**: Via `--leader-elect*` flags using Kubernetes lease resources
5. **Skipped Resources**: Via `--skipped-propagating-apis` and `--skipped-propagating-namespaces`

---

## Phase 4: Architecture Analysis

### Architecture Pattern: **Event-Driven Controller Architecture** (Operator Pattern on Steroids)

Karmada follows a **hub-and-spoke architecture** built entirely on the **Kubernetes controller/operator pattern**. It is NOT a traditional MVC, microservice, or layered architecture. Instead, it's an **event-driven reconciliation system** where:

- CRD objects represent desired state
- Controllers watch for changes and reconcile toward desired state
- The Kubernetes API server acts as both database and message bus

### Why This Pattern?

1. **Kubernetes-native**: Users already know Kubernetes APIs — Karmada extends them
2. **Declarative**: Describe *what* you want, not *how* to do it
3. **Eventually consistent**: Controllers continuously reconcile, handling failures naturally
4. **Extensible**: New CRDs and controllers can be added without changing existing ones

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      KARMADA CONTROL PLANE                          │
│                                                                     │
│  ┌───────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  karmada-      │    │  karmada-         │    │  karmada-        │  │
│  │  apiserver     │◄──►│  controller-      │    │  scheduler       │  │
│  │  (etcd-backed) │    │  manager          │    │                  │  │
│  └───────┬───────┘    │  ┌──────────────┐ │    │ ┌──────────────┐ │  │
│          │            │  │ Detector     │ │    │ │ Generic      │ │  │
│          │            │  │ (Resource    │ │    │ │ Scheduler    │ │  │
│          │            │  │  Watcher)    │ │    │ │ (Filter →    │ │  │
│          │            │  ├──────────────┤ │    │ │  Score →     │ │  │
│          │            │  │ 27+          │ │    │ │  Select →    │ │  │
│          │            │  │ Controllers  │ │    │ │  Assign)     │ │  │
│          │            │  ├──────────────┤ │    │ └──────────────┘ │  │
│          │            │  │ Override     │ │    └──────────────────┘  │
│          │            │  │ Manager      │ │                          │
│          │            │  ├──────────────┤ │    ┌──────────────────┐  │
│          │            │  │ Resource     │ │    │  karmada-webhook  │  │
│          │            │  │ Interpreter  │ │    │  (Validation &   │  │
│          │            │  └──────────────┘ │    │   Mutation)       │  │
│          │            └──────────────────┘    └──────────────────┘  │
│          │                                                          │
└──────────┼──────────────────────────────────────────────────────────┘
           │
     ┌─────┴──────────────────────────────────────────┐
     │            CLUSTER CONNECTIONS                   │
     │                                                  │
     │   PUSH MODE              PULL MODE               │
     │   (Direct API access)    (Agent-based)           │
     │                                                  │
     ▼                          ▼                       │
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐│
│ Member   │  │ Member   │  │ Member   │  │ Member   ││
│ Cluster  │  │ Cluster  │  │ Cluster  │  │ Cluster  ││
│ 1 (Push) │  │ 2 (Push) │  │ 3 (Pull) │  │ 4 (Pull) ││
│          │  │          │  │ ┌──────┐ │  │ ┌──────┐ ││
│          │  │          │  │ │Agent │ │  │ │Agent │ ││
│          │  │          │  │ └──────┘ │  │ └──────┘ ││
└──────────┘  └──────────┘  └──────────┘  └──────────┘│
     └─────────────────────────────────────────────────┘
```

### Resource Flow Pipeline

```
User creates                           Karmada Internal Pipeline
Deployment     ──────►  Detector   ──────►  PropagationPolicy Match
                            │
                            ▼
                     ResourceBinding    ──────►  Scheduler
                     (or ClusterRB)              │
                                                 ▼
                                          TargetCluster Selection
                                                 │
                                                 ▼
                                      BindingController ──────► Work Objects
                                                                  │
                                         ┌────────────────────────┤
                                         ▼                        ▼
                                   ExecutionController      Agent (Pull)
                                   (Push mode)               │
                                         │                    │
                                         ▼                    ▼
                                   Member Cluster        Member Cluster
                                   API Server            API Server
```

---

## Phase 5: Request Lifecycle — Deploying a Deployment to Multiple Clusters

Let's trace the complete flow of deploying an Nginx Deployment to 3 clusters.

### Step 1: User Creates Resources

```yaml
# 1. User creates a Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
---
# 2. User creates a PropagationPolicy
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-policy
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames: [member1, member2, member3]
```

### Step 2: Resource Detection

**File**: [pkg/detector/detector.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go)

1. `ResourceDetector.discoverResources()` (L187) periodically discovers API resources
2. Informers watch all discovered resources
3. `OnAdd()` (L311) fires when the Deployment is created
4. `Reconcile()` (L231) is called with the Deployment's key
5. `propagateResource()` calls `LookForMatchedPolicy()` (L381) — finds `nginx-policy`
6. `ApplyPolicy()` (L441) is called

### Step 3: ResourceBinding Creation

**File**: [pkg/detector/detector.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go#L441-L526)

1. `ClaimPolicyForObject()` (L762) labels the Deployment with policy metadata
2. `BuildResourceBinding()` creates a `ResourceBinding` object containing:
   - Resource reference (the Deployment)
   - Placement rules (from the PropagationPolicy)
   - Replica requirements (from ResourceInterpreter)
3. `controllerutil.CreateOrUpdate()` persists the ResourceBinding

### Step 4: Scheduling

**File**: [pkg/scheduler/scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go#L384-L469)

1. Scheduler's informer detects the new ResourceBinding
2. `doScheduleBinding()` (L395) checks if scheduling is needed
3. `scheduleResourceBinding()` (L571) calls `Algorithm.Schedule()`
4. **File**: [pkg/scheduler/core/generic_scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/core/generic_scheduler.go#L71-L116)
   - `findClustersThatFit()` — runs filter plugins (ClusterAffinity, TaintToleration, APIEnablement, etc.)
   - `prioritizeClusters()` — runs score plugins (ClusterLocality)
   - `selectClusters()` — applies spread constraints
   - `assignReplicas()` — distribute replicas across clusters
5. Result is patched to the ResourceBinding's `spec.clusters` field

### Step 5: Work Creation (Binding Controller)

**File**: [pkg/controllers/binding/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/controllers/binding)

1. `ResourceBindingController` watches ResourceBindings
2. When `spec.clusters` is set, it creates `Work` objects in execution namespaces
3. For each target cluster (e.g., `member1`):
   - Creates namespace `karmada-es-member1` (execution space)
   - Creates a `Work` object containing the Deployment manifest
   - Applies override policies (image changes, replica adjustments, etc.)

### Step 6: Execution (Push Mode)

**File**: [pkg/controllers/execution/execution_controller.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/controllers/execution/execution_controller.go)

1. `ExecutionController` watches `Work` objects
2. Uses `ObjectWatcher` to create/update the actual Deployment in the member cluster
3. Connects to member cluster's API server using stored credentials

### Step 7: Status Collection

**File**: [pkg/controllers/status/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/controllers/status)

1. `WorkStatusController` monitors resources in member clusters via informers
2. Collects status and writes it back to the `Work` object's status
3. `RBStatusController` / `CRBStatusController` aggregates status from all `Work` objects back to the `ResourceBinding`

---

## Phase 6: Database Analysis

### Database Used

Karmada does **NOT use a traditional database**. It uses **etcd** (via the Kubernetes API server) as its data store — the same pattern as vanilla Kubernetes.

```
┌────────────────────┐
│   Karmada API      │
│   Server           │
│   (kube-apiserver) │
│         │          │
│         ▼          │
│      etcd          │
│   (key-value DB)   │
└────────────────────┘
```

### ORM Used: **None** — Kubernetes uses a custom client library (`client-go`) with informers/listers as the data access layer.

### Data Model (Custom Resource Definitions)

The following CRDs form Karmada's "schema":

```
┌─────────────────────────────────────────────────────────┐
│                    POLICY LAYER                          │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │ PropagationPolicy   │  │ ClusterPropagationPolicy │  │
│  │ (Namespaced)        │  │ (Cluster-scoped)         │  │
│  └─────────────────────┘  └──────────────────────────┘  │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │ OverridePolicy      │  │ ClusterOverridePolicy    │  │
│  │ (Namespaced)        │  │ (Cluster-scoped)         │  │
│  └─────────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                  BINDING LAYER                           │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │ ResourceBinding     │  │ ClusterResourceBinding   │  │
│  │ (Namespaced)        │  │ (Cluster-scoped)         │  │
│  └─────────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                   WORK LAYER                             │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Work                                            │    │
│  │ (Namespaced — in execution space per cluster)   │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│               CLUSTER LAYER                              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Cluster                                         │    │
│  │ (Cluster-scoped — represents member clusters)   │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Entity-Relationship Diagram

```
PropagationPolicy ──selects──► Resource Template (Deployment, Service, etc.)
       │                              │
       │ creates                      │ owned by
       ▼                              ▼
ResourceBinding ◄─────────────── Resource Template
       │
       │ references
       ▼
    Cluster(s) ◄── scheduling result
       │
       │ creates Work per cluster
       ▼
     Work ──contains──► Resource Manifest
       │
       │ applied to
       ▼
  Member Cluster Resource (actual Deployment)
```

### Key CRD Types

| CRD | API Group | File | Purpose |
|-----|-----------|------|---------|
| `PropagationPolicy` | `policy.karmada.io/v1alpha1` | [propagation_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/policy/v1alpha1/propagation_types.go) | Namespace-scoped propagation rules |
| `ClusterPropagationPolicy` | `policy.karmada.io/v1alpha1` | Same file | Cluster-scoped propagation rules |
| `OverridePolicy` | `policy.karmada.io/v1alpha1` | [override_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/policy/v1alpha1/override_types.go) | Per-cluster resource overrides |
| `ResourceBinding` | `work.karmada.io/v1alpha2` | [binding_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/work/v1alpha2/binding_types.go) | Links resource to target clusters |
| `Work` | `work.karmada.io/v1alpha1` | `pkg/apis/work/v1alpha1/` | Carries resource manifest to member cluster |
| `Cluster` | `cluster.karmada.io/v1alpha1` | [types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/cluster/types.go) | Represents a registered member cluster |
| `FederatedResourceQuota` | `policy.karmada.io/v1alpha1` | [federatedresourcequota_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/policy/v1alpha1/federatedresourcequota_types.go) | Multi-cluster resource quotas |
| `ResourceInterpreterCustomization` | `config.karmada.io/v1alpha1` | `pkg/apis/config/v1alpha1/` | Custom resource interpreters |

---

## Phase 7: Component Dependency Graph

```
                    cmd/* (Entry Points)
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
  cmd/controller-   cmd/scheduler    cmd/agent
  manager/app           /app            /app
         │               │               │
         ▼               ▼               ▼
    ┌────┴────┐     pkg/scheduler   (mirrors controller-
    │ Core    │          │           manager for Pull)
    │ Setup   │          │
    │         │          ▼
    ▼         ▼    pkg/scheduler/core
pkg/detector  pkg/scheduler/framework
    │              │
    ▼              ▼
pkg/controllers/* ◄─────── pkg/resourceinterpreter
    │                          │
    ├── binding                │
    ├── execution              ▼
    ├── status            pkg/resourceinterpreter/
    ├── cluster           ├── default/native
    ├── namespace         ├── default/thirdparty
    ├── mcs               ├── customized/webhook
    ├── gracefuleviction  └── customized/declarative
    ├── applicationfailover
    ├── federatedhpa
    ├── ...
    │
    ▼
pkg/util/* (Shared Utilities)
    ├── helper/
    ├── fedinformer/ ← informer management
    ├── objectwatcher/ ← resource sync to member clusters
    ├── overridemanager/ ← apply override policies
    ├── names/ ← naming conventions
    ├── restmapper/ ← GVR ↔ GVK mapping
    ├── gclient/ ← generic client factory
    └── worker.go ← async work queue
         │
         ▼
    pkg/apis/* (CRD Type Definitions)
    ├── policy/v1alpha1/
    ├── work/v1alpha1/ & v1alpha2/
    ├── cluster/v1alpha1/
    ├── config/v1alpha1/
    └── ...
         │
         ▼
    pkg/generated/* (Auto-generated)
    ├── clientset/
    ├── informers/
    └── listers/
```

### Module Classification

| Category | Modules |
|----------|---------|
| **Core Modules** | `detector`, `scheduler`, `controllers/binding`, `controllers/execution`, `controllers/status` |
| **Infrastructure Modules** | `apis/*`, `generated/*`, `util/*`, `features` |
| **Extension Modules** | `resourceinterpreter`, `webhook/*`, `dependenciesdistributor` |
| **Feature Modules** | `controllers/federatedhpa`, `controllers/gracefuleviction`, `controllers/mcs`, `controllers/applicationfailover` |
| **User-Facing Modules** | `karmadactl/*`, `cmd/*` |
| **Deployment Modules** | `operator/*`, `charts/*`, `artifacts/*` |

---

## Phase 8: Feature Mapping

| Feature | Files Involved | Flow | Dependencies |
|---------|---------------|------|-------------|
| **Resource Propagation** | `detector/detector.go`, `controllers/binding/`, `controllers/execution/` | Detect → Match Policy → Create Binding → Create Work → Execute | `resourceinterpreter`, `overridemanager` |
| **Multi-Cluster Scheduling** | `scheduler/scheduler.go`, `scheduler/core/generic_scheduler.go`, `scheduler/framework/plugins/*` | Watch Bindings → Filter → Score → Select → Assign Replicas | `scheduler/cache`, `estimator` |
| **Override Policies** | `util/overridemanager/`, `webhook/overridepolicy/` | Binding Controller applies overrides before creating Work | `apis/policy` |
| **Cluster Failover** | `controllers/cluster/`, `controllers/gracefuleviction/`, `features.go` | Detect unhealthy → Taint → Evict → Reschedule | `features.Failover` gate |
| **Federated HPA** | `controllers/federatedhpa/`, `controllers/cronfederatedhpa/` | Collect metrics → Calculate replicas → Update bindings | `metrics-adapter`, `estimator` |
| **Multi-Cluster Service** | `controllers/mcs/`, `controllers/multiclusterservice/` | Export → Import → EndpointSlice sync | `features.MultiClusterService` |
| **Dependency Distribution** | `dependenciesdistributor/` | Detect deps (ConfigMap, Secret) → Auto-propagate | `resourceinterpreter.GetDependencies()` |
| **Cluster Registration** | `karmadactl/join/`, `karmadactl/register/`, `controllers/cluster/` | Join → Create Cluster object → Start status collection | `util/credential.go` |
| **Resource Interpreter** | `resourceinterpreter/interpreter.go` | Get replicas, health, status, dependencies for CRDs | `customized/`, `default/` |
| **Graceful Eviction** | `controllers/gracefuleviction/` | Delay eviction until replacement is healthy | `features.GracefulEviction` |
| **Application Failover** | `controllers/applicationfailover/` | Monitor app health → Trigger eviction from unhealthy clusters | Per-policy configuration |
| **Federated Resource Quota** | `controllers/federatedresourcequota/` | Sync quotas → Monitor usage → Enforce limits | `FederatedResourceQuota` CRD |
| **Workload Rebalancer** | `controllers/workloadrebalancer/` | Trigger reschedule of specific workloads | `WorkloadRebalancer` CRD |
| **Unified Auth** | `controllers/unifiedauth/` | Propagate RBAC to member clusters | `ClusterRole`, `ClusterRoleBinding` |
| **Policy Preemption** | `detector/preemption.go` | Higher priority policy takes over resources | `features.PolicyPreemption` |

---

## Phase 9: API Analysis

### Karmada Custom APIs (CRD-based)

Karmada's APIs are exposed as Kubernetes CRDs. There is no REST API server in the traditional sense — all operations go through the Kubernetes API server.

#### Policy API Group (`policy.karmada.io/v1alpha1`)

| Resource | Scope | Operations | Purpose |
|----------|-------|-----------|---------|
| `PropagationPolicy` | Namespaced | CRUD | Define which resources go to which clusters |
| `ClusterPropagationPolicy` | Cluster | CRUD | Cluster-scoped propagation rules |
| `OverridePolicy` | Namespaced | CRUD | Per-cluster resource customization |
| `ClusterOverridePolicy` | Cluster | CRUD | Cluster-scoped overrides |
| `FederatedResourceQuota` | Namespaced | CRUD | Cross-cluster resource quotas |
| `ClusterTaintPolicy` | Cluster | CRUD | Auto-taint clusters on conditions |

#### Work API Group (`work.karmada.io`)

| Resource | Version | Scope | Purpose |
|----------|---------|-------|---------|
| `ResourceBinding` | v1alpha2 | Namespaced | Bind namespaced resources to clusters |
| `ClusterResourceBinding` | v1alpha2 | Cluster | Bind cluster-scoped resources |
| `Work` | v1alpha1 | Namespaced | Carry resource manifests to member clusters |

#### Cluster API Group (`cluster.karmada.io/v1alpha1`)

| Resource | Scope | Purpose |
|----------|-------|---------|
| `Cluster` | Cluster | Represent a registered member cluster |

#### Config API Group (`config.karmada.io/v1alpha1`)

| Resource | Scope | Purpose |
|----------|-------|---------|
| `ResourceInterpreterCustomization` | Cluster | Define custom resource interpretation logic |
| `ResourceInterpreterWebhookConfiguration` | Cluster | Configure external interpretation webhooks |

### Admission Webhooks

All admission webhooks are defined under [pkg/webhook/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/webhook):

| Webhook | Type | Resource | Purpose |
|---------|------|----------|---------|
| `propagationpolicy` | Validating + Mutating | PropagationPolicy | Validate policy spec, assign permanent ID |
| `overridepolicy` | Validating | OverridePolicy | Validate override rules |
| `resourcebinding` | Validating | ResourceBinding | Validate binding spec |
| `clusterresourcebinding` | Validating | ClusterResourceBinding | Validate cluster binding |
| `work` | Mutating | Work | Ensure proper labels |
| `federatedhpa` | Validating | FederatedHPA | Validate autoscaling spec |
| `multiclusteringress` | Validating | MultiClusterIngress | Validate ingress rules |
| `resourcedeletionprotection` | Validating | Various | Prevent accidental deletion |
| `interpreter` | Custom | Any CRD | Resource interpretation webhook |

### Authentication

All API calls are authenticated via standard Kubernetes mechanisms:
- **Client certificates** (mTLS)
- **Service account tokens**
- **OIDC tokens**
- Configured via the Karmada API server's `--authentication-*` flags

---

## Phase 10: Authentication & Security

### Cluster Authentication Model

```
┌──────────────┐         ┌──────────────────┐        ┌─────────────────┐
│   User /     │  mTLS/  │  Karmada API     │  mTLS  │  Member Cluster │
│   kubectl    │──Token──►│  Server          │────────►│  API Server     │
│              │         │  (kube-apiserver) │        │                 │
└──────────────┘         └──────────────────┘        └─────────────────┘
                                │
                                │ RBAC
                                ▼
                         ClusterRole / Role
                         ClusterRoleBinding / RoleBinding
```

### How Member Cluster Credentials Are Managed

**File**: [pkg/util/credential.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/util/credential.go)

1. **Push Mode**: Member cluster kubeconfig is stored as a `Secret` in the Karmada control plane. The `Cluster` object's `spec.secretRef` points to this secret.
2. **Pull Mode**: The agent running in the member cluster uses a certificate-based identity. CSR (Certificate Signing Request) flow via [controllers/certificate/approver/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/controllers/certificate/approver).

### Unified Auth Controller

**File**: [pkg/controllers/unifiedauth/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/controllers/unifiedauth)

Propagates RBAC rules from Karmada control plane to member clusters, ensuring consistent authorization.

### Security Mechanisms

| Mechanism | Where | File |
|-----------|-------|------|
| mTLS | All inter-component communication | Kubernetes CA infrastructure |
| RBAC | API authorization | Kubernetes built-in + UnifiedAuth controller |
| Admission Webhooks | Input validation | `pkg/webhook/*` |
| Leader Election | Single-writer protection | controller-runtime lease-based |
| Resource Deletion Protection | Prevent accidental deletion | `pkg/webhook/resourcedeletionprotection/` |
| Namespace Isolation | Execution spaces per cluster | `karmada-es-{cluster-name}` namespaces |
| Certificate Rotation | Agent CSR approval | `controllers/certificate/approver` |
| Feature Gates | Disable risky features | `pkg/features/features.go` |

### Security Risks Identified

| Risk | Severity | Description |
|------|----------|-------------|
| Cluster credential exposure | **High** | Member cluster kubeconfig stored as Secret — if control plane is compromised, all clusters are compromised |
| No network policy enforcement | **Medium** | No built-in network segmentation between execution namespaces |
| Overly broad RBAC in agent mode | **Medium** | Agent requires significant permissions in member clusters |
| No audit logging built-in | **Low** | Relies on Kubernetes audit logging configuration |

---

## Phase 11: External Integrations

| Integration | Package/File | Purpose | Protocol |
|-------------|-------------|---------|----------|
| **Kubernetes API Server** | `k8s.io/client-go` | Core data store and API | HTTPS/REST |
| **etcd** | Via kube-apiserver | Persistent storage | gRPC |
| **gRPC (Scheduler Estimator)** | [pkg/estimator/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/estimator) | Replica estimation per cluster | gRPC with TLS |
| **Prometheus** | [pkg/metrics/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/metrics) | Metrics exposition | HTTP `/metrics` |
| **OpenSearch** | `opensearch-go` | Cross-cluster resource search backend | HTTPS/REST |
| **Cluster API (CAPI)** | [pkg/clusterdiscovery/clusterapi/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/clusterdiscovery/clusterapi) | Auto-discover clusters from CAPI | Kubernetes API |
| **Kubernetes Metrics API** | `k8s.io/metrics` | HPA metrics collection | Kubernetes API |
| **Custom Metrics API** | `custom_metrics`, `external_metrics` | Custom & external metrics for Federated HPA | Kubernetes API |
| **OpenTelemetry** | `go.opentelemetry.io/otel` | Distributed tracing | OTLP/gRPC |
| **Helm** | `charts/` | Package management for installation | Helm v3 |
| **KIND** | `sigs.k8s.io/kind` | Local development clusters | Docker |
| **Lua** | `gopher-lua` | Declarative resource interpreter scripting | Embedded |

---

## Phase 12: Design Patterns

| Pattern | Where Used | File(s) | Explanation |
|---------|-----------|---------|-------------|
| **Controller/Reconciler** | Every controller | All `pkg/controllers/*/` | Core Kubernetes pattern — watch events, reconcile desired vs actual state |
| **Factory** | Controller initialization | [controllermanager.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/controller-manager/app/controllermanager.go#L234-L267) | Map of `string → startFunc` for controller registration |
| **Strategy** | Scheduler plugins | [pkg/scheduler/framework/plugins/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/framework/plugins) | Filter/Score plugins are interchangeable strategies |
| **Chain of Responsibility** | Resource Interpreter | [interpreter.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/resourceinterpreter/interpreter.go) | Configurable → Customized → ThirdParty → Default fallback chain |
| **Observer/Event-Driven** | Informers + Event Handlers | `pkg/detector/`, `pkg/scheduler/event_handler.go` | Watch Kubernetes objects, react to changes |
| **Builder** | Options/Config construction | `cmd/*/app/options/` | Functional options pattern for scheduler, controllers |
| **Singleton** | Informer managers | `genericmanager.GetInstance()` | Single informer manager instance per process |
| **Adapter** | Resource interpreter adapters | `resourceinterpreter/default/native/` | Adapts diverse K8s resource types to unified interface |
| **Proxy** | Object Watcher | [pkg/util/objectwatcher/](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/util/objectwatcher) | Proxy that syncs resources to remote clusters |
| **Template Method** | Generic scheduler | [generic_scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/core/generic_scheduler.go) | Fixed algorithm skeleton (filter→score→select→assign) with plugin extension points |
| **Registry** | Plugin registry | [registry.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/framework/plugins/registry.go) | Register in-tree and out-of-tree scheduler plugins |
| **Async Worker Queue** | All controllers | [worker.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/util/worker.go) | Rate-limited work queues for event processing |
| **Functional Options** | Scheduler construction | [scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go#L150-L235) | `WithEnableSchedulerEstimator()`, `WithSchedulerName()`, etc. |

---

## Phase 13: Code Walkthrough — Top 10 Most Important Files

### 1. [controllermanager.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/controller-manager/app/controllermanager.go)

- **Purpose**: Bootstrap the entire controller manager — the heart of Karmada
- **Key Functions**: `NewControllerManagerCommand()`, `Run()`, `setupControllers()`, all `startXxxController()` functions
- **Logic**: Creates controller-runtime manager, initializes 27+ controllers, resource detector, dependencies distributor
- **Dependencies**: Nearly every package in `pkg/`
- **Why it exists**: Single entry point for all control plane controllers; modeled after `kube-controller-manager`

### 2. [detector.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go)

- **Purpose**: Watch ALL resources in the control plane and match them to propagation policies
- **Key Functions**: `Start()`, `Reconcile()`, `LookForMatchedPolicy()`, `ApplyPolicy()`, `BuildResourceBinding()`
- **Logic**: Discovers API resources, sets up informers, matches resources to policies, creates ResourceBindings
- **Dependencies**: `resourceinterpreter`, `apis/policy`, `apis/work`, `util/fedinformer`
- **Why it exists**: Bridge between user-created resources and Karmada's propagation pipeline

### 3. [scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go)

- **Purpose**: Schedule ResourceBindings to target clusters
- **Key Functions**: `NewScheduler()`, `Run()`, `doScheduleBinding()`, `scheduleResourceBinding()`
- **Logic**: Watches ResourceBindings, determines schedule type, calls Algorithm.Schedule(), patches results
- **Dependencies**: `scheduler/core`, `scheduler/framework`, `estimator/client`, `generated/`
- **Why it exists**: Decouples placement decisions from the detection and execution pipeline

### 4. [generic_scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/core/generic_scheduler.go)

- **Purpose**: Core scheduling algorithm — Filter → Score → Select → Assign
- **Key Functions**: `Schedule()`, `findClustersThatFit()`, `prioritizeClusters()`, `selectClusters()`, `assignReplicas()`
- **Logic**: Plugin-based scheduling pipeline inspired by kube-scheduler
- **Dependencies**: `scheduler/framework`, `scheduler/cache`
- **Why it exists**: Implements the scheduling algorithm as a composable pipeline

### 5. [interpreter.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/resourceinterpreter/interpreter.go)

- **Purpose**: Understand the structure of any Kubernetes resource (including CRDs)
- **Key Functions**: `GetReplicas()`, `ReviseReplica()`, `GetDependencies()`, `AggregateStatus()`, `InterpretHealth()`
- **Logic**: Chain-of-responsibility through 4 interpreter layers: Configurable → Webhook → ThirdParty → Default
- **Dependencies**: `customized/`, `default/native`, `default/thirdparty`
- **Why it exists**: Karmada needs to understand CRD structures without hard-coding each type

### 6. [execution_controller.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/controllers/execution/execution_controller.go)

- **Purpose**: Dispatch Work objects to member clusters (Push mode)
- **Key Functions**: `Reconcile()`, sync logic
- **Logic**: Watches Work objects, uses ObjectWatcher to create/update/delete resources in member clusters
- **Dependencies**: `objectwatcher`, `util/helper`
- **Why it exists**: The "last mile" — actually applies resources to member clusters

### 7. [features.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/features/features.go)

- **Purpose**: Define all feature gates that control optional functionality
- **Key Constants**: `Failover`, `GracefulEviction`, `PropagateDeps`, `MultiClusterService`, `PriorityBasedScheduling`, etc.
- **Logic**: Registry of feature flags with alpha/beta maturity levels
- **Why it exists**: Allows gradual rollout of features without breaking existing users

### 8. [propagation_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/policy/v1alpha1/propagation_types.go)

- **Purpose**: Define the PropagationPolicy and ClusterPropagationPolicy CRD schemas
- **Key Types**: `PropagationPolicy`, `PropagationSpec`, `Placement`, `ResourceSelector`, `ClusterAffinity`
- **Why it exists**: The user-facing API — this is what users write to control propagation

### 9. [binding_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/work/v1alpha2/binding_types.go)

- **Purpose**: Define ResourceBinding and ClusterResourceBinding CRD schemas
- **Key Types**: `ResourceBinding`, `ResourceBindingSpec`, `TargetCluster`, `AggregatedStatusItem`
- **Why it exists**: Internal CRD that bridges detection → scheduling → execution

### 10. [worker.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/util/worker.go)

- **Purpose**: Provide async work queue abstraction used by all controllers
- **Key Types**: `AsyncWorker`, `AsyncPriorityWorker`
- **Key Functions**: `NewAsyncWorker()`, `Run()`, `Enqueue()`, reconcile loop
- **Why it exists**: Unified rate-limited event processing — backbone of all controllers

---

## Phase 14: Data Flow Analysis

```
                                Data Flow Diagram
                                ==================

User (kubectl)
    │
    │  Creates Deployment + PropagationPolicy
    │  (kubectl apply -f ...)
    ▼
┌──────────────────────────────────────────────────────────────┐
│  KARMADA API SERVER (etcd-backed)                            │
│                                                              │
│  Stores: Deployment, PropagationPolicy, OverridePolicy,     │
│          ResourceBinding, Work, Cluster objects               │
└──────────────┬───────────────────────────────────────────────┘
               │
    ┌──────────┼──────────────────────┐
    │          │                      │
    ▼          ▼                      ▼
┌────────┐ ┌────────────────┐  ┌─────────────┐
│Detector│ │Webhook Server  │  │  Scheduler   │
│        │ │(validates on   │  │              │
│ Data:  │ │ create/update) │  │ Data:        │
│ - GVR  │ └────────────────┘  │ - Binding    │
│ - Key  │                      │   Spec       │
│ - Obj  │                      │ - Cluster    │
│        │                      │   List       │
│ Output:│                      │ - Scores     │
│ - RB   │                      │              │
│   with │                      │ Output:      │
│ policy │                      │ - Target     │
│ config │                      │   Clusters   │
└───┬────┘                      │ - Replica    │
    │                           │   Assign.    │
    │ creates ResourceBinding   └──────┬───────┘
    └──────────────────────────────────┤
                                       │ patches spec.clusters
                                       ▼
                              ┌──────────────────┐
                              │ Binding Controller│
                              │                  │
                              │ Data:            │
                              │ - Binding Spec   │
                              │ - Override Rules │
                              │                  │
                              │ Output:          │
                              │ - Work objects   │
                              │   (1 per cluster)│
                              └────────┬─────────┘
                                       │
                    ┌──────────────────┤───────────────────┐
                    ▼                  ▼                   ▼
            ┌──────────────┐  ┌──────────────┐   ┌──────────────┐
            │ExecutionCtrl │  │ExecutionCtrl │   │   Agent      │
            │(Push→member1)│  │(Push→member2)│   │ (Pull→member3│
            │              │  │              │   │              │
            │ Data:        │  │ Data:        │   │ Data:        │
            │ - K8s        │  │ - K8s        │   │ - Work       │
            │   manifest   │  │   manifest   │   │   manifest   │
            └──────┬───────┘  └──────┬───────┘   └──────┬───────┘
                   │                 │                   │
                   ▼                 ▼                   ▼
            Member Cluster 1  Member Cluster 2   Member Cluster 3
            (real resources)  (real resources)   (real resources)
                   │                 │                   │
                   └────────┬────────┘                   │
                            │ status flows back          │
                            ▼                            ▼
                    ┌──────────────────┐         ┌──────────────┐
                    │WorkStatus Ctrl   │         │  Agent       │
                    │(collects status) │         │ (reports     │
                    │                  │         │  status)     │
                    └────────┬─────────┘         └──────┬───────┘
                             │                          │
                             └──────────┬───────────────┘
                                        ▼
                              ┌──────────────────┐
                              │ RB/CRB Status    │
                              │ Controller       │
                              │ (aggregates      │
                              │  to Binding)     │
                              └──────────────────┘
```

---

## Phase 15: Execution Trace — Application Startup

### Controller Manager Startup Sequence

```
1. main() ─────────────────── cmd/controller-manager/controller-manager.go:30
   │
2. SetupSignalHandler() ──── controller-runtime (OS signal → context cancellation)
   │
3. NewControllerManagerCommand() ── cmd/controller-manager/app/controllermanager.go:101
   │  ├── Parse flags (kubeconfig, controllers, feature-gates, etc.)
   │  └── Register cobra command
   │
4. cli.Run(cmd) ──────────── k8s.io/component-base/cli
   │
5. Run(ctx, opts) ────────── cmd/controller-manager/app/controllermanager.go:163
   │
6. GetConfig() ───────────── Read kubeconfig, create REST config
   │
7. NewManager() ──────────── controller-runtime manager
   │  ├── Start cache sync
   │  ├── Configure leader election (Lease)
   │  ├── Set health/readiness probes
   │  └── Register metrics endpoint
   │
8. setupControllers() ────── cmd/controller-manager/app/controllermanager.go:843
   │
9.  ├── NewSingleClusterInformerManager() ─ pkg/util/fedinformer/genericmanager/
   │  │
10. ├── NewResourceInterpreter() ────────── pkg/resourceinterpreter/interpreter.go:84
   │  │   └── Start() → loads interpreter configs
   │  │
11. ├── ResourceDetector{} ─────────────── pkg/detector/detector.go:69
   │  │   └── mgr.Add(resourceDetector)
   │  │       └── Start() → L113
   │  │           ├── Setup policy reconcile workers
   │  │           ├── Setup resource detector worker
   │  │           ├── Watch PropagationPolicy informer
   │  │           ├── Watch ClusterPropagationPolicy informer
   │  │           └── discoverResources() goroutine (every 30s)
   │  │
12. ├── DependenciesDistributor{} ──────── pkg/dependenciesdistributor/ (if PropagateDeps enabled)
   │  │
13. ├── controllers.StartControllers() ─── Iterates controller map:
   │  │   ├── startClusterController
   │  │   ├── startClusterStatusController
   │  │   ├── startBindingController
   │  │   ├── startBindingStatusController
   │  │   ├── startExecutionController
   │  │   ├── startWorkStatusController
   │  │   ├── startNamespaceController
   │  │   ├── ... (20 more controllers)
   │  │   └── Each calls SetupWithManager(mgr)
   │  │
14. └── setupClusterAPIClusterDetector() ─ (if CAPI kubeconfig provided)
   │
15. controllerManager.Start(ctx) ─────── Blocks forever
       ├── Leader election acquired
       ├── Cache sync completed
       ├── All controllers start reconciling
       └── Blocks until ctx.Done() (OS signal)
```

---

## Phase 16: Learning Roadmap

### 🟢 Beginner Level — Read These First

| Order | File | Why |
|-------|------|-----|
| 1 | [README.md](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/README.md) | Understand the project's purpose and architecture overview |
| 2 | [features.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/features/features.go) | See all feature gates — understand what's toggleable |
| 3 | [propagation_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/policy/v1alpha1/propagation_types.go) | Core API — learn PropagationPolicy structure |
| 4 | [binding_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/work/v1alpha2/binding_types.go) | Core API — learn ResourceBinding structure |
| 5 | [controller-manager.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/controller-manager/controller-manager.go) | Entry point — see how the system boots |
| 6 | [constants.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/util/constants.go) | All constants and well-known values |
| 7 | `examples/` | Sample YAML files to understand user-facing API |
| 8 | [CONTRIBUTING.md](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/CONTRIBUTING.md) | Development setup and contribution process |

### 🟡 Intermediate Level — Read These Next

| Order | File | Why |
|-------|------|-----|
| 9 | [controllermanager.go (app)](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/controller-manager/app/controllermanager.go) | Full bootstrap logic — understand how controllers are wired |
| 10 | [detector.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go) | Resource detection — the "entry" to the propagation pipeline |
| 11 | [scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go) | Scheduler logic — how clusters are selected |
| 12 | [generic_scheduler.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/core/generic_scheduler.go) | Core algorithm — Filter → Score → Select → Assign |
| 13 | [execution_controller.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/controllers/execution/execution_controller.go) | How Work objects become real resources |
| 14 | [interpreter.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/resourceinterpreter/interpreter.go) | Resource understanding framework |
| 15 | [worker.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/util/worker.go) | Async work queue — used everywhere |
| 16 | [override_types.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/apis/policy/v1alpha1/override_types.go) | Override policy API |

### 🔴 Advanced Level — Master These

| Order | File | Why |
|-------|------|-----|
| 17 | [preemption.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/preemption.go) | Policy preemption logic |
| 18 | [dependencies_distributor.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/dependenciesdistributor/dependencies_distributor.go) | Auto-dependency detection |
| 19 | `pkg/scheduler/framework/` | Full scheduler framework with plugin system |
| 20 | `pkg/scheduler/core/assignment.go` | Replica assignment algorithms |
| 21 | `pkg/scheduler/core/division_algorithm.go` | Replica division strategies |
| 22 | `pkg/controllers/cluster/` | Cluster lifecycle with taint-based eviction |
| 23 | `pkg/controllers/gracefuleviction/` | Graceful workload migration |
| 24 | `pkg/controllers/federatedhpa/` | Cross-cluster HPA algorithm |
| 25 | `pkg/resourceinterpreter/customized/declarative/` | Lua-based interpreter |
| 26 | `operator/pkg/` | Karmada operator implementation |

---

## Phase 17: Contribution Guide

### Making Your First PR

1. **Fork** the repository on GitHub
2. **Clone** your fork: `git clone https://github.com/YOUR_USERNAME/karmada.git`
3. **Read** [CONTRIBUTING.md](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/CONTRIBUTING.md)
4. **Set up dev environment**: Requires Go 1.26+ (see [.go-version](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/.go-version))
5. **Run tests**: `make test`
6. **Run lints**: `make lint` (uses `.golangci.yml`)

### Safe Areas to Modify (Beginner-Friendly)

| Area | Risk Level | Description |
|------|-----------|-------------|
| `pkg/karmadactl/*` | 🟢 Low | CLI commands — isolated, well-tested |
| `test/e2e/*` | 🟢 Low | Test improvements — can't break production |
| `docs/` | 🟢 Low | Documentation |
| `pkg/util/` (new utilities) | 🟢 Low | Adding new helper functions |
| `pkg/webhook/*/validating_*.go` | 🟡 Medium | Adding validation rules |
| `pkg/scheduler/framework/plugins/*` | 🟡 Medium | Adding scheduler plugins |

### Risky Areas

| Area | Risk Level | Description |
|------|-----------|-------------|
| `pkg/detector/detector.go` | 🔴 High | Core detection loop — bugs affect all propagation |
| `pkg/scheduler/scheduler.go` | 🔴 High | Scheduling bugs cause workload misplacement |
| `pkg/controllers/execution/` | 🔴 High | Bugs here create/delete resources in production clusters |
| `pkg/apis/*` | 🔴 High | API changes affect backward compatibility |
| `pkg/resourceinterpreter/default/` | 🟡 Medium | Affects how standard K8s resources are understood |

### Testing Strategy

| Test Type | Location | Command |
|-----------|----------|---------|
| Unit tests | `*_test.go` alongside source | `make test` |
| E2E tests | `test/e2e/` | Requires local KIND clusters |
| Linting | `.golangci.yml` | `make lint` |
| Code generation | `hack/` | `make generate` |

---

## Phase 18: Interview Questions

### Architecture (5 Questions)

1. **Explain the hub-and-spoke architecture of Karmada. What are the key components in the control plane, and how do member clusters connect?**
2. **What is the difference between Push mode and Pull mode for member cluster registration? When would you choose one over the other?**
3. **Describe the resource propagation pipeline from when a user creates a Deployment to when it runs in a member cluster. Name every CRD involved.**
4. **How does Karmada's architecture compare to KubeFed (Kubernetes Federation v2)? What are the key design differences?**
5. **Why does Karmada use a separate karmada-apiserver instead of just extending an existing Kubernetes cluster? What are the trade-offs?**

### Scalability (4 Questions)

6. **How does Karmada handle rate limiting when managing hundreds of member clusters simultaneously?**
7. **Explain the role of the scheduler estimator. How does it prevent resource overcommitment across clusters?**
8. **What is the SchedulingOvercommitProtection feature gate, and what problem does it solve? How does the assumption cache work?**
9. **How would you scale Karmada to manage 1,000+ clusters? What are the bottlenecks?**

### Security (4 Questions)

10. **How are member cluster credentials stored and managed? What are the security implications?**
11. **Explain the Unified Auth Controller. How does it propagate RBAC rules to member clusters?**
12. **What admission webhooks does Karmada use, and what do they validate? How would you add a new validation rule?**
13. **In Pull mode, how does the agent authenticate with the Karmada control plane? Describe the CSR approval flow.**

### Database/Data Model (3 Questions)

14. **What is the relationship between PropagationPolicy, ResourceBinding, and Work objects? Draw the data flow.**
15. **Why does Karmada use execution namespaces (karmada-es-{cluster})? What problems does this solve?**
16. **How is status aggregated from multiple member clusters back to the ResourceBinding?**

### Design Patterns (4 Questions)

17. **Identify and explain the Chain of Responsibility pattern in Karmada's resource interpreter. Why was this pattern chosen?**
18. **How does the scheduler framework implement the Strategy pattern? How would you add a new scheduling plugin?**
19. **Explain the Functional Options pattern used in the scheduler constructor. What are its benefits?**
20. **How does the controller factory pattern in controllermanager.go work? Why is `init()` used to register controllers?**

### System Design (5 Questions)

21. **Design a graceful failover system. How does Karmada ensure zero-downtime when migrating workloads from a failed cluster?**
22. **How would you implement cross-cluster service discovery? Explain how Karmada's Multi-Cluster Service feature works.**
23. **Design a federated HPA system. How does Karmada collect metrics from multiple clusters and make scaling decisions?**
24. **Explain the override policy system. How can you customize a Deployment's image registry differently for each cluster?**
25. **How would you implement workload affinity/anti-affinity across clusters? What data structures would you use?**

---

## Phase 19: Improvement Suggestions

### 🔴 High Priority

| Issue | Type | Location | Description |
|-------|------|----------|-------------|
| Error handling in detector | Code Smell | [detector.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go) | Many `klog.Errorf` without structured logging; some errors silently ignored |
| `panic(err)` in Run() | Bug Risk | [controllermanager.go:170](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/controller-manager/app/controllermanager.go#L170) | REST config failure causes panic instead of graceful shutdown |
| Race condition TODO | Bug Risk | [scheduler.go:705-707](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/scheduler/scheduler.go#L704-L708) | Documented race condition in assumption cache `Add()` |
| `klog.Fatalf` in setupControllers | Bug Risk | [controllermanager.go:866-904](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/cmd/controller-manager/app/controllermanager.go#L866) | Fatal calls that crash the entire process instead of graceful degradation |
| Deprecated API usage | Code Smell | Throughout controllers | `GetEventRecorderFor` deprecated in controller-runtime v0.23.0 (noted in comments) |

### 🟡 Medium Priority

| Issue | Type | Location | Description |
|-------|------|----------|-------------|
| `nolint:gocyclo` | Code Smell | [detector.go:529](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go#L529) | `ApplyClusterPolicy` is too complex, should be refactored |
| Duplicated binding update logic | Code Smell | [detector.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go#L441-L666) | `ApplyPolicy` and `ApplyClusterPolicy` have nearly identical logic — extract shared function |
| Large file sizes | Code Smell | `detector.go` (1569L), `controllermanager.go` (981L), `scheduler.go` (1162L) | Should be split into smaller, focused files |
| Missing context propagation | Performance | Various | Some functions use `context.TODO()` instead of propagating parent context |
| Hardcoded retry backoff | Performance | Various | `retry.DefaultRetry` used everywhere; should be configurable |

### 🟢 Low Priority

| Issue | Type | Location | Description |
|-------|------|----------|-------------|
| `new(true)` pattern | Code Smell | [detector.go:390,417](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/pkg/detector/detector.go#L390) | `new(true)` is unusual Go — use `ptr.To(true)` consistently |
| Test coverage gaps | Testing | `pkg/detector/` | `detector.go` (1569L) vs `detector_test.go` (40470B) — complex functions need more edge case tests |
| Go version | Maintenance | [go.mod](file:///c:/Users/priya/OneDrive/Desktop/Coding/karmada/go.mod) | `go 1.26.4` — keep up with latest Go releases |
| Missing godoc on exported types | Documentation | Various | Some exported types/functions lack documentation comments |

---

## Phase 20: Final Summary

### 1. Project Mental Model

> **Karmada = Meta-Kubernetes for Multi-Cluster.** Think of it as a Kubernetes that manages other Kubernetes clusters. Users interact with it using familiar Kubernetes APIs, plus Karmada-specific CRDs (PropagationPolicy, OverridePolicy) that tell Karmada *where* and *how* to run workloads across clusters.

### 2. Architecture Summary

```
Architecture: Event-Driven Controller Pattern (Hub-and-Spoke)
Language: Go
Framework: controller-runtime + client-go + cobra
Data Store: etcd (via Kubernetes API server)
Communication: Kubernetes API (REST/HTTP2) + gRPC (scheduler estimator)
Installation: Helm charts, Operator, or CLI (karmadactl init)
```

### 3. Request Flow Summary

```
User → Deployment + PropagationPolicy
  → Detector matches policy
    → Creates ResourceBinding
      → Scheduler assigns clusters
        → Binding Controller creates Work objects
          → Execution Controller / Agent applies to member clusters
            → Status flows back up the chain
```

### 4. Database Summary

- **No traditional DB** — uses Kubernetes etcd via API server
- **9 API groups** with 10+ CRDs
- **Key CRDs**: PropagationPolicy, ResourceBinding, Work, Cluster
- **Execution spaces**: `karmada-es-{cluster}` namespaces isolate per-cluster Work objects

### 5. Key Files Summary

| Rank | File | Role |
|------|------|------|
| #1 | `cmd/controller-manager/app/controllermanager.go` | System bootstrap |
| #2 | `pkg/detector/detector.go` | Resource detection & policy matching |
| #3 | `pkg/scheduler/scheduler.go` | Multi-cluster scheduling |
| #4 | `pkg/scheduler/core/generic_scheduler.go` | Core scheduling algorithm |
| #5 | `pkg/resourceinterpreter/interpreter.go` | CRD structure understanding |
| #6 | `pkg/controllers/execution/execution_controller.go` | Resource dispatch to clusters |
| #7 | `pkg/apis/policy/v1alpha1/propagation_types.go` | PropagationPolicy API |
| #8 | `pkg/apis/work/v1alpha2/binding_types.go` | ResourceBinding API |
| #9 | `pkg/features/features.go` | Feature gate definitions |
| #10 | `pkg/util/worker.go` | Async work queue infrastructure |

### 6. What You MUST Understand First

1. **The CRD pipeline**: PropagationPolicy → ResourceBinding → Work → Member Cluster Resource
2. **The Detector's role**: It's the bridge between user resources and Karmada's internal pipeline
3. **Push vs Pull mode**: Two fundamentally different cluster connection models
4. **The scheduler framework**: Filter → Score → Select → Assign pipeline
5. **The resource interpreter**: How Karmada understands CRD structures

### 7. What You Can Ignore Initially

1. `vendor/` — Standard Go vendor directory
2. `pkg/generated/` — Auto-generated code (don't edit manually)
3. `operator/` — Only relevant if you're deploying Karmada via the operator
4. `pkg/karmadactl/` — CLI commands are isolated from core logic
5. `pkg/search/` — Optional search functionality
6. `pkg/metricsadapter/` — Optional metrics aggregation
7. `third_party/` — Vendored third-party code
8. `hack/` — Build scripts and code generation

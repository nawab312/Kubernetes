# Kubernetes Interview Mastery
# CATEGORY 6: SCHEDULING & RESOURCE MANAGEMENT

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Deep Answer → Gotchas → Interview Q&A → Connections.
> ⚠️ = High priority, frequently asked, commonly misunderstood.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 6.1 | Resource requests vs limits — CPU & memory ⚠️ | 🟢 Beginner |
| 6.2 | Namespaces — isolation and organization | 🟢 Beginner |
| 6.3 | LimitRange — defaults per namespace | 🟢 Beginner |
| 6.4 | ResourceQuota — namespace-level caps | 🟢 Beginner |
| 6.5 | QoS classes — Guaranteed, Burstable, BestEffort ⚠️ | 🟡 Intermediate |
| 6.6 | Node Affinity & Anti-Affinity | 🟡 Intermediate |
| 6.7 | Pod Affinity & Anti-Affinity ⚠️ | 🟡 Intermediate |
| 6.8 | Taints & Tolerations ⚠️ | 🟡 Intermediate |
| 6.9 | Priority Classes & Preemption | 🟡 Intermediate |
| 6.10 | Topology Spread Constraints ⚠️ | 🟡 Intermediate |
| 6.11 | Scheduler extenders & custom schedulers | 🔴 Advanced |
| 6.12 | Bin packing vs spreading strategies | 🔴 Advanced |
| 6.13 | Descheduler — rebalancing after the fact | 🔴 Advanced |
| 6.14 | Node pressure eviction — memory/disk/PID pressure | 🔴 Advanced |
| 6.15 | CPU Manager & Memory Manager (static policy) | 🔴 Advanced |

---

## Difficulty Legend
- 🟢 **Beginner** — Expected from ALL candidates
- 🟡 **Intermediate** — Expected from 3+ year engineers
- 🔴 **Advanced** — Differentiates senior/staff candidates
- ⚠️ **High Priority** — Frequently asked / commonly misunderstood

---

# ⚠️ 6.1 Resource Requests vs Limits — CPU & Memory

## 🟢 Beginner — HIGH PRIORITY

### What it is in simple terms

**Requests** are what the scheduler uses to place the pod — the guaranteed minimum a container gets. **Limits** are the maximum a container can use — the enforced ceiling. They are enforced by two completely different kernel mechanisms and behave very differently when exceeded.

---

### Requests vs Limits — The Core Distinction

```
REQUESTS: THE SCHEDULER'S PROMISE
═══════════════════════════════════════════════════════════════

Request = "I need at least this much to function correctly"
Scheduler finds a node where:
  allocatable_CPU    - already_requested_CPU    >= pod's CPU request
  allocatable_Memory - already_requested_Memory >= pod's memory request

Request is RESERVED on the node — even if the container uses less.
Node's "available capacity" = allocatable - sum of all requests
A node can be FULL (no schedulable pods) while CPUs sit idle.
  → This is expected and by design.

LIMITS: THE KERNEL'S CEILING
═══════════════════════════════════════════════════════════════

Limit = "You cannot use more than this"
Enforced by Linux kernel control groups (cgroups):
  CPU limit  → enforced by CFS bandwidth throttling
  Memory limit → enforced by OOM killer

Limit is NOT reserved on the node.
A node with 4 CPU cores can have total limits of 40 CPU cores.
(This is called "overcommitting limits" — common and normal)

WHEN EXCEEDED:
  CPU limit exceeded  → container THROTTLED (slowed, not killed)
                        Still running, but getting less CPU time
  Memory limit exceeded → container KILLED with OOMKilled
                          Pod restarts (if restartPolicy allows)
```

---

### CPU Units and Memory Units

```
CPU UNITS:
  1 CPU    = 1 vCPU = 1 AWS vCPU = 1 GCP vCPU = 1 Azure vCore
  1000m    = 1 CPU  (m = millicores)
  500m     = 0.5 CPU
  100m     = 0.1 CPU (one-tenth of a vCPU)
  250m     = 0.25 CPU
  1.5      = 1.5 CPUs (decimal notation)

  Minimum granularity: 1m (one millicore)
  You CANNOT request fractional millicores.

MEMORY UNITS:
  Ki = kibibyte = 1024 bytes
  Mi = mebibyte = 1024 Ki = 1,048,576 bytes
  Gi = gibibyte = 1024 Mi = 1,073,741,824 bytes

  K  = kilobyte  = 1000 bytes  (decimal)
  M  = megabyte  = 1000 KB
  G  = gigabyte  = 1000 MB

  In practice always use Ki/Mi/Gi to avoid ambiguity.
  512Mi = 512 × 1024 × 1024 bytes = 536,870,912 bytes

NODE ALLOCATABLE vs CAPACITY:
  Node capacity: 8 CPU, 32Gi RAM  (total hardware)
  System reserved: 200m CPU, 512Mi RAM  (OS + kubelet)
  Kube reserved:   100m CPU, 256Mi RAM  (K8s components)
  Allocatable:     7.7 CPU, ~31.25Gi RAM  (available for pods)

  kubectl describe node worker-1 | grep -A 6 Allocatable
  # Allocatable:
  #   cpu:     7700m
  #   memory:  31457280Ki
  #   pods:    110          ← hard pod limit per node
```

---

### Resource Spec YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:v1
    resources:
      requests:
        cpu: "250m"          # scheduler needs a node with 250m CPU free
        memory: "256Mi"      # scheduler needs a node with 256Mi memory free
      limits:
        cpu: "500m"          # kernel throttles if using > 500m CPU
        memory: "512Mi"      # kernel OOMKills if using > 512Mi memory

  - name: sidecar
    image: envoy:v1.28
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"

  # Total pod resource profile:
  # requests: 300m CPU, 320Mi memory  ← what scheduler sees
  # limits:   600m CPU, 640Mi memory  ← what kernel enforces

---
# Deployment with resources (production pattern)
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: api
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"    # allow burst to 4x request
            memory: "1Gi"   # memory limit = memory request for Guaranteed QoS
                            # or set limit > request for Burstable QoS
```

---

### CPU Throttling — How CFS Bandwidth Works

```
CPU THROTTLING MECHANISM (CFS = Completely Fair Scheduler)
═══════════════════════════════════════════════════════════════

Period: 100ms (default cfs_period_us = 100,000 microseconds)
Quota:  CPU limit × period
  Container with 500m limit:
  quota = 500m × 100ms = 50ms of CPU time per 100ms period

Timeline for 500m CPU limit:
  [0ms ─────── 50ms] Container runs freely
  [50ms ── 100ms]    Container THROTTLED (kernel suspends it)
  [100ms ─────150ms] New period starts, runs freely again
  [150ms ──200ms]    Throttled again
  → Container gets exactly 50% of a single CPU core

IMPACT OF THROTTLING:
  Container is alive and running — just slow
  Latency increases — requests take longer to complete
  CPU-intensive work gets done, just spread over more time
  For latency-sensitive apps: throttling = tail latency spikes
  P99 latency often dominated by CPU throttling at burst time

COMMON ANTIPATTERN: CPU limit too low
  Pod request: 100m, limit: 200m
  Traffic spike → app needs 300m
  App throttled to 200m → response latency doubles
  Looks like a slow app, not a resource problem
  Diagnosis: kubectl top pod + check CPU throttling metric:
    container_cpu_cfs_throttled_seconds_total

RECOMMENDATION:
  For latency-sensitive services (APIs, user-facing):
    Set NO CPU limit (only CPU request)
    → App gets to burst to full node CPU when available
    → No throttling artifacts
    → Use requests for scheduling accuracy
  For batch/background jobs:
    Set CPU limits — controlled resource usage is important
  For memory: ALWAYS set both request AND limit
    (memory limit = prevents OOM cascade across cluster)
```

---

### Memory OOM Kill — How It Works

```
MEMORY OOM KILL MECHANISM
═══════════════════════════════════════════════════════════════

When container exceeds memory limit:
  Linux kernel OOM killer activates
  Process inside container receives SIGKILL (uncatchable)
  Container exits with code 137 (128 + SIGKILL = 137)
  Pod status: OOMKilled
  kubelet restarts container (if restartPolicy: Always)

kubectl get pods
# NAME      READY  STATUS     RESTARTS  AGE
# myapp     1/1    Running    3         10m
#                             ↑ 3 OOMKills in 10 minutes = memory limit too low

kubectl describe pod myapp | grep -A 3 "Last State"
# Last State:  Terminated
#   Reason:    OOMKilled
#   Exit Code: 137

# Check actual memory usage
kubectl top pod myapp
# NAME    CPU(cores)  MEMORY(bytes)
# myapp   245m        498Mi   ← close to limit (512Mi) — will OOMKill soon

MEMORY UNLIKE CPU:
  CPU:    exceeds limit → throttled → slows down → keeps running
  Memory: exceeds limit → OOMKilled → process dies → pod restarts

Memory cannot be throttled — you cannot "slow down" memory usage.
Either you have the memory or you don't.
Set memory limit equal to memory request (Guaranteed QoS) for
critical workloads to prevent OOM kills.
```

---

### kubectl Resource Commands

```bash
# View resource usage of pods
kubectl top pods -n production
# NAME          CPU(cores)   MEMORY(bytes)
# api-abc123    245m         312Mi
# worker-def456 1200m        896Mi   ← near CPU limit if limit=1500m

# View resource usage of nodes
kubectl top nodes
# NAME        CPU(cores)  CPU%   MEMORY(bytes)  MEMORY%
# worker-1    3200m       40%    18Gi           56%
# worker-2    7100m       89%    28Gi           88%   ← near capacity

# View requests and limits on pods
kubectl get pods -n production -o custom-columns=\
  "NAME:.metadata.name,\
  CPU_REQ:.spec.containers[0].resources.requests.cpu,\
  CPU_LIM:.spec.containers[0].resources.limits.cpu,\
  MEM_REQ:.spec.containers[0].resources.requests.memory,\
  MEM_LIM:.spec.containers[0].resources.limits.memory"

# Check node allocatable vs requested
kubectl describe node worker-1 | grep -A 20 "Allocated resources"
# Allocated resources:
#   (Total limits may be over 100 percent, i.e., overcommitted.)
#   Resource  Requests   Limits
#   cpu       3200m(40%) 8000m(100%)  ← limits overcommitted
#   memory    18Gi(56%)  36Gi(112%)   ← limits overcommitted (OK)

# Find pods without resource requests (dangerous for scheduling)
kubectl get pods -A -o json | jq '.items[] |
  select(.spec.containers[].resources.requests == null) |
  .metadata.namespace + "/" + .metadata.name'

# Check CPU throttling metrics (requires Prometheus)
# container_cpu_cfs_throttled_periods_total
# container_cpu_cfs_periods_total
# Throttle ratio = throttled / total — alert at > 25%
```

---

### 🎤 Short Crisp Interview Answer

> *"Requests and limits serve completely different purposes. Requests are used by the scheduler to find a node with sufficient free capacity — they're the reserved minimum. Limits are enforced by Linux kernel cgroups as a hard ceiling. The critical difference is what happens when exceeded: CPU over limit means the container is throttled by CFS bandwidth control — slowed but not killed, which causes latency spikes. Memory over limit means the container is OOMKilled with signal 137. CPU can be overcommitted in limits, memory usually should not be. For latency-sensitive services I often set no CPU limit — only a request — to prevent throttling artifacts while still getting correct scheduling. Memory always needs both request and limit."*

---

### ⚠️ Gotchas

1. **CPU throttling is invisible in kubectl top** — `kubectl top pod` shows CPU usage in cores, not throttling percentage. A pod at 200m might be throttled 40% of the time if its limit is 300m. Use `container_cpu_cfs_throttled_seconds_total` Prometheus metric.
2. **No limits = BestEffort QoS = first evicted** — containers with no requests or limits get BestEffort QoS and are the first to be evicted under node pressure (topic 6.5).
3. **Memory limit should equal memory request for critical services** — setting limit > request means the container can use more memory until the node is stressed, at which point it gets killed. For databases, set them equal (Guaranteed QoS).
4. **Limit higher than node capacity means pod never schedules** — a pod with CPU request 100m on a 4-core node is fine. But a pod with CPU request 10 cores on a 4-core node is permanently Pending.
5. **Java JVM ignores cgroup limits by default (pre-Java 10)** — old JVMs size the heap based on total physical memory of the node, not the container's memory limit. Result: JVM allocates more heap than the container limit and gets OOMKilled. Use `-XX:+UseContainerSupport` (Java 10+, on by default Java 11+).

---

### Connections
- QoS classes derived from requests/limits — **6.5**
- LimitRange sets defaults when requests/limits omitted — **6.3**
- ResourceQuota caps total requests/limits per namespace — **6.4**
- Node pressure eviction based on actual usage vs limits — **6.14**
- CPU Manager for exclusive CPU pinning — **6.15**

---

---

# 6.2 Namespaces — Isolation and Organization

## 🟢 Beginner

### What it is in simple terms

A Namespace is a **logical partition within a Kubernetes cluster** that provides a scope for names, resource isolation, and policy application. The same resource name can exist in multiple namespaces. Most Kubernetes resources are namespaced; a few (Nodes, PVs, ClusterRoles) are cluster-scoped.

---

### What Namespaces Provide

```
NAMESPACE CAPABILITIES
═══════════════════════════════════════════════════════════════

1. NAME SCOPING:
   Two Deployments named "api" can coexist in different namespaces.
   production/api and staging/api are separate objects.

2. RESOURCE ISOLATION (soft):
   ResourceQuota limits total resources per namespace.
   LimitRange sets defaults per namespace.
   Namespace ≠ hard compute isolation (that requires node pools).

3. POLICY SCOPE:
   RBAC Roles apply within a namespace.
   NetworkPolicy applies within a namespace.
   LimitRange applies within a namespace.
   ResourceQuota applies within a namespace.

4. DNS SCOPING:
   service.namespace.svc.cluster.local (FQDN)
   Short name "service" resolves within the same namespace.

WHAT NAMESPACES DO NOT PROVIDE:
  ✗ Network isolation (pods in different namespaces can talk freely
    without NetworkPolicy)
  ✗ Node isolation (pods from all namespaces share nodes)
  ✗ Security isolation (without RBAC + NetworkPolicy + PSA)
  ✗ Performance isolation (all pods share the same scheduler)
```

---

### Default Namespaces

```
FOUR DEFAULT NAMESPACES:

default:
  Where resources go when no namespace is specified.
  kubectl get pods  → looks in "default" namespace
  Production workloads should NOT use "default" namespace.
  Use dedicated namespaces: production, staging, monitoring, etc.

kube-system:
  Kubernetes system components:
    kube-apiserver, kube-scheduler, kube-controller-manager (static pods)
    CoreDNS, kube-proxy (Deployments/DaemonSets)
    cloud-controller-manager, CNI components
  Do not deploy application workloads here.

kube-public:
  Publicly readable (even unauthenticated users).
  Contains cluster-info ConfigMap (bootstrap token discovery).
  Rarely used in practice.

kube-node-lease:
  Node heartbeat Lease objects.
  Each node has a Lease object here, updated every ~10s by kubelet.
  API Server uses these to detect node health faster than old method.
  Do not touch.
```

---

### Namespace YAML and Commands

```yaml
# Create namespace with labels (good practice)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
    kubernetes.io/metadata.name: production   # auto-added by K8s
  annotations:
    contact: platform-team@company.com
    cost-center: "1234"
```

```bash
# Create namespace
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace monitoring

# List all namespaces
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   30d
# kube-node-lease   Active   30d
# kube-public       Active   30d
# kube-system       Active   30d
# monitoring        Active   5d
# production        Active   10d
# staging           Active   10d

# Set default namespace for current context (avoid -n flag every time)
kubectl config set-context --current --namespace=production
# Now: kubectl get pods → looks in production namespace

# View everything in a namespace
kubectl get all -n production

# View resources across ALL namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # shorthand

# Resources that are NOT namespaced (cluster-scoped)
kubectl api-resources --namespaced=false
# NAME                    SHORTNAMES   APIVERSION   NAMESPACED
# nodes                   no           v1           false
# persistentvolumes       pv           v1           false
# clusterroles                         rbac...      false
# clusterrolebindings                  rbac...      false
# storageclasses          sc           storage...   false
# namespaces              ns           v1           false

# Delete namespace (deletes ALL resources inside it!)
kubectl delete namespace staging
# ⚠️ This cascade-deletes every pod, deployment, service, secret,
#    configmap, PVC in the namespace. Irreversible.
```

---

### Namespace Organization Patterns

```
COMMON PATTERNS:
═══════════════════════════════════════════════════════════════

BY ENVIRONMENT:
  production    ← prod workloads
  staging       ← pre-prod testing
  development   ← dev workloads
  Simple but mixes all teams in each namespace.

BY TEAM (recommended for medium/large orgs):
  team-payments-prod
  team-payments-staging
  team-auth-prod
  team-auth-staging
  Each team owns their namespaces. Quota per team.

BY APPLICATION:
  myapp-production
  myapp-staging
  myapp-development
  Good for monolithic apps or small teams.

SYSTEM NAMESPACES:
  monitoring    ← Prometheus, Grafana, AlertManager
  logging       ← Fluent Bit, Loki, Elasticsearch
  ingress       ← Nginx/ALB Ingress Controllers
  cert-manager  ← cert-manager
  external-secrets ← ESO controller
  argocd        ← Argo CD
  istio-system  ← Istio control plane
```

---

### 🎤 Short Crisp Interview Answer

> *"Namespaces provide logical partitioning within a cluster — name scoping, resource quota boundaries, and policy scope for RBAC, NetworkPolicy, and LimitRange. The four default namespaces are: default for unspecified resources, kube-system for K8s components, kube-public for cluster bootstrap info, and kube-node-lease for node heartbeats. Key thing is that namespaces are soft isolation — pods in different namespaces still run on shared nodes and can talk to each other without NetworkPolicy. For hard isolation you need separate node pools, NetworkPolicy, and Pod Security Admission."*

---

### ⚠️ Gotchas

1. **Deleting a namespace cascade-deletes everything in it** — all Deployments, Pods, Services, Secrets, ConfigMaps, PVCs. There is no recycle bin. Always confirm before `kubectl delete namespace`.
2. **Namespace deletion can hang** — if a namespace has resources with finalizers (CRDs, PVCs with Retain policy), it stays in Terminating state indefinitely. Must manually remove finalizers.
3. **Cross-namespace DNS requires FQDN** — service `myapp` in namespace `staging` is reached from `production` namespace as `myapp.staging.svc.cluster.local`, not just `myapp`.

---

---

# 6.3 LimitRange — Defaults Per Namespace

## 🟢 Beginner

### What it is in simple terms

A LimitRange is a **policy object that sets default resource requests/limits and min/max constraints** for containers, pods, or PVCs within a namespace. When a container is created without explicit resource specifications, LimitRange automatically injects the defaults.

---

### Why LimitRange Exists

```
THE PROBLEM: CONTAINERS WITHOUT RESOURCES
═══════════════════════════════════════════════════════════════

Developer creates pod without resource spec:
  containers:
  - name: app
    image: myapp:v1
    # no resources: block

  Result without LimitRange:
    Container has no request → scheduler treats it as 0 CPU, 0 memory
    Container has no limit → can use ALL node resources
    Multiple such containers = node resource starvation
    One misbehaving pod can starve all other pods on the node

  Result WITH LimitRange:
    Container automatically gets default request and limit
    Guaranteed some minimum resource guarantee
    Capped at maximum — can't monopolize node resources
    Developer doesn't need to know resource values (bad for them
    but protects the cluster from runaway containers)
```

---

### LimitRange YAML — All Scope Types

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:

  # ── Container-level limits ───────────────────────────────
  - type: Container
    # Default limit (applied if container has no limits:)
    default:
      cpu: "500m"
      memory: "256Mi"
    # Default request (applied if container has no requests:)
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    # Hard minimum — request/limit cannot be BELOW this
    min:
      cpu: "10m"
      memory: "32Mi"
    # Hard maximum — request/limit cannot EXCEED this
    max:
      cpu: "4"           # 4 CPUs max per container
      memory: "4Gi"      # 4 GiB max per container
    # Max limit-to-request ratio (prevents extreme overcommit)
    maxLimitRequestRatio:
      cpu: "10"          # limit cannot be > 10x request
      memory: "4"        # limit cannot be > 4x request

  # ── Pod-level limits (sum of all containers) ─────────────
  - type: Pod
    max:
      cpu: "8"           # all containers in pod cannot exceed 8 CPU
      memory: "8Gi"      # all containers in pod cannot exceed 8 GiB

  # ── PVC size limits ───────────────────────────────────────
  - type: PersistentVolumeClaim
    min:
      storage: "1Gi"     # PVC cannot request less than 1Gi
    max:
      storage: "50Gi"    # PVC cannot request more than 50Gi
```

---

### kubectl LimitRange Commands

```bash
# Create LimitRange
kubectl apply -f limitrange.yaml -n production

# View LimitRanges in namespace
kubectl get limitrange -n production
kubectl describe limitrange production-limits -n production
# Type       Resource  Min   Max   Default Request  Default Limit  Max L/R Ratio
# Container  cpu       10m   4     100m             500m           10
# Container  memory    32Mi  4Gi   128Mi            256Mi          4

# Test: create pod WITHOUT resource spec in limited namespace
kubectl run test --image=nginx -n production
# Pod gets defaultRequest (100m CPU, 128Mi mem) and
# defaultLimit (500m CPU, 256Mi mem) automatically injected

kubectl describe pod test -n production | grep -A 6 Limits
# Limits:
#   cpu:     500m     ← injected by LimitRange
#   memory:  256Mi
# Requests:
#   cpu:     100m     ← injected by LimitRange
#   memory:  128Mi

# Test: create pod exceeding max
kubectl run toobig --image=nginx \
  --limits='cpu=8,memory=10Gi' -n production
# Error: pods "toobig" is forbidden: maximum cpu usage per Container is 4,
#        but limit is 8
```

---

### 🎤 Short Crisp Interview Answer

> *"LimitRange is a namespace-level policy that injects default resource requests and limits into containers that don't specify them, and enforces min/max boundaries on what containers can request. Without it, containers without resource specs can starve the node. It also enforces the maxLimitRequestRatio to prevent extreme CPU overcommit — for example requiring that CPU limit cannot exceed 10x the CPU request. LimitRange operates at admission time — it's applied when the pod is created, not at runtime."*

---

### ⚠️ Gotchas

1. **LimitRange applies at admission — not retroactively** — existing pods are not affected when you create or change a LimitRange. Only new pods created after the LimitRange is applied get the defaults.
2. **Default only applies if container has NO resources block** — if a container specifies `requests: {cpu: "100m"}` but no limits, LimitRange injects the default limits. But it does NOT inject a default for cpu requests because the container already has a requests block.
3. **Injected defaults are invisible in pod YAML until created** — you won't see the injected values in your Deployment YAML. Check with `kubectl describe pod` after creation.

---

---

# 6.4 ResourceQuota — Namespace-Level Caps

## 🟢 Beginner

### What it is in simple terms

A ResourceQuota is a **hard ceiling on the total resource consumption** of a namespace — total CPU requested, total memory requested, total number of pods, PVCs, services, etc. It prevents one team or namespace from consuming all cluster resources.

---

### ResourceQuota YAML — Full Production Example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resource totals (REQUESTS — what scheduler reserves)
    requests.cpu: "20"          # all pods in namespace can request max 20 CPU
    requests.memory: "40Gi"     # all pods can request max 40 GiB RAM

    # Compute resource totals (LIMITS — what kernel caps at)
    limits.cpu: "40"            # all pods' CPU limits cannot exceed 40 CPU
    limits.memory: "80Gi"       # all pods' memory limits cannot exceed 80 GiB

    # Object count limits
    pods: "100"                 # max 100 pods in namespace
    services: "20"              # max 20 Services
    services.loadbalancers: "5" # max 5 LoadBalancer services (expensive!)
    services.nodeports: "5"     # max 5 NodePort services
    persistentvolumeclaims: "20"# max 20 PVCs
    secrets: "50"               # max 50 Secrets
    configmaps: "50"            # max 50 ConfigMaps

    # Storage quotas
    requests.storage: "500Gi"   # total storage across all PVCs
    gold.storageclass.storage.k8s.io/requests.storage: "100Gi"
    # ^ per-StorageClass storage quota (prevents expensive storage overuse)

---
# Scoped ResourceQuota — applies only to specific pod types
apiVersion: v1
kind: ResourceQuota
metadata:
  name: burst-quota
  namespace: production
spec:
  hard:
    pods: "10"
    requests.cpu: "10"
  # Only applies to pods that match the scope
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]  # quota for high-priority pods only
```

---

### ResourceQuota Enforcement

```bash
# View current quota usage
kubectl describe resourcequota production-quota -n production
# Name:            production-quota
# Namespace:       production
# Resource                   Used    Hard
# ──────────────────────────────────────────
# limits.cpu                 8       40
# limits.memory              16Gi    80Gi
# pods                       23      100
# requests.cpu               4       20
# requests.memory            8Gi     40Gi
# services                   8       20
# persistentvolumeclaims     5       20

# Test quota enforcement
# Try to create a deployment that would exceed quota:
kubectl create deployment bigapp --image=nginx \
  --replicas=90 -n production
# (Would create 90 pods) → fails when hitting pod limit of 100

# When requests.cpu quota is set, ALL pods MUST have cpu requests
kubectl run nospec --image=nginx -n production
# Error: pods "nospec" is forbidden: failed quota: production-quota:
#        must specify requests.cpu, requests.memory

# ← This is an important side-effect of ResourceQuota:
#    Once requests.cpu quota is set, every container must explicitly
#    set CPU requests (or LimitRange must provide defaults)

# View quota across all namespaces
kubectl get resourcequota --all-namespaces
```

---

### LimitRange vs ResourceQuota

```
LIMITRANGE vs RESOURCEQUOTA — COMPLEMENTARY, NOT ALTERNATIVES
═══════════════════════════════════════════════════════════════

LimitRange:
  Scope:    Per-container or per-pod
  What:     Min/max per container, default requests/limits
  When:     Applied at pod admission (per object)
  Enforces: Individual container resource bounds

ResourceQuota:
  Scope:    Whole namespace
  What:     Total sum of resources across all objects
  When:     Checked at every object creation/update
  Enforces: Aggregate namespace resource consumption

USED TOGETHER (recommended):
  LimitRange:    ensures every container has sane defaults
  ResourceQuota: ensures the whole namespace stays in budget

  Without LimitRange:
    ResourceQuota with requests.cpu requires every pod to
    specify CPU requests — or they fail. LimitRange provides
    those defaults automatically.
```

---

### 🎤 Short Crisp Interview Answer

> *"ResourceQuota sets hard limits on aggregate resource consumption within a namespace — total CPU requested, total memory, total pods, services, PVCs, and more. When a namespace has a requests.cpu quota, every pod creation must specify CPU requests — otherwise it's rejected. This is why LimitRange and ResourceQuota are used together: LimitRange provides the defaults, ResourceQuota enforces the aggregate cap. ResourceQuota is essential for multi-tenant clusters to prevent one team from consuming all cluster resources."*

---

### ⚠️ Gotchas

1. **requests.cpu quota forces explicit requests on all pods** — once you set a requests.cpu quota, every new pod must have CPU requests or it fails admission. Combine with LimitRange to inject defaults automatically.
2. **ResourceQuota does not apply to existing objects** — creating a ResourceQuota doesn't evict existing pods that exceed it. It only blocks new object creation.
3. **services.loadbalancers is per-namespace, not per-cluster** — you may need to set this low to prevent runaway cloud LB costs.

---

---

# ⚠️ 6.5 QoS Classes — Guaranteed, Burstable, BestEffort

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

QoS (Quality of Service) class is a **label Kubernetes automatically assigns to every pod** based on its resource requests and limits. It determines the eviction priority under node memory pressure — which pod gets killed first when the node runs out of memory.

---

### The Three QoS Classes

```
QOS CLASS DETERMINATION RULES
═══════════════════════════════════════════════════════════════

GUARANTEED (highest priority — evicted last):
  Rule: EVERY container in the pod must have:
    requests.cpu    set AND equals limits.cpu
    requests.memory set AND equals limits.memory
  (if only limit is set, request is automatically set equal to limit)

  Example:
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "500m"       ← must equal request
        memory: "512Mi"   ← must equal request

  ✓ Kernel gives this pod dedicated, non-sharable CPU/memory
  ✓ Evicted last under memory pressure
  ✓ OOM score adj = -998 (kernel less likely to kill)
  Use for: Databases, critical stateful services, latency-sensitive APIs

BURSTABLE (medium priority — evicted second):
  Rule: At least one container has cpu OR memory request/limit set
        BUT does not meet Guaranteed criteria

  Examples:
    - requests set but limits different (limit > request)
    - requests set but no limits (no ceiling)
    - limits set but no requests (limit copied to request)
    - only some containers have resources set

  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "1000m"     ← different from request → Burstable
      memory: "512Mi"  ← different from request → Burstable

  ✓ Can use more than request if node has spare capacity
  ✗ Evicted before Guaranteed pods when node is under pressure
  Use for: Most stateless applications, web servers, workers

BESTEFFORT (lowest priority — evicted first):
  Rule: NO container in the pod has ANY requests or limits set

  resources: {}   ← nothing set
  (or no resources block at all)

  ✗ Gets whatever CPU/memory happens to be available
  ✗ First to be evicted under ANY memory pressure
  ✗ OOM score adj = 1000 (kernel very likely to kill)
  Use for: Dev/test, truly non-critical batch jobs
           AVOID in production
```

---

### QoS and OOM Score

```
OOM SCORE ADJUSTER — HOW KERNEL DECIDES WHAT TO KILL
═══════════════════════════════════════════════════════════════

Linux OOM score: 0-1000. Higher = more likely to be killed by OOM killer.
Kubelet sets OOM score adj for containers based on QoS:

  BestEffort containers:    oom_score_adj = 1000
    → Kernel kills these FIRST under memory pressure

  Burstable containers:     oom_score_adj = 2 to 999
    → Score proportional to: 1000 × (container memory request / node capacity)
    → Containers using more than their request score higher (more killable)

  Guaranteed containers:    oom_score_adj = -998
    → Kernel almost never kills these
    → Only killed if nothing else available AND pod is over its limit

EVICTION ORDER (kubelet eviction, not OOM killer):
  1. BestEffort pods (no resources)
  2. Burstable pods exceeding their requests
  3. Burstable pods within their requests
  4. Guaranteed pods (only if over their own limits)

PRACTICAL IMPLICATION:
  Cluster under memory pressure → BestEffort pods evicted first
  Then Burstable pods that went over requests
  Guaranteed pods survive until the very end
  This is why production databases should be Guaranteed QoS
```

---

### Checking QoS Class

```bash
# View QoS class of a pod
kubectl describe pod myapp -n production | grep QoS
# QoS Class: Guaranteed

# View QoS via jsonpath
kubectl get pod myapp -n production \
  -o jsonpath='{.status.qosClass}'
# Guaranteed

# Find all BestEffort pods (no resources — dangerous in production)
kubectl get pods -A -o json | jq '
  .items[] |
  select(.status.qosClass == "BestEffort") |
  .metadata.namespace + "/" + .metadata.name'

# Find all pods and their QoS class
kubectl get pods -A \
  -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,QOS:.status.qosClass"
```

---

### QoS Class YAML Examples

```yaml
# GUARANTEED QoS — requests == limits for all containers
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
  - name: postgres
    image: postgres:15
    resources:
      requests:
        cpu: "1000m"
        memory: "2Gi"
      limits:
        cpu: "1000m"     # exact match = Guaranteed
        memory: "2Gi"    # exact match = Guaranteed

---
# BURSTABLE QoS — requests < limits
apiVersion: v1
kind: Pod
metadata:
  name: web-api
spec:
  containers:
  - name: api
    image: myapi:v1
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "1000m"     # limit > request = Burstable
        memory: "1Gi"    # limit > request = Burstable

---
# BESTEFFORT — no resources (AVOID in production)
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  containers:
  - name: job
    image: mybatch:v1
    # No resources block = BestEffort
```

---

### 🎤 Short Crisp Interview Answer

> *"QoS class is automatically assigned based on resource spec and determines eviction priority under node memory pressure. Guaranteed — every container has matching requests and limits — is evicted last and has the kernel OOM score of -998, making it almost unkillable. Burstable — requests set but not matching limits, or mixed containers — is evicted second. BestEffort — no resources at all — gets OOM score 1000 and is killed first. The practical implication: databases and critical stateful services should be Guaranteed QoS with matching requests and limits. Web APIs should be Burstable with limit higher than request to allow bursting. Never deploy production services as BestEffort."*

---

### ⚠️ Gotchas

1. **If you only set limits, requests auto-match** — `limits: {cpu: 500m}` with no requests block means Kubernetes copies limits to requests. This makes it Guaranteed QoS. Many engineers don't realize this.
2. **Mixed-container pods follow the worst container's QoS** — if one container is Guaranteed and another has no resources (BestEffort), the whole pod is BestEffort. All containers must meet Guaranteed criteria.
3. **Guaranteed does not mean unlimited** — Guaranteed QoS just means evicted last. The container still gets OOMKilled if it exceeds its own memory limit.

---

### Connections
- Requests/limits determine QoS class — **6.1**
- Eviction under node pressure — **6.14**
- Guaranteed QoS used for CPU Manager static policy — **6.15**

---

---

# 6.6 Node Affinity & Anti-Affinity

## 🟡 Intermediate

### What it is in simple terms

Node Affinity is a **set of rules that constrain which nodes a pod can or should be scheduled on**, based on node labels. It is the successor to the legacy `nodeSelector` field, offering richer expression syntax including `In`, `NotIn`, `Exists`, `Gt`, `Lt` operators.

---

### Node Affinity vs nodeSelector

```
EVOLUTION: nodeSelector → Node Affinity
═══════════════════════════════════════════════════════════════

nodeSelector (legacy — still works, limited):
  nodeSelector:
    kubernetes.io/os: linux
    node.kubernetes.io/instance-type: m5.xlarge
  Exact match only. AND logic between all selectors.
  Simple but inflexible.

Node Affinity (modern — prefer this):
  Supports: In, NotIn, Exists, DoesNotExist, Gt, Lt
  Supports: required (hard) AND preferred (soft) rules
  Supports: OR logic within a nodeSelectorTerm
  More expressive. Same scheduler outcome, more flexibility.
```

---

### Node Affinity Types

```
TWO TYPES OF NODE AFFINITY:

requiredDuringSchedulingIgnoredDuringExecution:
  REQUIRED = hard rule — pod will NOT schedule if not satisfied
  IgnoredDuringExecution = if node labels change AFTER pod is
  running, the pod is NOT evicted (it keeps running)
  Use: must run on certain node types (GPU nodes, specific AZ)

preferredDuringSchedulingIgnoredDuringExecution:
  PREFERRED = soft rule — scheduler tries to satisfy, skips if not
  Scores nodes: satisfying preference = higher score
  Pod WILL schedule even if no nodes satisfy the preference
  weight: 1-100 — how much this preference matters in scoring
  Use: try to run in same AZ for latency, but run anywhere if needed
```

---

### Node Affinity YAML

```yaml
spec:
  affinity:
    nodeAffinity:

      # REQUIRED: pod ONLY schedules on nodes matching this
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        # Multiple nodeSelectorTerms = OR logic (any one can match)
        - matchExpressions:
          # Multiple matchExpressions in one term = AND logic
          - key: kubernetes.io/os
            operator: In
            values: ["linux"]
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["m5.xlarge", "m5.2xlarge", "m5.4xlarge"]
            # Pod schedules on Linux m5.xlarge OR m5.2xlarge OR m5.4xlarge
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a", "us-east-1b"]
            # AND must be in us-east-1a or us-east-1b

        # Second term (OR with first term)
        - matchExpressions:
          - key: node-role
            operator: In
            values: ["gpu"]

      # PREFERRED: scheduler tries but not required
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80                 # high weight = strongly preferred
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a"]   # prefer us-east-1a for locality

      - weight: 20
        preference:
          matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["m5.2xlarge"]   # mildly prefer larger instance type

---
# Common real-world example: Schedule on GPU nodes only
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nvidia.com/gpu
            operator: Exists           # node must have this label (any value)

---
# Avoid spot/preemptible instances for critical workloads
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node.kubernetes.io/lifecycle
            operator: NotIn
            values: ["spot", "preemptible"]
```

---

### Common Node Labels Reference

```bash
# View all labels on a node
kubectl get node worker-1 --show-labels

# Common well-known node labels:
kubernetes.io/hostname: worker-1
kubernetes.io/os: linux
kubernetes.io/arch: amd64
topology.kubernetes.io/zone: us-east-1a
topology.kubernetes.io/region: us-east-1
node.kubernetes.io/instance-type: m5.xlarge
node.kubernetes.io/lifecycle: spot          # AWS Spot instances
eks.amazonaws.com/nodegroup: general-workers
eks.amazonaws.com/capacityType: SPOT        # EKS specific
nvidia.com/gpu: "true"                      # GPU nodes (label manually)
node-role.kubernetes.io/control-plane: ""   # control plane nodes

# Label a node manually
kubectl label node worker-3 \
  workload-type=gpu \
  hardware-gen=gen3-nvme

# Remove a label
kubectl label node worker-3 workload-type-
```

---

### 🎤 Short Crisp Interview Answer

> *"Node affinity constrains which nodes a pod schedules on using node label selectors. There are two types: required — a hard constraint the pod MUST satisfy or it stays Pending — and preferred — a soft preference the scheduler tries to satisfy with weighted scoring but ignores if not met. The key operators are In, NotIn, Exists, DoesNotExist. Multiple matchExpressions in one nodeSelectorTerm are AND logic; multiple nodeSelectorTerms are OR logic. Common uses include restricting pods to GPU nodes, specific AZs, non-spot instances, or nodes with local NVMe disks."*

---

### ⚠️ Gotchas

1. **IgnoredDuringExecution means no eviction on label change** — if you change a node label after pods are running, those pods are NOT evicted even if they no longer match the affinity. Pod continues running until manually removed.
2. **nodeSelectorTerms = OR, matchExpressions = AND** — a common source of bugs. Multiple items in `matchExpressions` are AND. Multiple items in `nodeSelectorTerms` (the outer list) are OR.
3. **Required affinity + no matching nodes = pod stuck in Pending** — if you add required affinity for a label that no node has, all pod replicas stay Pending. Check `kubectl describe pod` for scheduling failures.

---

---

# ⚠️ 6.7 Pod Affinity & Anti-Affinity

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Pod Affinity and Anti-Affinity express **rules about how pods should be placed relative to OTHER pods**, not relative to node labels. Affinity: "schedule me near pods with this label" (co-location). Anti-affinity: "schedule me AWAY from pods with this label" (spreading).

---

### Why Pod Affinity/Anti-Affinity Exists

```
USE CASES
═══════════════════════════════════════════════════════════════

POD AFFINITY (co-location):
  Web server should run on same node as its Redis cache
  → Eliminates network hop → lower latency
  Sidecar should run on same node as its primary
  → Guaranteed co-location for DaemonSet-like behavior

POD ANTI-AFFINITY (spreading):
  Database replicas must NOT run on the same node
  → One node failure kills at most one replica
  → High availability through physical separation

  Web API replicas must run on different nodes
  → Single node failure affects only a portion of capacity

  Database primary and replica must be in different AZs
  → AZ failure doesn't take down both primary and replica
```

---

### Pod Affinity/Anti-Affinity YAML

```yaml
spec:
  affinity:
    # POD AFFINITY — schedule near pods matching selector
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["redis-cache"]   # co-locate with redis pods
        topologyKey: kubernetes.io/hostname
        # topologyKey: defines the "unit" of co-location
        # hostname: same NODE as the redis pod
        # zone: same AZ as the redis pod
        # region: same region as the redis pod

    # POD ANTI-AFFINITY — schedule AWAY from pods matching selector
    podAntiAffinity:
      # REQUIRED: hard anti-affinity (strict separation)
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["my-database"]  # anti-affinity to own app label
        topologyKey: kubernetes.io/hostname
        # Result: no two "my-database" pods on the same node
        # If only 2 nodes and 3 replicas: 3rd pod stays PENDING

      # PREFERRED: soft anti-affinity (try to spread, continue if not possible)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: ["my-api"]
          topologyKey: topology.kubernetes.io/zone
          # Result: try to spread api pods across AZs
          # If only one AZ available: schedules anyway (preferred, not required)

---
# PRODUCTION PATTERN: HA for stateless service
# Spread replicas across nodes and AZs (soft — not strict)
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: ["my-api"]         # anti-affinity to self
          topologyKey: kubernetes.io/hostname  # try different nodes

      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: ["my-api"]
          topologyKey: topology.kubernetes.io/zone  # also try different AZs

---
# PRODUCTION PATTERN: DB HA with strict node separation
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: postgres
            role: replica
        topologyKey: kubernetes.io/hostname
      # Hard rule: replicas must be on different nodes
      # Trade-off: can't scale replicas beyond node count
```

---

### topologyKey — The Critical Field

```
topologyKey DETERMINES WHAT "SAME LOCATION" MEANS
═══════════════════════════════════════════════════════════════

kubernetes.io/hostname:
  Unit = individual node
  Anti-affinity: no two pods on the same node
  Affinity: same node as target pod

topology.kubernetes.io/zone:
  Unit = availability zone (us-east-1a, us-east-1b, us-east-1c)
  Anti-affinity: no two pods in the same AZ
  Affinity: same AZ as target pod

topology.kubernetes.io/region:
  Unit = region (us-east-1, us-west-2)
  Anti-affinity: no two pods in the same region
  (rarely useful — most clusters are single-region)

Custom topology key:
  You can use any node label as topology key
  Example: node-role=gpu → "same GPU pool"
  Example: rack=rack-A → "same physical rack"

⚠️  topologyKey MUST exist as a label on the nodes
    If the key doesn't exist on nodes → scheduler treats all nodes
    as same topology unit → anti-affinity becomes meaningless
```

---

### Pod Affinity vs Topology Spread Constraints

```
WHEN TO USE WHICH:
  Pod Anti-Affinity: "no two of MY pods on the same node/zone"
    All-or-nothing: strict separation or pending
    Simple: one rule, clear semantics

  Topology Spread Constraints (6.10): "spread my pods evenly"
    Allows some imbalance (maxSkew tolerance)
    More flexible: specify acceptable spread percentage
    Better for large replica counts
    Use this for most production spreading needs

  Both can be used together for fine-grained control.
```

---

### 🎤 Short Crisp Interview Answer

> *"Pod affinity and anti-affinity control pod placement relative to other pods. Affinity co-locates pods — for example, schedule my web server on the same node as Redis to eliminate the network hop. Anti-affinity spreads pods — for example, ensure no two database replicas run on the same node so a node failure doesn't kill multiple replicas. The topologyKey field is critical — it defines what 'same location' means: kubernetes.io/hostname means same node, topology.kubernetes.io/zone means same AZ. Required rules are hard — the pod stays Pending if unsatisfiable. Preferred rules are soft — scheduler tries but proceeds if it can't satisfy them."*

---

### ⚠️ Gotchas

1. **Required anti-affinity can block scaling** — if you have 3 nodes and required anti-affinity with topologyKey=hostname, you cannot scale beyond 3 replicas. The 4th pod is permanently Pending. Use preferred anti-affinity unless you truly need the hard guarantee.
2. **Pod affinity is expensive at scale** — the scheduler must check all running pods to evaluate affinity rules. With thousands of pods and complex affinity rules, scheduling latency increases significantly. Prefer Topology Spread Constraints for large clusters.
3. **Anti-affinity to self requires matching own labels** — the label selector in podAntiAffinity must match the pod's own labels for self-spreading to work. Using `app: my-api` in both the selector and the pod labels is the standard pattern.

---

---

# ⚠️ 6.8 Taints & Tolerations

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

**Taints** are applied to nodes to **repel pods**. **Tolerations** are applied to pods to **allow them to schedule on tainted nodes**. The combination lets you reserve certain nodes for specific workloads — for example, dedicating GPU nodes only to GPU workloads, or isolating critical infrastructure on dedicated nodes.

---

### Taint and Toleration Structure

```
TAINT STRUCTURE: key=value:effect
═══════════════════════════════════════════════════════════════

  key:     Identifier (any string)
  value:   Optional qualifier (any string)
  effect:  What happens to pods that DON'T tolerate this taint

THREE TAINT EFFECTS:
  NoSchedule:
    New pods WITHOUT matching toleration: NOT scheduled on this node
    Existing pods WITHOUT toleration:     STAY running (not evicted)
    Use: Reserve node for specific workloads, don't disrupt running pods

  PreferNoSchedule:
    Soft version of NoSchedule
    Scheduler tries to avoid this node but will use it if no choice
    Use: "soft" node reservation, graceful migration

  NoExecute:
    New pods WITHOUT toleration: NOT scheduled
    Existing pods WITHOUT toleration: EVICTED (within tolerationSeconds)
    Use: Node maintenance, draining, node health issues

TOLERATION STRUCTURE:
  key:      Must match taint key (or empty for any key)
  operator: Equal (match key+value) or Exists (match key only)
  value:    Must match taint value (if operator: Equal)
  effect:   Must match taint effect (or empty to match all effects)
  tolerationSeconds: How long to tolerate NoExecute before evicted
```

---

### Taint & Toleration YAML

```yaml
# Apply taint to a node (kubectl)
# kubectl taint nodes node-1 dedicated=gpu:NoSchedule
# kubectl taint nodes node-1 team=payments:NoSchedule
# kubectl taint nodes worker-3 spot-instance=true:NoExecute

# Remove taint (add trailing dash)
# kubectl taint nodes node-1 dedicated=gpu:NoSchedule-

---
# Pod with toleration — can schedule on gpu-dedicated node
spec:
  tolerations:
  # Exact match: key + value + effect
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  # Exists match: just key (any value)
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"

  # Match any taint with this key (any effect)
  - key: "team"
    operator: "Equal"
    value: "payments"
    effect: ""             # empty = matches all effects

  # Tolerate all taints (dangerous — use only for system DaemonSets)
  - operator: "Exists"    # matches ALL taints on any node

  # NoExecute with tolerationSeconds — stay on tainted node up to 60s
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 60  # pod evicted after 60s if node stays not-ready

---
# DaemonSet example — must tolerate all node taints to run everywhere
spec:
  tolerations:
  # Tolerate master/control plane taint
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "node-role.kubernetes.io/master"     # older K8s versions
    operator: "Exists"
    effect: "NoSchedule"
  # Tolerate not-ready and unreachable (stay running during node issues)
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
```

---

### Common Real-World Taint Patterns

```bash
# Pattern 1: Dedicate GPU nodes to GPU workloads only
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
kubectl taint nodes gpu-node-2 nvidia.com/gpu=true:NoSchedule
# Only pods with matching toleration schedule on gpu-node-1/2
# Combine with node affinity for GPU pods to prefer GPU nodes:
#   nodeAffinity: prefer/require nodes with nvidia.com/gpu label

# Pattern 2: Dedicated node for monitoring stack
kubectl taint nodes monitor-node-1 dedicated=monitoring:NoSchedule
# Add toleration to Prometheus, Grafana, AlertManager pods only

# Pattern 3: Drain node for maintenance (NoExecute evicts pods)
kubectl taint nodes worker-2 maintenance=true:NoExecute
# All pods without toleration evicted immediately
# Use tolerationSeconds for graceful eviction window

# Pattern 4: Spot instance taint (AWS)
# EKS managed node groups for spot instances automatically add:
kubectl taint nodes spot-node-1 \
  eks.amazonaws.com/capacityType=SPOT:NoSchedule
# Add toleration to batch jobs but not to critical services

# Pattern 5: Control plane taint (prevents app pods on masters)
# kubeadm automatically taints control plane nodes:
# node-role.kubernetes.io/control-plane:NoSchedule
# DaemonSets that need to run on control plane add toleration

# View taints on all nodes
kubectl get nodes -o custom-columns=\
  "NAME:.metadata.name,TAINTS:.spec.taints"
```

---

### Built-in System Tolerations

```
KUBERNETES AUTOMATICALLY ADDS THESE TOLERATIONS TO ALL PODS:
═══════════════════════════════════════════════════════════════

  node.kubernetes.io/not-ready:NoExecute (tolerationSeconds: 300)
    → Pod evicted after 300s if node becomes not-ready

  node.kubernetes.io/unreachable:NoExecute (tolerationSeconds: 300)
    → Pod evicted after 300s if node becomes unreachable

This is why pods are NOT immediately evicted when a node fails.
They wait 300 seconds (5 minutes) before rescheduling.
Reduces churn from brief network hiccups.

To make pods reschedule faster (at cost of more churn):
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30   # reschedule after only 30s

To make pods NEVER be evicted (DaemonSet, critical infra):
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    # No tolerationSeconds = tolerate indefinitely
```

---

### Taint vs Node Affinity — When to Use Which

```
TAINTS vs NODE AFFINITY — COMPLEMENTARY
═══════════════════════════════════════════════════════════════

TAINTS (node-centric — node repels pods):
  Applied to node to EXCLUDE most pods
  "This node is only for X workloads"
  Default behavior: pods don't schedule here
  Exception: pods that explicitly tolerate

NODE AFFINITY (pod-centric — pod selects nodes):
  Applied to pod to TARGET specific nodes
  "This pod wants to run on X type of node"
  Default behavior: pod can run anywhere
  Exception: pod prefers/requires specific node labels

USED TOGETHER (the standard pattern for node dedication):
  1. Taint GPU nodes: nvidia.com/gpu=true:NoSchedule
     → Prevents non-GPU pods from using GPU nodes (waste prevention)
  2. GPU pods add toleration: nvidia.com/gpu=true:NoSchedule
     → GPU pods CAN schedule on GPU nodes
  3. GPU pods add nodeAffinity: require nvidia.com/gpu=Exists label
     → GPU pods PREFER/REQUIRE GPU nodes (not just anywhere)

  Result: GPU nodes used exclusively by GPU workloads
  Taint = exclusion from the node side
  Affinity = attraction from the pod side
```

---

### 🎤 Short Crisp Interview Answer

> *"Taints are applied to nodes to repel pods, tolerations on pods allow scheduling on tainted nodes. Three taint effects: NoSchedule blocks new pods from scheduling without a matching toleration, PreferNoSchedule is a soft version, NoExecute evicts already-running pods that don't tolerate the taint. The common pattern is node dedication — taint GPU nodes with NoSchedule so only GPU workloads (which have the toleration) schedule there. Kubernetes automatically adds not-ready and unreachable tolerations with 300-second tolerance to all pods, which is why pods take 5 minutes to reschedule after a node failure. Taints and node affinity complement each other — taints exclude from the node side, affinity attracts from the pod side."*

---

### ⚠️ Gotchas

1. **NoSchedule doesn't evict existing pods** — if you taint a node with NoSchedule, pods already running on it continue running. Only NoExecute evicts running pods.
2. **Tolerating all taints (operator: Exists with no key)** — makes a pod schedule on ANY node including tainted ones. Correct for system DaemonSets but dangerous for app pods.
3. **Control plane nodes are tainted by default** — `node-role.kubernetes.io/control-plane:NoSchedule` prevents apps from running on control plane. DaemonSets that need control plane (log agents, monitors) must add this toleration.
4. **The 5-minute reschedule delay** — many engineers are surprised that pods don't immediately reschedule when a node fails. The 300-second default tolerationSeconds is the reason. Tune for your SLA requirements.

---

---

# ⚠️ 6.9 Priority Classes & Preemption

## 🟡 Intermediate

### What it is in simple terms

PriorityClasses let you **rank the importance of pods**. When the cluster is full and a high-priority pod needs to schedule, the scheduler can **preempt** (evict) lower-priority pods to make room. This ensures critical workloads always get scheduled even under resource pressure.

---

### PriorityClass YAML

```yaml
# System critical (built-in — DO NOT use in apps)
# system-cluster-critical: value 2000000000 (kube-system components)
# system-node-critical:    value 2000001000 (kubelet, node-critical daemonsets)

# Define your own PriorityClasses
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-production
value: 1000000           # higher = more important
globalDefault: false     # only one PriorityClass can be globalDefault: true
preemptionPolicy: PreemptLowerPriority  # default — can preempt lower pods
# preemptionPolicy: Never  → pod is high priority but won't preempt others
#                            (just gets scheduled before others waiting)
description: "Production critical services — database, auth, payments"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "Important production services — APIs, workers"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 0
globalDefault: true      # pods without priorityClassName use this
description: "Standard priority for most workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low-priority
value: -100              # negative — lower than default
globalDefault: false
preemptionPolicy: Never  # won't preempt anything
description: "Batch and background jobs — can be preempted"

---
# Pod using a PriorityClass
apiVersion: v1
kind: Pod
metadata:
  name: payments-api
spec:
  priorityClassName: critical-production  # reference PriorityClass name
  containers:
  - name: app
    image: payments:v1
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
```

---

### Preemption Flow

```
PREEMPTION — STEP BY STEP
═══════════════════════════════════════════════════════════════

Cluster state: All nodes full with medium-priority pods
Event: New critical-production pod needs scheduling (500m CPU, 512Mi mem)

STEP 1: Normal scheduling fails
  Scheduler finds NO node with 500m CPU and 512Mi free.
  Scheduler activates preemption logic.

STEP 2: Find preemption victims
  Scheduler looks for nodes where evicting pods would make room.
  Candidates: pods with lower priority than the incoming pod.
  Prefer: fewest victims, lowest priority victims, same node.

STEP 3: Remove nominated node annotation from any previous nomination
  Scheduler adds annotation to pending pod:
  pod.kubernetes.io/nominated-node: worker-2

STEP 4: Evict victim pods
  Scheduler sends delete to victim pods on worker-2.
  Victims respect their graceful termination period (SIGTERM).
  Victims may take up to terminationGracePeriodSeconds to die.

STEP 5: Incoming pod waits
  Critical pod stays PENDING until victims finish terminating.
  Takes longer for pods with long graceful periods.

STEP 6: Critical pod schedules on worker-2
  Once victims have freed enough resources, pod is bound.

KEY BEHAVIORS:
  Victims get graceful termination — not hard killed.
  Multiple lower-priority pods may be evicted for one high-priority pod.
  PodDisruptionBudget (PDB) respected during preemption — pods with
  PDB that would go below minAvailable are NOT preempted.
  Preemption does NOT guarantee the high-priority pod lands on the
  same node as the evicted pods (scheduling re-evaluated after eviction).
```

---

### kubectl PriorityClass Commands

```bash
# List PriorityClasses
kubectl get priorityclass
# NAME                      VALUE        GLOBAL-DEFAULT   AGE
# batch-low-priority        -100         false            5d
# critical-production       1000000      false            30d
# default-priority          0            true             30d
# high-priority             100000       false            30d
# system-cluster-critical   2000000000   false            30d
# system-node-critical      2000001000   false            30d

# Describe to see preemptionPolicy
kubectl describe priorityclass critical-production

# See priority class of running pods
kubectl get pods -n production \
  -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName,VALUE:.spec.priority"

# See if preemption is happening (events on preempted pods)
kubectl get events -A | grep -i preempt
```

---

### 🎤 Short Crisp Interview Answer

> *"PriorityClasses rank pod importance with integer values — higher value means higher priority. When the cluster is full and a high-priority pod needs to schedule, the scheduler preempts lower-priority pods on a candidate node by evicting them gracefully. The high-priority pod waits in Pending until victims finish terminating. PodDisruptionBudgets are respected during preemption — protected pods are not preemption victims if it would breach their PDB. preemptionPolicy: Never makes a high-priority pod get queued ahead of others without actually evicting anything."*

---

### ⚠️ Gotchas

1. **system-cluster-critical and system-node-critical are reserved** — don't use these for application pods. They're for core K8s components and node-critical DaemonSets.
2. **Preemption victims get graceful termination** — preemption can take 30-60 seconds because evicted pods run their termination handlers. The high-priority pod doesn't schedule instantly.
3. **PDB protects against preemption** — if a pod has a PDB and evicting it would drop below minAvailable, the scheduler won't use it as a preemption victim. Design PDBs carefully.

---

---

# ⚠️ 6.10 Topology Spread Constraints

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Topology Spread Constraints tell the scheduler to **distribute pods evenly across topology domains** (nodes, zones, regions) with a configurable maximum imbalance tolerance. It solves the high-availability problem more flexibly than pod anti-affinity — you don't need strict separation, just even-enough distribution.

---

### Why Topology Spread Over Pod Anti-Affinity

```
POD ANTI-AFFINITY LIMITATIONS
═══════════════════════════════════════════════════════════════

Problem with required pod anti-affinity for 10 replicas, 3 nodes:
  Rule: no two replicas on same node
  Nodes: 3
  Replicas: 10
  Result: 3 schedule (one per node), 7 stuck in Pending forever!
  Required anti-affinity = can't exceed node count

TOPOLOGY SPREAD CONSTRAINTS:
  Same scenario: 10 replicas, 3 nodes
  maxSkew: 1  (max imbalance between domains = 1)
  Optimal distribution: [4, 3, 3] ← all 10 schedule!
  vs anti-affinity: [1, 1, 1] + 7 Pending

  Topology spread allows multiple pods per node but ensures
  the COUNT DIFFERENCE between the most-loaded and
  least-loaded node is no greater than maxSkew.
```

---

### Topology Spread Constraints YAML

```yaml
spec:
  topologySpreadConstraints:

  # Constraint 1: Spread across AZs
  - maxSkew: 1
    # maxSkew: max allowed difference between most and least loaded domain
    # maxSkew: 1 → if zone-a has 4 pods, zone-b and zone-c must have ≥ 3 each
    # maxSkew: 2 → more relaxed spreading (allows bigger imbalance)

    topologyKey: topology.kubernetes.io/zone
    # The topology domain to spread across (must be a node label)

    whenUnsatisfiable: DoNotSchedule
    # DoNotSchedule: honor as a hard constraint — pod waits if violated
    # ScheduleAnyway: honor as a soft constraint — schedule but penalize violations

    labelSelector:
      matchLabels:
        app: my-api          # which pods count toward this spread
        # Must match the pods being scheduled (usually same labels as pod itself)

    # K8s 1.25+: match both pod template labels automatically
    matchLabelKeys:
    - pod-template-hash      # differentiates rollout versions
    # prevents old and new ReplicaSet pods from interfering with each other's spread

    # K8s 1.26+: minimum domains that must be available
    minDomains: 3            # require at least 3 zones to be available
    # if < 3 zones exist, whenUnsatisfiable applies

  # Constraint 2: Spread across nodes (within the AZ constraint)
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway    # soft — try but don't block
    labelSelector:
      matchLabels:
        app: my-api

---
# PRODUCTION PATTERN: HA across 3 AZs, soft node spreading
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 9
  template:
    metadata:
      labels:
        app: my-api
    spec:
      topologySpreadConstraints:
      # Hard AZ spreading — never allow AZ imbalance > 1
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: my-api

      # Soft node spreading — prefer different nodes
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: my-api

      containers:
      - name: api
        image: my-api:v1
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
```

---

### maxSkew Calculation

```
HOW maxSkew IS CALCULATED
═══════════════════════════════════════════════════════════════

Setup: 9 replicas, 3 zones (A, B, C), maxSkew: 1

  Current distribution: A=3, B=3, C=3  (max=3, min=3, skew=0) ✓
  New pod → any zone is fine (all would give skew=1 at most)

  After one pod dies in zone A:
  Distribution: A=2, B=3, C=3  (max=3, min=2, skew=1) ✓ (within maxSkew)
  New pod: MUST go to zone A (adding to B or C would skew=2)

  If zone A has 1, B has 3, C has 3:
  Skew = 3-1 = 2 = VIOLATES maxSkew: 1
  New pods ONLY go to zone A until balanced → A=2, B=3, C=3 → then rebalanced

⚠️  EXISTING SKEW IS NOT AUTOMATICALLY CORRECTED:
  If pods die unevenly, distribution becomes skewed.
  New pods scheduled to fix the skew.
  But existing pods are NOT rescheduled.
  Descheduler (topic 6.13) handles rebalancing existing pods.
```

---

### kubectl Topology Spread Commands

```bash
# Check pod distribution across zones
kubectl get pods -n production -o wide \
  --sort-by='.spec.nodeName' | awk '{print $7}' | sort | uniq -c
# Needs node→zone mapping — combine with:
kubectl get nodes -o custom-columns=\
  "NODE:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone"

# Check if topology spread constraints are causing Pending pods
kubectl describe pod my-api-xxx -n production | grep -A 5 "Events:"
# Events:
#   Warning  FailedScheduling  ...  0/3 nodes are available:
#            3 node(s) didn't match pod topology spread constraints

# Visualize pod distribution (useful script)
kubectl get pods -n production -l app=my-api -o wide | \
  awk 'NR>1 {print $7}' | sort | uniq -c
# 3 worker-1   ← 3 pods on node worker-1
# 3 worker-2   ← 3 pods on node worker-2
# 3 worker-3   ← 3 pods on node worker-3
```

---

### 🎤 Short Crisp Interview Answer

> *"Topology Spread Constraints distribute pods evenly across topology domains — zones, nodes, regions — with a maxSkew tolerance for acceptable imbalance. Unlike required pod anti-affinity which completely blocks scheduling when nodes are all occupied, topology spread allows multiple pods per domain but keeps the spread even. maxSkew of 1 means the busiest domain can have at most one more pod than the least busy. DoNotSchedule makes it a hard constraint, ScheduleAnyway makes it a scoring hint. The critical gotcha is that skew caused by pod terminations is not automatically corrected — you need the Descheduler for that."*

---

### ⚠️ Gotchas

1. **Skew imbalance is not auto-corrected** — if pods die and create an imbalance, new pods schedule to re-balance but existing pods don't move. Use Descheduler to rebalance the cluster.
2. **DoNotSchedule with insufficient zones = Pending** — if minDomains is 3 and only 2 zones exist, or if maxSkew cannot be satisfied, pods stay Pending. Start with ScheduleAnyway and monitor before hardening.
3. **labelSelector must match pod labels exactly** — the selector must match the pods you want to spread. If label is wrong, all pods count as 0 and spread constraint has no effect.
4. **Rolling updates temporarily increase skew** — during a Deployment rolling update, old ReplicaSet pods and new ReplicaSet pods both exist. Without matchLabelKeys, they can double-count and confuse the spread calculation.

---

---

# 6.11 Scheduler Extenders & Custom Schedulers

## 🔴 Advanced

### What it is in simple terms

The default Kubernetes scheduler handles the vast majority of placement decisions. **Scheduler extenders** let you add custom logic to the existing scheduler via webhook callbacks. **Custom schedulers** replace or supplement the default scheduler entirely — pods can opt into a custom scheduler by name. These exist when business or hardware constraints cannot be expressed through native scheduling primitives.

---

### When You Need Custom Scheduling Logic

```
REASONS TO EXTEND OR REPLACE THE DEFAULT SCHEDULER
═══════════════════════════════════════════════════════════════

1. HARDWARE-AWARE PLACEMENT:
   NUMA topology (memory + CPU on same socket)
   Specific NIC ports for RDMA / high-speed networking
   GPU topology — schedule on nodes where GPUs share NVLink
   These constraints cannot be expressed via node labels alone

2. EXTERNAL SYSTEM INTEGRATION:
   Check license server before scheduling (only 10 concurrent licenses)
   Consult CMDB to verify node's maintenance window
   Validate against enterprise capacity planner
   Cost-based scheduling (prefer cheapest available node)

3. CLUSTER-SPECIFIC POLICIES:
   Co-location with specific hardware not expressible via labels
   Compliance requirements (data residency)
   Custom bin-packing or spreading strategies

4. MACHINE LEARNING / GANG SCHEDULING:
   All pods of a job must schedule simultaneously or not at all
   Batch schedulers: Volcano, Apache YuniKorn
   ML training: Kubeflow's gang scheduling
```

---

### Scheduler Extension Points

```
DEFAULT SCHEDULER PIPELINE — WHERE YOU CAN HOOK IN
═══════════════════════════════════════════════════════════════

kubectl apply Pod → API Server → Scheduler Queue

Scheduler pipeline (per pod, in order):

  Sort          → order pending pods in queue (by priority)
  PreFilter     → pre-process pod info, reject early if impossible
  Filter        → eliminate nodes that don't satisfy requirements
                  (nodeAffinity, taints, resource fit, etc.)
  PostFilter    → handle Filter failures (find preemption victims)
  PreScore      → prepare data for scoring phase
  Score         → rank remaining nodes (0-100 per plugin)
                  Plugins: NodeResourcesFit, ImageLocality,
                           InterPodAffinity, NodeAffinity, etc.
  NormalizeScore→ normalize plugin scores to 0-100 range
  Reserve       → reserve resources (optimistic locking)
  Permit        → wait / approve / reject scheduling decision
                  Used for gang scheduling (wait for all pods)
  PreBind       → pre-binding actions
  Bind          → actually write node assignment to API Server
  PostBind      → cleanup after successful bind

EXTENSION MECHANISM 1: Scheduler Framework Plugins
  Write Go code implementing plugin interfaces
  Compile INTO the scheduler binary
  Full access to all extension points above
  Highest performance, lowest latency
  Examples: Volcano, Koordinator, scheduler-plugins repo

EXTENSION MECHANISM 2: Scheduler Extenders (webhook)
  External HTTP webhook called during Filter and Prioritize phases
  Scheduler calls your HTTP endpoint with pod + candidate nodes
  Your service returns: filtered node list or score map
  No recompile needed — runs as separate service
  Higher latency (~ms per scheduling decision)
  Cannot access Reserve/Permit/Bind extension points
```

---

### Scheduler Extender Configuration

```yaml
# KubeSchedulerConfiguration — enable extenders
# /etc/kubernetes/scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    # Disable built-in plugins if needed
    score:
      disabled:
      - name: NodeResourcesBalancedAllocation
  extenders:
  - urlPrefix: "https://my-scheduler-extender.kube-system.svc:8080"
    filterVerb: "filter"         # POST /filter with pending pods
    prioritizeVerb: "prioritize" # POST /prioritize for scoring
    preemptVerb: "preempt"       # POST /preempt for preemption
    weight: 5                    # extender's score weight (vs built-in)
    enableHTTPS: true
    tlsConfig:
      caFile: /etc/ssl/ca.crt
    nodeCacheCapable: false       # true = extender gets node info upfront
    ignorable: false              # false = if extender is down, fail scheduling
                                  # true = if extender is down, ignore it
    managedResources:             # extended resources this extender manages
    - name: "company.com/fpga"
      ignoredByScheduler: true
```

---

### Custom Scheduler

```yaml
# Deploy a second scheduler alongside default
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-scheduler
  namespace: kube-system
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: gpu-scheduler-sa
      containers:
      - name: scheduler
        image: my-company/gpu-scheduler:v1.0
        command:
        - /usr/local/bin/kube-scheduler
        - --config=/etc/kubernetes/gpu-scheduler-config.yaml
        # Custom scheduler built from scheduler framework
        # implements GPU topology-aware placement

---
# Pod opts into custom scheduler
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training-job
spec:
  schedulerName: gpu-scheduler  # use custom scheduler, not default-scheduler
  containers:
  - name: training
    image: pytorch:2.0
    resources:
      limits:
        nvidia.com/gpu: 8       # request 8 GPUs
```

---

### Gang Scheduling with Volcano

```yaml
# Volcano — batch scheduler for ML/HPC workloads
# Implements gang scheduling: all pods schedule or none do
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: pytorch-training
spec:
  minAvailable: 4          # need at least 4 pods to start (gang)
  schedulerName: volcano
  queue: default
  plugins:
    ssh: []
    pytorch: []
  tasks:
  - replicas: 1
    name: master
    template:
      spec:
        containers:
        - name: master
          image: pytorch:2.0
          resources:
            limits:
              nvidia.com/gpu: 1
  - replicas: 3
    name: worker
    template:
      spec:
        containers:
        - name: worker
          image: pytorch:2.0
          resources:
            limits:
              nvidia.com/gpu: 1
  # All 4 pods (1 master + 3 workers) schedule simultaneously
  # or none schedule (avoids partial allocations causing deadlock)
```

---

### 🎤 Short Crisp Interview Answer

> *"The default scheduler has a plugin framework with extension points at every stage — Filter, Score, Reserve, Permit, Bind. For adding custom logic without recompiling, scheduler extenders are HTTP webhooks called during Filter and Score phases. For deeper integration, you compile a custom plugin into the scheduler binary using the scheduler framework. For full replacement, you deploy a custom scheduler as a separate pod and pods opt in via schedulerName. The main production use cases are GPU topology-aware scheduling for ML workloads, gang scheduling where all training pods must start simultaneously, and cost-aware scheduling. Volcano is the most common custom scheduler for ML/HPC batch workloads."*

---

### ⚠️ Gotchas

1. **Extender latency multiplies with pod count** — each scheduling decision calls the extender HTTP endpoint. If you're scheduling 100 pods and the extender takes 10ms, that's 1 second just in extender calls. Keep extenders fast.
2. **ignorable: false means extender outage = scheduling outage** — if your extender service goes down and ignorable is false, NO pods can be scheduled. Run extenders with high availability.
3. **Custom schedulers don't see each other's decisions** — two schedulers can both schedule pods to the same node without knowing about each other, causing overcommit. Coordinate through ResourceVersion and optimistic locking.

---

### Connections
- Default scheduler pipeline — **Category 1 (1.5)**
- GPU node scheduling with node affinity + taints — **6.6, 6.8**
- Gang scheduling interacts with PriorityClass — **6.9**

---

---

# 6.12 Bin Packing vs Spreading Strategies

## 🔴 Advanced

### What it is in simple terms

**Bin packing** (also called least-requested) fills nodes to maximum utilization before using new ones — like filling boxes completely before opening a new one. **Spreading** (most-requested inverse) distributes pods across as many nodes as possible. These are competing strategies with very different cost and reliability implications.

---

### The Trade-Off

```
BIN PACKING vs SPREADING
═══════════════════════════════════════════════════════════════

SPREADING (Kubernetes default):
  Goal: distribute load evenly across all nodes
  Scheduler scores: nodes with MORE free resources score HIGHER
  Result: pods spread thinly across many nodes

  Cluster: 10 nodes, 100 pods
  Distribution: ~10 pods per node

  PROS:
  ✓ High availability — node failure affects fewer pods
  ✓ Latency consistency — no hot nodes
  ✓ Simple to reason about

  CONS:
  ✗ All 10 nodes must run even if only 3 would suffice
  ✗ Higher cloud cost (can't scale down idle nodes)
  ✗ Node autoscaler cannot consolidate — nodes look "in use"

CONSOLIDATION / BIN PACKING:
  Goal: fill nodes to maximum before using new ones
  Scheduler scores: nodes with LESS free resources score HIGHER
  Result: pods concentrate on fewer nodes

  Cluster: 10 nodes, 100 pods
  Distribution: 3 nodes fully packed, 7 nodes empty (can scale down)

  PROS:
  ✓ Fewer nodes running → lower cloud cost
  ✓ Cluster autoscaler can scale down empty nodes
  ✓ Better for variable/bursty workloads
  ✓ GPU node utilization maximized

  CONS:
  ✗ Node failure affects more pods
  ✗ Hot nodes — latency variability
  ✗ Less resilient (fewer pods per failure domain)

RECOMMENDATION:
  Production HA services:     Spreading + Topology Spread Constraints
  Batch / ML workloads:       Bin packing (cost optimization)
  Mixed cluster:              Bin pack batch, spread production
                              Using node pools + taints/tolerations
```

---

### Configuring Scheduler Scoring Strategy

```yaml
# KubeSchedulerConfiguration — change scoring strategy
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        # SPREADING (default — MostAllocated inverse):
        type: LeastAllocated
        # Scores = (cpu_free/cpu_total + mem_free/mem_total) / 2
        # Node with MORE free resources = higher score = pods spread out

        # BIN PACKING:
        # type: MostAllocated
        # Scores = (cpu_used/cpu_total + mem_used/mem_total) / 2
        # Node with LESS free resources = higher score = pods pack in

        # BALANCED ALLOCATION (prevent CPU/memory fragmentation):
        # type: RequestedToCapacityRatio
        # resources:
        # - name: cpu
        #   weight: 1
        # - name: memory
        #   weight: 1

        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

---

### Cost Optimization Bin Packing on EKS

```
EKS NODE CONSOLIDATION STRATEGY
═══════════════════════════════════════════════════════════════

SCENARIO: 20 m5.xlarge nodes, each at 30% utilization
  Cost: 20 × $0.192/hr = $3.84/hr
  With bin packing: could fit same workload on 6 nodes
  Savings: 14 × $0.192/hr = $2.69/hr = $1,969/month

KARPENTER (EKS-native) — better than bin packing scheduler setting:
  Karpenter does bin packing natively during provisioning.
  It calculates the minimum number of nodes needed.
  It provisions the EXACT right instance type for the workload.
  It runs CONSOLIDATION: periodically reschedules pods to fewer nodes.

  kubectl get nodeclaim -n karpenter  # see nodes being consolidated
  kubectl get nodes                   # watch nodes being terminated

CLUSTER AUTOSCALER + BIN PACKING:
  Set scheduler to MostAllocated scoring.
  Cluster Autoscaler scale-down:
    Finds nodes where ALL pods can be moved to other nodes.
    Cordons and drains those nodes.
    Terminates the EC2 instance.
  This only works if nodes are ACTUALLY near empty.
  MostAllocated scoring helps concentrate pods.

NODE CONSOLIDATION APPROACHES:
  1. Scheduler bin packing:   New pods go to busiest nodes
  2. Descheduler consolidation: Moves existing pods (topic 6.13)
  3. Karpenter consolidation:  Directly manages node lifecycle
```

---

### 🎤 Short Crisp Interview Answer

> *"Bin packing concentrates pods onto fewer nodes to maximize utilization and enable scale-down, while spreading distributes pods thinly for resilience. The Kubernetes default scheduler uses LeastAllocated scoring — preferring nodes with more free resources — which spreads pods. For cost optimization, you switch to MostAllocated scoring which fills nodes first. In practice, the choice depends on workload type: production HA services should spread for resilience, batch jobs and ML training should bin pack for cost. On EKS with Karpenter, consolidation is handled natively and more aggressively than the scheduler-level bin packing."*

---

### ⚠️ Gotchas

1. **MostAllocated alone doesn't consolidate existing pods** — it only affects new pod placement. Existing pods on sparse nodes stay there. Use Descheduler for consolidation of existing pods.
2. **Bin packing increases blast radius** — a densely packed node failure evicts far more pods than a spread node failure. Balance cost savings against HA requirements.
3. **Karpenter consolidation can cause disruption** — Karpenter's consolidation feature reschedules pods and terminates nodes. Ensure PDBs are set correctly to limit disruption during consolidation.

---

---

# 6.13 Descheduler — Rebalancing After the Fact

## 🔴 Advanced

### What it is in simple terms

The Kubernetes scheduler only makes decisions when pods are being placed — it never moves running pods. The **Descheduler** is a separate component that **periodically evicts pods** that violate current policies, are badly placed, or could be better distributed, allowing the scheduler to re-place them on better nodes.

---

### Why Descheduler is Needed

```
THE SCHEDULER'S BLIND SPOT — STALE PLACEMENT
═══════════════════════════════════════════════════════════════

SCENARIO: Pod was placed correctly, but cluster state changed

Day 1: Pod-A placed on worker-3 (only node in us-east-1c)
       This was correct — good AZ spread.

Day 7: New nodes added: worker-4, worker-5 in us-east-1c
       Pods in us-east-1a and us-east-1b still evenly distributed.
       Worker-3 still has Pod-A and now also has 8 new pods.
       Original pods in us-east-1a and us-east-1b have 0 new pods.
       Distribution: a=3, b=3, c=9 ← severely imbalanced

Day 14: Node affinity rules changed — Pod-B no longer needs
        to be on GPU node but is still running there.
        GPU node is fully occupied by non-GPU pods.
        New GPU pod cannot schedule — all GPU nodes occupied.

THE SCHEDULER DOES NOTHING IN THESE CASES:
  Pod-A is running and healthy — no reason to evict and reschedule.
  Pod-B is running and healthy — no reason to evict.
  But both are badly placed relative to current desired state.

DESCHEDULER FIXES THIS:
  Runs periodically (cron-like) or as continuous loop.
  Evaluates running pods against current policies.
  Evicts pods that violate policies or are sub-optimally placed.
  The scheduler re-places them on now-correct nodes.
```

---

### Descheduler Strategies

```
BUILT-IN DESCHEDULER STRATEGIES (profiles/plugins)
═══════════════════════════════════════════════════════════════

RemoveDuplicates:
  Evicts pods where more than one replica of the same
  ReplicaSet/Deployment is on the same node.
  Use case: HPA scaled up quickly, all pods landed on one node.

LowNodeUtilization:
  Finds UNDERUTILIZED nodes (all resources below thresholds).
  Finds OVERUTILIZED nodes (any resource above thresholds).
  Evicts pods from overutilized nodes.
  Scheduler replaces them on underutilized nodes.
  Use case: Bin packing and cost reduction.

HighNodeUtilization:
  Inverse of LowNodeUtilization.
  Evicts pods from underutilized nodes.
  Scheduler consolidates them on more utilized nodes.
  Use case: Enabling Cluster Autoscaler to scale down sparse nodes.

RemovePodsViolatingNodeAffinity:
  Evicts pods that no longer satisfy their node affinity rules.
  Use case: Node labels changed after pod placement.

RemovePodsViolatingNodeTaints:
  Evicts pods that are on nodes with taints they don't tolerate.
  Use case: Taint added to node after pod was running there.

RemovePodsViolatingInterPodAntiAffinity:
  Evicts pods that violate their anti-affinity rules.
  Use case: Pods ended up on same node due to scheduling race.

RemovePodsViolatingTopologySpreadConstraint:
  Evicts pods where topology spread is violated.
  Use case: Pods distributed unevenly after terminations.
  This is the most important strategy for spread rebalancing.

PodLifeTime:
  Evicts pods older than a configured lifetime.
  Use case: Force periodic restart of long-running pods.
  Useful for: clearing memory leaks, refreshing credentials.
```

---

### Descheduler YAML Configuration

```yaml
# Install via Helm
# helm repo add descheduler https://kubernetes-sigs.github.io/descheduler/
# helm install descheduler descheduler/descheduler -n kube-system

# Descheduler policy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: descheduler-policy
  namespace: kube-system
data:
  policy.yaml: |
    apiVersion: "descheduler/v1alpha2"
    kind: "DeschedulerPolicy"
    profiles:
    - name: production-rebalancing
      pluginConfig:
      - name: "DefaultEvictor"
        args:
          # NEVER evict these (protected)
          ignorePvcPods: false        # allow evicting PVC pods (careful!)
          evictLocalStoragePods: false # don't evict pods with local storage
          evictSystemCriticalPods: false
          nodeFit: true               # only evict if pod can reschedule elsewhere
          minReplicas: 2              # only evict if replica count > 2
          minPodAge: 120              # pod must be 2+ minutes old to be evicted

      - name: "RemovePodsViolatingTopologySpreadConstraint"
        args:
          includeSoftConstraints: false  # only enforce hard constraints
          constraints:
          - DoNotSchedule

      - name: "LowNodeUtilization"
        args:
          thresholds:          # node is "underutilized" if ALL below these
            cpu: 20            # < 20% CPU used
            memory: 20         # < 20% memory used
            pods: 20           # < 20% pod capacity used
          targetThresholds:    # node is "overutilized" if ANY above these
            cpu: 50
            memory: 50
            pods: 50
          # Evicts pods from overutilized nodes
          # Scheduler places them on underutilized nodes

      - name: "RemoveDuplicates"  # remove duplicate replicas on same node

      - name: "PodLifeTime"
        args:
          maxPodLifeTimeSeconds: 604800   # evict pods older than 7 days
          podStatusPhases:
          - "Running"
          labelSelector:
            matchLabels:
              auto-recycle: "true"        # only pods with this label

      plugins:
        balance:
          enabled:
          - "RemovePodsViolatingTopologySpreadConstraint"
          - "LowNodeUtilization"
          - "RemoveDuplicates"
        deschedule:
          enabled:
          - "RemovePodsViolatingNodeAffinity"
          - "RemovePodsViolatingNodeTaints"
          - "RemovePodsViolatingInterPodAntiAffinity"
          - "PodLifeTime"

---
# Descheduler as CronJob (run every 2 hours)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: descheduler
  namespace: kube-system
spec:
  schedule: "0 */2 * * *"    # every 2 hours
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: descheduler
          containers:
          - name: descheduler
            image: registry.k8s.io/descheduler/descheduler:v0.28.0
            command:
            - /bin/descheduler
            args:
            - --policy-config-file=/policy-dir/policy.yaml
            - --v=3
            volumeMounts:
            - name: policy-volume
              mountPath: /policy-dir
          volumes:
          - name: policy-volume
            configMap:
              name: descheduler-policy
```

---

### 🎤 Short Crisp Interview Answer

> *"The scheduler only places pods — it never moves running pods. The Descheduler runs periodically and evicts pods that are badly placed relative to current state: pods violating topology spread constraints, pods on nodes with taints they don't tolerate, pods whose node affinity no longer matches after label changes, and duplicate pods on the same node. After eviction, the scheduler re-places them correctly. The most commonly used strategies are RemovePodsViolatingTopologySpreadConstraint — which fixes spread imbalance after pod terminations — and LowNodeUtilization — which consolidates pods from sparse nodes to enable cluster autoscaler scale-down. Critical: Descheduler uses the DefaultEvictor which respects PDBs — it won't evict pods if it would breach a PodDisruptionBudget."*

---

### ⚠️ Gotchas

1. **Descheduler respects PDBs but only partially** — by default, Descheduler checks PDBs before eviction. But with aggressive settings and many pods, it can still cause disruption. Set minReplicas in DefaultEvictor to prevent evicting the last replica.
2. **nodeFit: true is critical** — without it, Descheduler can evict pods that have nowhere to reschedule (no fitting node), causing them to stay Pending forever.
3. **PodLifeTime strategy causes regular disruption** — if configured too aggressively (short lifetime), it creates a churn loop. Use only for specific labeled pods that need periodic refresh.

---

---

# 6.14 Node Pressure Eviction — Memory/Disk/PID Pressure

## 🔴 Advanced

### What it is in simple terms

When a node runs low on resources — memory, disk space, or process IDs — kubelet proactively **evicts pods** to protect the node from becoming completely unusable. This is kubelet-driven eviction, distinct from scheduler-driven eviction (priority preemption). Understanding eviction thresholds and ordering is essential for cluster stability.

---

### Eviction Thresholds — Hard and Soft

```
NODE PRESSURE CONDITIONS AND EVICTION TRIGGERS
═══════════════════════════════════════════════════════════════

RESOURCES KUBELET MONITORS:
  memory.available:   available node memory
  nodefs.available:   available disk space for node filesystem (kubelet, logs)
  nodefs.inodesFree:  available inodes on node filesystem
  imagefs.available:  available disk for container images and overlayfs
  imagefs.inodesFree: available inodes on image filesystem
  pid.available:      available process IDs on the node

EVICTION THRESHOLD TYPES:

  SOFT EVICTION (graceful — give pods time to terminate):
    Trigger: threshold breached AND maintained for evictionSoftGracePeriod
    Action:  Kubelet waits evictionSoftGracePeriod, then evicts gracefully
    Pod gets: SIGTERM → grace period → SIGKILL if still running

  HARD EVICTION (immediate — protect node):
    Trigger: threshold breached (immediate, no waiting)
    Action:  Kubelet evicts immediately
    Pod gets: SIGKILL (no grace period — immediate termination)

DEFAULT HARD EVICTION THRESHOLDS (kubelet defaults):
  memory.available  < 100Mi   → evict pods immediately
  nodefs.available  < 10%     → evict pods immediately
  nodefs.inodesFree < 5%      → evict pods immediately
  imagefs.available < 15%     → evict pods immediately

DEFAULT SOFT EVICTION THRESHOLDS (none configured by default):
  Must be explicitly configured in kubelet config
  Production recommendation:
    memory.available  < 500Mi  (soft, 1m grace) + < 100Mi (hard)
    nodefs.available  < 20%    (soft, 2m grace) + < 10%   (hard)
```

---

### Kubelet Eviction Configuration

```yaml
# /etc/kubernetes/kubelet-config.yaml (or via KubeletConfiguration)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# SOFT EVICTION THRESHOLDS (trigger warning, wait before evicting)
evictionSoft:
  memory.available: "500Mi"   # warn when < 500Mi free memory
  nodefs.available: "20%"     # warn when < 20% disk free
  nodefs.inodesFree: "10%"    # warn when < 10% inodes free
  imagefs.available: "20%"    # warn when < 20% image disk free

# How long to sustain soft threshold before evicting
evictionSoftGracePeriod:
  memory.available: "1m30s"   # wait 90 seconds before evicting
  nodefs.available: "2m"      # wait 2 minutes before evicting
  nodefs.inodesFree: "2m"
  imagefs.available: "2m"

# HARD EVICTION THRESHOLDS (immediate eviction, no grace period)
evictionHard:
  memory.available: "100Mi"   # evict immediately below 100Mi free
  nodefs.available: "5%"      # evict immediately below 5% disk
  nodefs.inodesFree: "2%"     # evict immediately below 2% inodes
  imagefs.available: "5%"     # evict immediately below 5% image disk
  pid.available: "100"        # evict immediately if < 100 PIDs available

# Minimum eviction reclaim (how much to free per eviction cycle)
evictionMinimumReclaim:
  memory.available: "200Mi"   # try to reclaim 200Mi per cycle (not just 1Mi)
  nodefs.available: "1Gi"     # reclaim 1Gi disk per cycle
  imagefs.available: "2Gi"    # reclaim 2Gi image disk per cycle

# Prevent eviction thrashing (wait before re-evaluating after eviction)
evictionPressureTransitionPeriod: "5m"
# Node waits 5 minutes after pressure resolves before removing MemoryPressure condition

# Image garbage collection (separate from eviction)
imageGCHighThresholdPercent: 85  # start GC when image disk > 85% full
imageGCLowThresholdPercent: 80   # GC until image disk < 80% full
```

---

### Pod Eviction Order Under Memory Pressure

```
WHICH PODS GET EVICTED FIRST
═══════════════════════════════════════════════════════════════

Kubelet eviction order (when memory pressure triggers):

STEP 1: Evict BestEffort pods
  All pods with QoS class BestEffort evicted first.
  Rationale: No resources reserved, no contract, lowest priority.
  Ordered by: amount of memory OVER their limit (most over = evicted first)
  Since BestEffort has no limit: ordered by raw memory usage (most = first)

STEP 2: Evict Burstable pods exceeding their requests
  Burstable pods using MORE memory than requested.
  Ordered by: amount over their request (most over = evicted first)
  Rationale: They're using "borrowed" resources.

STEP 3: Evict Burstable pods within their requests
  Burstable pods still within their memory request.
  Ordered by: pod priority (lower priority = evicted first)
  Then by: memory usage

STEP 4: Evict Guaranteed pods
  Only evicted as last resort when nothing else can free enough memory.
  Ordered by: pod priority (lower priority = evicted first)
  Rationale: They paid for the resources with matching requests=limits.

MEMORY PRESSURE INDICATOR ON NODE:
  kubectl describe node worker-1 | grep -A 3 Conditions
  # Conditions:
  #   Type              Status    Reason
  #   MemoryPressure    True      KubeletHasInsufficientMemory
  #   DiskPressure      False
  #   PIDPressure       False
  #   Ready             True      KubeletReady

NODE CONDITIONS AND TAINTS:
  When kubelet detects pressure, it adds a TAINT to the node:
  MemoryPressure → node.kubernetes.io/memory-pressure:NoSchedule
  DiskPressure   → node.kubernetes.io/disk-pressure:NoSchedule
  PIDPressure    → node.kubernetes.io/pid-pressure:NoSchedule

  This PREVENTS new pods from scheduling on the pressured node.
  Existing pods without toleration for these taints are EVICTED.
  (NoSchedule taint added during pressure)
```

---

### Disk Pressure — Image and Node Filesystem

```
DISK PRESSURE DETAILS
═══════════════════════════════════════════════════════════════

TWO SEPARATE FILESYSTEMS MONITORED:

nodefs (node filesystem — /var/lib/kubelet):
  Stores: pod logs, emptyDir volumes, container writable layers
  Fills up when: logs not rotated, emptyDir overuse, too many pods

imagefs (image filesystem — /var/lib/containerd):
  Stores: pulled container images, overlay layers
  Fills up when: large images, many images never cleaned up

DISK PRESSURE MITIGATION:
  1. Increase evictionHard.nodefs.available threshold
     → Evict sooner before full disk crashes node

  2. Enable container log rotation
     kubectl apply -f - <<EOF
     apiVersion: v1
     kind: ConfigMap
     metadata:
       name: kubelet-config
     data:
       containerLogMaxSize: "10Mi"    # max log file size
       containerLogMaxFiles: "5"      # keep last 5 rotated files
     EOF

  3. Image garbage collection (already configured above)
     kubelet automatically deletes old unused images
     when imagefs.available hits imageGCHighThresholdPercent

  4. emptyDir sizeLimit in pod spec
     volumes:
     - name: scratch
       emptyDir:
         sizeLimit: 1Gi   # pod evicted if emptyDir exceeds 1Gi

PID PRESSURE:
  Caused by: fork bombs, runaway processes, misconfigured apps
  Node PID max: typically 32768 or 262144 (kernel.pid_max)
  Alert at < 1000 PIDs available
  Diagnosis: kubectl exec into suspect pod → ps aux | wc -l
```

---

### kubectl Eviction Monitoring

```bash
# Check node conditions (memory, disk, PID pressure)
kubectl describe node worker-1 | grep -A 10 Conditions:
# Type              Status  Reason
# MemoryPressure    False   KubeletHasSufficientMemory
# DiskPressure      True    KubeletHasDiskPressure    ← ALERT!
# PIDPressure       False   KubeletHasSufficientPID
# Ready             False   KubeletNotReady

# Check for evicted pods
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted
kubectl get pods -n production | grep Evicted

# See eviction reason
kubectl describe pod evicted-pod-xxx -n production | tail -20
# Events:
#   Warning  Evicted  The node was low on resource: memory.
#            Threshold quantity: 100Mi, available: 86Mi.

# Watch memory usage on nodes
kubectl top nodes --sort-by=memory
# NAME        CPU(cores)  CPU%   MEMORY(bytes)  MEMORY%
# worker-1    4200m       52%    28Gi           87%    ← near eviction
# worker-2    2100m       26%    15Gi           46%

# Check eviction threshold settings on a node
kubectl proxy &
curl http://localhost:8001/api/v1/nodes/worker-1/proxy/configz \
  2>/dev/null | python3 -m json.tool | grep -A 3 eviction
```

---

### 🎤 Short Crisp Interview Answer

> *"Kubelet eviction is node-level protection — when memory, disk, or PID resources are critically low, kubelet evicts pods to save the node. There are two thresholds: soft eviction waits for a configurable grace period before evicting gracefully with SIGTERM, hard eviction is immediate with SIGKILL. Eviction order follows QoS: BestEffort first, then Burstable pods over their requests, then Burstable within requests, Guaranteed pods last. When pressure is detected, kubelet also adds a taint — memory-pressure, disk-pressure, or pid-pressure with NoSchedule — preventing new pods from landing on the stressed node. This is separate from priority-based preemption which is scheduler-driven."*

---

### ⚠️ Gotchas

1. **Hard eviction = SIGKILL, no graceful shutdown** — pods evicted by hard thresholds get no chance to clean up. Ensure critical data is committed before eviction by keeping memory usage well below hard thresholds.
2. **Eviction thrashing** — if you evict one pod to free memory, the scheduler places a new pod, triggering eviction again. evictionPressureTransitionPeriod prevents immediate re-evaluation.
3. **Log accumulation causes disk pressure** — application logs in /var/log/containers grow unboundedly without log rotation. This is a common cause of DiskPressure on nodes. Set containerLogMaxSize and containerLogMaxFiles in kubelet config.
4. **emptyDir without sizeLimit causes disk pressure** — a single pod with runaway emptyDir writes can fill the node filesystem and trigger eviction of ALL pods on the node.

---

### Connections
- QoS class determines eviction order — **6.5**
- Node conditions visible in cluster events — **Category 1 (1.6)**
- ResourceQuota doesn't prevent eviction — **6.4**
- Memory limits prevent individual OOM but not eviction — **6.1**

---

---

# 6.15 CPU Manager & Memory Manager (Static Policy)

## 🔴 Advanced

### What it is in simple terms

By default, all containers share CPUs on a time-sliced basis — any container can run on any CPU core at any time. The **CPU Manager** with static policy assigns **exclusive, dedicated CPU cores** to specific containers, eliminating CPU context switching, cache thrashing, and NUMA cross-socket latency. The **Memory Manager** complements this by ensuring memory pages come from the NUMA node local to the assigned CPUs.

---

### Why Exclusive CPUs Matter

```
THE PROBLEM: CPU TIME-SLICING AND CACHE THRASHING
═══════════════════════════════════════════════════════════════

DEFAULT (none/shared CPU policy):
  Container A runs on CPU 0 for 5ms
  Container B preempts — runs on CPU 0
  Container A scheduled back on CPU 2 (different core)
  Container A's L1/L2 cache is now COLD on CPU 2
  Container A must reload all working data into cache

  For latency-sensitive workloads:
    Cache miss penalty: 100-300 ns (L1 miss to L3 hit)
    NUMA cross-socket: 100-300 ns additional
    P99 latency spikes from cache contention: 1-10ms artifacts

STATIC CPU POLICY:
  Container A is assigned CPU 0 and CPU 1 exclusively
  Container A ALWAYS runs on CPU 0 and CPU 1
  No other container uses CPU 0 or CPU 1
  CPU cache stays warm → consistent, low latency
  Linux scheduler respects cpuset cgroup binding

USE CASES:
  ✓ Low-latency NFV (Network Function Virtualization)
  ✓ Real-time telco workloads (5G packet processing)
  ✓ High-frequency trading applications
  ✓ Latency-sensitive database engines (custom RDBMS)
  ✓ ML inference requiring consistent P99 latency
  ✓ DPDK (Data Plane Development Kit) applications
```

---

### Requirements for Static CPU Policy

```
PREREQUISITES FOR EXCLUSIVE CPU PINNING
═══════════════════════════════════════════════════════════════

1. CPU Manager policy: static (not default "none")
2. Pod must be Guaranteed QoS class
   (requests.cpu == limits.cpu, requests.memory == limits.memory)
3. CPU request must be an INTEGER (not fractional)
   limits.cpu: "2"    ← 2 whole CPUs → gets dedicated CPU 4 and CPU 5
   limits.cpu: "500m" ← half a CPU  → NOT pinned (shared, CFS throttled)

WHY INTEGER ONLY:
  You cannot "pin half a CPU" — either a core is exclusively yours or
  it's shared. Fractional CPU requests use CFS bandwidth (quota/period)
  which requires the core to be shared.

CPU POOL MANAGEMENT:
  Kubelet divides node CPUs into three pools:
    Reserved CPUs:  system-reserved + kube-reserved CPUs
                    For OS processes, kubelet, container runtime
    Shared pool:    CPUs for all non-pinned containers (Burstable/BestEffort)
                    Also time-sliced among pinned init containers
    Exclusive pool: CPUs allocated one-by-one to pinned containers

  Example: 16-core node
    Reserved: CPU 0, CPU 1 (2 CPUs)
    Shared pool: initially CPU 2-15 (14 CPUs)
    Container requests 4 CPUs (Guaranteed, integer):
    → CPUs 2, 3, 4, 5 moved from shared to exclusive
    → Shared pool: CPU 6-15 (10 CPUs remaining)
```

---

### CPU Manager Configuration

```yaml
# kubelet configuration — enable static CPU policy
# /etc/kubernetes/kubelet-config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# CPU Manager policy
cpuManagerPolicy: "static"    # "none" (default) or "static"
cpuManagerPolicyOptions:
  full-pcpus-only: "true"     # allocate whole physical CPUs
                               # (not SMT/hyper-threading pairs separately)
                               # ensures true isolation (no HT sibling sharing)

# Reserve CPUs for system (cannot be allocated to containers)
reservedSystemCPUs: "0-1"     # CPU 0 and 1 reserved for OS/kubelet

# Memory Manager policy (complementary to CPU Manager)
memoryManagerPolicy: "Static" # "None" or "Static"

# State files (persist CPU/Memory assignments across kubelet restart)
# cpuManagerState: /var/lib/kubelet/cpu_manager_state
# memoryManagerState: /var/lib/kubelet/memory_manager_state

---
# Pod using static CPU policy — MUST be Guaranteed QoS
apiVersion: v1
kind: Pod
metadata:
  name: latency-sensitive-app
spec:
  containers:
  - name: app
    image: realtime-processor:v1
    resources:
      requests:
        cpu: "4"          # integer CPUs = eligible for pinning
        memory: "8Gi"
      limits:
        cpu: "4"          # limits == requests → Guaranteed QoS
        memory: "8Gi"     # memory also must match for Guaranteed

    # Container gets EXCLUSIVE use of 4 specific CPU cores
    # e.g., CPUs 4, 5, 6, 7 (kubelet decides which ones)
    # Linux cpuset cgroup: cpuset.cpus = "4-7"
    # No other container touches these CPUs

---
# NON-ELIGIBLE — fractional CPU (stays in shared pool)
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "500m"       # NOT an integer → not eligible for pinning
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    # This is still Guaranteed QoS (req == lim)
    # But NOT pinned — stays in shared CPU pool with CFS scheduling
```

---

### Memory Manager — NUMA-Aware Memory Allocation

```
MEMORY MANAGER STATIC POLICY — NUMA TOPOLOGY
═══════════════════════════════════════════════════════════════

NUMA (Non-Uniform Memory Access):
  Modern multi-socket servers have 2+ NUMA nodes.
  Example: 2-socket server, 32 CPUs, 256 GiB RAM
    NUMA node 0: CPUs 0-15, RAM banks 0-7 (128 GiB)
    NUMA node 1: CPUs 16-31, RAM banks 8-15 (128 GiB)

  Memory access latency:
    Local NUMA (CPU on same node as RAM):  ~80 ns
    Remote NUMA (CPU crosses socket):      ~120-150 ns

  Without Memory Manager:
    Container pinned to CPUs 0-3 (NUMA node 0)
    Memory allocated from NUMA node 1 (whatever Linux decides)
    → All memory accesses cross the QPI bus (slower)

  With Memory Manager static policy:
    Container pinned to CPUs 0-3 (NUMA node 0)
    Memory Manager allocates 8 GiB from NUMA node 0
    → All memory accesses local (faster, consistent)

TOPOLOGYMANAGER:
  Coordinates CPU Manager + Memory Manager + Device Manager
  Ensures CPUs, memory, and devices (GPU, RDMA NIC) all come
  from the same NUMA node for a container.

  topologyManagerPolicy options:
    none:          No topology constraints (default)
    best-effort:   Try to align, continue if impossible
    restricted:    Align or fail scheduling
    single-numa-node: Must fit entirely within one NUMA node
```

---

### Topology Manager Configuration

```yaml
# kubelet configuration for full topology alignment
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

cpuManagerPolicy: "static"
memoryManagerPolicy: "Static"
topologyManagerPolicy: "single-numa-node"
# All resources (CPU + memory + devices) must come from same NUMA node
# Pod fails to schedule on node if it spans NUMA nodes

topologyManagerScope: "container"  # or "pod"
# container: each container gets independent NUMA alignment
# pod: all containers in pod must fit in same NUMA topology

# Reserve memory per NUMA node for system
reservedMemory:
- numaNode: 0
  limits:
    memory: "2Gi"    # reserve 2Gi on NUMA node 0 for kubelet/system
- numaNode: 1
  limits:
    memory: "2Gi"    # reserve 2Gi on NUMA node 1 for kubelet/system
```

---

### Monitoring CPU Manager

```bash
# Check CPU Manager state file
cat /var/lib/kubelet/cpu_manager_state
# {"policyName":"static","defaultCpuSet":"2-7,10-15",
#  "entries":{"pod-uid-abc":{"container-name":"4-5"}}}
# ← Shows which CPUs are pinned to which containers

# Verify CPU pinning inside container
kubectl exec -it latency-app -- cat /proc/self/status | grep Cpus_allowed
# Cpus_allowed:  00000030   ← hex bitmask = CPUs 4 and 5 (bits 4 and 5 set)
kubectl exec -it latency-app -- taskset -p 1
# pid 1's current affinity mask: 30  ← hex 0x30 = CPUs 4,5

# Check NUMA topology of node
kubectl exec -it latency-app -- numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 128720 MB
# node distances: 0  1
#                 0: 10 21   ← local=10, remote=21 (2.1x penalty)

# View node CPU topology
kubectl describe node worker-1 | grep -A 5 "Capacity:"
# Capacity:
#   cpu:    32           ← total CPUs
# kubectl get node worker-1 -o jsonpath='{.status.allocatable.cpu}'
# 30   ← 2 reserved for system, 30 allocatable

# After static policy: allocatable may reduce
# (pinned containers reduce shared pool)
```

---

### 🎤 Short Crisp Interview Answer

> *"CPU Manager static policy assigns exclusive, non-shared CPU cores to containers, eliminating cache thrashing and context switching that cause P99 latency spikes. It requires the pod to be Guaranteed QoS AND request integer CPU counts — fractional CPUs can't be pinned, they use CFS bandwidth which requires sharing. Kubelet divides node CPUs into a reserved pool for system processes, a shared pool for normal containers, and an exclusive pool for pinned containers. Memory Manager complements this by ensuring memory is allocated from the NUMA node local to the pinned CPUs, avoiding cross-socket memory access latency. The Topology Manager coordinates both so CPU, memory, and devices like GPUs or RDMA NICs all come from the same NUMA node."*

---

### ⚠️ Gotchas

1. **Must delete CPU Manager state on policy change** — changing cpuManagerPolicy from none to static requires deleting `/var/lib/kubelet/cpu_manager_state` before restarting kubelet, or it will fail to start.
2. **Integer CPUs consumed from shared pool** — when containers claim exclusive CPUs, the shared pool shrinks. If too many pinned containers run on a node, the shared pool becomes so small that other pods can't fit.
3. **full-pcpus-only prevents SMT sibling sharing** — without this option, a pinned container might get CPU 4 while CPU 4's HyperThreading sibling CPU 20 is in the shared pool. Other containers on CPU 20 still share the physical core's execution units, degrading performance.
4. **single-numa-node policy can cause Pending pods** — if a pod requests resources that span NUMA nodes, it will be rejected by Topology Manager. The pod stays Pending with topology violation. Reduce resource requests to fit within one NUMA node.

---

### Connections
- Guaranteed QoS required for CPU pinning — **6.5**
- Resource requests must equal limits (integer) — **6.1**
- Node-level resource management feeds into — **6.14 eviction**
- GPU topology (complementary use case) — **6.11 custom schedulers**

---

---

# 🏁 Category 6 — Complete Scheduling Decision Map

```
SCHEDULING DECISION TREE
═══════════════════════════════════════════════════════════════════

NEW POD ARRIVES → SCHEDULER
  │
  ├── FEASIBILITY (Filter phase)
  │     Does node have enough resources?
  │       requests.cpu + requests.memory fit?     → 6.1
  │       ResourceQuota not exceeded?             → 6.4
  │     Does pod tolerate all node taints?        → 6.8
  │     Does node satisfy required nodeAffinity?  → 6.6
  │     Does node satisfy required podAffinity?   → 6.7
  │     Does scheduling violate topologySpread?   → 6.10
  │       (DoNotSchedule only)
  │
  ├── SCORING (Score phase)
  │     Node resource fit score (spread vs pack)  → 6.12
  │     Preferred nodeAffinity score              → 6.6
  │     Preferred podAffinity score               → 6.7
  │     Preferred topologySpread score            → 6.10
  │     Priority class weight                     → 6.9
  │     Custom extender score                     → 6.11
  │
  ├── NO SUITABLE NODE FOUND?
  │     Preemption: evict lower-priority pods?    → 6.9
  │     Custom scheduler (GPU topology, gang)?    → 6.11
  │     Pod stays PENDING
  │
  └── NODE SELECTED → POD BOUND

RUNNING PODS
  │
  ├── NODE UNDER MEMORY/DISK/PID PRESSURE
  │     QoS class determines eviction order       → 6.5
  │     BestEffort → Burstable → Guaranteed        → 6.14
  │     Kubelet adds pressure taint to node        → 6.14
  │
  ├── CPU SHARING (default) vs PINNED (static)    → 6.15
  │     Integer Guaranteed QoS → exclusive CPUs
  │     All others → shared CFS scheduling
  │
  └── DESCHEDULER PERIODIC EVALUATION
        Spread violated?  → evict for rebalance   → 6.13
        Node taint added? → evict violating pods  → 6.13
        Low utilization?  → evict for consolidation → 6.13

POLICY OBJECTS GOVERNING ALL OF THE ABOVE
  LimitRange   → per-container defaults and bounds   → 6.3
  ResourceQuota → per-namespace aggregate caps       → 6.4
  PriorityClass → pod importance ranking             → 6.9
  Namespace     → policy scope boundary              → 6.2
```

---

# Quick Reference — Category 6 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **Requests** | Scheduler uses to find fit. Reserved on node. CPU: throttled if over. Memory: OOMKilled if over |
| **Limits** | Kernel enforces. CPU: CFS throttle (slow, not kill). Memory: OOMKill (exit 137) |
| **No CPU limit** | Recommended for latency-sensitive services — prevents throttle artifacts |
| **Namespace** | Logical partition. Soft isolation. Name scope + policy scope |
| **LimitRange** | Injection of defaults + min/max bounds at admission time (per container) |
| **ResourceQuota** | Aggregate caps per namespace. requests.cpu quota forces all pods to set CPU requests |
| **Guaranteed QoS** | req == lim, ALL containers. Evicted last. OOM score -998 |
| **Burstable QoS** | At least one container has req/lim set but not matching. Middle priority |
| **BestEffort QoS** | No resources at all. Evicted first. OOM score 1000. Avoid in production |
| **Node Affinity** | Pod selects node via node labels. Required=hard, Preferred=soft |
| **Pod Anti-Affinity** | Required: strict separation (can cause Pending). Preferred: soft spread |
| **topologyKey** | Defines domain unit for affinity/anti-affinity/spread (hostname=node, zone=AZ) |
| **Taints** | Node repels pods. NoSchedule/PreferNoSchedule/NoExecute |
| **Tolerations** | Pod allows scheduling on tainted node. Built-in: 300s not-ready tolerance |
| **PriorityClass** | Higher value = more important. Preemption evicts lower-priority for higher |
| **Topology Spread** | maxSkew = max imbalance. DoNotSchedule=hard. ScheduleAnyway=soft |
| **Bin packing** | MostAllocated scoring — fill nodes for cost reduction |
| **Spreading** | LeastAllocated scoring (default) — spread for resilience |
| **Descheduler** | Evicts badly-placed running pods. Respects PDBs. Fix topology drift |
| **Node eviction** | Kubelet protects node. BestEffort first. Hard=SIGKILL. Soft=SIGTERM |
| **CPU Manager** | Static policy: exclusive CPUs for Guaranteed+integer pods. Eliminates cache thrash |
| **Memory Manager** | NUMA-local memory for CPU-pinned containers |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| Default pod reschedule delay after node failure | 300 seconds (5 min) — not-ready/unreachable tolerationSeconds |
| OOM score — BestEffort | 1000 (most killable) |
| OOM score — Guaranteed | -998 (least killable) |
| Hard eviction default — memory | < 100Mi free |
| Hard eviction default — nodefs | < 10% free |
| Hard eviction default — imagefs | < 15% free |
| CPU Manager: min request for pinning | 1 (integer) |
| topologyKey for node-level spread | kubernetes.io/hostname |
| topologyKey for zone-level spread | topology.kubernetes.io/zone |
| Default scheduler scoring strategy | LeastAllocated (spreading) |
| Bin packing strategy name | MostAllocated |
| PriorityClass system-cluster-critical value | 2,000,000,000 |
| Max pods per node (default) | 110 |
| Local NUMA memory access | ~80 ns |
| Remote NUMA memory access | ~120-150 ns |

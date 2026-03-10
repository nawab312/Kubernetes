# Kubernetes Interview Mastery
# CATEGORY 1: CORE ARCHITECTURE & FUNDAMENTALS

---

> **How to use this document:**
> Each topic follows the same structure: Simple Explanation → Why It Exists → Internal Working → Key Components → Interview Answers (Short + Deep) → Production Example → Interview Questions → Gotchas → Connections to Other Topics.
> ⚠️ markers indicate frequently asked / commonly misunderstood topics — HIGH PRIORITY.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 1.1 | What is Kubernetes & Why It Exists | 🟢 Beginner |
| 1.2 | Kubernetes Architecture Overview | 🟢 Beginner |
| 1.3 | The API Server | 🟢 Beginner |
| 1.4 | etcd — The Brain / Source of Truth | 🟢 Beginner |
| 1.5 | Scheduler — How Pods Get Placed | 🟢 Beginner |
| 1.6 | Controller Manager — The Reconciliation Engine | 🟢 Beginner |
| 1.7 | Kubelet — The Node Agent | 🟢 Beginner |
| 1.8 | Kube-proxy — Networking on Nodes | 🟢 Beginner |
| 1.9 | Container Runtime (containerd, CRI-O) | 🟢 Beginner |
| 1.10 | How a Pod Creation Request Flows End-to-End ⚠️ | 🟡 Intermediate |
| 1.11 | Leader Election in Control Plane Components | 🟡 Intermediate |
| 1.12 | API Server Admission Flow | 🟡 Intermediate |
| 1.13 | Watch & Informer Pattern ⚠️ | 🟡 Intermediate |
| 1.14 | etcd Internals — Raft Consensus, Compaction, Defrag | 🔴 Advanced |
| 1.15 | API Server HA — How Multiple API Servers Coordinate | 🔴 Advanced |
| 1.16 | Custom API Servers & API Aggregation Layer | 🔴 Advanced |

---

## Difficulty Legend

- 🟢 **Beginner** — Expected from ALL candidates
- 🟡 **Intermediate** — Expected from 3+ year engineers
- 🔴 **Advanced** — Differentiates senior/staff candidates
- ⚠️ **High Priority** — Frequently asked / commonly misunderstood

---

---

# 1.1 What is Kubernetes & Why It Exists

## 🟢 Beginner

---

### What it is in simple terms

Kubernetes is a **container orchestration platform**. It takes your containerized applications and handles running them, keeping them alive, scaling them, and connecting them — across a fleet of machines — automatically.

Think of it as an **operating system for your datacenter**. Just like Linux manages processes on one machine, Kubernetes manages containers across many machines.

---

### Why it exists — The Evolution of Deployment

To understand Kubernetes, you need to understand the full evolution of how we ran software:

```
═══════════════════════════════════════════════════════════════════
EVOLUTION OF DEPLOYMENT
═══════════════════════════════════════════════════════════════════

ERA 1: BARE METAL
──────────────────
  [App A + App B + App C]  ←  all crammed onto one physical server

  Problems:
  ✗ App A's memory leak kills App B & C
  ✗ Dependency hell — App A needs Python 2, App B needs Python 3
  ✗ Scaling = physically buying & racking new servers (weeks)
  ✗ No portability — "works on my machine" syndrome
  ✗ Utilization is terrible — avg 10–15% CPU usage


ERA 2: VIRTUAL MACHINES
────────────────────────
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  VM1        │  │  VM2        │  │  VM3        │
  │  [App A]    │  │  [App B]    │  │  [App C]    │
  │  [Guest OS] │  │  [Guest OS] │  │  [Guest OS] │
  └─────────────┘  └─────────────┘  └─────────────┘
  ═══════════════════════════════════════════════════
                   [Hypervisor]
                  [Physical Server]

  Improvements: isolation ✓, portability ✓, multiple apps ✓
  Problems:
  ✗ Each VM carries a full OS = 1–4 GB RAM overhead per app
  ✗ Slow boot times — minutes to start
  ✗ Still manual: who decides which VM goes on which server?
  ✗ Scaling still slow — provision VM, install OS, configure...
  ✗ Underutilization — reserved but not fully used


ERA 3: CONTAINERS (Docker alone)
─────────────────────────────────
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │[App A]   │ │[App B]   │ │[App C]   │
  │[libs]    │ │[libs]    │ │[libs]    │
  └──────────┘ └──────────┘ └──────────┘
  ═══════════════════════════════════════
            [Host OS Kernel]          ← shared, not duplicated
            [Physical Server]

  Improvements:
  ✓ Lightweight (~MBs vs GBs)
  ✓ Start in milliseconds
  ✓ Portable — same image runs everywhere
  ✓ Isolated dependencies

  BUT — Docker manages ONE machine.
  New problems at scale:
  ✗ You have 500 containers across 50 servers — who manages them?
  ✗ Container dies at 3am — who restarts it?
  ✗ Traffic spikes — who adds more containers?
  ✗ Server dies — who moves containers to healthy servers?
  ✗ How do 500 containers find and talk to each other?
  ✗ How do you deploy v2 without downtime?
  ✗ How do you roll back when v2 is broken?


ERA 4: KUBERNETES — Container Orchestration
─────────────────────────────────────────────
  Kubernetes solves ALL of the above:

  ✓ Bin packing      — places containers optimally across nodes
  ✓ Self-healing     — restarts/reschedules dead containers
  ✓ Auto-scaling     — adds/removes containers based on load
  ✓ Service discovery— containers find each other by name
  ✓ Load balancing   — distributes traffic across container replicas
  ✓ Rolling updates  — deploy new versions with zero downtime
  ✓ Rollbacks        — one command to go back
  ✓ Config mgmt      — inject secrets/config without rebuilding images
  ✓ Storage mgmt     — attach/detach persistent storage automatically

═══════════════════════════════════════════════════════════════════
```

---

### Key Design Philosophy

**Declarative over Imperative** — You tell Kubernetes *what* you want, not *how* to do it.

```bash
# Imperative (Docker/shell):
docker run app
docker run app
docker run app   # you do the work manually

# Declarative (Kubernetes):
replicas: 3   # K8s figures out how to make it so, and KEEPS it so
```

**Control Loop / Reconciliation** — The entire system is built on one foundational idea:

```
Observe current state → Compare to desired state → Act to close the gap
         ↑___________________________________________________|
```

This loop runs **continuously, forever**. It is why Kubernetes is self-healing.

**Origin** — Born from Google's internal system called **Borg**, which ran approximately 2 billion containers per week. Open-sourced in 2014. Now governed by the **CNCF (Cloud Native Computing Foundation)**.

---

### 🎤 Short Crisp Interview Answer

> *"Kubernetes is a container orchestration system that automates deployment, scaling, and management of containerized applications across a cluster of machines. It exists because Docker alone only manages containers on a single host — at scale you need something to handle scheduling, self-healing, service discovery, rolling updates, and resource management across hundreds of nodes. Kubernetes solves all of that declaratively."*

---

### 📖 Deeper Interview Answer

> *"Beyond orchestration, Kubernetes embodies two core design principles. First, declarative state management — you describe what you want and Kubernetes continuously reconciles reality to match it. Second, the control loop pattern — every controller in the system runs an infinite observe-diff-act loop. This is why Kubernetes is self-healing: a controller always notices drift from desired state and corrects it. The system originated from Google's Borg, which proved this model at planetary scale. Kubernetes also introduces a universal API model where every resource — pods, services, configs — is an API object with a spec (desired) and status (actual), and controllers close the gap between them."*

---

### ⚠️ Common Gotcha

Kubernetes is **NOT** a PaaS. It does not manage your code, your CI/CD pipeline, or your application logic. It manages *infrastructure* for containers. Many candidates confuse it with Heroku-style platforms. Kubernetes gives you primitives — you build the platform on top of it.

---

### Common Interview Questions

**Q: How is Kubernetes different from Docker Compose?**
> Docker Compose manages multi-container apps on a **single machine**. Kubernetes orchestrates containers across **many machines**, with self-healing, auto-scaling, and production-grade networking built in.

**Q: What problem does Kubernetes solve that Docker does not?**
> Docker runs containers on one host. Kubernetes solves cluster-wide scheduling, self-healing, service discovery, rolling updates, autoscaling, and storage management across a fleet of nodes.

**Q: What is the core design principle of Kubernetes?**
> Declarative desired state combined with continuous reconciliation loops. You declare what you want; controllers continuously ensure reality matches it.

---

### Connections to Other Topics

- The **declarative model** underpins every resource type: Deployments (1.6), Services, ConfigMaps
- The **reconciliation loop** is what the Controller Manager (1.6) implements
- The **API model** (spec vs status) is why the API Server (1.3) and etcd (1.4) are designed the way they are

---

---

# 1.2 Kubernetes Architecture Overview

## 🟢 Beginner

---

### What it is in simple terms

A Kubernetes cluster has two planes:

- **Control Plane** — the brain. Makes all decisions. Stores all state.
- **Data Plane (Worker Nodes)** — the muscle. Runs your actual workloads.

---

### Full Architecture Diagram

```
═══════════════════════════════════════════════════════════════════════════
KUBERNETES CLUSTER ARCHITECTURE
═══════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────┐
│                         CONTROL PLANE                                   │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────────┐ │
│  │  API Server  │  │     etcd     │  │      Controller Manager       │ │
│  │  (kube-      │  │  (key-value  │  │  (Deployment, Node, ReplicaSet│ │
│  │  apiserver)  │  │   store)     │  │   controllers, etc.)          │ │
│  └──────┬───────┘  └──────────────┘  └───────────────────────────────┘ │
│         │                                                               │
│  ┌──────┴───────┐                                                       │
│  │  Scheduler   │                                                       │
│  │(kube-        │                                                       │
│  │ scheduler)   │                                                       │
│  └──────────────┘                                                       │
│                                                                         │
│  ★ KEY RULE: Only the API Server talks to etcd directly ★               │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │  HTTPS / API calls
           ┌────────────────────┼─────────────────────┐
           ▼                    ▼                      ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   WORKER NODE 1  │  │   WORKER NODE 2  │  │   WORKER NODE 3  │
│                  │  │                  │  │                  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │  Kubelet   │  │  │  │  Kubelet   │  │  │  │  Kubelet   │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │ Kube-proxy │  │  │  │ Kube-proxy │  │  │  │ Kube-proxy │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │  Container │  │  │  │  Container │  │  │  │  Container │  │
│  │  Runtime   │  │  │  │  Runtime   │  │  │  │  Runtime   │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │  [Pod]     │  │  │  │  [Pod]     │  │  │  │  [Pod]     │  │
│  │  [Pod]     │  │  │  │  [Pod]     │  │  │  │  [Pod]     │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
└──────────────────┘  └──────────────────┘  └──────────────────┘

═══════════════════════════════════════════════════════════════════════════
```

---

### Control Plane Components

| Component | Role |
|---|---|
| **kube-apiserver** | Front door. All requests go through here. Validates, authenticates, persists to etcd |
| **etcd** | Distributed KV store. Single source of truth for ALL cluster state |
| **kube-scheduler** | Watches for unscheduled Pods, picks the best node for each |
| **kube-controller-manager** | Runs all built-in control loops (Node, Deployment, ReplicaSet, etc.) |
| **cloud-controller-manager** | Talks to cloud provider API (AWS, GCP) for nodes, LBs, storage |

---

### Worker Node Components

| Component | Role |
|---|---|
| **kubelet** | Agent on every node. Ensures containers described in PodSpec are running |
| **kube-proxy** | Manages iptables/IPVS rules for Service networking on each node |
| **container runtime** | Actually runs containers. containerd or CRI-O (Docker removed in K8s 1.24+) |

---

### 🎤 Short Crisp Interview Answer

> *"A Kubernetes cluster has a control plane and worker nodes. The control plane has four main components: the API server which is the central gateway, etcd which stores all cluster state, the scheduler which assigns pods to nodes, and the controller manager which runs reconciliation loops. Worker nodes have the kubelet which manages pods, kube-proxy which handles service networking, and a container runtime like containerd. The critical architectural rule is that only the API server talks to etcd — every other component communicates through the API server."*

---

### ⚠️ Gotchas

1. The **control plane itself runs on nodes** — in production you typically have 3 or 5 dedicated control plane nodes for HA.
2. In **managed clusters like EKS**, the control plane is fully hidden — AWS runs it, you only see worker nodes.
3. The **cloud-controller-manager** is separate from kube-controller-manager and handles cloud-provider-specific resources like LoadBalancers and node lifecycle via cloud APIs.

---

### Common Interview Questions

**Q: What happens if the control plane goes down?**
> Running workloads continue running on nodes — the kubelet keeps containers alive. But you cannot schedule new pods, scale deployments, or make any API changes until the control plane recovers. This is why HA control planes (3+ nodes) are critical in production.

**Q: What is the single most important architectural rule in Kubernetes?**
> Only the API server talks to etcd. All other components communicate through the API server. This centralizes all state access and enables consistent authentication, authorization, and admission control.

---

### Connections to Other Topics

- Each control plane component is detailed in topics 1.3 through 1.9
- The HA architecture of the control plane is covered in 1.11 and 1.15
- Worker node networking connects to kube-proxy (1.8) and the CNI layer (Category 3)

---

---

# 1.3 The API Server

## 🟢 Beginner

---

### What it is in simple terms

The API server is the **single front door** to the entire Kubernetes cluster. Every action — creating a pod, scaling a deployment, checking node status — goes through it. Nothing communicates directly with anything else. Everything goes through the API server.

---

### Why it exists and what problem it solves

Without a central gateway you would have chaos: every component would need to know where every other component is, implement its own auth, handle its own consistency. The API server centralizes all of this:

- **Authentication** — proves who you are
- **Authorization** — proves you're allowed
- **Admission** — enforces policy
- **Validation** — ensures correctness
- **Persistence** — writes to etcd
- **Watch API** — notifies all components of changes

---

### How it works internally — The Request Pipeline

```
═══════════════════════════════════════════════════════════════
API SERVER REQUEST PIPELINE
═══════════════════════════════════════════════════════════════

Client (kubectl / controller / kubelet / webhook)
         │
         ▼ HTTPS request
┌─────────────────────────────────────────────────────────────┐
│                      API SERVER                             │
│                                                             │
│  STEP 1: AUTHENTICATION  ──→  Who are you?                  │
│     • x509 client certificates                              │
│     • Bearer tokens (ServiceAccount JWT)                    │
│     • OIDC tokens (external identity provider)              │
│     • Bootstrap tokens (node join)                          │
│                                                             │
│  STEP 2: AUTHORIZATION   ──→  Are you allowed?              │
│     • RBAC (most common)                                    │
│     • ABAC (legacy)                                         │
│     • Node authorizer                                       │
│     • Webhook authorizer                                    │
│                                                             │
│  STEP 3: ADMISSION CONTROL ──→ Should this be mutated?      │
│     • Mutating webhooks  (CAN change the object)            │
│     • Validating webhooks (CAN only reject the object)      │
│     • Built-in: LimitRanger, ResourceQuota, PSA             │
│                                                             │
│  STEP 4: VALIDATION      ──→  Is the schema valid?          │
│     • OpenAPI schema validation                             │
│     • Type checks, required field checks                    │
│                                                             │
│  STEP 5: PERSIST TO etcd ──→  Write to source of truth      │
│                                                             │
│  STEP 6: RETURN response ──→  201 Created / 200 OK / 4xx    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼  etcd watch events trigger controllers and kubelet
```

---

### Key Properties of the API Server

**It is stateless** — holds no state itself. All state lives in etcd. This is why you can run multiple API servers for HA — any one of them can handle any request.

**It is RESTful** — every Kubernetes resource is a REST resource:

```bash
GET    /api/v1/namespaces/default/pods           # list pods
GET    /api/v1/namespaces/default/pods/mypod     # get specific pod
POST   /api/v1/namespaces/default/pods           # create pod
PUT    /api/v1/namespaces/default/pods/mypod     # full update
PATCH  /api/v1/namespaces/default/pods/mypod     # partial update
DELETE /api/v1/namespaces/default/pods/mypod     # delete pod
WATCH  /api/v1/namespaces/default/pods?watch=1   # stream changes
```

**kubectl is just an HTTP client:**

```bash
# See exactly what kubectl sends to the API server
kubectl get pods -v=8    # verbosity level 8 — shows full HTTP request/response
kubectl get pods -v=9    # verbosity level 9 — shows request body too
```

**API Groups — How the API is organized:**

```
Core API group:   /api/v1              → Pods, Services, ConfigMaps, Secrets, Nodes
Named API groups: /apis/apps/v1        → Deployments, ReplicaSets, StatefulSets
                  /apis/batch/v1       → Jobs, CronJobs
                  /apis/rbac.../v1     → Roles, ClusterRoles, Bindings
                  /apis/networking/v1  → NetworkPolicies, Ingress
                  /apis/custom.io/v1   → Your CRDs live here
```

---

### 🎤 Short Crisp Interview Answer

> *"The API server is the central gateway for all Kubernetes operations. Every component — kubectl, controllers, kubelets — communicates exclusively through it. A request flows through authentication, authorization, admission control, validation, and then gets persisted to etcd. The API server is stateless, which is how you can run multiple instances for high availability. It exposes a RESTful API and supports watch semantics, which is how controllers get notified of changes."*

---

### 📖 Deeper Interview Answer

> *"The API server is the only component that directly reads and writes etcd, which means it's the single point of consistency enforcement. Its stateless design is key — because all state is in etcd, you can run 2 or 3 API servers behind a load balancer and any one handles any request. The watch mechanism is especially important: rather than polling, clients open a long-lived HTTP connection with a resourceVersion parameter, and the API server streams change events as they happen. This is what makes the entire controller and informer architecture efficient. The API server also runs the admission pipeline — mutating webhooks run first to modify objects, then validating webhooks run to approve or reject. This is where policy engines like OPA/Gatekeeper plug in."*

---

### ⚠️ Gotcha

The API server **does not push work to kubelets**. Kubelets **watch** (via long-lived HTTP watch connection) the API server for pods assigned to their node. This is a **pull model, not push**. Many candidates get this backwards.

---

### Common Interview Questions

**Q: What happens when the API server is unavailable?**
> Running pods continue running because kubelet keeps them alive independently. But nothing new can be scheduled, no deployments can be updated, and kubectl commands fail. Controllers stop reconciling. The cluster is in a read-only frozen state from an operational perspective.

**Q: How does kubectl authenticate to the API server?**
> By default via client certificate in kubeconfig. The certificate's Common Name becomes the username, the Organization field maps to groups. Service accounts use JWT bearer tokens. OIDC integrates with external identity providers like AWS IAM or Google.

**Q: What is the difference between authentication and authorization in K8s?**
> Authentication answers "who are you?" — validates the identity via cert/token/OIDC. Authorization answers "are you allowed?" — RBAC checks if that identity has permission to perform the requested verb (get/create/delete) on the requested resource in the requested namespace.

---

### Connections to Other Topics

- The admission pipeline connects to **Admission Controllers (7.4)** and **Webhook Admission (7.5)**
- The watch mechanism is the foundation of the **Informer Pattern (1.13)**
- RBAC authorization connects to **RBAC deep dive (7.1)**
- API Groups are how **CRDs (10.1)** extend the Kubernetes API

---

---

# 1.4 etcd — The Brain / Source of Truth

## 🟢 Beginner

---

### What it is in simple terms

etcd is a **distributed key-value store** that holds the entire state of your Kubernetes cluster. Every object — every pod spec, every service, every secret, every node status — lives in etcd. If you lose etcd without a backup, your cluster state is gone permanently.

---

### Why it exists and what problem it solves

Kubernetes needs a place to store cluster state that is:

- **Consistent** — all readers see the same data at the same time
- **Highly available** — survives individual node failures
- **Strongly ordered** — writes are applied in a deterministic sequence
- **Watchable** — clients can subscribe to changes

etcd provides all of these through the Raft consensus algorithm.

---

### How it works internally

```
═══════════════════════════════════════════════════════════════
etcd IN KUBERNETES
═══════════════════════════════════════════════════════════════

WHAT'S STORED AND WHERE:
──────────────────────────────────────────────────────────────
  Resource Type         Key in etcd
  ─────────────────     ──────────────────────────────────────
  Pod specs             /registry/pods/default/mypod
  Deployment specs      /registry/deployments/default/myapp
  Node registrations    /registry/minions/node1
  Secrets (encrypted)   /registry/secrets/default/mysecret
  ConfigMaps            /registry/configmaps/default/myconfig
  RBAC rules            /registry/clusterroles/admin
  Service definitions   /registry/services/specs/default/mysvc
  Lease objects         /registry/leases/kube-system/kube-scheduler

CLUSTER TOPOLOGY — Production (always odd number):
────────────────────────────────────────────────────────────

  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  etcd-1  │◄──►│  etcd-2  │◄──►│  etcd-3  │
  │ (LEADER) │    │(FOLLOWER)│    │(FOLLOWER)│
  └──────────┘    └──────────┘    └──────────┘

  • All WRITES go to the LEADER
  • READs can be served by any member
  • QUORUM required: (n/2)+1 nodes must be alive

  Quorum math:
  3-node cluster → needs 2 alive to function (tolerates 1 failure)
  5-node cluster → needs 3 alive to function (tolerates 2 failures)
  4-node cluster → needs 3 alive (same tolerance as 3-node — wasteful)

  WHY ALWAYS ODD:
  A 4-node cluster tolerates only 1 failure — same as 3-node.
  4 nodes buys you NOTHING over 3. Always use 3 or 5.

═══════════════════════════════════════════════════════════════
```

---

### Critical etcd Operations for SREs

```bash
# Check etcd cluster health
etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check cluster member list
etcdctl member list --write-out=table

# Take a snapshot backup — CRITICAL for disaster recovery
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db

# Verify backup integrity
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table

# Restore from snapshot
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# Compact old revisions (prevents unbounded growth)
etcdctl compact $(etcdctl endpoint status \
  --write-out="json" | jq '.[0].Status.header.revision')

# Defrag after compaction (physically reclaims disk space)
etcdctl defrag --endpoints=https://127.0.0.1:2379

# Check database size — monitor this in production
etcdctl endpoint status --write-out=table
```

---

### 🎤 Short Crisp Interview Answer

> *"etcd is a distributed key-value store that serves as the single source of truth for all Kubernetes cluster state. Every object — pods, deployments, secrets, node registrations — is stored as a key-value entry under /registry/. It uses the Raft consensus algorithm to stay consistent across a cluster of typically 3 or 5 nodes. Only the API server talks to etcd directly. In production, regular etcd snapshots are critical — if you lose etcd without a backup, the entire cluster state is unrecoverable."*

---

### 📖 Deeper Interview Answer

> *"etcd uses MVCC — Multi-Version Concurrency Control — which means every write creates a new revision rather than overwriting the previous one. This is how the API server's watch mechanism works: clients subscribe from a specific resourceVersion and receive only incremental changes. etcd is also the basis for leader election in Kubernetes — the scheduler and controller manager create Lease objects in etcd and use optimistic concurrency to compete for leadership. Operationally, the two most important practices are: backup (etcdctl snapshot save) for disaster recovery, and monitoring db size because etcd has a default 2GB quota — if it fills up, etcd goes read-only and the cluster stops accepting writes. You should compact old revisions regularly and defrag to physically reclaim space, always running defrag on followers before the leader."*

---

### ⚠️ Gotchas

1. **etcd is NOT a general-purpose database** — it has a default **2GB size limit** (max recommended 8GB) and is not designed for high-throughput writes. Storing large CRD objects or high-frequency updates will cause problems.
2. **Slow disk kills the cluster** — etcd is extremely sensitive to disk I/O latency. In production, etcd nodes must have **dedicated SSDs**. etcd should **never share a disk** with application workloads.
3. **Compaction vs Defrag are different** — compaction removes old revisions *logically* (marks them deleted); defrag *physically* reclaims disk space. You need both. Defrag alone without compaction does nothing meaningful.
4. **etcd going read-only is a P0 incident** — the entire cluster stops accepting new writes. Alert when db size exceeds 70% of quota.

---

### Common Interview Questions

**Q: How do you back up and restore etcd?**
> Back up using `etcdctl snapshot save`. Restore with `etcdctl snapshot restore`, then update the etcd pod manifest to point to the restored data directory. After restore, restart the control plane components.

**Q: Why must etcd clusters always have an odd number of nodes?**
> Because of quorum — you need (n/2)+1 nodes alive to function. A 4-node cluster needs 3 alive, which is the same as a 3-node cluster. The extra node adds cost with no additional fault tolerance. Always use 3 or 5.

**Q: What happens if etcd loses quorum?**
> The etcd cluster goes into a read-only state. The API server can still serve reads from its watch cache but cannot persist new writes. The entire cluster is effectively frozen until quorum is restored.

---

### Connections to Other Topics

- etcd's watch mechanism enables the **Informer Pattern (1.13)**
- Lease objects in etcd power **Leader Election (1.11)**
- **API Server HA (1.15)** depends on etcd's optimistic concurrency
- **etcd Raft internals** are covered in depth in topic 1.14

---

---

# 1.5 Scheduler — How Pods Get Placed

## 🟢 Beginner

---

### What it is in simple terms

The scheduler watches for **newly created Pods that have no node assigned** and picks the best node for each one. That is literally its only job — but the decision of "best node" involves a sophisticated two-phase pipeline.

---

### Why it exists and what problem it solves

Without a scheduler, someone would have to manually decide which server to run each container on. At scale with hundreds of nodes and thousands of pods, this is impossible manually. The scheduler automates optimal placement considering resources, topology, affinity rules, and policies.

---

### How it works internally — The Scheduling Pipeline

```
═══════════════════════════════════════════════════════════════
SCHEDULER DECISION PIPELINE
═══════════════════════════════════════════════════════════════

New Pod created with spec.nodeName = "" (empty — not scheduled)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1: FILTERING  (Predicates)                           │
│                                                             │
│  Eliminate nodes that CANNOT run this pod.                  │
│                                                             │
│  Checks performed:                                          │
│  • Does node have enough CPU and memory?                    │
│  • Does node satisfy nodeSelector labels?                   │
│  • Does node satisfy nodeAffinity rules?                    │
│  • Does node have taints the pod doesn't tolerate?          │
│  • Are requested ports available on the node?               │
│  • Does node satisfy volume topology requirements?          │
│  • Is the node in Ready state?                              │
│  • Does pod fit within node's allocatable resources?        │
│                                                             │
│  Input:  ALL nodes in the cluster                           │
│  Output: Subset of FEASIBLE nodes                           │
│                                                             │
│  If 0 feasible nodes → Pod status = Pending                 │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2: SCORING  (Priorities)                             │
│                                                             │
│  Rank feasible nodes to find the BEST one.                  │
│                                                             │
│  Scoring plugins (each scores 0-100):                       │
│  • LeastAllocated      — prefer nodes with more free space  │
│  • MostAllocated       — bin-pack onto fewer nodes          │
│  • ImageLocality       — prefer nodes with image cached     │
│  • InterPodAffinity    — honor pod affinity preferences     │
│  • TopologySpread      — spread across zones and nodes      │
│  • NodeAffinity        — bonus for preferred node labels    │
│                                                             │
│  Weighted sum of all plugin scores = final node score.      │
│  Highest score wins.                                        │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3: BINDING                                           │
│                                                             │
│  Scheduler writes spec.nodeName = "winning-node" to         │
│  the Pod object via the API Server.                         │
│  API Server persists to etcd.                               │
│  Kubelet on that node detects the assignment and starts     │
│  the container.                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Key Scheduling Concepts with YAML

```yaml
# ── nodeSelector — simplest hard requirement ──────────────────
spec:
  nodeSelector:
    disktype: ssd

# ── Node Affinity — expressive hard and soft rules ────────────
spec:
  affinity:
    nodeAffinity:
      # HARD rule — pod will not schedule if not satisfied
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: In
            values: [us-east-1a, us-east-1b]
      # SOFT rule — scheduler prefers but doesn't require
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: [ssd]

# ── Taints on a node (via kubectl) ───────────────────────────
# NoSchedule    — won't schedule NEW pods without toleration
# PreferNoSchedule — tries not to, but will if necessary
# NoExecute     — evicts EXISTING pods too (if no toleration)
kubectl taint nodes node1 env=prod:NoSchedule

# ── Toleration on pod — allows scheduling on tainted node ─────
spec:
  tolerations:
  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"

# ── Topology Spread Constraints ───────────────────────────────
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

---

### 🎤 Short Crisp Interview Answer

> *"The scheduler watches for pods with no node assigned and selects the best node through a two-phase process. First, filtering eliminates nodes that can't satisfy the pod's requirements — insufficient resources, missing labels, incompatible taints, failed affinity rules. Second, scoring ranks the remaining nodes based on priorities like least allocated resources, image locality, and topology spread. The scheduler then binds the pod to the winning node by writing the node name to the pod spec via the API server."*

---

### ⚠️ Gotcha — "IgnoredDuringExecution"

The phrase `IgnoredDuringExecution` in affinity rules means: if a pod is already running and the node's labels **change**, Kubernetes will **NOT evict the pod**. The rule is only enforced at scheduling time. A future `RequiredDuringExecution` mode is planned. Most candidates miss this nuance.

---

### Common Interview Questions

**Q: What happens when a pod stays in Pending state?**
> The scheduler cannot find a feasible node. Common causes: insufficient CPU/memory across all nodes, nodeSelector labels don't match any node, taints without matching tolerations, PVC can't be bound, or pod anti-affinity conflicts. Debug with `kubectl describe pod <name>` — the Events section shows the exact reason.

**Q: What is the difference between nodeSelector and nodeAffinity?**
> nodeSelector is simple key-value matching — hard requirement only. nodeAffinity supports richer expressions (In, NotIn, Exists, Gt, Lt operators), and supports both hard (required) and soft (preferred) rules with configurable weights.

**Q: How do taints and tolerations differ from node affinity?**
> Taints and tolerations are about **repelling** pods from nodes unless they explicitly opt in. Node affinity is about **attracting** pods to nodes. Taints prevent pods from landing unless they tolerate the taint. Affinity rules pull pods toward preferred nodes.

---

### Connections to Other Topics

- Filtering uses **resource requests (6.1)** to check capacity
- Taints connect to **node lifecycle — cordon and drain (9.4)**
- TopologySpread is critical for **multi-AZ reliability**
- Custom schedulers and scheduler extenders are covered in topic **6.11**

---

---

# 1.6 Controller Manager — The Reconciliation Engine

## 🟢 Beginner

---

### What it is in simple terms

The controller manager is a **single binary that runs dozens of independent control loops**. Each loop watches the state of some resource and takes action to make reality match what you declared. It is the engine of Kubernetes' self-healing capability.

---

### The Reconciliation Loop — Core Pattern

```
═══════════════════════════════════════════════════════════════
THE RECONCILIATION LOOP  (runs for EVERY controller, forever)
═══════════════════════════════════════════════════════════════

                 ┌─────────────────────────┐
                 │      Desired State       │
                 │  (what's declared in     │
                 │   your YAML / etcd)      │
                 │   e.g., replicas: 3      │
                 └──────────┬──────────────┘
                            │
                            ▼
  ┌──────────────────────────────────────────────────────────┐
  │                  CONTROLLER LOOP                         │
  │                                                          │
  │  1. OBSERVE   — watch API server for resource changes    │
  │  2. DIFF      — compare current state vs desired state   │
  │  3. ACT       — take action to close the gap             │
  │  4. REPEAT    — forever, continuously                    │
  └──────────────────────────────────────────────────────────┘
                            │
                            ▼
                 ┌─────────────────────────┐
                 │      Current State       │
                 │  (what's actually        │
                 │   running right now)     │
                 │   e.g., running: 2       │
                 └─────────────────────────┘

  Gap detected: need 1 more pod
  → create pod object in API server
  → scheduler picks a node
  → kubelet starts it
  → loop runs again: current = desired → no action
```

---

### Key Built-in Controllers

```
Controller                  What It Does
─────────────────────────── ──────────────────────────────────────────────
ReplicaSet Controller       Ensures correct number of pod replicas running
Deployment Controller       Manages ReplicaSets for rolling updates
StatefulSet Controller      Manages ordered, stable pod identities
DaemonSet Controller        Ensures one pod per matching node
Job Controller              Ensures batch jobs run to completion
CronJob Controller          Creates Jobs on a cron schedule
Node Controller             Monitors node health, taints unreachable nodes
Service Controller          Creates cloud LBs for LoadBalancer services
EndpointSlice Controller    Keeps EndpointSlices in sync with pod IPs
PVC Controller              Binds PVCs to matching PVs
Namespace Controller        Cleans up resources when namespace is deleted
ServiceAccount Controller   Creates default SA in new namespaces
```

---

### Real World Production Example — Node Failure Self-Healing

You have a Deployment with `replicas: 3`. A worker node dies. Here is exactly what happens automatically:

```
TIME 0:    Node hardware failure — node stops responding

TIME +40s: Node Controller detects node unreachable
           (node-monitor-grace-period = 40s default)
           → marks node condition: Ready=False

TIME +5min: Node Controller applies NoExecute taint to node
            (pod-eviction-timeout = 5min default)
            → pods on dead node get eviction notice

TIME +5min: ReplicaSet Controller observes:
            running replicas = 1 (was 3, two on dead node)
            desired = 3, current = 1 → need 2 more pods
            → creates 2 new pod objects in API server

TIME +5min+1s: Scheduler sees 2 unscheduled pods
               → places them on remaining healthy nodes
               → writes node name to pod specs

TIME +5min+3s: Kubelet on healthy nodes sees new pods assigned
               → pulls images (from cache: milliseconds)
               → starts containers

TIME +5min+5s: Pods pass readiness probes
               → EndpointSlice controller adds their IPs to Service
               → kube-proxy updates rules on all nodes
               → traffic routes to the new pods

TOTAL RECOVERY TIME: approximately 5-6 minutes, fully automatic
```

---

### 🎤 Short Crisp Interview Answer

> *"The controller manager runs dozens of independent reconciliation loops — one for each resource type like Deployments, ReplicaSets, Nodes, Jobs. Each loop continuously compares desired state from etcd to actual state in the cluster, and takes actions to close any gap. This is the core self-healing mechanism of Kubernetes. For example, if a node dies and pods are lost, the ReplicaSet controller detects the shortfall and creates replacement pods, which the scheduler then places on healthy nodes."*

---

### ⚠️ Gotcha

Controllers **only act through the API server** — they never directly touch nodes or containers. A controller creates a pod object, and the scheduler + kubelet handle the rest. This indirection is intentional and is what makes the architecture composable and auditable.

---

### Common Interview Questions

**Q: What is the difference between a controller and an operator?**
> A controller is a reconciliation loop managing a built-in resource type. An operator is a custom controller paired with a CRD — it manages a custom resource representing a complex application (like a database cluster), encoding operational knowledge into code.

**Q: Why does Kubernetes not act immediately on changes? What is eventual consistency?**
> Kubernetes is eventually consistent — it guarantees it will reach the desired state, but not instantaneously. There is latency in watch propagation, scheduling decisions, image pulls, and probe checks. This is by design and is acceptable for distributed systems.

---

### Connections to Other Topics

- Reconciliation connects to **Informer Pattern (1.13)** — how controllers observe changes efficiently
- **Leader Election (1.11)** ensures only one controller manager instance is active
- Custom controllers and operators are covered in **Category 10 (CRDs & Extensibility)**

---

---

# 1.7 Kubelet — The Node Agent

## 🟢 Beginner

---

### What it is in simple terms

The kubelet is an **agent that runs on every worker node**. It is the bridge between the control plane's intentions and what actually runs on a machine. It takes a PodSpec and makes it real on that node.

---

### How it works internally

```
═══════════════════════════════════════════════════════════════
KUBELET RESPONSIBILITIES
═══════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────┐
│                         KUBELET                              │
│                                                              │
│  INPUTS:                                                     │
│  • Watches API Server for pods assigned to THIS node         │
│  • Static pod manifests from /etc/kubernetes/manifests/      │
│                                                              │
│  CORE ACTIONS:                                               │
│  1. Calls Container Runtime via CRI to start/stop containers │
│  2. Manages pod volumes — mounts and unmounts                │
│  3. Runs liveness, readiness, and startup probes             │
│  4. Reports pod status back to API server                    │
│  5. Reports node status (CPU/mem available) to API server    │
│  6. Enforces resource limits via Linux cgroups               │
│  7. Manages pod logs (writes to /var/log/pods/)              │
│  8. Evicts pods under node resource pressure                 │
│                                                              │
│  DOES NOT:                                                   │
│  ✗ Manage containers not created by Kubernetes               │
│  ✗ Talk to etcd directly                                     │
│  ✗ Talk to other kubelets directly                           │
└──────────────────────────────────────────────────────────────┘
         │
         │  CRI — Container Runtime Interface (gRPC)
         ▼
┌──────────────────────────────────────────────────────────────┐
│                    Container Runtime                         │
│                  (containerd / CRI-O)                        │
│  Pulls images, creates/runs/stops containers                 │
└──────────────────────────────────────────────────────────────┘
         │
         │  OCI — Open Container Initiative
         ▼
┌──────────────────────────────────────────────────────────────┐
│                       runc / kata                            │
│          (calls Linux kernel to create the process)          │
└──────────────────────────────────────────────────────────────┘
```

---

### ⚠️ Static Pods — Critical Detail

```
Control plane components (API server, etcd, scheduler, controller-manager)
run as STATIC PODS — their manifests live in /etc/kubernetes/manifests/
on the control plane node.

The kubelet on the control plane node reads these files DIRECTLY,
without going through the API server.

This is how Kubernetes bootstraps itself — the kubelet is the one
component that does NOT need Kubernetes running to start Kubernetes.

ls /etc/kubernetes/manifests/
  etcd.yaml
  kube-apiserver.yaml
  kube-controller-manager.yaml
  kube-scheduler.yaml

If you need to troubleshoot a control plane component, edit these files
directly — kubelet will detect the change and restart the component.
```

---

### Kubelet Useful Commands

```bash
# Check kubelet status on a node
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -f

# View kubelet configuration
cat /var/lib/kubelet/config.yaml

# Force kubelet to re-evaluate pod (by deleting pod — kubelet restarts it)
kubectl delete pod <pod-name>

# Check what kubelet is doing via API server
kubectl describe node <node-name>    # Conditions, Capacity, Allocatable
kubectl get node <node-name> -o yaml # Full node object including status
```

---

### 🎤 Short Crisp Interview Answer

> *"The kubelet is the agent running on every node that makes pod specs into running containers. It watches the API server for pods assigned to its node, then calls the container runtime via the CRI interface to start them. It also runs health probes, manages volumes, enforces resource limits through cgroups, and reports pod and node status back to the API server. An important detail is that control plane components themselves run as static pods — the kubelet on the control plane node reads their manifests directly from disk, which is how Kubernetes bootstraps itself."*

---

### ⚠️ Gotchas

1. **Kubelet does not restart itself** — if kubelet crashes, it requires systemd or another process manager to restart it. It is the only Kubernetes component that runs outside of Kubernetes.
2. **Node allocatable vs capacity** — kubelet reserves resources for the OS and itself. The allocatable resources (what pods can use) is always less than the node's total capacity.
3. **Probe failures cause restarts and traffic removal** — liveness probe failure → container restart. Readiness probe failure → pod removed from Service endpoints (traffic stops). Many production outages are caused by misconfigured probes.

---

### Common Interview Questions

**Q: How does kubelet know which pods to run on its node?**
> It watches the API server for pod objects where `spec.nodeName` matches its node name. The watch is a long-lived HTTP connection that streams events as they happen.

**Q: What is the difference between liveness, readiness, and startup probes?**
> Liveness: is the container alive? Failure → container is killed and restarted. Readiness: is the container ready to receive traffic? Failure → pod removed from Service endpoints but NOT restarted. Startup: is the app done starting? Disables liveness/readiness until it passes, preventing premature restarts of slow-starting apps.

---

### Connections to Other Topics

- Probes connect to **Observability — Pod health (8.2)**
- Resource enforcement via cgroups relates to **QoS classes (6.5)**
- Static pods are key to understanding **control plane HA (1.15)**
- CRI connects to **Container Runtime topic (1.9)**

---

---

# 1.8 Kube-proxy — Networking on Nodes

## 🟢 Beginner

---

### What it is in simple terms

Kube-proxy runs on every node and **implements Kubernetes Service networking**. When you create a Service with a ClusterIP, kube-proxy is what makes traffic to that virtual IP actually reach your pods — by programming the Linux kernel's packet routing.

---

### Why it exists and what problem it solves

Kubernetes Services have virtual IPs (ClusterIPs) that don't exist on any real network interface. Traffic destined for these virtual IPs needs to be intercepted and redirected to real pod IPs. Kube-proxy programs the Linux kernel to do this transparently.

---

### How it works — Two Main Modes

```
═══════════════════════════════════════════════════════════════
KUBE-PROXY MODES
═══════════════════════════════════════════════════════════════

THE PROBLEM:
  Service ClusterIP:  10.96.0.50    (virtual — no interface owns this)
  Pod IPs:            10.244.1.5
                      10.244.2.3
                      10.244.3.8
  
  Traffic to 10.96.0.50:80 must be:
  1. Intercepted
  2. DNAT'd (Destination NAT) to one real pod IP
  3. Load balanced across all healthy pods

──────────────────────────────────────────────────────────────
MODE 1: iptables (default mode)
──────────────────────────────────────────────────────────────
  kube-proxy watches Services and Endpoints.
  Programs iptables rules in the kernel:

  KUBE-SERVICES chain:
  → dst 10.96.0.50:80 → jump KUBE-SVC-XXXX

  KUBE-SVC-XXXX chain (probabilistic load balancing):
  → 33% chance → jump KUBE-SEP-AAA → DNAT to 10.244.1.5:8080
  → 50% chance → jump KUBE-SEP-BBB → DNAT to 10.244.2.3:8080
  → otherwise  → jump KUBE-SEP-CCC → DNAT to 10.244.3.8:8080

  Performance:
  → Rule lookup is O(n) — every packet traverses ALL rules
  → At 10,000 services: tens of thousands of rules per packet
  → CPU overhead grows linearly with cluster size

──────────────────────────────────────────────────────────────
MODE 2: IPVS (recommended for large clusters, 1000+ services)
──────────────────────────────────────────────────────────────
  Uses Linux kernel's IPVS (IP Virtual Server) module.
  Uses hash tables internally → O(1) lookup.

  Supports real load balancing algorithms:
  → rr        (round-robin)
  → lc        (least connections)
  → sh        (source hash — session affinity)
  → wrr       (weighted round-robin)

  Enable in kubeadm:
  kubeadm init --config=config.yaml
  # config.yaml: mode: ipvs

  Enable on existing cluster:
  kubectl edit configmap kube-proxy -n kube-system
  # Set mode: "ipvs"

──────────────────────────────────────────────────────────────
MODE 3: eBPF via Cilium (replaces kube-proxy entirely)
──────────────────────────────────────────────────────────────
  No iptables. No IPVS. eBPF programs in kernel do everything.
  Best performance. Covered in depth in networking category.

═══════════════════════════════════════════════════════════════
```

---

### 🎤 Short Crisp Interview Answer

> *"Kube-proxy runs on every node and implements Service networking by programming the kernel's packet filtering rules. When you create a Service, kube-proxy writes iptables or IPVS rules that DNAT traffic destined for the Service's ClusterIP to one of the backing pod IPs. In iptables mode, this is O(n) — performance degrades with many services. IPVS mode uses kernel hash tables for O(1) lookup, which is preferred for large clusters. Modern setups like Cilium replace kube-proxy entirely with eBPF for even better performance and observability."*

---

### ⚠️ Gotchas

1. **kube-proxy does not handle pod-to-pod traffic** — it only handles Service-based traffic. Direct pod IP communication is handled by the CNI plugin.
2. **iptables is not truly random load balancing** — it uses probability rules that approximate equal distribution, but connections are not perfectly balanced. IPVS supports true least-connections.
3. **Cilium replaces kube-proxy** — when using Cilium in full eBPF mode, kube-proxy is not deployed at all. This is increasingly common in production.

---

### Common Interview Questions

**Q: What is the difference between ClusterIP, NodePort, and LoadBalancer service types?**
> ClusterIP: virtual IP accessible only within the cluster (kube-proxy programs it). NodePort: exposes a static port on every node's IP, external traffic can reach the service. LoadBalancer: provisions a cloud load balancer (via cloud-controller-manager) in front of NodePort, gives you an external IP.

**Q: When would you choose IPVS over iptables?**
> For clusters with more than 1,000 services, or where you need real load balancing algorithms (least connections, source hash for session affinity). iptables performance degrades linearly; IPVS stays constant.

---

### Connections to Other Topics

- Connects to **Services and DNS (3.2, 3.3)**
- **Cilium and eBPF (3.11)** replaces kube-proxy entirely
- **EndpointSlices (3.9)** are what kube-proxy watches to know which pod IPs back each service

---

---

# 1.9 Container Runtime (containerd, CRI-O)

## 🟢 Beginner

---

### What it is in simple terms

The container runtime is the software that **actually runs containers** on a node. Kubernetes tells the runtime what to run via the CRI (Container Runtime Interface), and the runtime handles the low-level Linux work of creating isolated processes.

---

### The Full Container Runtime Stack

```
═══════════════════════════════════════════════════════════════
CONTAINER RUNTIME STACK
═══════════════════════════════════════════════════════════════

Kubernetes (kubelet)
      │
      │  CRI — Container Runtime Interface (gRPC)
      │  Defines: ImageService + RuntimeService APIs
      │  kubelet calls: RunPodSandbox, CreateContainer,
      │                 StartContainer, PullImage, etc.
      ▼
┌─────────────────────────────────────────────────────────────┐
│           High-Level Runtime (manages images + lifecycle)   │
│                                                             │
│  containerd   ← Industry default                           │
│               ← Used by EKS, GKE, AKS, most distros        │
│               ← Donated to CNCF by Docker, Inc.            │
│                                                             │
│  CRI-O        ← Red Hat / OpenShift                        │
│               ← Kubernetes-only (no Docker CLI support)    │
│               ← Leaner, purpose-built for K8s              │
└─────────────────────────────────────────────────────────────┘
      │
      │  OCI — Open Container Initiative spec
      │  (image format + runtime spec — standardized)
      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Low-Level Runtime                        │
│                                                             │
│  runc     ← Standard. Uses Linux namespaces + cgroups.     │
│             Most containers use this.                       │
│                                                             │
│  kata     ← VM-based containers. Each pod gets its own     │
│             lightweight VM. Strong isolation. Used where   │
│             multi-tenant security is critical.             │
│                                                             │
│  gVisor   ← Google's user-space kernel. Intercepts        │
│             syscalls in user space. Used in GKE Sandbox.   │
└─────────────────────────────────────────────────────────────┘
      │
      ▼
Linux Kernel
(namespaces: pid, net, mnt, uts, ipc, user)
(cgroups: CPU, memory, I/O limits)
(seccomp: syscall filtering)

═══════════════════════════════════════════════════════════════
WHAT HAPPENED TO DOCKER IN KUBERNETES?
═══════════════════════════════════════════════════════════════

Docker = docker CLI + dockerd daemon + containerd + runc

K8s 1.20: Docker deprecated as a runtime (dockershim deprecated)
K8s 1.24: dockershim FULLY REMOVED from kubelet

"Docker is dead in K8s" means: the Docker DAEMON (dockerd) is gone.
Docker IMAGES still work perfectly — OCI image format is standard.
containerd runs the same images Docker builds.

Docker build → OCI image → pushed to registry → pulled by containerd
Everything still works. Only dockerd as the runtime is gone.
```

---

### 🎤 Short Crisp Interview Answer

> *"The container runtime is what actually executes containers on a node. Kubernetes communicates with it via the CRI — Container Runtime Interface — a gRPC API. The dominant runtime today is containerd, used by EKS, GKE, and AKS. containerd delegates actual process creation to runc, which uses Linux namespaces and cgroups. Docker was removed as a supported runtime in Kubernetes 1.24, but Docker-built images still work because they follow the OCI image specification that containerd understands natively."*

---

### ⚠️ Gotcha

**The pause container** — every Kubernetes pod starts with a hidden "pause" (or "sandbox") container. This container holds the network namespace for the entire pod. All app containers share its network namespace. If you kill the pause container, all containers in the pod restart. Most candidates have never heard of this.

---

### Common Interview Questions

**Q: What is the difference between containerd and Docker?**
> Docker is a developer toolchain (CLI, build tool, daemon). containerd is the low-level runtime component within Docker, extracted and donated to CNCF. Kubernetes uses containerd directly, bypassing the Docker daemon, because the CRI interface is all that's needed.

**Q: What are Linux namespaces and cgroups and how do they relate to containers?**
> Namespaces provide isolation — each container gets its own view of PIDs, network interfaces, filesystem mounts, hostnames. cgroups enforce resource limits — they restrict how much CPU, memory, and I/O a process can consume. Together, namespaces + cgroups = containers. There's no magic — containers are just Linux processes with these constraints applied.

---

### Connections to Other Topics

- CRI interface is called by the **Kubelet (1.7)**
- Runtime security (kata, gVisor) connects to **Pod Security (7.3)**
- **containerd** is the runtime behind EKS node groups **(12.1)**

---

---

# ⚠️ 1.10 How a Pod Creation Request Flows End-to-End

## 🟡 Intermediate — MOST COMMONLY ASKED DEEP-DIVE QUESTION

---

### Why this matters

This single question reveals whether a candidate understands how all components of Kubernetes work together. Senior interviewers ask this to see if you can trace a request through the entire system. Know every step cold.

---

### The Complete End-to-End Flow

```
═══════════════════════════════════════════════════════════════════════════
POD CREATION: COMPLETE END-TO-END FLOW
═══════════════════════════════════════════════════════════════════════════

You run: kubectl apply -f pod.yaml

────────────────────────────────────────────────────────────────────────
STEP 1: kubectl → API Server
────────────────────────────────────────────────────────────────────────
  • kubectl reads kubeconfig: gets API server URL + TLS certs
  • Serializes your YAML → JSON
  • Sends HTTP POST to:
    https://<api-server>:6443/api/v1/namespaces/default/pods
  • Content-Type: application/json

────────────────────────────────────────────────────────────────────────
STEP 2: API Server — Auth + Admission + Persist
────────────────────────────────────────────────────────────────────────
  • Authentication:  validates client cert / bearer token / OIDC
  • Authorization:   RBAC check — can this user create pods here?
  • Mutating Admission Webhooks run (may modify the request):
    - Inject Istio sidecar container
    - Add default resource limits
    - Add labels or annotations
  • Schema Validation: is the PodSpec structurally valid?
  • Validating Admission Webhooks run (may reject):
    - Image not from approved registry → REJECT
    - Missing required labels → REJECT
  • PERSIST to etcd:
    Key:    /registry/pods/default/mypod
    Value:  full pod JSON
    Note:   spec.nodeName = ""  ← unscheduled

  → Returns 201 Created to kubectl
  → kubectl prints: pod/mypod created

────────────────────────────────────────────────────────────────────────
STEP 3: Scheduler Watches and Binds
────────────────────────────────────────────────────────────────────────
  • Scheduler has open WATCH on API server:
    GET /api/v1/pods?watch=true&fieldSelector=spec.nodeName=""
  • Receives ADDED event for new pod
  • Runs filtering: eliminate nodes that can't fit pod
  • Runs scoring: rank remaining nodes
  • Selects winning node (e.g., worker-node-2)
  • Sends binding to API server:
    POST /api/v1/namespaces/default/pods/mypod/binding
    Body: {"target": {"name": "worker-node-2"}}
  • API Server writes spec.nodeName = "worker-node-2" to etcd

────────────────────────────────────────────────────────────────────────
STEP 4: Kubelet on worker-node-2 Starts the Pod
────────────────────────────────────────────────────────────────────────
  • kubelet on worker-node-2 has open WATCH:
    GET /api/v1/pods?watch=true&fieldSelector=spec.nodeName=worker-node-2
  • Receives MODIFIED event: pod now assigned to this node
  
  kubelet actions:
  1. Calls CNI plugin: set up pod networking
     - Allocate pod IP (e.g., 10.244.2.15) from node CIDR
     - Create veth pair: one end in pod, one in host network
     - Program routing tables
  
  2. Calls containerd via CRI:
     a. RunPodSandbox: create pause container (holds network NS)
     b. PullImage: pull app image if not in local cache
     c. CreateContainer: create container spec
     d. StartContainer: start the container process
  
  3. Runs init containers first (sequentially, wait for each to complete)
  4. Starts app containers
  5. Starts startup probe (if configured)
  6. Starts liveness and readiness probes
  
  7. Reports status back to API server:
     phase: Running
     podIP: 10.244.2.15
     containerStatuses: [{ready: false}]  ← not ready yet

────────────────────────────────────────────────────────────────────────
STEP 5: Readiness Probe Passes — Pod Joins Service
────────────────────────────────────────────────────────────────────────
  • kubelet's readiness probe succeeds
  • kubelet updates pod status: containerStatuses: [{ready: true}]
  • API server persists to etcd
  
  • EndpointSlice controller watches pod status changes
  • Sees pod is now Ready
  • Adds pod IP (10.244.2.15) to Service's EndpointSlice
  
  • kube-proxy on ALL nodes watches EndpointSlices
  • Updates iptables/IPVS rules on every node
  • Now traffic to Service ClusterIP routes to this pod

────────────────────────────────────────────────────────────────────────
TIMELINE (typical fast path with cached image):
────────────────────────────────────────────────────────────────────────
  0ms       kubectl sends request
  50ms      API server persists to etcd
  100ms     Scheduler picks node
  150ms     kubelet sees assignment
  200ms     CNI sets up networking
  300ms     containerd starts container (image cached)
  300ms+    Readiness probe polling begins
  varies    Readiness probe passes (depends on app startup time)
  +50ms     EndpointSlice updated
  +50ms     kube-proxy updates all nodes
  TOTAL:    ~1-3 seconds (cached image), minutes (cold pull)

═══════════════════════════════════════════════════════════════════════════
```

---

### 🎤 Short Crisp Interview Answer

> *"When you apply a pod manifest, kubectl sends an HTTP POST to the API server. The API server authenticates the request, checks RBAC, runs mutating then validating admission webhooks, validates the schema, and persists the pod to etcd with no node assigned. The scheduler is watching for exactly this — unscheduled pods — filters nodes by feasibility, scores them, picks the best one, and writes the node name back to the pod. The kubelet on that node is watching for pods assigned to it, calls CNI to set up networking, then calls containerd to pull the image and start containers. Once readiness probes pass, the EndpointSlice controller adds the pod to the service, and kube-proxy updates its rules on all nodes."*

---

### ⚠️ Gotchas

1. **Image pull can dominate the timeline** — a 2GB image on a cold node takes minutes. This is why pre-pulling images and using small images matters in production.
2. **Readiness ≠ Running** — a pod can be `Running` phase but not `Ready`. Traffic only goes to Ready pods. Many candidates confuse these.
3. **The scheduler does not start the container** — it only writes a node name. The kubelet does the actual work.
4. **kube-proxy updates happen on ALL nodes** — when a new pod joins a service, every node in the cluster updates its iptables rules. At scale (1000+ nodes), this propagation has latency.

---

### Common Interview Questions

**Q: Walk me through what happens when you run kubectl apply.**
> Use the full flow above — API server pipeline, scheduler two phases, kubelet CRI calls, CNI setup, probe → endpoint lifecycle.

**Q: Why might a pod get stuck in Pending state?**
> Scheduler can't find a feasible node: insufficient resources, no matching node labels, taints without tolerations, PVC can't be bound, pod anti-affinity with all existing pods. Debug with `kubectl describe pod`.

**Q: What is the role of the CNI plugin in pod creation?**
> CNI (Container Network Interface) is called by kubelet when a pod sandbox is created. It allocates an IP from the node's CIDR, creates a virtual network interface inside the pod's namespace, connects it to the node network, and programs routing so the pod can reach other pods and services.

---

### Connections to Other Topics

- Admission webhooks in the flow connect to **Admission Controllers (7.4, 7.5)**
- CNI setup connects to **Kubernetes Networking Model (3.1)**
- Readiness/liveness probes connect to **Probes (8.2)**
- EndpointSlice update connects to **Services and kube-proxy (3.2, 1.8)**

---

---

# 1.11 Leader Election in Control Plane Components

## 🟡 Intermediate

---

### What it is in simple terms

In an HA control plane you run multiple replicas of the scheduler and controller manager. But you **don't want two schedulers scheduling the same pod simultaneously**. Leader election ensures only ONE instance is active at any time, while others wait as hot standbys.

---

### How it works — Lease Objects

```
═══════════════════════════════════════════════════════════════
LEADER ELECTION VIA LEASE OBJECTS
═══════════════════════════════════════════════════════════════

3 scheduler replicas running on control plane nodes:
  [scheduler-pod-1]  [scheduler-pod-2]  [scheduler-pod-3]

They compete by trying to ATOMICALLY write a Lease object:
  API path: /apis/coordination.k8s.io/v1/namespaces/kube-system/leases/kube-scheduler

Lease object contents:
┌────────────────────────────────────────────────────────────┐
│ apiVersion: coordination.k8s.io/v1                         │
│ kind: Lease                                                │
│ metadata:                                                  │
│   name: kube-scheduler                                     │
│   namespace: kube-system                                   │
│ spec:                                                      │
│   holderIdentity: "scheduler-pod-1-abc123"  ← current leader│
│   leaseDurationSeconds: 15                  ← TTL          │
│   acquireTime:  "2024-01-01T10:00:00Z"                     │
│   renewTime:    "2024-01-01T10:00:10Z"      ← updated every│
│   leaseTransitions: 3                        ← 10 seconds  │
└────────────────────────────────────────────────────────────┘

LEADER behavior:
  • Actively does work (schedules pods)
  • Renews its Lease every ~10 seconds (leaseDuration/2)

FOLLOWER behavior:
  • Watches the Lease, stays completely idle
  • Does NOT schedule pods

FAILOVER when leader dies:
  1. Leader stops renewing Lease (because it crashed)
  2. After 15 seconds (leaseDurationSeconds), Lease is considered expired
  3. All followers attempt to acquire Lease simultaneously
  4. They use etcd's optimistic concurrency (resourceVersion check)
     → Only ONE write succeeds
     → Others get 409 Conflict → retry
  5. Winner becomes new leader, starts working immediately

FAILOVER TIME: approximately 15-20 seconds

CHECK CURRENT LEADER:
  kubectl get lease kube-scheduler -n kube-system -o yaml
  kubectl get lease kube-controller-manager -n kube-system -o yaml

═══════════════════════════════════════════════════════════════
```

---

### 🎤 Short Crisp Interview Answer

> *"In an HA control plane, you run multiple replicas of the scheduler and controller manager, but only one should be active at a time. They use a leader election mechanism backed by Lease objects in the kube-system namespace. The current leader continuously renews its Lease every ~10 seconds. Followers watch the Lease and stay completely idle. If the leader dies and stops renewing, the Lease expires after 15 seconds and followers race to acquire it using etcd's optimistic concurrency — only one wins and becomes the new leader. Failover typically takes 15–20 seconds."*

---

### ⚠️ Gotcha

**API servers do NOT use leader election** — all API server replicas are active simultaneously. Only the scheduler and controller manager use leader election. This is a very common interview confusion point. API servers are stateless and can all run in parallel safely.

---

### Common Interview Questions

**Q: Why do API servers not need leader election but the scheduler does?**
> API servers are pure request handlers — they're stateless, consistency is guaranteed by etcd's optimistic locking, and multiple can safely handle requests in parallel. The scheduler and controller manager perform *actions* (creating pods, managing resources) that would cause conflicts if two instances acted simultaneously on the same objects.

**Q: How long does it take to recover if a scheduler crashes in HA setup?**
> Approximately 15-20 seconds — the Lease TTL (default 15s) must expire before followers compete, plus the time to elect a new leader and resume scheduling.

---

### Connections to Other Topics

- Lease objects are stored in **etcd (1.4)** via the **API server (1.3)**
- The optimistic concurrency underpinning election connects to **API Server HA (1.15)**
- This is why managed control planes like **EKS (12.1)** handle HA for you

---

---

# 1.12 API Server Admission Flow

## 🟡 Intermediate

---

### What it is in simple terms

Every object creation or modification request passes through the admission control pipeline. This is where Kubernetes enforces policy — injecting sidecars, validating images, enforcing security standards, applying defaults.

---

### The Full Admission Pipeline

```
═══════════════════════════════════════════════════════════════
API SERVER ADMISSION CONTROL — FULL PIPELINE
═══════════════════════════════════════════════════════════════

REQUEST ARRIVES AT API SERVER
      │
      ▼
 ① AUTHENTICATION
  ─────────────────
  Q: Who is this request from?
  Methods:
  • x509 client certificate    (kubectl, kubelet)
  • Bearer token               (ServiceAccount JWT)
  • OIDC token                 (AWS IAM, Google, Okta)
  • Bootstrap token            (node joining cluster)
  • Webhook authenticator      (custom external IdP)
  
  Output: UserInfo { username, groups[], uid, extra }
  On failure: 401 Unauthorized
      │
      ▼
 ② AUTHORIZATION
  ──────────────────
  Q: Is this identity allowed to do this?
  Modes (checked in order, first ALLOW wins):
  • RBAC         — most common, role-based
  • Node         — kubelets can only access their own node's data
  • ABAC         — legacy, file-based attribute rules
  • Webhook      — delegate to external authorization service
  
  On failure: 403 Forbidden
      │
      ▼
 ③ MUTATING ADMISSION WEBHOOKS
  ──────────────────────────────
  Q: Should this object be MODIFIED before acceptance?
  
  • All registered mutating webhooks run IN PARALLEL
  • Each webhook receives the object and returns patches
  • Patches are JSON Patch (RFC 6902) format
  • Final object is merge of all patches
  
  Common uses:
  • Istio: inject envoy sidecar container
  • Pod defaults: set resource limits from LimitRange
  • Labels/annotations: add team labels, cost center tags
  • Image tag normalization: :latest → :sha256:abc123
  
  If webhook unavailable: depends on failurePolicy (Fail or Ignore)
      │
      ▼
 ④ OBJECT SCHEMA VALIDATION
  ──────────────────────────
  • Validate the (now possibly mutated) object against OpenAPI schema
  • Check required fields, types, format constraints
  • Unknown fields handling (depends on server-side apply)
      │
      ▼
 ⑤ VALIDATING ADMISSION WEBHOOKS
  ─────────────────────────────────
  Q: Should this object be REJECTED?
  
  • All registered validating webhooks run IN PARALLEL
  • Each can only ALLOW or DENY — cannot modify
  • If ANY webhook returns deny → ENTIRE REQUEST FAILS
  
  Common uses:
  • OPA/Gatekeeper: enforce custom policies
  • Kyverno: policy engine
  • Image policy: reject non-approved registries
  • Required labels: reject deployments without team label
  • Resource limits: reject pods without CPU/memory limits
      │
      ▼
 ⑥ PERSIST TO etcd
  ──────────────────
  • Object is written to etcd
  • resourceVersion assigned (monotonically increasing)
  • creationTimestamp set
      │
      ▼
 ⑦ RETURN RESPONSE
  ─────────────────
  • 201 Created (new object)
  • 200 OK (update)
  • 4xx (client error — rejected by admission, schema invalid)
  • 5xx (server error — webhook unavailable with failurePolicy=Fail)

KEY BUILT-IN ADMISSION CONTROLLERS:
─────────────────────────────────────────────────────────────
NamespaceLifecycle        Reject resources in terminating NS
LimitRanger               Apply default limits from LimitRange
ResourceQuota             Enforce namespace resource quotas
ServiceAccount            Auto-assign default SA to pods
DefaultStorageClass       Set default storage class on PVCs
PodSecurity               Enforce Pod Security Standards (PSS)
MutatingAdmissionWebhook  Call external mutating webhooks
ValidatingAdmissionWebhook Call external validating webhooks
```

---

### 🎤 Short Crisp Interview Answer

> *"Every API server request passes through authentication, then authorization, then admission control. Admission has two webhook phases — mutating webhooks run first and can modify the incoming object, for example to inject sidecars or set default resource limits. Then validating webhooks run and can only approve or reject — they cannot modify. If any validating webhook rejects, the whole request fails. After admission, the object is validated against the OpenAPI schema and persisted to etcd."*

---

### ⚠️ Gotchas

1. **Mutating runs BEFORE validating** — this is deliberate. You want to normalize the object (set defaults, inject containers) before validating it against policy.
2. **failurePolicy=Fail vs Ignore** — if a webhook is unreachable and failurePolicy=Fail, ALL requests of that type are rejected. This has caused production outages where a broken admission webhook took down the entire cluster. Always set up webhook HA.
3. **Webhooks add latency** — every webhook call adds ~5-50ms to request processing. Many webhooks = slow cluster. Monitor webhook latency in production.

---

### Common Interview Questions

**Q: What is the difference between a MutatingWebhook and a ValidatingWebhook?**
> Mutating webhooks can modify the object (inject containers, add labels, set defaults). Validating webhooks can only approve or reject. Mutating always runs before validating. The separation ensures policy validation happens on the final object after all mutations.

**Q: What happens if an admission webhook is down?**
> Depends on `failurePolicy`. If `Fail` (default for most security webhooks), all requests of matching types are rejected. If `Ignore`, requests proceed without the webhook check. This is why webhook HA and proper failure policies are critical in production.

---

### Connections to Other Topics

- Mutating webhooks power **Istio sidecar injection (11.4)**
- Validating webhooks power **OPA/Gatekeeper (7.10)** and **Kyverno (7.11)**
- Admission controllers include **Pod Security Admission (7.3)**
- Webhooks are defined as Kubernetes objects — they use the **API Server (1.3)** themselves

---

---

# ⚠️ 1.13 Watch & Informer Pattern

## 🟡 Intermediate

---

### What it is in simple terms

Every Kubernetes controller needs to know when things change. The watch & informer pattern is the **efficient mechanism** by which controllers get notified of changes without constantly polling the API server — and it is the foundation of how the entire Kubernetes control plane stays synchronized.

---

### The Problem — Why Not Just Poll?

```
NAIVE APPROACH (DO NOT DO THIS):

while True:
    pods = GET /api/v1/pods        ← full list every second
    reconcile(pods)
    sleep(1)

Problems:
✗ 50 controllers × 10,000 pods × every 1s = API server meltdown
✗ Full list is expensive (serialization, network, etcd scan)
✗ You can miss events between polls
✗ Latency — up to 1s to react to changes
```

---

### The Watch Mechanism

```
═══════════════════════════════════════════════════════════════
KUBERNETES WATCH — LONG-POLL STREAMING
═══════════════════════════════════════════════════════════════

Step 1: Initial LIST (get current state + resourceVersion)
  GET /api/v1/pods
  Response: {items: [...all pods...], metadata: {resourceVersion: "5000"}}

Step 2: WATCH from that resourceVersion (never ends)
  GET /api/v1/pods?watch=true&resourceVersion=5000

  Server holds HTTP connection OPEN and streams events:
  {"type":"ADDED",    "object": {...new pod...}}
  {"type":"MODIFIED", "object": {...updated pod...}}
  {"type":"DELETED",  "object": {...deleted pod...}}
  {"type":"BOOKMARK", "object": {...}, resourceVersion: "5100"}

  Connection streams indefinitely.
  If connection drops: reconnect with last known resourceVersion
  If resourceVersion too old (compacted): do a fresh LIST + WATCH

KEY PROPERTY:
  resourceVersion is monotonically increasing.
  By providing it on reconnect, client receives ONLY changes
  since last seen event — nothing is missed, nothing is duplicated.
```

---

### The Informer Pattern

```
═══════════════════════════════════════════════════════════════
INFORMER ARCHITECTURE (used by ALL controllers)
═══════════════════════════════════════════════════════════════

                    API SERVER
                        │
                        │  LIST + WATCH (one connection per resource type)
                        │
┌───────────────────────▼─────────────────────────────────────┐
│                     INFORMER                                │
│                                                             │
│  ┌─────────────────┐                                        │
│  │   ListWatcher   │  ← does initial LIST, then WATCH       │
│  │                 │    auto-reconnects on failure           │
│  │                 │    handles resourceVersion bookmarks    │
│  └────────┬────────┘                                        │
│           │ events (ADDED, MODIFIED, DELETED)               │
│           ▼                                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Local In-Memory Cache (Indexer / Store)            │    │
│  │                                                     │    │
│  │  • Thread-safe cache of all objects                 │    │
│  │  • Controllers READ FROM HERE not API server        │    │
│  │  • Indexed for fast lookup (by namespace, label...) │    │
│  │  • Stays in sync via the watch stream               │    │
│  └──────────────────────────┬──────────────────────────┘    │
│                             │ on change                     │
│           ┌─────────────────┴────────────────┐              │
│           │  Event Handlers (optional)       │              │
│           │  OnAdd / OnUpdate / OnDelete      │              │
│           │  → push object KEY to work queue │              │
│           └──────────────────────────────────┘              │
└───────────────────────────────────────────────────────────  │
                             │
                             ▼
           ┌─────────────────────────────────┐
           │         Work Queue              │
           │                                 │
           │  • FIFO with DEDUPLICATION      │
           │  • Object changed 5 times while │
           │    reconciler is busy → only    │
           │    ONE entry in the queue       │
           │  • Rate-limited retry on errors │
           └─────────────────┬───────────────┘
                             │
                             ▼
           ┌─────────────────────────────────┐
           │      Controller / Reconciler    │
           │                                 │
           │  func Reconcile(key string) {   │
           │    // get CURRENT state from    │
           │    // CACHE (not API server)    │
           │    obj := cache.Get(key)        │
           │    // take action               │
           │    // update real world         │
           │  }                              │
           └─────────────────────────────────┘

WHY THE LOCAL CACHE IS CRITICAL:
  Without it:
  10 controllers × GET /api/v1/pods = 10 API server calls per reconcile
  
  With informer cache:
  10 controllers share ONE watch connection to API server
  All reads go to local cache = 0 additional API server load
  
  The cache is effectively a local read replica of etcd
  for all controllers on that machine.
```

---

### 🎤 Short Crisp Interview Answer

> *"Controllers don't poll the API server in a loop. Instead they use the informer pattern. An informer does an initial LIST of all resources, then opens a long-lived WATCH stream to receive incremental change events — Added, Modified, Deleted. It maintains a local in-memory cache that controllers read from instead of hitting the API server directly. Changes are pushed into a deduplicated work queue — if an object changes five times while the reconciler is busy, there's still only one entry. The reconciler processes items from the queue and always reads current state from the cache, not replaying every intermediate change."*

---

### ⚠️ Gotchas

1. **resourceVersion gap / Too Old** — if the watch is disconnected for too long and the resourceVersion has been compacted out of etcd, the informer falls back to a full relist. This is expensive and can spike API server load. This is why etcd compaction intervals matter.
2. **Work queue deduplication** — the queue stores the object's **key** (namespace/name), not the object itself. The reconciler always fetches current state from the cache when processing. This means intermediate states are silently skipped — the reconciler sees only the latest state.
3. **Shared informers** — in production controllers built with controller-runtime, multiple controllers sharing the same resource type share a single informer and cache. This is critical for efficiency. Don't create per-controller informers for the same resource.
4. **Stale cache** — there is a brief window after a write where the cache may not reflect the latest state. Production controllers must handle this and use optimistic concurrency (resourceVersion) when updating objects.

---

### Common Interview Questions

**Q: Why does Kubernetes use a watch-based model instead of polling?**
> Watch is O(events) — you only process actual changes. Polling is O(objects × frequency) regardless of whether anything changed. Watch gives lower latency, lower API server load, and no missed events.

**Q: What happens when a watch connection to the API server drops?**
> The informer records the last resourceVersion it saw. On reconnect it resumes the watch from that version, receiving only missed events. If the version is too old (compacted from etcd), it falls back to a full LIST + new WATCH, ensuring no state is permanently missed.

**Q: What is the purpose of the work queue in the informer pattern?**
> Decoupling and deduplication. The event handler just enqueues a key. The reconciler processes keys at its own pace. If the same object changes multiple times, the queue deduplicates to one entry. Failed reconciliations get rate-limited retry. This prevents thundering herds and handles errors gracefully.

---

### Connections to Other Topics

- Informers are used by every controller in **Controller Manager (1.6)**
- The underlying watch mechanism is served by the **API Server (1.3)**
- Informers are the foundation for building **Custom Controllers and Operators (10.3, 10.4)**
- The work queue retry mechanism connects to error handling in **operator development (10.5)**

---

---

# 1.14 etcd Internals — Raft Consensus, Compaction, Defrag

## 🔴 Advanced

---

### Raft Consensus Algorithm

```
═══════════════════════════════════════════════════════════════
RAFT CONSENSUS IN etcd — HOW WRITES ARE COMMITTED
═══════════════════════════════════════════════════════════════

ROLES:
  Leader   — 1 node. Handles ALL writes. Coordinates replication.
  Follower — All other nodes. Accept replicated logs from leader.
  Candidate — Temporary role during election.

WRITE PATH (strictly serialized via the leader):
─────────────────────────────────────────────────

  Client → API Server → etcd Leader

  Step 1: Client sends write to Leader
  Step 2: Leader appends entry to its local log (uncommitted)
  Step 3: Leader sends AppendEntries RPC to ALL followers IN PARALLEL
  Step 4: Leader waits for QUORUM acknowledgment (majority respond)
  Step 5: Leader marks entry COMMITTED in its log
  Step 6: Leader applies entry to state machine (updates value)
  Step 7: Leader responds SUCCESS to client
  Step 8: Leader sends commit notification to followers on next heartbeat

  ┌──────────┐  1.Write  ┌──────────┐
  │  Client  │ ─────────►│  LEADER  │
  └──────────┘           └────┬─────┘
                              │ 2. AppendEntries (parallel RPC)
              ┌───────────────┼──────────────┐
              ▼               ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │Follow-1  │   │Follow-2  │   │Follow-3  │
        └──────────┘   └──────────┘   └──────────┘
              │               │
              └───────┬───────┘
                      │ 3. Quorum ACK (2 of 3 responded)
                      ▼
                Leader commits + responds to client

TERMS AND ELECTIONS:
─────────────────────
  Time in Raft is divided into terms (monotonically increasing integers).
  Each term begins with an election.
  
  Election trigger:
  • Follower hasn't heard from leader in election timeout (150-300ms)
  • Follower increments its term, becomes Candidate
  • Sends RequestVote RPC to all other nodes
  • Receives majority votes → becomes new Leader
  
  SPLIT BRAIN PREVENTION:
  Network partition: [Node1] ─ ✂ ─ [Node2, Node3]
  
  Partition of 1 (Node1): cannot get majority votes → stays follower
  Partition of 2 (Node2+3): can get majority votes → elects leader
  
  Only ONE partition can make progress.
  Old leader (Node1 if it was leader) cannot commit writes:
  → cannot replicate to majority → cannot confirm writes to clients
  → clients get errors / timeouts → no data loss, no corruption

CONSISTENCY GUARANTEE: LINEARIZABILITY
  Once a write is acknowledged to the client, ALL subsequent
  reads (from any node) will see that write.
  etcd guarantees this by routing reads through the leader or
  using the ReadIndex mechanism to verify leadership before serving.
```

---

### MVCC, Compaction, and Defrag

```
═══════════════════════════════════════════════════════════════
etcd MVCC AND STORAGE MANAGEMENT
═══════════════════════════════════════════════════════════════

MVCC — MULTI-VERSION CONCURRENCY CONTROL:
──────────────────────────────────────────
  Every write creates a NEW revision. Old revisions are retained.
  This is how the WATCH mechanism works — clients subscribe from
  a specific revision and receive only changes after that point.

  Example history for /registry/pods/default/mypod:
  ┌────────────┬──────────────────────────────────────────────┐
  │ Revision   │ Value                                        │
  ├────────────┼──────────────────────────────────────────────┤
  │ Rev 100    │ {image: nginx:1.0, replicas: 1}              │
  │ Rev 105    │ {image: nginx:1.1, replicas: 1}  ← update   │
  │ Rev 112    │ {image: nginx:1.1, replicas: 3}  ← scaled   │
  │ Rev 120    │ <tombstone>                       ← deleted  │
  └────────────┴──────────────────────────────────────────────┘

  Without compaction: unlimited revision history = unbounded disk usage

COMPACTION: Remove old revisions (logical operation)
──────────────────────────────────────────────────────
  etcdctl compact 120
  → Revisions 100, 105, 112 are marked deleted
  → Data not physically freed yet (still on disk)
  → Clients can no longer watch from before rev 120
  
  Auto-compaction by kube-apiserver:
  --etcd-compaction-interval=5m   ← every 5 minutes (default)
  
  After compaction, watch clients with old resourceVersion
  receive a "410 Gone" and must do a fresh LIST.

DEFRAGMENTATION: Reclaim physical disk space
─────────────────────────────────────────────
  etcdctl defrag
  → Physically rewrites the bbolt database file
  → Reclaims space freed by compaction
  → Database file shrinks on disk
  
  ⚠️  DEFRAG IS BLOCKING on the node being defragged:
  → Takes a write lock
  → Temporarily stops serving writes (milliseconds to seconds)
  → Do followers FIRST, leader LAST
  → Or: etcdctl defrag --cluster  (handles order automatically)

MONITORING etcd SIZE:
─────────────────────
  Key metrics:
  etcd_mvcc_db_total_size_in_bytes        ← current db size
  etcd_mvcc_db_total_size_in_use_in_bytes ← actual used size
  etcd_server_quota_backend_bytes         ← quota limit
  
  Default quota: 2GB
  Recommended max: 8GB (--quota-backend-bytes=8589934592)
  
  When quota is hit:
  → etcd returns "etcdserver: mvcc: database space exceeded"
  → API server cannot write → cluster accepts NO new changes
  → This is a P0 production incident
  
  Alert threshold: > 70% of quota
  Action: compact + defrag immediately

PRODUCTION BEST PRACTICES:
────────────────────────────
  • Run etcd on dedicated SSDs (p99 fsync < 10ms required)
  • Never share etcd disk with application workloads
  • Backup daily minimum: etcdctl snapshot save
  • Monitor db size, fsync latency, leader elections
  • Compact every 5 minutes (kube-apiserver default)
  • Defrag weekly or when in_use << total_size by >30%
  • Use 3 nodes for most clusters, 5 for large/critical clusters
```

---

### 🎤 Short Crisp Interview Answer

> *"etcd uses Raft consensus — all writes go to the leader, which replicates to followers and commits only after a quorum acknowledges. This prevents split-brain — a minority partition cannot get enough votes to commit. etcd uses MVCC internally, keeping all historical revisions, which means it grows unboundedly without compaction. Compaction removes old revisions logically; defrag physically reclaims disk space. Defrag should be done on followers before the leader since it briefly blocks writes. If etcd hits its size quota — default 2GB, max 8GB — it goes read-only, which is a critical cluster outage requiring immediate compaction and defrag."*

---

### ⚠️ Gotchas

1. **Default quota is only 2GB** — large clusters with many resources or high-frequency CRD updates hit this quickly. Always set `--quota-backend-bytes` to 4GB or 8GB.
2. **Defrag without compaction does nothing** — you must compact first to mark old data for deletion, then defrag to physically reclaim it.
3. **Leader's disk is bottleneck** — the leader handles all writes and fsyncs. A slow leader disk slows the entire cluster. Monitor `wal_fsync_duration_seconds` metric.
4. **Never use even-numbered etcd clusters** — 4-node cluster has same fault tolerance as 3-node but costs more. Always use 3 or 5.

---

---

# 1.15 API Server HA — How Multiple API Servers Coordinate

## 🔴 Advanced

---

### Architecture

```
═══════════════════════════════════════════════════════════════
API SERVER HIGH AVAILABILITY
═══════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │              LOAD BALANCER (L4)                          │
  │   (AWS NLB / HAProxy / kube-vip / cloud LB)              │
  │   Single endpoint: https://k8s.internal:6443             │
  └────────────┬──────────────────────────┬──────────────────┘
               │                          │
    ┌──────────┴──────────┐    ┌──────────┴──────────┐
    │    API Server 1     │    │    API Server 2     │
    │    (ACTIVE)         │    │    (ACTIVE)         │
    │    stateless        │    │    stateless        │
    └──────────┬──────────┘    └──────────┬──────────┘
               │                          │
               └─────────────┬────────────┘
                             │  (BOTH read and write)
                    ┌────────┴────────┐
                    │  etcd cluster   │
                    │  (3 or 5 nodes) │
                    └─────────────────┘

KEY DIFFERENCES FROM SCHEDULER / CONTROLLER MANAGER:
──────────────────────────────────────────────────────
  Scheduler / CM:    1 ACTIVE  +  N standby (leader election)
  API Servers:       ALL ACTIVE simultaneously (no election)

WHY API SERVERS DON'T NEED LEADER ELECTION:
  • Pure request handlers — stateless
  • No "actions" that could conflict if done twice
  • Consistency guaranteed by etcd, not by which server handles it
  • Two API servers writing same object simultaneously →
    etcd optimistic concurrency (resourceVersion check) →
    one write wins, other gets 409 Conflict → client retries

OPTIMISTIC CONCURRENCY:
  Every object has a resourceVersion field (an etcd revision number).
  When updating, client must include current resourceVersion.
  etcd checks: is this still the current version?
    YES → write succeeds, new resourceVersion assigned
    NO  → 409 Conflict → client must re-GET then re-PUT

  This prevents:
  → Two API servers updating same object → last write wins problem
  → Controllers reading stale cache → safe concurrent updates

CLIENT WATCH RECONNECTION:
  When a watch client's API server dies:
  1. HTTP connection drops
  2. Client has last seen resourceVersion
  3. Client reconnects to LB → lands on different API server
  4. Resumes watch from last resourceVersion
  5. Receives all missed events → no events lost

KUBECONFIG FOR HA:
  # Multiple API server endpoints in kubeconfig
  clusters:
  - cluster:
      server: https://api-lb.internal:6443  ← single LB address
      certificate-authority-data: ...
    name: production
  
  OR with client-side load balancing:
  - cluster:
      server: https://api1.internal:6443
```

---

### 🎤 Short Crisp Interview Answer

> *"Unlike the scheduler and controller manager, API servers don't need leader election because they're completely stateless — all state is in etcd. In an HA setup you run 2 or 3 API servers behind a load balancer, all active simultaneously. Consistency is guaranteed by etcd's optimistic concurrency control via resourceVersion — if two API servers try to update the same object simultaneously, etcd rejects one with a conflict and the client retries. The key distinction is: API servers are all active at once, but scheduler and controller manager use leader election so only one is active at a time."*

---

---

# 1.16 Custom API Servers & API Aggregation Layer

## 🔴 Advanced

---

### What it is and why it exists

```
═══════════════════════════════════════════════════════════════
API AGGREGATION LAYER
═══════════════════════════════════════════════════════════════

PROBLEM CRDs cannot solve:
  • Need storage backend OTHER than etcd
    (e.g., time-series DB for metrics, relational DB)
  • Need custom business logic on EVERY read/write
  • Need very high read throughput CRDs can't provide
  • Need real-time computed data (metrics, pod logs)

SOLUTION: API Aggregation
  Register a custom API server that handles specific API groups.
  Main API server proxies matching requests to your server.

ARCHITECTURE:
  ┌─────────────────────────────────────────────────────────┐
  │              Main kube-apiserver                        │
  │                                                         │
  │  /api/v1/*              → handles internally            │
  │  /apis/apps/v1/*        → handles internally            │
  │                                                         │
  │  /apis/metrics.k8s.io/v1beta1/*  ──── proxy ──────────►│──┐
  │  /apis/mycompany.io/v1/*         ──── proxy ──────────►│  │
  └─────────────────────────────────────────────────────────┘  │
                                                               │
                         ┌─────────────────────────────────────┘
                         ▼
         ┌─────────────────────────┐
         │  metrics-server         │
         │  (in-memory storage,    │
         │   not etcd)             │
         │  Handles:               │
         │  kubectl top pods       │
         │  kubectl top nodes      │
         │  HPA metric queries     │
         └─────────────────────────┘

REGISTERING A CUSTOM API SERVER:
  # APIService object tells main API server to proxy this group
  apiVersion: apiregistration.k8s.io/v1
  kind: APIService
  metadata:
    name: v1beta1.metrics.k8s.io
  spec:
    service:
      name: metrics-server
      namespace: kube-system
      port: 443
    group: metrics.k8s.io
    version: v1beta1
    groupPriorityMinimum: 100
    versionPriority: 100
    insecureSkipTLSVerify: false   # always false in production
    caBundle: <base64-ca-cert>

CRD vs API AGGREGATION — WHEN TO USE WHICH:
──────────────────────────────────────────────
  CRDs:
  ✓ Simple to implement — no code needed for basic CRD
  ✓ Uses etcd — consistency guaranteed
  ✓ Works with all standard K8s tooling (kubectl, RBAC, watch)
  ✓ Good for: operators, config objects, custom workloads
  ✗ Limited to etcd storage model
  ✗ etcd size limits apply
  ✗ No custom storage or computation on reads

  API Aggregation:
  ✓ Custom storage backend
  ✓ Custom business logic on every request
  ✓ High-throughput reads (in-memory, specialized DB)
  ✓ Good for: metrics, logs, service catalog, webhook-heavy APIs
  ✗ Requires writing and operating a full API server
  ✗ Complex to implement and maintain
  ✗ Must implement auth delegation, RBAC integration yourself
```

---

### 🎤 Short Crisp Interview Answer

> *"The API aggregation layer lets you extend the Kubernetes API with custom API servers that handle specific API groups. The main API server proxies requests for registered groups to your server. The most common example is metrics-server, which serves the metrics.k8s.io API group from in-memory storage rather than etcd — which is why kubectl top works. You register a custom API server by creating an APIService object. The difference from CRDs is that aggregated API servers can use any storage backend, implement custom admission logic, and serve high-throughput or computed data that would be impractical to store in etcd."*

---

---

# 🏁 Category 1 — Complete Architecture Connection Map

```
═══════════════════════════════════════════════════════════════════════════
HOW ALL CATEGORY 1 COMPONENTS CONNECT
═══════════════════════════════════════════════════════════════════════════

  USER / AUTOMATION
         │
         │  kubectl / CI-CD / controllers
         ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                       API SERVER (1.3)                              │
  │  authn → authz → mutating webhooks → schema validate →              │
  │  validating webhooks → persist                                       │
  │                                                                     │
  │  [Admission Flow: 1.12]  [Watch API: 1.13]  [HA: 1.15]             │
  └──────┬─────────────────────────────────────────────────────────────┘
         │
         ▼  (ONLY component that reads/writes etcd)
  ┌─────────────────────────────────────────────────────────────────────┐
  │                          etcd (1.4)                                 │
  │  Raft consensus, MVCC, source of truth                              │
  │  [Internals: 1.14]                                                  │
  └─────────────────────────────────────────────────────────────────────┘

  All other components WATCH the API server (Informer Pattern: 1.13)

  ┌──────────────────────────────────────────────────────────────────┐
  │  SCHEDULER (1.5)                                                 │
  │  Filter → Score → Bind                                           │
  │  Leader elected via Lease (1.11)                                 │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  CONTROLLER MANAGER (1.6)                                        │
  │  Reconciliation loops for all resource types                     │
  │  Leader elected via Lease (1.11)                                 │
  └──────────────────────────────────────────────────────────────────┘

  Per-Node (every worker node):
  ┌──────────────────────────────────────────────────────────────────┐
  │  KUBELET (1.7)           KUBE-PROXY (1.8)                        │
  │  Watches pods for        Watches Services/EndpointSlices         │
  │  its node → calls CRI    → programs iptables/IPVS                │
  │  (Static pods for        (or replaced by Cilium eBPF)            │
  │   control plane)                                                 │
  │         │                                                        │
  │         ▼ CRI                                                    │
  │  CONTAINER RUNTIME (1.9)                                         │
  │  containerd / CRI-O → runc → Linux kernel (namespaces, cgroups) │
  └──────────────────────────────────────────────────────────────────┘

  POD CREATION FULL FLOW (1.10):
  kubectl → API Server → Scheduler → Kubelet → CRI → CNI → Running
  └─── all the above components working in sequence ───────────────┘

  EXTENSIBILITY:
  Custom API Servers via Aggregation Layer (1.16)
  Custom Controllers via CRD + Informers (Category 10)

═══════════════════════════════════════════════════════════════════════════
```

---

# Quick Reference — Interview Cheat Sheet

| Topic | One-Line Answer |
|-------|----------------|
| What is K8s | Container orchestration — runs containers across machines, self-heals, scales |
| Control Plane | API Server + etcd + Scheduler + Controller Manager |
| Worker Node | Kubelet + kube-proxy + Container Runtime |
| API Server | Single front door, stateless, all components talk through it |
| etcd | Distributed KV store, source of truth, Raft consensus, only API server talks to it |
| Scheduler | Watches unbound pods, Filter → Score → Bind |
| Controller Manager | Runs reconciliation loops, closes gap between desired and actual state |
| Kubelet | Node agent, makes pod specs real, calls CRI, runs probes |
| kube-proxy | Programs iptables/IPVS for Service routing on each node |
| Container Runtime | containerd/CRI-O, runs containers via OCI/runc |
| Pod creation flow | kubectl → API Server → etcd → Scheduler → Kubelet → CRI → CNI → Ready |
| Leader election | Scheduler + CM use Lease objects, one active at a time, API servers all active |
| Admission | Mutating (modify) runs before Validating (approve/reject) |
| Watch/Informer | LIST + WATCH stream → local cache → dedup work queue → reconciler |
| Raft | Leader gets quorum ack before committing, odd-numbered clusters, split-brain safe |
| Compaction/Defrag | Compact removes old revisions logically, defrag physically frees disk |
| API HA | All API servers active behind LB, optimistic concurrency via resourceVersion |
| API Aggregation | Proxy custom API groups to custom server (e.g. metrics-server) |

---

*Document generated for Kubernetes Interview Mastery — Category 1: Core Architecture & Fundamentals*
*All 16 topics covered: 1.1 through 1.16*

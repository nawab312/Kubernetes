# Kubernetes Interview Mastery
# CATEGORY 2: WORKLOADS

---

> **How to use this document:**
> Each topic follows the same structure: Simple Explanation → Why It Exists → Internal Working → Key Components → Interview Answers (Short + Deep) → Production Example → Interview Questions → Gotchas → Connections to Other Topics.
> ⚠️ markers indicate frequently asked / commonly misunderstood topics — HIGH PRIORITY.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 2.1 | Pods — The Atomic Unit | 🟢 Beginner |
| 2.2 | ReplicaSet — Ensuring Desired Count | 🟢 Beginner |
| 2.3 | Deployment — Rolling Updates, Rollbacks | 🟢 Beginner |
| 2.4 | StatefulSet — Stateful Workloads ⚠️ | 🟢 Beginner |
| 2.5 | DaemonSet — One Pod Per Node | 🟢 Beginner |
| 2.6 | Job & CronJob — Batch Workloads | 🟢 Beginner |
| 2.7 | Init Containers | 🟢 Beginner |
| 2.8 | Sidecar Containers (Native Sidecar Pattern, K8s 1.29+) | 🟢 Beginner |
| 2.9 | Deployment Strategies — RollingUpdate vs Recreate ⚠️ | 🟡 Intermediate |
| 2.10 | StatefulSet — Headless Services, Stable Network IDs, PVC Retention | 🟡 Intermediate |
| 2.11 | Pod Lifecycle — Phases, Conditions, restartPolicy | 🟡 Intermediate |
| 2.12 | Pod Disruption Budgets (PDB) ⚠️ | 🟡 Intermediate |
| 2.13 | Horizontal Pod Autoscaler (HPA) | 🟡 Intermediate |
| 2.14 | Vertical Pod Autoscaler (VPA) | 🟡 Intermediate |
| 2.15 | KEDA — Event-Driven Autoscaling | 🔴 Advanced |
| 2.16 | Deployment — maxSurge/maxUnavailable Tuning for Zero-Downtime | 🔴 Advanced |
| 2.17 | StatefulSet — Parallel Pod Management, Update Strategies | 🔴 Advanced |
| 2.18 | Custom Workload Controllers (Writing Operators) | 🔴 Advanced |

---

## Difficulty Legend

- 🟢 **Beginner** — Expected from ALL candidates
- 🟡 **Intermediate** — Expected from 3+ year engineers
- 🔴 **Advanced** — Differentiates senior/staff candidates
- ⚠️ **High Priority** — Frequently asked / commonly misunderstood

---

---

# 2.1 Pods — The Atomic Unit

## 🟢 Beginner

---

### What it is in simple terms

A Pod is the **smallest deployable unit in Kubernetes**. It is a wrapper around one or more containers that share the same network namespace, the same IP address, and optionally the same storage volumes. You never interact with containers directly in Kubernetes — you always work through Pods.

---

### Why it exists and what problem it solves

Docker runs individual containers. But real applications often need tightly coupled processes — an app container plus a log shipper, or a web server plus a config reloader. These processes need to share localhost networking and filesystem. A Pod models exactly this: **a group of containers that must run together on the same machine and communicate as if they were on the same host**.

---

### How it works internally

```
═══════════════════════════════════════════════════════════════
POD INTERNALS
═══════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────┐
│                          POD                                 │
│                                                              │
│  ┌─────────────┐   Shared across ALL containers:             │
│  │   pause     │   • Network namespace (same IP, same ports) │
│  │  container  │   • localhost — containers talk via 127.0.0.1│
│  │  (sandbox)  │   • IPC namespace (shared memory)           │
│  └─────────────┘   • Optionally: shared volumes              │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  Container  │  │  Container  │  │  Container  │          │
│  │     A       │  │     B       │  │     C       │          │
│  │  (app)      │  │  (sidecar)  │  │  (adapter)  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                              │
│  Pod IP: 10.244.2.15  (unique across entire cluster)         │
│  Containers share ONE IP — they differ by port only:         │
│    Container A: 10.244.2.15:8080                             │
│    Container B: 10.244.2.15:9090                             │
└──────────────────────────────────────────────────────────────┘

THE PAUSE CONTAINER:
  Created FIRST by kubelet when pod starts.
  Holds the network namespace for the entire pod.
  All other containers join this namespace.
  If pause dies → ALL containers in pod restart.
  Its PID = 1 in the network namespace.
  Never shown in kubectl get pods output.
```

---

### Full Pod YAML — Every Important Field

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: v2
  annotations:
    prometheus.io/scrape: "true"
spec:
  # ── Scheduling ────────────────────────────────────────────
  nodeName: worker-node-1          # direct assignment (bypasses scheduler)
  nodeSelector:
    disktype: ssd
  serviceAccountName: myapp-sa

  # ── Security ──────────────────────────────────────────────
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  # ── Init containers run first, sequentially ───────────────
  initContainers:
  - name: wait-for-db
    image: busybox:1.35
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']

  # ── Main containers ────────────────────────────────────────
  containers:
  - name: app
    image: myapp:v2.1.0
    ports:
    - containerPort: 8080
      name: http

    # Resources — ALWAYS set these in production
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"

    # Environment variables
    env:
    - name: DB_HOST
      value: "postgres.production.svc.cluster.local"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    envFrom:
    - configMapRef:
        name: app-config

    # Health checks
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30
      periodSeconds: 10       # allows up to 300s for app to start

    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 3

    # Volume mounts
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
      readOnly: true
    - name: tmp-dir
      mountPath: /tmp

    # Container-level security
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]

    # Lifecycle hooks
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # allow LB drain before kill

  # ── Volumes ────────────────────────────────────────────────
  volumes:
  - name: config-vol
    configMap:
      name: app-config
  - name: tmp-dir
    emptyDir: {}

  # ── Termination ────────────────────────────────────────────
  terminationGracePeriodSeconds: 30
  restartPolicy: Always       # Always | OnFailure | Never

  # ── DNS ────────────────────────────────────────────────────
  dnsPolicy: ClusterFirst
```

---

### Essential kubectl Commands for Pods

```bash
# Create / apply
kubectl apply -f pod.yaml
kubectl run mypod --image=nginx:1.25 --restart=Never   # bare pod, no controller

# Inspect
kubectl get pods -o wide                   # shows node and IP
kubectl describe pod mypod                 # full details + events
kubectl get pod mypod -o yaml              # full YAML with status
kubectl get pod mypod -o jsonpath='{.status.podIP}'

# Debug
kubectl logs mypod                         # stdout of first container
kubectl logs mypod -c sidecar             # specific container
kubectl logs mypod --previous            # logs from crashed container ⚠️
kubectl exec -it mypod -- /bin/bash      # interactive shell
kubectl exec -it mypod -c sidecar -- sh # shell in specific container
kubectl port-forward pod/mypod 8080:8080 # local port access

# Copy files
kubectl cp mypod:/app/logs/error.log ./error.log

# Delete
kubectl delete pod mypod
kubectl delete pod mypod --grace-period=0 --force   # immediate (use sparingly)

# Watch pod events (critical for debugging startup failures)
kubectl get events --field-selector involvedObject.name=mypod --sort-by='.lastTimestamp'
```

---

### 🎤 Short Crisp Interview Answer

> *"A Pod is the smallest deployable unit in Kubernetes — a wrapper around one or more containers that share the same network namespace, IP address, and optionally volumes. Every pod gets a unique cluster IP. Containers within a pod communicate via localhost. Pods are ephemeral — when they die they don't come back with the same IP or identity. That's why you never run bare pods in production; you use higher-level controllers like Deployments and StatefulSets that manage pod lifecycle and ensure the desired number are always running."*

---

### 📖 Deeper Interview Answer

> *"The key design insight of a pod is the shared network namespace. This is implemented using a pause container — a tiny container that Kubernetes creates first for every pod. The pause container holds the network namespace, and all other containers in the pod join that namespace. This is why all containers share the same IP and can communicate via localhost. The pause container's existence also means if your app container crashes and restarts, the pod IP stays the same — the network namespace is maintained by the pause container which didn't restart. Another important detail: pod resources are scheduled atomically — the scheduler must find a node with enough room for all containers combined. You cannot split one pod across multiple nodes."*

---

### ⚠️ Gotchas

1. **Pods are ephemeral** — when a bare pod dies, it is gone. It does NOT reschedule itself. Only controllers (Deployment, ReplicaSet) reschedule pods. Never run bare pods in production for anything that matters.
2. **One IP per pod, not per container** — all containers in a pod share the pod's IP and differ only by port. Two containers cannot bind the same port or they will conflict.
3. **The pause container is invisible but critical** — always running in every pod, holds the network namespace. Never shown in `kubectl get pods`. Killing the pause container restarts the entire pod.
4. **Pod restarts ≠ Pod replacement** — when a container crashes and restarts (same pod), the IP stays the same. When a pod is evicted and rescheduled (new pod), it gets a completely new IP. These are completely different events.
5. **--previous logs** — one of the most useful debugging tools. When a container crashes, its logs are gone from `kubectl logs`. `kubectl logs --previous` retrieves the previous container's logs before the crash.

---

### Common Interview Questions

**Q: Why can't two containers in the same pod both listen on port 8080?**
> They share the same network namespace — the same IP address. Port 8080 on that IP can only be bound by one process. The second container trying to bind the same port will fail at startup.

**Q: What is the pause container and why does it exist?**
> The pause container is a minimal container that runs in every pod and holds the network namespace. All other containers join this namespace, which is why they share the same IP. If app containers crash and restart, the network namespace (and therefore pod IP) is preserved by the still-running pause container.

**Q: When should you use multiple containers in a pod vs separate pods?**
> Multiple containers in a pod when they are tightly coupled — must share filesystem, must communicate via localhost, have the same lifecycle. Examples: app + sidecar log shipper, app + config reloader. Separate pods when containers are independent services, have different scaling requirements, or should fail independently.

---

### Connections to Other Topics

- Pod scheduling is handled by the **Scheduler (1.5)**
- Pod health is monitored via **Probes (8.2)**
- Pod networking identity (IP) is set up by the **CNI plugin (3.5)**
- Pods are managed by **ReplicaSet (2.2)**, **Deployment (2.3)**, **StatefulSet (2.4)**, **DaemonSet (2.5)**
- Pod termination lifecycle connects to **Deployment zero-downtime (2.16)**

---

---

# 2.2 ReplicaSet — Ensuring Desired Count

## 🟢 Beginner

---

### What it is in simple terms

A ReplicaSet ensures that a **specified number of pod replicas are running at all times**. If pods die, it creates new ones. If there are too many, it deletes some. It uses label selectors to identify which pods it owns.

---

### Why it exists and what problem it solves

Without a ReplicaSet, if a pod crashes it's simply gone. You'd have to manually notice and recreate it. A ReplicaSet automates this continuously — it's the basic self-healing primitive for stateless pod groups.

---

### How it works internally

```
═══════════════════════════════════════════════════════════════
REPLICASET RECONCILIATION LOOP
═══════════════════════════════════════════════════════════════

spec.replicas: 3
spec.selector.matchLabels:
  app: myapp     ← RS "owns" any pod with this label

Current state: 2 pods running (one crashed)

RECONCILIATION:
  desired: 3
  actual:  2  (counted by label selector scan)
  diff:   +1
     │
     ▼
  Create 1 new pod from spec.template
     │
     ▼
  actual: 3  ✓  no action needed until next change

LABEL SELECTOR OWNERSHIP — CRITICAL DETAIL:
  RS does NOT track pod UUIDs.
  It COUNTS pods matching its selector at reconciliation time.

  ⚠️  If you manually create a pod with matching labels:
  Running: [rs-pod-1] [rs-pod-2] [rs-pod-3] [manual-pod] = 4 total
  RS sees: 4 > 3 desired → DELETES one pod (might delete your manual one)

  ⚠️  If you change a running pod's labels:
  RS can no longer see that pod → creates a replacement
  Now you have 4 pods: 3 owned by RS + 1 orphan
```

---

### ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp          # must EXACTLY match pod template labels
  template:               # pod template — RS creates pods from this
    metadata:
      labels:
        app: myapp        # must match selector above
    spec:
      containers:
      - name: app
        image: myapp:v1
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

---

### ReplicaSet kubectl Commands

```bash
# Check RS status
kubectl get rs
# NAME        DESIRED   CURRENT   READY   AGE
# myapp-rs    3         3         3       5d

# See which pods belong to RS (RS name in pod name prefix)
kubectl get pods -l app=myapp

# Scale (rarely done directly — use Deployment)
kubectl scale rs myapp-rs --replicas=5

# See RS details including events
kubectl describe rs myapp-rs

# See RS owned by a Deployment
kubectl get rs -l app=myapp
```

---

### 🎤 Short Crisp Interview Answer

> *"A ReplicaSet ensures a specified number of pod replicas are always running. It uses label selectors to identify its pods, and its reconciliation loop continuously checks actual vs desired count, creating or deleting pods to close the gap. In practice you almost never create ReplicaSets directly — Deployments create and manage ReplicaSets for you, adding rolling update and rollback capabilities on top. The main thing to understand is that RS tracks ownership by label selector, not by pod UUID — which means pods with matching labels get adopted whether you intended it or not."*

---

### ⚠️ Gotchas

1. **Label selector is immutable after creation** — you cannot change `spec.selector` on an existing ReplicaSet. This is why Deployments create a new RS for each update rather than modifying the existing one.
2. **Orphaned pods** — if you delete a ReplicaSet without deleting its pods (`kubectl delete rs --cascade=orphan`), the pods keep running but are now unmanaged. The next RS with the same selector will adopt them.
3. **You rarely use RS directly** — use Deployment instead. RS alone has no rolling update mechanism.

---

### Connections to Other Topics

- ReplicaSets are managed by **Deployments (2.3)**
- Label selectors connect to **pod label fundamentals**
- The reconciliation loop that drives RS connects to **Controller Manager (1.6)**
- RS interacts with **PodDisruptionBudgets (2.12)** during disruptions

---

---

# 2.3 Deployment — Rolling Updates, Rollbacks

## 🟢 Beginner

---

### What it is in simple terms

A Deployment is a **higher-level controller that manages ReplicaSets**. It adds rolling updates, rollbacks, and declarative update strategies on top of ReplicaSet. It is the standard and most common way to run stateless applications in Kubernetes.

---

### Why it exists and what problem it solves

ReplicaSets keep pods running but have no concept of versions or updates. If you change the pod template in a ReplicaSet, it applies to newly created pods only — existing pods are not updated. Deployment solves this by managing multiple ReplicaSets — one per version — and orchestrating the transition between them.

---

### How it works internally

```
═══════════════════════════════════════════════════════════════
DEPLOYMENT → REPLICASET → PODS HIERARCHY
═══════════════════════════════════════════════════════════════

Deployment: myapp
  │
  ├── ReplicaSet: myapp-7d4f8b9c6  (current — v2)  replicas: 3
  │       ├── Pod: myapp-7d4f8b9c6-abc12  [v2]
  │       ├── Pod: myapp-7d4f8b9c6-def34  [v2]
  │       └── Pod: myapp-7d4f8b9c6-ghi56  [v2]
  │
  └── ReplicaSet: myapp-6b3a9d8f1  (previous — v1) replicas: 0
          (kept at 0 for rollback — NOT deleted)

WHAT TRIGGERS A NEW REPLICASET:
  Only changes to spec.template (the pod spec) create a new RS:
  ✓ image change         → new RS
  ✓ env var change       → new RS
  ✓ resource limit change→ new RS
  ✓ label change on pod  → new RS

  Changes that do NOT create a new RS:
  ✗ replicas change       → updates existing RS directly
  ✗ strategy change       → updates Deployment metadata only
  ✗ annotation change     → updates Deployment metadata only

ROLLOUT HISTORY:
  kubectl rollout history deployment/myapp
  REVISION  CHANGE-CAUSE
  1         Initial deploy nginx:1.0
  2         Updated to nginx:1.1
  3         Updated to nginx:1.2  ← current

  Old ReplicaSets are kept up to revisionHistoryLimit (default: 10)
  Each old RS = one rollback revision available
```

---

### Deployment YAML — Production Ready

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  annotations:
    kubernetes.io/change-cause: "Updated to v2.1.0 — hotfix #4521"
spec:
  replicas: 3
  revisionHistoryLimit: 5       # keep last 5 RS for rollback (default 10)
  progressDeadlineSeconds: 600  # fail deployment if no progress in 10min

  selector:
    matchLabels:
      app: myapp

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # max pods ABOVE desired count during update
      maxUnavailable: 0    # max pods BELOW desired count during update
                           # 0 = zero-downtime rolling update

  template:
    metadata:
      labels:
        app: myapp
        version: v2.1.0
    spec:
      containers:
      - name: app
        image: myapp:v2.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 5
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
      terminationGracePeriodSeconds: 60
```

---

### Essential Deployment Operations

```bash
# Deploy / update image
kubectl apply -f deployment.yaml
kubectl set image deployment/myapp app=myapp:v2.2.0  # quick image update

# Monitor rollout progress
kubectl rollout status deployment/myapp
# Waiting for deployment "myapp" rollout to finish:
# 1 out of 3 new replicas have been updated...

# Pause rollout at current state (canary: partial update)
kubectl rollout pause deployment/myapp

# Resume paused rollout
kubectl rollout resume deployment/myapp

# View rollout history
kubectl rollout history deployment/myapp
kubectl rollout history deployment/myapp --revision=2  # see what changed

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2

# Scale
kubectl scale deployment/myapp --replicas=10

# Restart all pods (triggers rolling update with same image)
kubectl rollout restart deployment/myapp

# Check all ReplicaSets managed by this Deployment
kubectl get rs -l app=myapp
# NAME               DESIRED   CURRENT   READY
# myapp-7d4f8b9c6    3         3         3     ← current
# myapp-6b3a9d8f1    0         0         0     ← old (for rollback)
```

---

### 🎤 Short Crisp Interview Answer

> *"A Deployment manages ReplicaSets and adds rolling updates and rollbacks. When you update a Deployment — change image, env vars, or any pod spec field — it creates a new ReplicaSet and gradually scales it up while scaling down the old one. Old ReplicaSets are retained at zero replicas for rollback. The rollout strategy is controlled by maxSurge and maxUnavailable — setting maxUnavailable to 0 gives you zero-downtime deployments by always keeping full capacity while the update progresses."*

---

### 📖 Deeper Interview Answer

> *"The Deployment controller watches for changes to the pod template hash — any change to spec.template generates a new hash, triggering creation of a new ReplicaSet. The rolling update then runs two reconciliation loops simultaneously: scaling up the new RS and scaling down the old RS, subject to the maxSurge and maxUnavailable constraints. The progressDeadlineSeconds field is critical in production — without it, a deployment that's stuck due to an image pull error or failing readiness probe will appear as 'Progressing' indefinitely. With the deadline, it transitions to 'Failed' and triggers alerting. Old ReplicaSets are retained for rollback up to revisionHistoryLimit — but in large clusters with many deployments and frequent releases, these accumulated RS objects add etcd overhead, so setting revisionHistoryLimit to 3-5 is a good practice."*

---

### ⚠️ Gotchas

1. **Only pod template changes trigger a rollout** — changing `replicas` or `strategy` does NOT create a new ReplicaSet. Only changes to `spec.template` (image, env, resources, labels) trigger a new RS and therefore a new rollout.
2. **progressDeadlineSeconds is critical** — if not set (default 600s), a stuck deployment eventually reports `False` on the Progressing condition but doesn't fail loudly. Always set this and alert on it.
3. **revisionHistoryLimit default is 10** — in clusters with many deployments and CI/CD releasing frequently, this creates hundreds of dead RS objects accumulating in etcd. Set to 3-5.
4. **The default strategy is NOT zero-downtime** — default `maxUnavailable: 25%` means with 4 replicas, 1 can be unavailable. Explicitly set `maxUnavailable: 0` for zero-downtime.
5. **rollout restart is a rolling restart** — `kubectl rollout restart` triggers a rolling update with the exact same spec. Useful to force pod recreation (to pick up new ConfigMap from mounted volume, etc.).

---

### Common Interview Questions

**Q: What is the difference between a ReplicaSet and a Deployment?**
> ReplicaSet only ensures a desired number of pods run. Deployment manages ReplicaSets to provide rolling updates, rollbacks, and update history. A Deployment creates a new RS for each version, keeping old ones for rollback.

**Q: How do you roll back a bad deployment?**
> `kubectl rollout undo deployment/myapp` reverts to the previous RS. `kubectl rollout undo deployment/myapp --to-revision=N` reverts to a specific revision. The rollback itself is a rolling update from new RS back to old RS.

**Q: What happens to a Deployment when you update only the replica count?**
> The replica count change is applied directly to the current ReplicaSet. No new RS is created, no rollout is triggered, no new revision appears in history. The change takes effect immediately.

---

### Connections to Other Topics

- Rolling update internals connect to **Deployment Strategies (2.9)**
- maxSurge/maxUnavailable zero-downtime tuning is in **topic 2.16**
- HPA controls replicas on top of Deployment in **topic 2.13**
- PDB protects pods during Deployment rollouts in **topic 2.12**
- Deployment is managed via GitOps by **ArgoCD/Flux (10.12)**

---

---

# ⚠️ 2.4 StatefulSet — Stateful Workloads

## 🟢 Beginner — HIGH PRIORITY

---

### What it is in simple terms

A StatefulSet manages **stateful applications** where pods need stable identities. Unlike Deployments where pods are interchangeable and ephemeral, StatefulSet pods have **stable names, stable network identities, and stable persistent storage** that survives pod restarts and rescheduling.

---

### Why it exists — The Problem with Deployments for Stateful Apps

```
═══════════════════════════════════════════════════════════════
WHY DEPLOYMENT FAILS FOR DATABASES
═══════════════════════════════════════════════════════════════

Imagine running a 3-node MySQL cluster with a Deployment:

  PROBLEM 1: UNSTABLE NAMES
  ─────────────────────────
  Deployment pods: myapp-7d4f8b-abc (random suffix)
  After reschedule: myapp-9a2b3c-pqr (completely new name)
  MySQL nodes refer to each other by hostname → cluster config breaks

  PROBLEM 2: UNSTABLE STORAGE
  ────────────────────────────
  Deployment: one shared PVC OR no dedicated storage per pod
  After pod reschedule: pod gets a fresh empty volume
  All data is GONE

  PROBLEM 3: ORDERING MATTERS
  ────────────────────────────
  MySQL: node 0 (primary) must be fully up before replicas join
  Deployment: starts all pods simultaneously — replicas try to join
  before primary exists → cluster formation fails

  PROBLEM 4: SCALE-DOWN IS DANGEROUS
  ────────────────────────────────────
  Deployment: randomly deletes any pod during scale-down
  Deleting the primary MySQL node could corrupt the cluster

  StatefulSet solves ALL four of these.
```

---

### How StatefulSet Works — The Four Guarantees

```
═══════════════════════════════════════════════════════════════
STATEFULSET GUARANTEES
═══════════════════════════════════════════════════════════════

StatefulSet: mysql  (replicas: 3)

  ┌──────────────────────────────────────────────────────────┐
  │  Pod: mysql-0                                            │
  │    PVC:  data-mysql-0  (dedicated — never reassigned)    │
  │    DNS:  mysql-0.mysql.production.svc.cluster.local      │
  │    Role: PRIMARY (by app convention — ordinal 0)         │
  ├──────────────────────────────────────────────────────────┤
  │  Pod: mysql-1                                            │
  │    PVC:  data-mysql-1  (dedicated)                       │
  │    DNS:  mysql-1.mysql.production.svc.cluster.local      │
  │    Role: REPLICA                                         │
  ├──────────────────────────────────────────────────────────┤
  │  Pod: mysql-2                                            │
  │    PVC:  data-mysql-2  (dedicated)                       │
  │    DNS:  mysql-2.mysql.production.svc.cluster.local      │
  │    Role: REPLICA                                         │
  └──────────────────────────────────────────────────────────┘

GUARANTEE 1: STABLE NAMES
  Pod name: always <statefulset-name>-<ordinal>
  mysql-0, mysql-1, mysql-2 — never random suffixes
  After reschedule: still mysql-0

GUARANTEE 2: STABLE DNS (requires headless service)
  mysql-0.mysql.production.svc.cluster.local → always resolves to mysql-0's IP
  After reschedule to different node with new IP → DNS updates automatically
  Other pods reconnect using the same DNS name

GUARANTEE 3: STABLE STORAGE
  PVC: data-mysql-0 follows pod mysql-0 forever
  Pod reschedules to node-3: PVC rebinds to pod on node-3
  Data persists across all pod restarts and reschedules
  PVCs NOT deleted when pod is deleted (safety feature)

GUARANTEE 4: ORDERED OPERATIONS (default)
  Scale UP:    create 0 first → wait Running+Ready → create 1 → wait → create 2
  Scale DOWN:  delete 2 first → wait terminated → delete 1 → wait → delete 0
  Update:      update 2 first → wait Ready → update 1 → wait → update 0

  Why reverse order for scale-down?
  mysql-0 is the primary — you always want to remove replicas before primary
```

---

### StatefulSet YAML — Production MySQL

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: production
spec:
  serviceName: "mysql"         # REQUIRED: references the headless service
  replicas: 3
  podManagementPolicy: OrderedReady    # or Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0             # update all pods (canary: set to 2 first)

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 60    # time to flush writes to disk
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        # Pod knows its own ordinal from its hostname
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name    # "mysql-0", "mysql-1", etc.
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "127.0.0.1"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 5

  # volumeClaimTemplates — creates ONE PVC per pod automatically
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "gp3"
      resources:
        requests:
          storage: 100Gi

---
# Headless service — REQUIRED for stable per-pod DNS
apiVersion: v1
kind: Service
metadata:
  name: mysql          # must match StatefulSet.spec.serviceName
  namespace: production
spec:
  clusterIP: None      # THIS makes it headless — no virtual IP assigned
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql

---
# Regular service for primary writes (points to mysql-0)
apiVersion: v1
kind: Service
metadata:
  name: mysql-primary
  namespace: production
spec:
  selector:
    app: mysql
    role: primary      # label set by app/operator on mysql-0
  ports:
  - port: 3306
```

---

### StatefulSet kubectl Operations

```bash
# Create
kubectl apply -f statefulset.yaml

# Check status — shows ordered creation
kubectl get pods -l app=mysql -w
# NAME      READY   STATUS    RESTARTS   AGE
# mysql-0   1/1     Running   0          30s
# mysql-1   0/1     Init:0/1  0          5s   ← waiting for mysql-0

# Scale up (creates mysql-3 after mysql-2 is ready)
kubectl scale sts mysql --replicas=4

# Scale down (removes mysql-3 first, then mysql-2, etc.)
kubectl scale sts mysql --replicas=2

# Update image (rolling, reverse order: 2→1→0)
kubectl set image sts/mysql mysql=mysql:8.0.35

# Monitor rolling update
kubectl rollout status sts/mysql

# Rollback
kubectl rollout undo sts/mysql

# Delete StatefulSet but KEEP pods running (dangerous)
kubectl delete sts mysql --cascade=orphan

# Force delete stuck pod (use carefully — may corrupt state)
kubectl delete pod mysql-0 --grace-period=0 --force
```

---

### 🎤 Short Crisp Interview Answer

> *"StatefulSet is for applications that need stable, persistent identity — databases, message queues, distributed systems. Unlike Deployments where pods are interchangeable, StatefulSet gives each pod a stable ordinal name like mysql-0 and mysql-1, a stable DNS entry via a headless service, and a dedicated PVC that survives pod restarts and rescheduling. Pods are created and deleted in order — scale-up goes 0,1,2 sequentially waiting for each to be Ready; scale-down reverses 2,1,0. This ordering matters for applications like Kafka or MySQL where the primary node must be the last to go down."*

---

### 📖 Deeper Interview Answer

> *"The key implementation detail is the volumeClaimTemplate — for each pod with ordinal N, StatefulSet automatically creates a PVC named template-name-pod-name. This PVC has the same lifecycle as the pod's data, not the pod object itself. When you delete mysql-0 and it gets recreated, the new mysql-0 pod binds to the existing data-mysql-0 PVC — your data is intact. A critical distinction: the PVC is NOT deleted when the pod is deleted or even when the StatefulSet is deleted, by default. This is a safety mechanism. The persistentVolumeClaimRetentionPolicy field (stable in 1.27) lets you configure whether PVCs are deleted on scale-down or StatefulSet deletion. For updates, the partition parameter acts as a progressive canary — set partition to 2 to update only pod-2, validate it, then lower partition to 1, then 0."*

---

### ⚠️ Gotchas

1. **Headless service is REQUIRED** — `clusterIP: None`. Without it, per-pod DNS entries don't exist. The StatefulSet still works for pod creation but stable network identity is broken.
2. **PVCs are NOT deleted when StatefulSet is deleted** — this is a safety feature but surprises people. You'll see orphaned PVCs after deleting a StatefulSet. Use `persistentVolumeClaimRetentionPolicy: {whenDeleted: Delete}` to auto-clean if desired.
3. **Stuck StatefulSet rollout** — if pod-0 fails to become Ready, the entire rolling update halts. The cluster doesn't move to pod-1. Unlike Deployment which could continue, StatefulSet halts to protect ordered state. Fix the pod-0 issue or manually delete it.
4. **Force-deleting a StatefulSet pod is dangerous** — `kubectl delete pod --force --grace-period=0` bypasses the normal termination. For stateful apps this can cause split-brain (two pods think they own the same data). Always let the graceful termination complete.
5. **Pod knows its identity** — apps can determine their ordinal from `metadata.name` or the `MY_POD_NAME` env var pattern. mysql-0 knows it's the primary; mysql-1 knows it's a replica. Build this into your container startup logic.

---

### Common Interview Questions

**Q: What is the difference between a Deployment and a StatefulSet?**
> Deployment: pods are interchangeable, random names, shared or no persistent storage, parallel start/stop. StatefulSet: pods have stable ordinal names, dedicated per-pod PVCs, stable DNS via headless service, ordered sequential operations. Use Deployment for stateless apps, StatefulSet for databases, queues, and any app needing stable identity.

**Q: Why does StatefulSet need a headless service?**
> The headless service (clusterIP: None) causes CoreDNS to create individual A records for each pod — mysql-0.mysql, mysql-1.mysql, etc. A regular service would only give you one ClusterIP that load-balances across all pods. Databases need to reach specific pods (the primary vs replicas), not a random one.

**Q: What happens to PVCs when you delete a StatefulSet?**
> By default, PVCs are retained — they are NOT deleted. This prevents accidental data loss. The PVCs remain as orphaned resources. You must delete them manually, or configure `persistentVolumeClaimRetentionPolicy: {whenDeleted: Delete}` to have them auto-cleaned.

**Q: What are some real-world applications that use StatefulSet?**
> MySQL, PostgreSQL, MongoDB, Cassandra, Kafka (via Strimzi operator), ZooKeeper, Elasticsearch, Redis (in cluster mode), etcd itself, Consul.

---

### Connections to Other Topics

- Headless services and per-pod DNS are covered in depth in **topic 2.10**
- PVC templates connect to **Persistent Volumes and Storage Classes (4.2, 4.3)**
- PVC retention policy is covered in **topic 2.10**
- Update strategies and partition canary are covered in **topic 2.17**
- Operators often manage StatefulSets on your behalf — see **topic 2.18**

---

---

# 2.5 DaemonSet — One Pod Per Node

## 🟢 Beginner

---

### What it is in simple terms

A DaemonSet ensures that **exactly one copy of a pod runs on every node** (or every node matching a label selector). When new nodes join the cluster, the DaemonSet pod is automatically scheduled on them. When nodes are removed, pods are garbage collected.

---

### Why it exists and what problem it solves

Some infrastructure components need to run on every single node — log collectors, monitoring agents, network plugins, security scanners. A Deployment doesn't guarantee one-per-node placement and doesn't automatically deploy to new nodes. DaemonSet was purpose-built for this.

```
═══════════════════════════════════════════════════════════════
DAEMONSET — ONE POD PER NODE, AUTO-EXPANDS
═══════════════════════════════════════════════════════════════

  Node 1      Node 2      Node 3      Node 4 (new!)
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │fluentd│   │fluentd│   │fluentd│   │fluentd│ ← auto-created
  └──────┘    └──────┘    └──────┘    └──────┘
     ↑           ↑           ↑
  Existing nodes                     Auto-provisioned on join

  Real-world DaemonSet use cases:
  ─────────────────────────────────────────────────────────────
  fluentd / filebeat / promtail   → log collection per node
  node-exporter                   → node hardware metrics
  kube-proxy                      → Service networking (IS a DaemonSet)
  calico-node / cilium-agent      → CNI network plugin
  falco                           → runtime security scanning
  nvidia-device-plugin            → GPU driver on GPU nodes
  aws-node (vpc-cni)              → EKS pod networking
  datadog-agent                   → APM and infrastructure monitoring
```

---

### DaemonSet YAML — node-exporter

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter

  updateStrategy:
    type: RollingUpdate       # or OnDelete
    rollingUpdate:
      maxUnavailable: 1       # update 1 node at a time

  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      # Run on ALL nodes including control plane
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node.kubernetes.io/not-ready
        operator: Exists
        effect: NoExecute
      - key: node.kubernetes.io/disk-pressure
        operator: Exists
        effect: NoSchedule

      # Access host resources for metrics
      hostPID: true
      hostNetwork: true

      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)
        ports:
        - containerPort: 9100
          hostPort: 9100        # binds directly to node's port
        resources:
          requests:
            cpu: "50m"
            memory: "50Mi"
          limits:
            cpu: "200m"
            memory: "200Mi"
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root-fs
          mountPath: /rootfs
          readOnly: true

      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root-fs
        hostPath:
          path: /

      # Optional: run only on specific nodes (e.g., GPU nodes only)
      # Remove nodeSelector to run on ALL nodes
      nodeSelector:
        node-type: worker

      # Priority to ensure it runs even under resource pressure
      priorityClassName: system-node-critical
```

---

### DaemonSet kubectl Commands

```bash
# Check DaemonSet status
kubectl get daemonset -n monitoring
# NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# node-exporter   5         5         5       5            5

# See on which nodes pods are running
kubectl get pods -l app=node-exporter -o wide

# Check rollout status during update
kubectl rollout status daemonset/node-exporter -n monitoring

# Rollback DaemonSet update
kubectl rollout undo daemonset/node-exporter -n monitoring
```

---

### 🎤 Short Crisp Interview Answer

> *"A DaemonSet ensures exactly one pod runs on every node — or every node matching a selector. New nodes automatically get the pod; removed nodes clean up automatically. It's used for infrastructure components that need cluster-wide coverage: log collectors like Fluentd, monitoring agents like node-exporter, the CNI plugin like Calico or Cilium, and kube-proxy is itself a DaemonSet. DaemonSet pods are placed directly by the DaemonSet controller — it sets nodeName directly on the pod spec, bypassing normal scheduler scoring. But taints and tolerations still apply — a DaemonSet must explicitly tolerate node taints to run on tainted nodes."*

---

### ⚠️ Gotchas

1. **DaemonSet bypasses scheduler scoring** — the DaemonSet controller directly sets `spec.nodeName`, bypassing the scheduler's filter and score phases. This means features like least-allocated scoring don't apply. However, taints/tolerations and nodeSelector are still respected.
2. **Control plane nodes need explicit tolerations** — by default, control plane nodes have a `node-role.kubernetes.io/control-plane:NoSchedule` taint. Your DaemonSet pod won't run there unless you add a matching toleration. For monitoring agents you usually want this.
3. **OnDelete update strategy** — with `updateStrategy: OnDelete`, pods only get the new spec when you manually delete them. Useful for critical node-level daemons where you want to control the exact update timing per node.

---

### Connections to Other Topics

- DaemonSet tolerations connect to **Taints and Tolerations (6.8)**
- `hostNetwork: true` and `hostPath` volumes are security concerns flagged by **Pod Security Standards (7.3)**
- kube-proxy as a DaemonSet connects to **kube-proxy networking (1.8)**
- Cilium and Calico run as DaemonSets — covered in **CNI plugins (3.5)**

---

---

# 2.6 Job & CronJob — Batch Workloads

## 🟢 Beginner

---

### What they are in simple terms

- **Job** — runs one or more pods to **completion**. Ensures a task finishes successfully, retrying on failure. When done, pods are not restarted.
- **CronJob** — creates Jobs on a **cron schedule**. Like Unix cron, but Kubernetes-native with full pod management.

---

### Why they exist and what problem they solve

Deployments run pods forever — they restart on exit. But batch workloads (database migrations, report generation, data processing, cleanup tasks) need to run once, finish, and stop. Jobs provide the "run to completion" semantic that Deployments deliberately lack.

---

### Job Patterns

```
═══════════════════════════════════════════════════════════════
JOB PATTERNS — THREE COMMON CONFIGURATIONS
═══════════════════════════════════════════════════════════════

PATTERN 1: Single Completion (completions: 1, parallelism: 1)
──────────────────────────────────────────────────────────────
  Run one task once:
  [Pod] → runs → exits 0 → Job COMPLETE ✓
        → exits non-0 → restart (up to backoffLimit times)
        → exits non-0 after backoffLimit → Job FAILED ✗

  Use for: DB migrations, one-off scripts, setup tasks

PATTERN 2: Fixed Completion Count (completions: N, parallelism: M)
───────────────────────────────────────────────────────────────────
  completions: 5, parallelism: 2
  Run 5 successful completions total, 2 at a time:

  [Pod-1] [Pod-2] → both exit 0  → count: 2/5
  [Pod-3] [Pod-4] → both exit 0  → count: 4/5
  [Pod-5]         → exits 0      → count: 5/5 → Job COMPLETE ✓

  Use for: batch processing with known item count

PATTERN 3: Work Queue (completions: unset, parallelism: N)
──────────────────────────────────────────────────────────
  completions not set → job completes when ANY pod exits 0
                        and signals all work is done
  Pods pull work from Redis/SQS/Kafka queue independently

  [Pod-1] [Pod-2] [Pod-3]  ← all pulling from SQS
  Pod-1 exits 0 (queue empty) → Job controller signals completion
  Remaining pods terminated

  Use for: queue processors, map-reduce style workloads

RETRY BEHAVIOR:
  backoffLimit: 4          → retry up to 4 times
  backoff delays: 10s, 20s, 40s, 80s  (exponential)
  After backoffLimit exhausted → Pod status = Failed, Job = Failed
```

---

### Job YAML — Database Migration

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v2
  namespace: production
spec:
  completions: 1              # need exactly 1 successful completion
  parallelism: 1              # run 1 pod at a time
  backoffLimit: 4             # retry up to 4 times on failure
  activeDeadlineSeconds: 300  # kill job if not complete in 5 minutes
  ttlSecondsAfterFinished: 86400   # auto-delete job + pods after 24h

  template:
    metadata:
      labels:
        job: db-migration
    spec:
      restartPolicy: OnFailure    # REQUIRED: must be OnFailure or Never
      containers:
      - name: migration
        image: myapp:v2.1.0
        command: ["python", "manage.py", "migrate", "--no-input"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"

---
# Parallel batch job with indexed completion mode
apiVersion: batch/v1
kind: Job
metadata:
  name: image-processor
spec:
  completions: 100              # process 100 images
  parallelism: 10               # 10 workers at a time
  completionMode: Indexed       # each pod gets a unique index 0-99
                                # via JOB_COMPLETION_INDEX env var
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: processor
        image: image-processor:v1
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

---

### CronJob YAML — Scheduled Report

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  namespace: production
spec:
  schedule: "0 2 * * *"           # 2:00 AM UTC every day
  timeZone: "America/New_York"    # K8s 1.27+ stable timezone support
  concurrencyPolicy: Forbid       # Forbid | Allow | Replace
  successfulJobsHistoryLimit: 3   # keep last 3 successful job records
  failedJobsHistoryLimit: 3       # keep last 3 failed job records
  startingDeadlineSeconds: 300    # skip if can't start within 5min of schedule

  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800   # 30 minute max runtime
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: reporter
            image: myreporter:v1
            command: ["python", "generate_report.py", "--date=$(date +%Y-%m-%d)"]
            resources:
              requests:
                cpu: "100m"
                memory: "256Mi"

---
# concurrencyPolicy options:
# Allow   → if previous run is still running, start new one anyway (dangerous)
# Forbid  → if previous run is still running, SKIP this scheduled run
# Replace → if previous run is still running, DELETE it and start new one
```

---

### Job kubectl Commands

```bash
# View jobs
kubectl get jobs -n production
# NAME               COMPLETIONS   DURATION   AGE
# db-migration-v2    1/1           23s        5m

# Watch job progress
kubectl get jobs -w

# View pods created by a job
kubectl get pods -l job-name=db-migration-v2

# View job logs
kubectl logs -l job-name=db-migration-v2

# Manually trigger a CronJob immediately (for testing)
kubectl create job --from=cronjob/daily-report manual-run-$(date +%s)

# View CronJob history
kubectl get cronjob daily-report
# NAME           SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# daily-report   0 2 * * *   False     0        8h              30d

# Suspend a CronJob (stop future scheduled runs)
kubectl patch cronjob daily-report -p '{"spec":{"suspend":true}}'
```

---

### 🎤 Short Crisp Interview Answer

> *"A Job runs pods to completion — unlike Deployments which restart pods forever. It retries on failure up to backoffLimit times and supports parallel execution patterns for batch processing. A CronJob creates Jobs on a schedule. Key settings: restartPolicy must be OnFailure or Never for Jobs — not Always; concurrencyPolicy controls whether overlapping executions are allowed — use Forbid for most jobs to prevent pile-up; and ttlSecondsAfterFinished prevents completed jobs from cluttering the cluster and etcd."*

---

### ⚠️ Gotchas

1. **restartPolicy: Always is forbidden in Jobs** — Jobs must use `OnFailure` or `Never`. `Always` would restart a successfully completed pod indefinitely, defeating the purpose. Kubernetes rejects this at admission.
2. **CronJob misfire on cluster downtime** — if the cluster is down when a job should run and `startingDeadlineSeconds` has passed, the execution is SKIPPED — not queued for later. If startingDeadlineSeconds is not set, missed runs that stack up (>100) cause the CronJob to stop scheduling entirely.
3. **concurrencyPolicy: Allow is dangerous** — if a job regularly runs longer than its schedule interval, you accumulate running pods consuming resources indefinitely. Always use `Forbid` or `Replace`.
4. **activeDeadlineSeconds vs backoffLimit** — activeDeadlineSeconds is a hard wall-clock limit for the entire job. backoffLimit is the number of pod-level retries. Both can kill a job. Know which one triggered.
5. **ttlSecondsAfterFinished default is nil** — completed Jobs and their pods accumulate forever without this set. In active clusters this creates significant etcd bloat. Set to 86400 (24h) or similar.

---

### Common Interview Questions

**Q: What is the difference between activeDeadlineSeconds and backoffLimit?**
> backoffLimit is the number of pod retry attempts before the Job is marked Failed. activeDeadlineSeconds is a hard time limit for the entire Job — if it hasn't finished in that many seconds, it's killed regardless of remaining retries. Both can cause a Job to fail; they're complementary, not duplicates.

**Q: How do you run a CronJob immediately for testing?**
> `kubectl create job --from=cronjob/<name> <manual-job-name>` — this creates a one-off Job from the CronJob's jobTemplate without waiting for the schedule.

**Q: What happens if a CronJob's previous run is still running when the next schedule fires?**
> Depends on `concurrencyPolicy`. Forbid skips the new run. Allow starts both simultaneously. Replace deletes the old run and starts fresh. Use Forbid for most production jobs.

---

### Connections to Other Topics

- Jobs running alongside Istio sidecars — solved by **Native Sidecar Containers (2.8)**
- Jobs use the same **Pod lifecycle (2.11)** mechanics
- KEDA can trigger Jobs based on events in **topic 2.15**
- Helm hooks use Jobs for pre/post install tasks — **Helm (9.7)**

---

---

# 2.7 Init Containers

## 🟢 Beginner

---

### What it is in simple terms

Init containers are **special containers that run to completion sequentially before any app containers start**. Each must succeed (exit code 0) before the next begins. If an init container fails, the pod restarts it (based on restartPolicy) until it succeeds or the pod fails.

---

### Why they exist and what problem they solve

App containers often need prerequisites before they can start safely. You could put prerequisite logic inside the app itself, but that couples infrastructure concerns to application code, makes the app image heavier, and duplicates logic across services. Init containers externalize these concerns cleanly, with access to different tools and images.

---

### Execution Flow

```
═══════════════════════════════════════════════════════════════
INIT CONTAINER EXECUTION — SEQUENTIAL GATE PATTERN
═══════════════════════════════════════════════════════════════

Pod starts → kubelet runs init containers in ORDER:

  ┌──────────────────────────────────────────────────────────┐
  │  INIT CONTAINER 1: "wait-for-postgres"                   │
  │  Command: until pg_isready -h postgres; do sleep 2; done │
  │  Running... Running... Running... SUCCESS (exit 0)        │
  └──────────────────────────────────────────────────────────┘
                    │  (ONLY if exit 0)
                    ▼
  ┌──────────────────────────────────────────────────────────┐
  │  INIT CONTAINER 2: "run-migrations"                      │
  │  Command: python manage.py migrate                       │
  │  Running... SUCCESS (exit 0)                             │
  └──────────────────────────────────────────────────────────┘
                    │  (ONLY if exit 0)
                    ▼
  ┌──────────────────────────────────────────────────────────┐
  │  INIT CONTAINER 3: "fetch-config-from-vault"             │
  │  Command: vault kv get ... > /shared/config.json         │
  │  Running... SUCCESS (exit 0)                             │
  └──────────────────────────────────────────────────────────┘
                    │  (ONLY if all init containers succeeded)
                    ▼
  ┌──────────────────────────────────────────────────────────┐
  │  APP CONTAINER starts NOW                                │
  │  • Database guaranteed to be ready                       │
  │  • Migrations guaranteed to be complete                  │
  │  • Config guaranteed to exist at /etc/myapp/config.json  │
  └──────────────────────────────────────────────────────────┘

POD STATUS DURING INIT:
  kubectl get pods
  NAME     READY   STATUS            RESTARTS   AGE
  myapp    0/1     Init:0/3          0          5s   ← waiting on init 1
  myapp    0/1     Init:1/3          0          30s  ← init 1 done, init 2 running
  myapp    0/1     Init:2/3          0          45s  ← init 2 done, init 3 running
  myapp    0/1     PodInitializing   0          50s  ← all init done, app starting
  myapp    1/1     Running           0          55s  ← app up

KEY PROPERTIES:
  ✓ Run sequentially — one at a time, in order defined
  ✓ Must exit 0 to proceed — exit 1 triggers pod restart
  ✓ Can use DIFFERENT images than app (e.g., curl, vault CLI)
  ✓ Can share volumes with app containers to pass data
  ✗ Do NOT run liveness or readiness probes
  ✗ Not restarted individually — entire pod restarts
```

---

### Init Container YAML — Full Production Example

```yaml
spec:
  initContainers:

  # 1. Wait for database to be available
  - name: wait-for-postgres
    image: postgres:15-alpine       # has pg_isready tool
    command:
    - sh
    - -c
    - |
      echo "Waiting for postgres at postgres.production:5432..."
      until pg_isready -h postgres.production -p 5432 -U appuser; do
        echo "Postgres not ready yet, sleeping 2s..."
        sleep 2
      done
      echo "Postgres is ready!"
    resources:
      requests:
        cpu: "10m"
        memory: "32Mi"

  # 2. Run database schema migrations
  - name: run-migrations
    image: myapp:v2.1.0             # same image as app
    command: ["python", "manage.py", "migrate", "--no-input"]
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"

  # 3. Fetch config from HashiCorp Vault
  - name: fetch-vault-config
    image: vault:1.15               # different image — has vault CLI
    command:
    - sh
    - -c
    - |
      vault login -method=kubernetes role=myapp
      vault kv get -field=config secret/production/myapp > /shared/config.json
      echo "Config fetched successfully"
    env:
    - name: VAULT_ADDR
      value: "https://vault.internal:8200"
    volumeMounts:
    - name: shared-config
      mountPath: /shared            # write config here

  # 4. Set correct permissions on data directory
  - name: fix-permissions
    image: busybox:1.35
    command: ["sh", "-c", "chown -R 1000:1000 /data && chmod 700 /data"]
    securityContext:
      runAsUser: 0                  # need root just for this init task
    volumeMounts:
    - name: data-volume
      mountPath: /data

  containers:
  - name: app
    image: myapp:v2.1.0
    volumeMounts:
    - name: shared-config
      mountPath: /etc/myapp         # reads config written by init-3
    - name: data-volume
      mountPath: /data
    securityContext:
      runAsUser: 1000               # non-root for app

  volumes:
  - name: shared-config
    emptyDir: {}                    # shared between init and app
  - name: data-volume
    persistentVolumeClaim:
      claimName: myapp-data
```

---

### 🎤 Short Crisp Interview Answer

> *"Init containers run to completion sequentially before any app container starts. They're used for prerequisites — waiting for dependencies to be ready, running migrations, fetching secrets, or fixing filesystem permissions. Each init container must exit with code 0 before the next starts. If one fails, the entire pod restarts it. They're powerful because you can use completely different images with specialized tooling than your app, and they can share emptyDir volumes to pass data to app containers. The pod shows Init:0/N, Init:1/N status during initialization."*

---

### ⚠️ Gotchas

1. **Init containers restart the WHOLE POD on failure** — not just the init container itself. If `restartPolicy: Always`, the pod keeps restarting and you see CrashLoopBackOff with status `Init:CrashLoopBackOff`.
2. **Resource limits for scheduling purposes** — the scheduler uses the max of (largest init container limits, sum of all app container limits). Init containers run sequentially so their resources are not additive. A large init container can cause the pod to need more resources than the app alone would.
3. **Long-running init blocks readiness** — an init container waiting for a dependency that never starts (circular dependency, network misconfiguration) keeps pods in `Init:0/N` forever. Always build in timeouts in your wait logic.
4. **Volumes are accessible** — init containers can write to emptyDir volumes that app containers read. This is the clean pattern for injecting config files, initializing database schemas, or setting up credentials without baking them into the app image.

---

### Common Interview Questions

**Q: What is the difference between an init container and a regular container?**
> Init containers run sequentially before app containers start, must complete successfully (exit 0), don't run probes, and don't restart individually on failure (pod restarts). Regular containers run concurrently, run indefinitely, support probes, and can be restarted individually.

**Q: How do init containers communicate data to app containers?**
> Via shared volumes — typically emptyDir. The init container writes to a volume mountPath; the app container mounts the same volume and reads from it. This enables patterns like fetching secrets, generating config files, or downloading assets at startup.

**Q: Can an init container use a different image from the app?**
> Yes, and this is one of their key advantages. You can use a heavyweight image with curl, vault CLI, or psql tools in the init container, and keep your app image minimal. Each init container specifies its own image independently.

---

### Connections to Other Topics

- Init containers are the setup phase before **Sidecar Containers (2.8)**
- Contrast with native sidecars which run ALONGSIDE app — **topic 2.8**
- The Pod lifecycle phases during init — **Pod Lifecycle (2.11)**
- Init containers running Vault config fetch connects to **External Secrets (5.4)**

---

---

# 2.8 Sidecar Containers (Native Sidecar Pattern, K8s 1.29+)

## 🟢 Beginner

---

### What it is in simple terms

A sidecar is a **helper container that runs alongside the main app container** in the same pod. It extends or supports the app without modifying the app code. Before K8s 1.29, sidecars were just regular containers with no special lifecycle guarantees. K8s 1.29 introduced **native sidecar containers** as a formal first-class concept.

---

### Why it exists and what problem it solves

The sidecar pattern separates cross-cutting concerns from application logic. Instead of every application implementing logging, metrics collection, mTLS, and secret rotation, these run as independent containers alongside the app. This enables language-agnostic infrastructure capabilities.

---

### Old Pattern vs Native Sidecar — The Key Difference

```
═══════════════════════════════════════════════════════════════
OLD SIDECAR (Before K8s 1.29) — Regular container, no ordering
═══════════════════════════════════════════════════════════════

Pod startup:
  [init-container] → COMPLETES → [APP + SIDECAR start SIMULTANEOUSLY]

Problems:
  ✗ Race condition at startup:
    App starts before Envoy sidecar is ready
    App's first outbound calls fail (Envoy not listening yet)
    → 500 errors, failed health checks, slow start

  ✗ Termination ordering problem:
    App receives SIGTERM → starts draining
    Sidecar (log-shipper) receives SIGTERM simultaneously
    Sidecar stops BEFORE app finishes draining
    → Last seconds of logs lost

  ✗ Job completion problem:
    Job's main container finishes → exits 0
    Sidecar (Envoy/Istio) keeps running → NEVER EXITS
    → Job pod never terminates → Job never completes
    → Cluster accumulates stuck Job pods forever


═══════════════════════════════════════════════════════════════
NATIVE SIDECAR (K8s 1.29+) — restartPolicy: Always in initContainers
═══════════════════════════════════════════════════════════════

Pod startup:
  [init-containers] → [SIDECAR starts, stays running] → [APP starts]
                              ↑
                     restartPolicy: Always in the initContainers section
                     Starts BEFORE app. Stays running throughout pod lifecycle.

Benefits:
  ✓ Sidecar starts BEFORE app — Envoy ready before app's first request
  ✓ On pod termination: APP stops FIRST, then sidecar stops
    → log-shipper flushes all logs before exiting
  ✓ Job completion: sidecar exits automatically when main container exits
    → No more stuck Job pods with Istio sidecars
  ✓ Sidecar restarts independently if it crashes (restartPolicy: Always)
    without restarting the whole pod

EXECUTION ORDER:
  STARTUP:     init-containers (sequential) → sidecars (parallel) → app containers
  TERMINATION: app containers → sidecars → (never the other way)
```

---

### Native Sidecar YAML (K8s 1.29+)

```yaml
spec:
  initContainers:

  # Regular init container — runs and EXITS before app starts
  - name: wait-for-db
    image: busybox:1.35
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 1; done']
    # No restartPolicy field → regular init container behavior

  # NATIVE SIDECAR — starts before app, stays running alongside it
  - name: log-shipper
    image: fluent-bit:2.2
    restartPolicy: Always          # ← THIS makes it a native sidecar
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
    - name: fluent-bit-config
      mountPath: /fluent-bit/etc

  # Another native sidecar — Vault Agent for secret renewal
  - name: vault-agent
    image: vault:1.15
    restartPolicy: Always          # ← native sidecar
    command: ["vault", "agent", "-config=/vault/config.hcl"]
    volumeMounts:
    - name: vault-config
      mountPath: /vault
    - name: shared-secrets
      mountPath: /secrets

  # App container starts AFTER all native sidecars are Ready
  containers:
  - name: app
    image: myapp:v2.1.0
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
    - name: shared-secrets
      mountPath: /etc/secrets      # secrets written by vault-agent sidecar

  volumes:
  - name: log-volume
    emptyDir: {}
  - name: shared-secrets
    emptyDir: {}
  - name: fluent-bit-config
    configMap:
      name: fluent-bit-config
  - name: vault-config
    configMap:
      name: vault-agent-config
```

---

### Common Sidecar Use Cases

```
USE CASE            SIDECAR IMAGE           WHAT IT DOES
────────────────────────────────────────────────────────────────────
Service mesh        Envoy (Istio inject)    mTLS, circuit breaking, tracing
Log collection      Fluent Bit / Fluentd    Tail app logs → Loki/Elasticsearch
Secret injection    Vault Agent             Renew + write secrets to filesystem
Config refresh      configmap-reload        Watch ConfigMap, send SIGHUP to app
Metrics adapter     prometheus-exporter     Expose app metrics in Prom format
Auth proxy          oauth2-proxy            Handle OIDC auth in front of app
Debug tools         netshoot               Temporary network debugging tools
```

---

### The Job + Istio Problem — Solved by Native Sidecars

```
═══════════════════════════════════════════════════════════════
BEFORE NATIVE SIDECARS: JOBS WITH ISTIO WERE BROKEN
═══════════════════════════════════════════════════════════════

Job pod with Istio sidecar (old way):
  [app container: exits 0]  ← job is "done"
  [envoy sidecar: still running]  ← envoy never exits
  → Job pod stays in Running state forever
  → Job completion count never increments
  → kubectl get jobs shows 0/1 COMPLETIONS indefinitely

Workaround (ugly, common in production):
  app container, before exit:
  curl -X POST http://127.0.0.1:15020/quitquitquit  ← kill Envoy manually

AFTER NATIVE SIDECARS: Jobs work properly
  [app container: exits 0]  ← job is "done"
  [envoy sidecar: receives signal, exits gracefully]  ← respects lifecycle
  → Job pod terminates cleanly
  → Job shows 1/1 COMPLETIONS

This was one of the primary motivations for native sidecar support.
```

---

### 🎤 Short Crisp Interview Answer

> *"A sidecar container runs alongside the main app container to provide supporting functionality without modifying app code — examples include Envoy for service mesh, Fluent Bit for log shipping, and Vault Agent for secret injection. Before K8s 1.29, sidecars were regular containers with no lifecycle ordering guarantees — causing startup race conditions and Job completion problems with Istio. K8s 1.29 introduced native sidecars: by setting restartPolicy: Always in the initContainers section, the sidecar starts before the app, stays running throughout, and exits after the app terminates. This fixes both the startup ordering problem and the Job + Istio completion problem."*

---

### ⚠️ Gotchas

1. **Native sidecar = `restartPolicy: Always` inside `initContainers`** — it's counter-intuitive. You declare it in the init containers section but it behaves like a regular container that persists for the pod's lifetime.
2. **Sidecar resource overhead adds up** — Envoy sidecar consumes ~50-100m CPU and 100-200Mi memory per pod. At 1000 pods, that's 50-100 CPU cores and 100-200GB RAM just for sidecars. This is one reason Istio ambient mesh (sidecarless) is gaining traction.
3. **Old sidecar pattern still common** — K8s 1.29 is relatively recent. Most clusters in production still use the old pattern. Know both approaches and when the old workaround (`quitquitquit`) is needed.

---

### Common Interview Questions

**Q: Why did Kubernetes add native sidecar support? What was broken before?**
> Three problems: startup ordering (sidecar not ready when app starts), termination ordering (sidecar stops before app drains), and Job completion (sidecars like Envoy never exited, preventing Jobs from completing). Native sidecars with `restartPolicy: Always` in initContainers solve all three.

**Q: How is a native sidecar different from a regular init container?**
> A regular init container runs, exits, and never runs again. A native sidecar (init container with restartPolicy: Always) starts before the app and keeps running alongside it for the pod's entire lifetime — it has the same lifecycle as a regular container but with startup ordering guarantees.

---

### Connections to Other Topics

- Envoy sidecar injection by Istio is covered in **Istio Architecture (11.2, 11.3, 11.4)**
- Vault Agent sidecar connects to **External Secrets (5.4)**
- Native sidecars solve the **Job (2.6)** + service mesh problem
- Resource overhead of sidecars connects to **Resource Requests and Limits (6.1)**

---

---

# ⚠️ 2.9 Deployment Strategies — RollingUpdate vs Recreate

## 🟡 Intermediate — HIGH PRIORITY

---

### What it is in simple terms

When you update a Deployment, Kubernetes replaces old pods with new ones. The strategy controls **how** that replacement happens — whether you have downtime, how many pods run simultaneously, and how long the update takes.

---

### Strategy 1: RollingUpdate (Default)

```
═══════════════════════════════════════════════════════════════
ROLLING UPDATE — GRADUAL REPLACEMENT
═══════════════════════════════════════════════════════════════

Deployment: replicas=4, maxSurge=1, maxUnavailable=1

BEFORE UPDATE:   [v1] [v1] [v1] [v1]   (4 pods, capacity=100%)

STEP 1:  Create 1 new v2 pod (surge allows 5 total)
         [v1] [v1] [v1] [v1] [v2]      (5 pods — maxSurge=1)

STEP 2:  Wait for v2 pod to pass readiness probe
         [v1] [v1] [v1] [v1] [v2✓]

STEP 3:  Terminate 1 old v1 pod (maxUnavailable=1 allows 3 ready)
         [v1] [v1] [v1] [v2]           (back to 4)

STEP 4:  Repeat — create another v2, wait ready, kill v1
         [v1] [v1] [v2] [v2]
         [v1] [v2] [v2] [v2]
         [v2] [v2] [v2] [v2]           ← UPDATE COMPLETE ✓

DURING THE UPDATE:
  Service sends traffic to BOTH v1 and v2 pods simultaneously
  ⚠️  Your app MUST be backward compatible during this window!
  Old API clients hitting v2, new API clients hitting v1 → both must work

═══════════════════════════════════════════════════════════════
ZERO-DOWNTIME: maxUnavailable=0
═══════════════════════════════════════════════════════════════

  replicas=4, maxSurge=1, maxUnavailable=0

  [v1] [v1] [v1] [v1]
  [v1] [v1] [v1] [v1] [v2]    ← create v2 FIRST (5 pods)
  [v1] [v1] [v1] [v2]         ← only kill v1 AFTER v2 is ready
  [v1] [v1] [v2] [v2]
  [v1] [v2] [v2] [v2]
  [v2] [v2] [v2] [v2]         ← always had at least 4 ready ✓

  Cost: 25% extra capacity during update (acceptable trade-off)
```

---

### Strategy 2: Recreate

```
═══════════════════════════════════════════════════════════════
RECREATE STRATEGY — FULL DOWNTIME, CLEAN CUTOVER
═══════════════════════════════════════════════════════════════

  BEFORE:  [v1] [v1] [v1] [v1]  ← all serving traffic

  STEP 1:  Kill ALL v1 pods
           [   ] [   ] [   ] [   ]  ← DOWNTIME STARTS
           Service returns no endpoints → connection refused

  STEP 2:  Create all v2 pods
           [v2] [v2] [v2] [v2]  ← DOWNTIME ENDS
           After readiness probes pass, traffic resumes

YAML configuration:
  strategy:
    type: Recreate      # no rollingUpdate section

USE CASES — when Recreate is the RIGHT choice:
  ✓ Database schema change that BREAKS old version
    (column renamed, type changed — v1 crashes with new schema)
  ✓ App acquires an exclusive file lock or port
    (v2 can't start while v1 holds the lock)
  ✓ Single-writer configurations
    (only one writer allowed at a time — message queue consumer)
  ✓ Memory-constrained nodes
    (can't run old + new simultaneously — cluster would OOM)
  ✓ Fundamental architecture change
    (v2 is completely incompatible with v1's data format)
```

---

### maxSurge and maxUnavailable — The Complete Math

```
═══════════════════════════════════════════════════════════════
maxSurge AND maxUnavailable — TUNING REFERENCE
═══════════════════════════════════════════════════════════════

  replicas: N

  maxSurge:        max pods ABOVE N allowed during update
                   Adds capacity → costs more resources
  maxUnavailable:  max pods BELOW N allowed during update
                   Reduces capacity → saves resources but risky

  Accepted formats:
    Absolute:    maxSurge: 2          (always 2 extra)
    Percentage:  maxSurge: "25%"      (ceil of 25% of N)
    Zero:        maxUnavailable: 0    (zero-downtime mode)

  COMMON PRESETS:
  ─────────────────────────────────────────────────────────────
  Zero-downtime (production recommended):
    maxSurge: 1
    maxUnavailable: 0

  Fast rollout (accepts brief capacity reduction):
    maxSurge: 0
    maxUnavailable: "25%"

  Balanced (default — NOT zero-downtime):
    maxSurge: "25%"
    maxUnavailable: "25%"   ← ⚠️ NOT zero-downtime!

  Canary via pause (1 canary pod):
    maxSurge: 1
    maxUnavailable: 0
    kubectl rollout pause after first pod updates

  Big Bang (same as Recreate but with surge):
    maxSurge: "100%"
    maxUnavailable: "100%"
  ─────────────────────────────────────────────────────────────

  CONSTRAINT: maxSurge and maxUnavailable CANNOT BOTH BE 0
  Kubernetes rejects this — no progress would be possible.

  ⚠️  DEFAULT IS NOT ZERO-DOWNTIME:
  Default: maxSurge=25%, maxUnavailable=25%
  With 4 replicas: 1 pod can be unavailable during update
  → Capacity drops to 75% briefly
  → Set maxUnavailable: 0 EXPLICITLY for zero-downtime
```

---

### 🎤 Short Crisp Interview Answer

> *"Deployment has two strategies. RollingUpdate gradually replaces old pods with new ones, controlled by maxSurge and maxUnavailable. maxSurge sets how many extra pods can exist above desired count; maxUnavailable sets how many can be below desired capacity. For zero-downtime set maxUnavailable=0 — capacity never drops below desired, new pods must be Ready before old ones are removed. Recreate kills all old pods first then creates new — has downtime but is necessary when two versions cannot coexist, like a database schema migration that breaks the old app version."*

---

### 📖 Deeper Interview Answer

> *"The critical but often missed issue with rolling updates is that both versions run simultaneously and receive traffic. This requires backward compatibility — if you're adding a required database column and deploying a new app version that depends on it, during the rollout you have old app pods talking to the new schema. This is why the expand-contract migration pattern exists: first deploy a migration that adds a nullable column (old app still works), then deploy new app that uses it, then deploy a migration to make it non-nullable. For maxSurge and maxUnavailable tuning, the key insight is that maxSurge costs money (extra pods) while maxUnavailable costs availability. In cost-sensitive environments with good PDBs, maxSurge=0 maxUnavailable=1 is common. In availability-critical services, maxSurge=1 maxUnavailable=0 is the standard."*

---

### ⚠️ Gotchas

1. **The default is NOT zero-downtime** — `maxUnavailable: 25%` is the default. With 4 replicas, 1 pod can be unavailable. Explicitly set `maxUnavailable: 0` for zero-downtime.
2. **Both versions serve traffic simultaneously** — during rolling update, your service load-balances to both old and new pods. Your app must handle this — backward-compatible APIs, backward-compatible database changes.
3. **maxSurge=0 AND maxUnavailable=0 is invalid** — Kubernetes rejects this configuration as it would prevent any progress.
4. **Percentages are rounded differently** — maxSurge rounds UP (ceil), maxUnavailable rounds DOWN (floor). With replicas=3 and 25%: maxSurge=1 (ceil(0.75)), maxUnavailable=0 (floor(0.75)).

---

### Common Interview Questions

**Q: When would you use Recreate strategy instead of RollingUpdate?**
> When two versions of the app cannot coexist: schema migrations that break old app, exclusive resource acquisition, single-writer constraints, or when you simply cannot afford the extra capacity for maxSurge and readiness probe window.

**Q: How do you achieve zero-downtime rolling updates?**
> Set `maxUnavailable: 0` and `maxSurge: 1` (or higher). Ensure readiness probes accurately reflect when the app is ready to serve traffic. Add a preStop sleep hook to handle the kube-proxy propagation delay. Set terminationGracePeriodSeconds large enough for the app to drain in-flight requests.

**Q: What is the difference between maxSurge and maxUnavailable?**
> maxSurge adds extra capacity during the update — you temporarily have more pods than desired. maxUnavailable removes capacity — you temporarily have fewer ready pods than desired. Zero-downtime requires maxUnavailable=0, accepting the cost of maxSurge extra pods.

---

### Connections to Other Topics

- Zero-downtime advanced tuning including SIGTERM race condition — **topic 2.16**
- PDB works alongside rolling updates to prevent over-disruption — **topic 2.12**
- Rolling updates trigger the **preStop hook and SIGTERM flow (2.11)**
- StatefulSet has its own update strategy with partitions — **topic 2.17**

---


---

# 2.10 StatefulSet — Headless Services, Stable Network IDs, PVC Retention

## 🟡 Intermediate

---

### Headless Services — DNS Deep Dive

```
REGULAR SERVICE (clusterIP assigned):
  mysql.production.svc.cluster.local → 10.96.45.23 (single ClusterIP)
  Traffic: ClusterIP → kube-proxy → random pod. You never know which pod.

HEADLESS SERVICE (clusterIP: None):
  mysql.production.svc.cluster.local → [10.244.1.5, 10.244.2.6, 10.244.3.7]
  DNS returns ALL pod IPs directly as A records. Client picks.

PER-POD DNS (StatefulSet + headless service):
  mysql-0.mysql.production.svc.cluster.local → 10.244.1.5
  mysql-1.mysql.production.svc.cluster.local → 10.244.2.6
  mysql-2.mysql.production.svc.cluster.local → 10.244.3.7

  After pod reschedule (new IP 10.244.4.8):
  mysql-0.mysql.production.svc.cluster.local → 10.244.4.8 (auto-updated)
  Application reconnects using same name — no config change needed.
```

---

### PVC Retention Policy (K8s 1.27+ Stable)

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain    # PVCs survive StatefulSet deletion (DEFAULT)
    whenScaled: Delete     # PVCs deleted when pod is scaled away

# whenDeleted: Retain  → never auto-delete data on StatefulSet delete
# whenScaled:  Delete  → clean up PVCs for pods you intentionally removed
```

---

### 🎤 Short Crisp Interview Answer

> *"StatefulSet headless service provides stable DNS — each pod gets a DNS entry in the format pod-name.service.namespace.svc.cluster.local that resolves to that specific pod's IP even after rescheduling. This is how distributed databases know how to reach specific nodes by stable addresses. PVC retention policy controls whether PVCs are deleted when the StatefulSet is deleted or scaled down — the default retains PVCs to prevent accidental data loss. The partition parameter acts as a canary gate for updates — you update only pods at or above the partition ordinal to validate before rolling out to all."*

---

### ⚠️ Gotchas

1. **Headless service must exist BEFORE StatefulSet pods start** — apply the Service first or DNS entries won't be created.
2. **PVC not found on reschedule with local storage** — local PVs are node-affinity-bound. Always use network storage (EBS, EFS) with StatefulSets unless pods are pinned to specific nodes.
3. **Deleting StatefulSet does NOT delete PVCs** — PVCs remain as orphaned resources. Delete manually or configure `whenDeleted: Delete`.

---

---

# ⚠️ 2.11 Pod Lifecycle — Phases, Conditions, restartPolicy

## 🟡 Intermediate

---

### Pod Phases

```
Pending     → Accepted by K8s, not yet running (scheduling, image pull, init containers)
Running     → Bound to node, at least one container running
              ⚠️  Running ≠ Ready! Pod may not be serving traffic.
Succeeded   → All containers exited 0. Will not restart. Terminal for Jobs.
Failed      → All containers terminated, at least one non-zero exit.
Unknown     → Node communication lost. Pod state cannot be determined.
```

---

### Pod Conditions (More Granular Than Phase)

```
PodScheduled:               True  → scheduler assigned a node
PodReadyToStartContainers:  True  → sandbox and network ready
Initialized:                True  → all init containers completed
ContainersReady:            True  → all containers passed readiness probes
Ready:                      True  → pod receives Service traffic
                            False → pod excluded from Service endpoints

A pod can be phase=Running but condition Ready=False
→ It is alive but NOT receiving traffic
```

---

### restartPolicy

```yaml
restartPolicy: Always     # default — restart on any exit (services)
restartPolicy: OnFailure  # restart on non-zero exit only (jobs)
restartPolicy: Never      # never restart (one-shot, debug)

# Restart backoff: 10s → 20s → 40s → 80s → ... → 5min max
# Visible as: CrashLoopBackOff in kubectl get pods
# Resets after pod runs successfully for 10 minutes
```

---

### Pod Termination Sequence

```
T+0s:  deletionTimestamp set. EndpointSlice controller removes pod from endpoints.
       kube-proxy starts propagating iptables changes (1-5s across cluster).
T+0s:  kubelet executes preStop hook simultaneously:
         lifecycle:
           preStop:
             exec:
               command: ["sleep", "5"]   ← wait for kube-proxy propagation
T+5s:  SIGTERM sent to containers. App drains in-flight requests.
T+30s: terminationGracePeriodSeconds expires → SIGKILL. Forced termination.

⚠️  preStop time IS counted inside terminationGracePeriodSeconds.
    preStop(5s) + drain(25s) must fit within grace period(30s).

RACE CONDITION FIX:
  kube-proxy propagation and kubelet SIGTERM happen simultaneously.
  Without preStop sleep: new connections arrive at terminating pod.
  With preStop sleep 5s: kube-proxy finishes before app stops accepting.
```

---

### 🎤 Short Crisp Interview Answer

> *"Pod lifecycle has five phases: Pending, Running, Succeeded, Failed, and Unknown. Critical distinction: Running phase does not mean Ready condition is true — readiness probes control the Ready condition which determines whether the pod receives traffic. restartPolicy controls container restart: Always for services, OnFailure for jobs. For graceful termination, the preStop hook runs before SIGTERM and is counted inside terminationGracePeriodSeconds — a preStop sleep of 5 seconds buys time for kube-proxy to propagate endpoint removal before the app stops accepting connections."*

---

### ⚠️ Gotchas

1. **Running ≠ Ready ≠ Serving traffic** — three separate states. Only condition Ready=True means traffic flows.
2. **CrashLoopBackOff is not a phase** — it's shown in REASON column, phase is still Running.
3. **OOMKilled** — container hit memory limit. Kernel's OOM killer terminates it. Visible in `lastState.terminated.reason: OOMKilled`.
4. **preStop counts against grace period** — preStop(30s) + drain(30s) with terminationGracePeriodSeconds=30 means app gets 0 seconds to drain.

---

---

# ⚠️ 2.12 Pod Disruption Budgets (PDB)

## 🟡 Intermediate — HIGH PRIORITY

---

### What it is in simple terms

A PodDisruptionBudget limits how many pods of an application can be **voluntarily disrupted simultaneously**. It protects availability during node drains, cluster upgrades, and maintenance.

---

### The Problem It Solves

```
WITHOUT PDB — cluster upgrade scenario:
  3 replicas across 3 nodes
  node-1 drains: evicts pod-A → 2 pods left
  node-2 drains: evicts pod-B → 1 pod left
  node-3 drains: evicts pod-C → 0 pods → OUTAGE
  All three drains can run simultaneously.

WITH PDB (minAvailable: 2):
  node-1 drains: evicts pod-A → 2 pods ✓ (at minimum)
  node-2 drain: BLOCKED — would leave only 1 pod
  Waits until pod-A replacement is Running+Ready elsewhere
  Then evicts pod-B → still 2 pods ✓ → safe maintenance
```

---

### PDB YAML

```yaml
# Form 1: minAvailable — minimum pods that MUST remain
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2           # with replicas=3: 1 disruption allowed
  selector:
    matchLabels:
      app: myapp

# Form 2: maxUnavailable — maximum pods that CAN be down
spec:
  maxUnavailable: 1         # regardless of scale: only 1 at a time

# Percentage form (scales with HPA)
spec:
  minAvailable: "50%"       # always keep at least half available

# Unhealthy pod eviction (K8s 1.27+)
spec:
  minAvailable: 2
  unhealthyPodEvictionPolicy: AlwaysAllow   # evict unhealthy pods even if at minAvailable
  # Use to unblock stuck drains caused by unhealthy pods
```

---

### PDB Operations

```bash
# Check PDB status — ALWAYS check before cluster maintenance
kubectl get pdb -n production
# NAME        MIN AVAILABLE   ALLOWED DISRUPTIONS
# myapp-pdb   2               1      ← safe to proceed
# cache-pdb   2               0      ← BLOCKED — investigate before proceeding

# What's blocking a drain?
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# error: Cannot evict pod as it would violate the pod's disruption budget.

# Check events for eviction failures
kubectl get events --field-selector reason=FailedEviction -A

# Emergency: force drain bypassing PDB (causes potential outage)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --force
```

---

### 🎤 Short Crisp Interview Answer

> *"A PodDisruptionBudget limits voluntary disruptions to an application. minAvailable sets a floor on how many pods must remain; maxUnavailable sets a ceiling on how many can be down simultaneously. When someone drains a node, Kubernetes checks all PDBs before evicting each pod — if eviction would violate the PDB it blocks and waits for the pod to be replaced elsewhere. The critical mistake is setting minAvailable equal to replicas — this creates zero allowed disruptions and permanently blocks all cluster maintenance operations."*

---

### ⚠️ Gotchas

1. **PDB only protects voluntary disruptions** — node failures, OOM kills, liveness probe restarts all bypass PDB.
2. **minAvailable = replicas → deadlock** — zero allowed disruptions. kubectl drain waits forever. Fix: always leave 1 disruption allowed.
3. **ALLOWED DISRUPTIONS = 0 blocks everything** — check `kubectl get pdb` before any maintenance window.
4. **Unhealthy pods can block maintenance** — use `unhealthyPodEvictionPolicy: AlwaysAllow` to allow eviction of unhealthy pods.

---

---

# 2.13 Horizontal Pod Autoscaler (HPA)

## 🟡 Intermediate

---

### How HPA Works Internally

```
HPA CONTROL LOOP — Every 15 seconds:

1. Query metrics-server (CPU/memory) or custom metrics API
2. Calculate desired replicas:
   desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))

   Example (scale up):
   currentReplicas=3, currentCPU=90%, targetCPU=50%
   desired = ceil(3 × 90/50) = ceil(5.4) = 6  → scale to 6

   Example (scale down):
   currentReplicas=6, currentCPU=20%, targetCPU=50%
   desired = ceil(6 × 20/50) = ceil(2.4) = 3  → scale to 3

3. Apply scale-down stabilization window (default 5min — prevents thrashing)
4. Clamp: minReplicas ≤ desired ≤ maxReplicas
5. Update Deployment/SS replica count via API server
```

---

### HPA YAML — autoscaling/v2

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  minReplicas: 2
  maxReplicas: 50

  metrics:
  # CPU utilization (% of request)
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60

  # Memory (absolute per pod)
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi

  # Custom metric from Prometheus Adapter
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"

  # External metric (SQS queue depth)
  - type: External
    external:
      metric:
        name: sqs_messages_visible
        selector:
          matchLabels:
            queue: order-processing
      target:
        type: AverageValue
        averageValue: "50"

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300     # wait 5min before scaling down
      policies:
      - type: Percent
        value: 10                          # max 10% scale-down per minute
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0       # scale up immediately
      policies:
      - type: Percent
        value: 100                         # can double per minute
        periodSeconds: 60
```

---

### 🎤 Short Crisp Interview Answer

> *"HPA automatically adjusts replica count based on metrics. It runs every 15 seconds, queries CPU or memory from metrics-server or custom metrics from Prometheus Adapter, and calculates desired replicas as current replicas times the ratio of current to target metric. Scale-up is aggressive — acts immediately. Scale-down has a 5-minute stabilization window to prevent thrashing. For HPA to work, pods must have CPU or memory requests set — HPA calculates utilization as a percentage of requests. Without requests, HPA shows unknown in TARGETS and cannot scale."*

---

### ⚠️ Gotchas

1. **HPA requires resource requests** — without CPU requests, HPA shows `<unknown>/60%` and does nothing.
2. **HPA owns the replica count** — manually scaling a Deployment overridden within 15 seconds.
3. **Scale-to-zero NOT supported** — HPA minimum is 1. Use KEDA for zero scaling.
4. **Multiple metrics use maximum** — HPA computes desired replicas for each metric independently and uses the highest result.
5. **VPA + HPA CPU conflict** — never target the same resource with both.

---

---

# 2.14 Vertical Pod Autoscaler (VPA)

## 🟡 Intermediate

---

### What it is in simple terms

VPA automatically adjusts CPU and memory **requests and limits** for containers based on actual historical usage — right-sizing pods instead of adding more of them.

---

### VPA Modes

```
Off      → Recommendations only. NO changes. Start here always.
           kubectl describe vpa myapp-vpa → see Target/LowerBound/UpperBound

Initial  → Apply recommendations only at pod CREATION.
           Running pods NOT touched. Safe for production.

Auto     → Apply continuously. Evicts pods to apply new values.
           ⚠️ DISRUPTIVE — causes pod restarts.

Recreate → Evict only when limits exceeded (conservative Auto).
```

---

### VPA YAML

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"             # start with Off, observe first
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
      controlledValues: RequestsAndLimits
    - containerName: sidecar
      mode: "Off"                 # do not modify sidecar resources
```

---

### 🎤 Short Crisp Interview Answer

> *"VPA automatically right-sizes CPU and memory requests based on actual historical usage. It has four modes: Off just provides recommendations, Initial applies only at pod creation, Auto continuously evicts pods to apply new values, and Recreate evicts only when limits are exceeded. In production most teams use Off mode first to observe recommendations, then graduate to Initial. Never run VPA in Auto mode alongside HPA targeting CPU on the same deployment — VPA changes requests which alters HPA's utilization calculation causing oscillation."*

---

### ⚠️ Gotchas

1. **VPA Auto evicts pods** — without PDB, can take down too many simultaneously.
2. **VPA + HPA CPU conflict** — combine only: VPA on memory (Off/Initial) + HPA on CPU/custom.
3. **Needs time to learn** — requires days of usage history. Don't evaluate fresh deployments.

---

---

# 2.15 KEDA — Event-Driven Autoscaling

## 🔴 Advanced

---

### What it is in simple terms

KEDA (Kubernetes Event-Driven Autoscaling) scales deployments based on **external event sources** — message queue depth, stream consumer lag, cron schedules, HTTP request rate. Its killer feature: **scale-to-zero**, which native HPA cannot do.

---

### How KEDA Works Internally

```
External Event Source          KEDA Operator            Kubernetes
(SQS, Kafka, Redis...)              │
        │                           │
        │  1. Poll metric            │
        │◄──────────────────────────┤
        │  sqs depth = 0             │
        │──────────────────────────►│  2. Scale to 0 replicas
        │                           │─────────────────────────► Deployment
        │  500 messages arrive       │                           replicas: 0
        │──────────────────────────►│  3. 500/50 = 10 pods
        │                           │─────────────────────────► Deployment
        │                           │                           replicas: 10

INTERNALS:
  KEDA creates and manages an HPA object behind the scenes.
  ScaledObject → KEDA operator → HPA with external metric source
  KEDA exposes the external metric value to the HPA.
  HPA does the actual pod scaling.
  KEDA = external metric adapter + scale-to-zero extension on top of HPA.
```

---

### ScaledObject YAML

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor        # Deployment to scale

  minReplicaCount: 0             # scale to ZERO when no events
  maxReplicaCount: 100
  pollingInterval: 15            # check every 15 seconds
  cooldownPeriod: 300            # wait 5min before scaling to 0

  triggers:
  # SQS Queue depth
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456/orders
      queueLength: "50"          # 1 pod per 50 messages
      awsRegion: us-east-1
    authenticationRef:
      name: keda-aws-creds       # IRSA via TriggerAuthentication

  # Kafka consumer group lag
  - type: kafka
    metadata:
      bootstrapServers: kafka.production:9092
      consumerGroup: order-processors
      topic: orders
      lagThreshold: "100"        # 1 pod per 100 messages lag

  # Cron — guarantee minimum capacity during business hours
  - type: cron
    metadata:
      timezone: America/New_York
      start: 0 8 * * 1-5         # 8am weekdays
      end: 0 20 * * 1-5          # 8pm weekdays
      desiredReplicas: "10"      # min 10 during business hours

---
# TriggerAuthentication — credentials for KEDA scalers
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-aws-creds
  namespace: production
spec:
  podIdentity:
    provider: aws                # use IRSA — no static credentials
```

---

### KEDA vs HPA Comparison

```
Feature                   HPA           KEDA
─────────────────────────────────────────────────────
Scale to zero             ✗             ✓
CPU/memory scaling        ✓             ✓ (via HPA underneath)
External event sources    Limited       50+ scalers
SQS queue depth           ✗             ✓
Kafka consumer lag        ✗             ✓
Cron-based scaling        ✗             ✓
Redis list length         ✗             ✓
Custom Prometheus query   With adapter  ✓ (built-in)
Replaces HPA              —             No (extends HPA)
```

---

### 🎤 Short Crisp Interview Answer

> *"KEDA extends Kubernetes autoscaling to event sources — SQS queue depth, Kafka consumer lag, Redis list length, HTTP request rate, and 50+ other scalers. Its killer feature is scale-to-zero: when there are no events, KEDA scales the deployment to zero pods saving cost. When events arrive it scales up. Internally KEDA works by creating and managing an HPA object with external metric sources — it's a layer on top of HPA not a replacement. It's the standard solution for event-driven microservices especially SQS-based workers on EKS."*

---

### ⚠️ Gotchas

1. **Scale-to-zero cold start latency** — when messages arrive at zero replicas, wait for scheduling + image pull + readiness = 30s to several minutes. Set `minReplicaCount: 1` for latency-sensitive workloads.
2. **KEDA creates HPA objects** — don't create your own HPA for the same Deployment. They'll conflict.
3. **TriggerAuthentication is namespace-scoped** — use ClusterTriggerAuthentication for cross-namespace reuse.
4. **cooldownPeriod** — the wait time before scaling from some replicas to zero. Set this generously (300s+) to avoid ping-ponging.

---

---

# 2.16 Deployment — maxSurge/maxUnavailable Tuning for Zero-Downtime

## 🔴 Advanced

---

### Zero-Downtime Scenarios

```
SCENARIO 1: High-traffic API — cannot lose any capacity
  replicas: 10, maxSurge: 2, maxUnavailable: 0
  Creates 2 new pods, waits for ready, removes 2 old. Always 10 serving.
  Cost: 20% extra capacity during update.

SCENARIO 2: Resource-constrained — no spare node capacity
  replicas: 10, maxSurge: 0, maxUnavailable: 2
  Removes 2 old first, then creates 2 new. Brief window at 80%.
  Risk: if new pods fail, only 8 running.

SCENARIO 3: Large deployment — parallel fast update
  replicas: 100, maxSurge: "20%", maxUnavailable: "20%"
  Updates 20 at a time. Fast. Always 80-100 pods serving.

SCENARIO 4: Canary via rollout pause
  kubectl set image deployment/myapp app=myapp:v2
  kubectl rollout pause deployment/myapp   # stop at 1 new pod
  # Monitor metrics — check error rates, latency
  kubectl rollout resume deployment/myapp  # if good: continue
  kubectl rollout undo deployment/myapp    # if bad: rollback
```

---

### The SIGTERM Race Condition — Critical Production Detail

```
THE RACE (happens on every pod termination):
  Step A: EndpointSlice controller removes pod IP from endpoints
          → kube-proxy starts updating iptables on ALL nodes
          → This propagation takes 1-5 seconds across a large cluster

  Step B: kubelet sends SIGTERM to pod (happens SIMULTANEOUSLY with A)

  Window: For 1-5 seconds, new connections STILL arrive at the terminating pod
          because kube-proxy hasn't finished updating iptables everywhere.
          Pod is already receiving SIGTERM — about to stop accepting!
          → Connection reset errors for clients

THE FIX — preStop sleep:
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 5"]

  With preStop sleep 5s:
  T+0:    preStop starts → sleep 5 seconds
  T+0-5s: kube-proxy propagates endpoint removal to all nodes
  T+5s:   preStop completes → SIGTERM sent NOW
  T+5s+:  By now, no new connections arriving → clean drain

SIZING terminationGracePeriodSeconds:
  Must be greater than: preStop time + app drain time
  Example:
    preStop:                    10s  (generous kube-proxy propagation)
    App drain (in-flight reqs): 45s  (depends on max request duration)
    Total needed:               55s
    terminationGracePeriodSeconds: 60  (with 5s safety buffer)

  ⚠️  preStop time IS counted inside terminationGracePeriodSeconds
  ⚠️  If preStop=30s and terminationGracePeriodSeconds=30s: app gets 0s to drain
```

---

### 🎤 Short Crisp Interview Answer

> *"For true zero-downtime deployments, set maxUnavailable=0 so capacity never drops below desired. The critical but often missed issue is the SIGTERM race condition — when Kubernetes removes a pod from service endpoints and sends SIGTERM simultaneously, kube-proxy hasn't finished propagating the endpoint removal. New connections still arrive at the terminating pod for 1-5 seconds. The fix is a preStop sleep of 5-10 seconds, which delays SIGTERM long enough for iptables rules to propagate across all nodes. terminationGracePeriodSeconds must be larger than preStop time plus application drain time."*

---

### ⚠️ Gotchas

1. **preStop is inside the grace period countdown** — common mistake: preStop=30s + drain=30s = needs 60s grace period, not 30s.
2. **Propagation time grows with cluster size** — a 10-node cluster propagates in ~1s. A 1000-node cluster can take 5-10 seconds. Size your preStop accordingly.
3. **SIGTERM handling in your app is mandatory** — the app must catch SIGTERM and drain gracefully. Default behavior in many frameworks: immediate exit. Add SIGTERM handler explicitly.

---

---

# 2.17 StatefulSet — Parallel Pod Management, Update Strategies

## 🔴 Advanced

---

### podManagementPolicy

```yaml
# Default: OrderedReady
# Scale up:   0 → ready → 1 → ready → 2
# Scale down: 2 terminated → 1 terminated → 0
# Update:     N → N-1 → ... → 0 (reverse order)
# Use for: databases, Kafka (ordering matters)
podManagementPolicy: OrderedReady

# Parallel: all pods created/deleted simultaneously
# Loses ordering guarantee — faster
# Use for: Cassandra ring (any peer can join), stateless-like stateful apps
podManagementPolicy: Parallel
```

---

### Partition-Based Canary Updates

```
StatefulSet: kafka (replicas: 5, pods 0-4)
Current image: kafka:3.4, want to upgrade to kafka:3.5

STEP 1: Set partition=4 (update only pod-4, highest ordinal)
  kubectl patch sts kafka -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":4}}}}'
  kubectl set image sts/kafka kafka=kafka:3.5

  Result: pod-0 to pod-3 stay on kafka:3.4
          pod-4 updates to kafka:3.5   ← validate this one

STEP 2: Validate pod-4 (canary)
  kubectl logs kafka-4 | grep -i error
  Check Kafka metrics: under-replicated partitions, ISR changes

STEP 3: Roll to partition=2 (pods 2,3,4 update)
  kubectl patch sts kafka -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
  pods 2,3,4: kafka:3.5   pods 0,1: kafka:3.4

STEP 4: Full rollout (partition=0)
  kubectl patch sts kafka -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
  All pods: kafka:3.5

ROLLBACK AT ANY STEP:
  kubectl set image sts/kafka kafka=kafka:3.4
  Only already-updated pods revert
  Partition controls which pods are affected

OnDelete strategy (full manual control):
  updateStrategy:
    type: OnDelete   # pods update only when YOU manually delete them
  kubectl delete pod kafka-4   # triggers update of pod-4 only
  Use for: databases where you orchestrate leader failover manually
```

---

### 🎤 Short Crisp Interview Answer

> *"StatefulSet podManagementPolicy controls ordering. OrderedReady is the default — sequential creation and deletion, each pod must be Ready before the next. Parallel creates and deletes all simultaneously, useful when ordering doesn't matter like Cassandra. For updates, the partition parameter is the canary mechanism — set partition to the highest ordinal, update one pod, validate it, then progressively lower the partition number to roll out to all. OnDelete strategy gives complete manual control — pods only get the new spec when you explicitly delete them, which is useful for databases where you want to orchestrate failover manually."*

---

---

# 2.18 Custom Workload Controllers (Writing Operators)

## 🔴 Advanced

---

### What is an Operator

An Operator = **CRD (Custom Resource Definition) + Custom Controller** that automates management of a complex application by encoding human operational knowledge into code.

```
WITHOUT OPERATOR (human SRE):
  Manually: provision DB, set up replication, handle failover,
            schedule backups, scale storage, rotate passwords

WITH OPERATOR (software SRE):
  CRD: MySQLCluster
  spec:
    replicas: 3
    version: "8.0"
    backupSchedule: "0 2 * * *"
    storage: 100Gi

  Controller automatically:
  → Creates StatefulSet with correct MySQL config
  → Sets up primary/replica replication
  → Promotes replica on primary failure
  → Creates CronJob for backups per schedule
  → Expands PVCs when storage field increases
  → Updates status.primaryPod and status.replicationLag
```

---

### Operator Maturity Levels

```
Level 1: Basic Install        → deploys the application
Level 2: Seamless Upgrades    → manages version upgrades safely
Level 3: Full Lifecycle       → backup, restore, configuration tuning
Level 4: Deep Insights        → Prometheus metrics, alerts, log analysis
Level 5: Autopilot            → auto-scales, auto-heals, auto-tunes
```

---

### Reconciler Pattern (controller-runtime / Go)

```go
func (r *MySQLClusterReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {

    // 1. Fetch the custom resource (desired state)
    cluster := &mysqlv1.MySQLCluster{}
    if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Fetch current state (StatefulSet)
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, req.NamespacedName, sts)
    stsNotFound := errors.IsNotFound(err)

    // 3. Diff and Act
    if stsNotFound {
        // Create the StatefulSet
        newSts := buildStatefulSet(cluster)
        if err := r.Create(ctx, newSts); err != nil {
            return ctrl.Result{}, err
        }
    } else if *sts.Spec.Replicas != cluster.Spec.Replicas {
        // Update replica count
        sts.Spec.Replicas = &cluster.Spec.Replicas
        if err := r.Update(ctx, sts); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 4. Update status
    cluster.Status.Phase = "Running"
    cluster.Status.ReadyReplicas = sts.Status.ReadyReplicas
    r.Status().Update(ctx, cluster)

    // 5. Requeue for periodic reconciliation
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```

---

### Popular Production Operators

```
Operator                    Manages
────────────────────────────────────────────────────────
Prometheus Operator         Prometheus + Alertmanager + Grafana
Strimzi Kafka Operator      Apache Kafka clusters
CloudNativePG               PostgreSQL clusters
Percona Operator            MySQL, MongoDB, PostgreSQL
Vault Operator (Banzai)     HashiCorp Vault clusters
ArgoCD                      GitOps CD pipeline
Cert-Manager                TLS certificates (Let's Encrypt)
External Secrets Operator   Sync secrets from Vault/AWS SM/GCP SM
```

---

### 🎤 Short Crisp Interview Answer

> *"An Operator is a custom Kubernetes controller that manages a complex application by combining a CRD — representing the application's desired state — with a controller that reconciles actual state to desired. The controller uses the same reconciliation loop pattern as built-in Kubernetes controllers: watch for changes, compare desired to actual, take action. Operators encode operational knowledge — database failover, backup scheduling, rolling upgrades — into code. Popular examples include the Prometheus Operator, Strimzi for Kafka, and cert-manager. They're built using frameworks like kubebuilder or Operator SDK which scaffold the controller-runtime boilerplate."*

---

### ⚠️ Gotchas

1. **Reconciler must be idempotent** — it can be called multiple times for the same state. Running `Create` twice should not fail or create duplicates. Always check existence before creating.
2. **Status subresource is separate from spec** — always update status via `r.Status().Update()`, not `r.Update()`. Mixing them causes RBAC issues and conflicts.
3. **Finalizers for cleanup** — add a finalizer to your CRD if the controller creates external resources (cloud LB, DNS record). Without finalizer, the CR is deleted before cleanup runs, leaving orphaned external resources.
4. **Watch breadth** — watch only the resources you own. Over-watching (e.g., all Pods in cluster) floods the work queue and degrades controller performance.

---

---

# 🏁 Category 2 — Complete Workloads Connection Map

```
═══════════════════════════════════════════════════════════════════════════
WORKLOADS HIERARCHY AND RELATIONSHIPS
═══════════════════════════════════════════════════════════════════════════

                        YOUR APPLICATION
                              │
         ┌────────────────────┼──────────────────────┐
         ▼                    ▼                       ▼
  STATELESS APPS        STATEFUL APPS           BATCH / EVENTS
         │                    │                       │
   Deployment            StatefulSet            Job / CronJob
         │                    │                  KEDA ScaledObject
    ReplicaSet           Headless Svc                 │
         │               PVC Template           (runs to completion)
        Pod ◄────────────────────────────────────────┘
         │
   ┌─────┴──────┐
   │  Init Ctr  │  (setup, wait, migrations — run before app, exit)
   │  Sidecar   │  (log ship, proxy, secrets — run alongside app)
   │  App Ctr   │  (your code)
   └────────────┘

INFRASTRUCTURE LAYER (one per node):
  DaemonSet → log collectors, node-exporter, CNI plugins, kube-proxy

AUTOSCALING:
  HPA    → scale OUT (more pods) based on CPU/memory/custom metrics
  VPA    → scale UP (bigger pods) based on usage history
  KEDA   → scale based on events (queues, streams) + scale-to-zero

PROTECTION:
  PDB    → guard against too many voluntary disruptions at once

LIFECYCLE:
  Phases:     Pending → Running → Succeeded / Failed / Unknown
  Conditions: PodScheduled → Initialized → ContainersReady → Ready
  Termination: preStop hook → SIGTERM → grace period → SIGKILL

DEPLOYMENT STRATEGY:
  RollingUpdate (maxSurge + maxUnavailable) → zero-downtime with tuning
  Recreate → full downtime, required when versions cannot coexist

STATEFULSET ORDERING:
  OrderedReady: sequential (0→1→2 up, 2→1→0 down) — databases
  Parallel:     all at once — Cassandra ring, performance-first

OPERATOR PATTERN:
  CRD + Custom Controller = Operator
  Encodes operational knowledge (backup, failover, scaling) into code

═══════════════════════════════════════════════════════════════════════════
```

---

# Quick Reference — Category 2 Cheat Sheet

| Workload | Use For | Key Property |
|----------|---------|--------------|
| **Pod** | Atomic unit | Ephemeral, shared network namespace, not self-healing alone |
| **ReplicaSet** | Replica management | Label selector ownership, rarely used directly |
| **Deployment** | Stateless apps | Rolling updates, rollbacks, revision history |
| **StatefulSet** | Databases, queues | Stable names, dedicated PVCs, ordered operations |
| **DaemonSet** | Node-level agents | One per node, auto-deploys to new nodes, bypasses scheduler |
| **Job** | Batch tasks | Run to completion, retries, restartPolicy must be OnFailure/Never |
| **CronJob** | Scheduled tasks | Creates Jobs on schedule, concurrencyPolicy controls overlap |
| **Init Container** | Prerequisites | Sequential, runs before app, must exit 0 |
| **Native Sidecar** | App augmentation | restartPolicy: Always in initContainers, lifecycle ordering |
| **HPA** | Scale out | CPU/mem/custom metrics, needs requests set, no scale-to-zero |
| **VPA** | Right-size pods | Adjusts requests, use Off mode first, conflicts with HPA on CPU |
| **KEDA** | Event-driven scale | External sources, scale-to-zero, extends HPA |
| **PDB** | Protect availability | Voluntary disruption guard, minAvailable must be < replicas |

---

## Key Numbers to Remember

| Setting | Default | Production Recommendation |
|---------|---------|--------------------------|
| terminationGracePeriodSeconds | 30s | 60s for most services |
| preStop sleep | none | 5-10s for kube-proxy propagation |
| revisionHistoryLimit | 10 | 3-5 |
| HPA sync period | 15s | 15s (don't change) |
| HPA scale-down stabilization | 300s | 300s (keep to prevent thrashing) |
| PDB minAvailable | none | replicas - 1 minimum |
| Job ttlSecondsAfterFinished | nil (never) | 86400 (24h) |

---

*Document generated for Kubernetes Interview Mastery — Category 2: Workloads*
*All 18 topics covered: 2.1 through 2.18*

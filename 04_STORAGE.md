# Kubernetes Interview Mastery
# CATEGORY 4: STORAGE

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Deep Answer → Gotchas → Interview Q&A → Connections.
> ⚠️ = High priority, frequently asked, commonly misunderstood.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 4.1 | Volumes — emptyDir, hostPath, configMap, secret | 🟢 Beginner |
| 4.2 | Persistent Volumes (PV) & Persistent Volume Claims (PVC) | 🟢 Beginner |
| 4.3 | Storage Classes | 🟢 Beginner |
| 4.4 | Dynamic provisioning — how PVC → StorageClass → PV works ⚠️ | 🟡 Intermediate |
| 4.5 | Access modes — RWO, ROX, RWX, RWOP | 🟡 Intermediate |
| 4.6 | Reclaim policies — Retain, Delete, Recycle | 🟡 Intermediate |
| 4.7 | CSI (Container Storage Interface) — how it works | 🟡 Intermediate |
| 4.8 | Volume expansion — online vs offline | 🟡 Intermediate |
| 4.9 | StatefulSet PVC retention policies | 🔴 Advanced |
| 4.10 | Volume snapshots & restore | 🔴 Advanced |
| 4.11 | CSI driver architecture (node plugin, controller plugin) | 🔴 Advanced |
| 4.12 | Local PVs — performance trade-offs, node affinity binding | 🔴 Advanced |

---

## Difficulty Legend
- 🟢 **Beginner** — Expected from ALL candidates
- 🟡 **Intermediate** — Expected from 3+ year engineers
- 🔴 **Advanced** — Differentiates senior/staff candidates
- ⚠️ **High Priority** — Frequently asked / commonly misunderstood

---

# 4.1 Volumes — emptyDir, hostPath, configMap, secret

## 🟢 Beginner

### What it is in simple terms

A Volume is a **directory accessible to containers in a pod**. Unlike a container's filesystem which is ephemeral (destroyed on container restart), volumes have their own lifetime — either tied to the pod, the node, or completely independent (persistent). Volumes solve the problem of container-local storage being temporary and isolated.

---

### Why Volumes Exist

```
THE CONTAINER STORAGE PROBLEM
═══════════════════════════════════════════════════════════════

Container filesystem (overlay/union):
  [Container writable layer]    ← new files written here
  [Image layer 3]               ← read-only
  [Image layer 2]               ← read-only
  [Image layer 1]               ← read-only

  Container restarts → writable layer is DESTROYED
  New container = fresh empty writable layer
  All data from previous run: GONE

FIVE PROBLEMS VOLUMES SOLVE:
  1. App writes logs → container crashes → logs lost
     FIX: emptyDir persists across container restarts (not pod restarts)

  2. Two containers in same pod need to share files
     FIX: emptyDir is shared across all containers in the pod

  3. App needs config files at runtime (can't bake into image)
     FIX: configMap/secret volumes mount K8s objects as files

  4. App needs access to node-level files (log collection, /proc, /sys)
     FIX: hostPath mounts node filesystem into pod

  5. App needs durable storage that survives pod restarts
     FIX: PersistentVolumeClaims (topics 4.2+)
```

---

### Volume Type 1: emptyDir

```
WHAT IT IS:
  Created empty when pod is ASSIGNED to a node.
  Lives for the LIFETIME OF THE POD — not individual containers.
  Container crash → data SURVIVES (container restarts, emptyDir intact)
  Pod deleted, evicted, or rescheduled → data GONE

STORAGE BACKENDS:
  emptyDir: {}            → backed by node disk (default)
  emptyDir:
    medium: Memory        → backed by tmpfs (RAM)
                            Faster reads/writes than disk
                            Counts against container memory limit
                            Cleared when pod terminates

USE CASES:
  ✓ Scratch space for large data transformations
  ✓ Sharing files between init containers and app containers
  ✓ Sharing logs between app container and sidecar log-shipper
  ✓ Checkpoint storage for crash recovery within a pod
  ✓ High-speed in-memory cache (medium: Memory)
```

```yaml
# emptyDir YAML example
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: scratch
      mountPath: /tmp/work        # app writes processed data here
    - name: logs
      mountPath: /var/log/app     # app writes logs here

  - name: log-shipper
    image: fluent-bit:2.2
    volumeMounts:
    - name: logs
      mountPath: /var/log/app     # sidecar reads same log directory
      readOnly: true

  volumes:
  - name: scratch
    emptyDir:
      medium: Memory              # RAM-backed tmpfs
      sizeLimit: 500Mi            # cap to prevent OOM if app writes too much

  - name: logs
    emptyDir: {}                  # disk-backed, shared between containers
```

---

### Volume Type 2: hostPath

```
WHAT IT IS:
  Mounts a file or directory from the HOST NODE's filesystem.
  Bypasses pod isolation. Direct access to node-level resources.

⚠️  SECURITY CONCERN:
  hostPath: / → container has access to ENTIRE node filesystem
  Privileged container + hostPath = near-root-on-node access
  Pod Security Standards (baseline+) restrict or forbid hostPath

LEGITIMATE USE CASES:
  ✓ Log collection agents reading /var/log/containers
  ✓ Monitoring agents reading /proc, /sys (node metrics)
  ✓ Container runtime access: /var/run/containerd/containerd.sock
  ✓ Node-local storage for single-node dev environments

NOT RECOMMENDED FOR:
  ✗ Persistent storage for stateful apps (breaks on pod reschedule)
  ✗ Sharing data between pods on different nodes
  ✗ Production databases

hostPath.type options:
  ""                → no check, create if needed
  DirectoryOrCreate → create dir if not exists (mode 0755)
  Directory         → must already exist, fail if not
  FileOrCreate      → create file if not exists
  File              → must be an existing file
  Socket            → must be a Unix socket
  CharDevice        → must be a character device file
  BlockDevice       → must be a block device file
```

```yaml
# hostPath YAML example — node-exporter DaemonSet
spec:
  containers:
  - name: node-exporter
    image: prom/node-exporter:v1.6.1
    volumeMounts:
    - name: proc
      mountPath: /host/proc
      readOnly: true
    - name: sys
      mountPath: /host/sys
      readOnly: true

  volumes:
  - name: proc
    hostPath:
      path: /proc
      type: Directory         # must already exist

  - name: sys
    hostPath:
      path: /sys
      type: Directory
```

---

### Volume Type 3: configMap

```
WHAT IT IS:
  ConfigMap keys become filenames.
  ConfigMap values become file contents.
  Mounted read-only in the container filesystem.

AUTOMATIC UPDATE:
  When ConfigMap changes → mounted files update automatically.
  Update propagation time: up to 60 seconds (kubelet sync period).
  Files updated atomically via symlink swap — no partial reads.
  ⚠️  App must explicitly re-read the file to pick up the change.
  nginx needs: nginx -s reload
  Java apps need: re-read properties file
```

```yaml
# ConfigMap definition
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: production
data:
  nginx.conf: |
    server {
      listen 8080;
      location / {
        root /usr/share/nginx/html;
      }
    }
  app.properties: |
    log.level=INFO
    cache.ttl=300
    max.connections=100

---
# Pod mounting configMap as volume
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: nginx-config-vol
      mountPath: /etc/nginx/conf.d     # all keys mounted as files here
      readOnly: true

  volumes:
  - name: nginx-config-vol
    configMap:
      name: nginx-config
      items:                            # optional: select specific keys only
      - key: nginx.conf
        path: nginx.conf                # file will be named nginx.conf
        mode: 0644                      # file permissions
```

---

### Volume Type 4: secret

```
WHAT IT IS:
  Secret keys become filenames, values become file contents.
  Mounted as tmpfs (RAM) by default — NEVER touches node disk.
  Files deleted when pod terminates.

VOLUME MOUNT vs ENV VAR:
  Volume mount:   ✓ Not visible in process env (ps aux shows nothing)
                  ✓ Can be updated (with pod restart for most apps)
                  ✓ tmpfs — never hits disk
  Env var:        ✗ Exposed in env dumps and process listings
                  ✗ Cannot update without restarting pod
                  ✓ Simpler for basic values

COMMON USE CASES:
  TLS certificates:    /etc/tls/tls.crt  + /etc/tls/tls.key
  Database passwords:  /etc/secrets/db-password
  API keys:            /etc/secrets/api-key
  SSH private keys:    /etc/ssh/id_rsa
```

```yaml
# Secret definition
apiVersion: v1
kind: Secret
metadata:
  name: tls-cert
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

---
# Pod mounting secret as volume
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: tls-vol
      mountPath: /etc/tls
      readOnly: true
    - name: db-creds
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: tls-vol
    secret:
      secretName: tls-cert
      defaultMode: 0400           # restrictive permissions — owner read-only

  - name: db-creds
    secret:
      secretName: database-credentials
      items:
      - key: password
        path: db-password         # mounts as /etc/secrets/db-password
```

---

### Volume Type 5: projected

```yaml
# Combine multiple sources into one mountPath
spec:
  containers:
  - name: app
    volumeMounts:
    - name: combined
      mountPath: /etc/app

  volumes:
  - name: combined
    projected:
      sources:
      - configMap:
          name: app-config
      - secret:
          name: app-secrets
      - serviceAccountToken:          # auto-rotated K8s service account token
          path: token
          expirationSeconds: 3600
      - downwardAPI:                  # pod metadata as files
          items:
          - path: pod-name
            fieldRef:
              fieldPath: metadata.name
          - path: memory-limit
            resourceFieldRef:
              containerName: app
              resource: limits.memory
```

---

### Volume Comparison Table

```
Volume Type    Lifetime        Shared    Persistent    Use For
─────────────────────────────────────────────────────────────────
emptyDir       Pod lifetime    Yes       No            Scratch, inter-container sharing
hostPath       Node lifetime   No*       Node-local    DaemonSet access to node resources
configMap      ConfigMap obj   Yes       K8s object    Non-sensitive config as files
secret         Pod lifetime    Yes       K8s object    Credentials, TLS, mounted as tmpfs
PVC            Independent     Depends   Yes           Databases, durable storage
projected      Sources         Yes       K8s objects   Combined sources in one mount

*hostPath data NOT shared between pods on different nodes
```

---

### 🎤 Short Crisp Interview Answer

> *"Volumes attach storage to containers within a pod. emptyDir is ephemeral scratch space — created empty when the pod starts, shared between all containers in the pod, deleted when the pod is deleted but survives individual container restarts. hostPath mounts node filesystem directories into the pod — used for DaemonSets needing node-level access like log collectors and monitoring agents, but is a security risk and should never be used for persistent storage. configMap and secret volumes mount Kubernetes objects as files — configMap for non-sensitive config, secret for credentials mounted as tmpfs. For durable storage that survives pod deletion, you need PersistentVolumeClaims."*

---

### ⚠️ Gotchas

1. **emptyDir survives container restart, NOT pod deletion** — a container crashing and restarting sees emptyDir data intact. Pod eviction, deletion, or rescheduling wipes it entirely.
2. **configMap updates propagate but apps don't auto-reload** — files update atomically within ~60 seconds, but the application must explicitly re-read the file. nginx needs a reload signal, Java apps need to re-read properties.
3. **secret as env var exposes in process list** — `kubectl exec pod -- env` shows all env vars including secret values. Volume mount is more secure — value is in tmpfs, not in process environment.
4. **hostPath is node-specific** — data written on node-1 is inaccessible to a pod rescheduled to node-2. Never use hostPath for stateful app data.

---

### Common Interview Questions

**Q: What is the difference between emptyDir and a PVC?**
> emptyDir is tied to the pod's lifetime — deleted when the pod is deleted or rescheduled. A PVC is independent of the pod — the data survives pod deletion, rescheduling, and even namespace deletion (with Retain policy). emptyDir is for temporary intra-pod sharing. PVCs are for durable storage.

**Q: Why is mounting a secret as a volume more secure than env vars?**
> Env vars are visible in process listings, environment dumps, and any tool that can inspect the process environment. Secret volumes mount as tmpfs (RAM), never touch disk, and are not in the process environment — only accessible by reading the file explicitly.

---

### Connections
- Durable storage beyond pod lifetime → **4.2 PV/PVC**
- configMap update behavior → **Category 5 (Configuration)**
- hostPath used in DaemonSets → **Category 2 (2.5 DaemonSet)**

---

---

# 4.2 Persistent Volumes (PV) & Persistent Volume Claims (PVC)

## 🟢 Beginner

### What it is in simple terms

The PV/PVC system separates **storage provisioning from storage consumption** using a two-object model. Admins create PersistentVolumes representing real storage. Developers create PersistentVolumeClaims requesting storage by size and access mode. Kubernetes binds them — the developer never needs to know which AWS EBS volume or NFS share they got.

---

### The Two-Object Model

```
PV / PVC — SEPARATION OF CONCERNS
═══════════════════════════════════════════════════════════════

STORAGE ADMIN (infrastructure team):
  Creates PersistentVolumes — actual real storage:
  PV: ebs-volume-001
    Capacity:    100Gi
    AccessModes: ReadWriteOnce
    StorageClass: gp3
    AWS EBS VolumeID: vol-0abc123def456789

DEVELOPER (application team):
  Creates PersistentVolumeClaim — storage request:
  PVC: my-database-storage
    Capacity request: 50Gi
    AccessModes: ReadWriteOnce
    StorageClass: gp3

KUBERNETES (binding controller):
  Finds PV that satisfies the PVC request.
  Binds: PV ebs-volume-001 ←→ PVC my-database-storage
  Developer's pod mounts PVC — gets the 100Gi EBS volume.

WHY THIS DESIGN:
  Developer doesn't need AWS console access.
  Admin can pre-provision pools of storage.
  PVCs are portable — same YAML works in different environments.
  StorageClasses enable automatic dynamic provisioning (topic 4.4).
```

---

### PersistentVolume YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-production-001
  labels:
    type: ssd
    environment: production
spec:
  # Storage capacity
  capacity:
    storage: 100Gi

  # Who can use this PV (covered in detail in topic 4.5)
  accessModes:
  - ReadWriteOnce          # can be mounted by ONE node at a time

  # What happens to data when PVC is deleted (topic 4.6)
  persistentVolumeReclaimPolicy: Retain

  # Which StorageClass this PV belongs to
  storageClassName: gp3

  # Volume mode: Filesystem (default) or Block
  volumeMode: Filesystem   # formats and mounts as filesystem
  # volumeMode: Block      # raw block device, app manages filesystem

  # Bind to this specific PVC only (static binding, optional)
  claimRef:
    namespace: production
    name: my-database-pvc

  # Backend — AWS EBS (in-tree, legacy)
  awsElasticBlockStore:
    volumeID: vol-0abc123def456789
    fsType: ext4

  # Backend — CSI (modern, recommended)
  # csi:
  #   driver: ebs.csi.aws.com
  #   volumeHandle: vol-0abc123def456789
  #   fsType: ext4

  # Backend — NFS
  # nfs:
  #   server: nfs.production.internal
  #   path: /exports/databases/prod-001
```

---

### PersistentVolumeClaim YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 50Gi           # minimum capacity required

  storageClassName: gp3       # which StorageClass to use
                               # "" = no dynamic provisioning (static only)
                               # omit = cluster default StorageClass

  volumeMode: Filesystem

  # Optional: bind to specific PV by label
  selector:
    matchLabels:
      type: ssd
      environment: production

---
# Pod mounting a PVC
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql    # MySQL data directory
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mysql-data        # references the PVC
      readOnly: false
```

---

### PV/PVC Lifecycle and Status

```
PV STATUS TRANSITIONS:
  Available → Bound → Released → Available or Failed

  Available:  PV exists, no PVC bound to it. Waiting for a claim.

  Bound:      PVC is bound to this PV.
              Only that PVC can use it.
              1:1 relationship — one PV per PVC, always.

  Released:   PVC was deleted, PV still exists.
              Data still intact on the underlying storage.
              Cannot be bound to a NEW PVC until manually reclaimed.
              (With Retain policy — admin must clean up claimRef)

  Failed:     Automatic reclamation failed.
              Manual intervention required.

PVC STATUS TRANSITIONS:
  Pending → Bound

  Pending:  No matching PV found yet.
            Waiting for dynamic provisioning,
            OR waiting for admin to create matching PV.
            ⚠️  Pod using a Pending PVC will NOT start!
                Pod shows: "waiting for volume binding"

  Bound:    PVC is bound to a PV. Pod can start and mount.
```

---

### kubectl PV/PVC Commands

```bash
# List PVs (cluster-scoped — no namespace flag)
kubectl get pv
# NAME            CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM
# pv-prod-001     100Gi     RWO           Retain          Bound   production/mysql-data
# pv-prod-002     50Gi      RWO           Delete          Available

# List PVCs (namespaced)
kubectl get pvc -n production
# NAME        STATUS  VOLUME       CAPACITY  ACCESS MODES  STORAGECLASS
# mysql-data  Bound   pv-prod-001  100Gi     RWO           gp3

# Describe PVC — shows binding events, capacity, storage class
kubectl describe pvc mysql-data -n production

# Debug why PVC is stuck in Pending
kubectl describe pvc mysql-data -n production | grep -A 10 Events:
# Events:
#   Warning  ProvisioningFailed  5s  failed to provision volume:
#            no matching StorageClass found

# Delete PVC (PV behavior depends on reclaim policy)
kubectl delete pvc mysql-data -n production

# Check PV after PVC deletion with Retain policy
kubectl get pv pv-prod-001
# STATUS: Released  ← data intact, admin must clean up
```

---

### 🎤 Short Crisp Interview Answer

> *"The PV/PVC system separates storage provisioning from consumption. A PersistentVolume represents actual storage — an EBS volume, NFS share, or cloud disk. A PersistentVolumeClaim is a request for storage by size and access mode. Kubernetes binds a PVC to a matching PV. The pod mounts the PVC without knowing which actual storage it got. PVs are cluster-scoped, PVCs are namespaced. Key lifecycle: PV is Available until bound to a PVC, becomes Bound, and after PVC deletion becomes Released — then based on reclaim policy either deleted, manually cleaned, or recycled. A pod with a Pending PVC will not start."*

---

### ⚠️ Gotchas

1. **PVC Pending = pod does not start** — pod stays in Pending with "waiting for volume binding". Always check PVC status first when pods won't start.
2. **1:1 PV to PVC binding** — a 1000Gi PV bound to a 10Gi PVC wastes 990Gi. That capacity is unavailable to any other PVC.
3. **Released PV cannot auto-rebind** — even with data intact, the old claimRef prevents a new PVC from binding. Admin must manually clear it.
4. **PVs are cluster-scoped, PVCs are namespaced** — a PVC in namespace A can grab a PV intended for namespace B if labels don't restrict it.

---

### Common Interview Questions

**Q: What is the difference between a PV and a PVC?**
> A PV is the actual storage resource — it represents real underlying storage like an EBS volume or NFS share, and is cluster-scoped. A PVC is a claim or request for storage — it specifies size and access mode requirements, and is namespaced. Kubernetes binds them together. The pod always mounts a PVC, never a PV directly.

**Q: What happens to a PV when its PVC is deleted?**
> It depends on the reclaimPolicy. With Delete, the PV and underlying storage are deleted immediately. With Retain, the PV moves to Released state — data is intact but the PV cannot be bound to a new PVC until an admin clears the claimRef field.

---

### Connections
- StorageClass enables automatic PV creation → **4.3**
- Dynamic provisioning end-to-end → **4.4**
- Access modes on PV and PVC must match → **4.5**
- Reclaim policy controls lifecycle → **4.6**

---

---

# 4.3 Storage Classes

## 🟢 Beginner

### What it is in simple terms

A StorageClass is a **Kubernetes object that defines a type of storage and the provisioner that creates it**. It's the bridge between PVC requests and actual storage creation. StorageClasses enable automatic dynamic provisioning — when a PVC is created, a StorageClass automatically provisions a new PV and real cloud storage.

---

### Why StorageClasses Exist

```
BEFORE STORAGECLASSES — MANUAL PROVISIONING
═══════════════════════════════════════════════════════════════

Admin workflow (every time a developer needs storage):
  1. Log into AWS console
  2. Create EBS volume in correct AZ
  3. Note the volume ID
  4. Write PV YAML with that exact volume ID
  5. Apply PV YAML
  6. Wait for developer's PVC to bind

  10 developers × 3 PVCs each = 30 manual cloud operations
  Slow, error-prone, doesn't scale, terrible developer experience.

WITH STORAGECLASSES — SELF-SERVICE DYNAMIC PROVISIONING:
  Admin defines StorageClass ONCE:
    provisioner: ebs.csi.aws.com
    type: gp3, iops: 3000, encrypted: true

  Developer creates PVC with storageClassName: gp3
  Kubernetes automatically:
    → calls the gp3 provisioner (EBS CSI driver)
    → provisions a new EBS gp3 volume in the right AZ
    → creates a PV object in Kubernetes
    → binds the PVC to that PV
    → pod can start with storage in ~15-30 seconds

  Zero admin involvement after initial StorageClass creation.
  Developers get storage on demand, self-service.
```

---

### StorageClass YAML — EKS Examples

```yaml
# gp3 StorageClass — recommended default for EKS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # default for PVCs
                                                          # without explicit class
provisioner: ebs.csi.aws.com           # AWS EBS CSI driver

volumeBindingMode: WaitForFirstConsumer  # ← CRITICAL for EKS
# Immediate:             provision PV when PVC is created
#                        Problem: EBS might provision in wrong AZ
# WaitForFirstConsumer:  provision PV when pod claiming PVC is scheduled
#                        EBS created in SAME AZ as scheduled pod ✓

allowVolumeExpansion: true             # allow PVC resize later

reclaimPolicy: Delete                  # delete EBS when PVC deleted
# reclaimPolicy: Retain                # keep EBS when PVC deleted

parameters:
  type: gp3                            # EBS volume type
  iops: "3000"                         # IOPS (gp3 range: 3000-16000)
  throughput: "125"                    # MB/s (gp3 range: 125-1000)
  encrypted: "true"                    # encrypt at rest with AWS KMS

---
# io2 — high IOPS for production databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2-high-iops
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"

---
# EFS — ReadWriteMany, shared across nodes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap          # use EFS Access Points
  fileSystemId: fs-0abc123def       # existing EFS filesystem ID
  directoryPerms: "700"

---
# Local SSD — high performance, node-local, manual provisioning only
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
provisioner: kubernetes.io/no-provisioner  # no dynamic provisioning
volumeBindingMode: WaitForFirstConsumer
```

---

### kubectl StorageClass Commands

```bash
# List all StorageClasses
kubectl get storageclass
# NAME              PROVISIONER             RECLAIMPOLICY  BINDINGMODE            DEFAULT
# gp2               kubernetes.io/aws-ebs   Delete         Immediate              false
# gp3 (default)     ebs.csi.aws.com         Delete         WaitForFirstConsumer   true
# io2-high-iops     ebs.csi.aws.com         Retain         WaitForFirstConsumer   false

# PVC using default StorageClass (storageClassName omitted)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-storage
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
  # storageClassName not set → uses the marked default (gp3)

# PVC with storageClassName: "" (empty string)
# → NO dynamic provisioning, must match a manually created PV

# Change default StorageClass
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass gp3 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

### 🎤 Short Crisp Interview Answer

> *"A StorageClass defines a type of storage and the provisioner that creates it. When a PVC references a StorageClass, the provisioner automatically creates a PV and actual cloud storage — eliminating manual storage provisioning. On EKS, the critical parameter is volumeBindingMode: WaitForFirstConsumer — this delays EBS volume creation until a pod is scheduled, ensuring the EBS volume is in the same AZ as the pod. Using Immediate mode on EKS can create EBS volumes in the wrong AZ, causing pods to permanently fail to attach storage."*

---

### ⚠️ Gotchas

1. **Immediate mode on EKS = AZ mismatch** — EBS volumes are AZ-specific. With Immediate binding, the volume is created in an arbitrary AZ. If the pod schedules to a different AZ, it cannot attach. Always use WaitForFirstConsumer on EKS.
2. **Two default StorageClasses cause failures** — only one can be marked default. If two have `is-default-class: "true"`, PVCs without explicit storageClassName fail with a conflict error.
3. **Cannot change StorageClass on bound PVC** — once created and bound, the StorageClass is immutable. To migrate: backup data, delete PVC, create new PVC with new class, restore.

---

### Connections
- StorageClass feeds into dynamic provisioning flow → **4.4**
- volumeBindingMode interacts with pod scheduling → **Category 1 Scheduler**
- EFS StorageClass for RWX access → **4.5**
- reclaimPolicy on StorageClass → **4.6**

---

---

# ⚠️ 4.4 Dynamic Provisioning — How PVC → StorageClass → PV Works

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Dynamic provisioning is the **automated chain** that turns a developer's PVC request into real cloud storage with zero admin involvement. Understanding the exact sequence is essential for debugging storage issues.

---

### Complete Dynamic Provisioning Flow

```
DYNAMIC PROVISIONING — STEP BY STEP
═══════════════════════════════════════════════════════════════

Developer applies:
  kubectl apply -f pvc.yaml
  (PVC: mysql-data, size: 100Gi, storageClass: gp3)

STEP 1: API Server accepts PVC
  PVC created in etcd with status: Pending

STEP 2: PV Controller watches PVCs (in kube-controller-manager)
  Sees: new PVC with storageClass: gp3, no bound PV
  Checks: does a matching PV already exist? (static binding check)
  Answer: No → trigger dynamic provisioning

STEP 3: WaitForFirstConsumer — wait for pod scheduling
  (If volumeBindingMode: WaitForFirstConsumer)
  PV Controller pauses — waits for a pod using this PVC to be scheduled.
  Scheduler picks node: worker-2 in us-east-1b
  Scheduler adds annotation to PVC:
    volume.kubernetes.io/selected-node: worker-2
  PV Controller sees annotation → proceeds with provisioning

STEP 4: External Provisioner called
  external-provisioner sidecar in ebs-csi-controller watches PVCs.
  Sees: PVC mysql-data needs provisioning in us-east-1b
  Reads StorageClass parameters: type=gp3, iops=3000, encrypted=true
  Calls CSI Controller: CreateVolume(100Gi, AZ=us-east-1b)
  CSI Controller calls: AWS API ec2.CreateVolume()
  AWS creates: EBS volume vol-0xyz789abc, 100GiB, gp3, us-east-1b

STEP 5: Provisioner creates PV object
  external-provisioner creates PV in Kubernetes:
    PV: pvc-abc123def456  (auto-named by provisioner)
      capacity: 100Gi
      accessModes: [ReadWriteOnce]
      csi.driver: ebs.csi.aws.com
      csi.volumeHandle: vol-0xyz789abc
      nodeAffinity: us-east-1b   (EBS volume is AZ-pinned)
      claimRef: mysql-data in production

STEP 6: PV Controller binds PVC to PV
  PVC status: Pending → Bound
  PVC.spec.volumeName = pvc-abc123def456

STEP 7: Kubelet mounts volume on node
  Pod scheduled to worker-2 (same AZ us-east-1b as EBS volume)
  Kubelet calls CSI Node plugin: NodeStageVolume
    → CSI calls: ec2.AttachVolume(vol-0xyz789abc, worker-2)
    → Volume appears as /dev/nvme1n1 on worker-2
  Kubelet calls CSI Node plugin: NodePublishVolume
    → Formats /dev/nvme1n1 as ext4 (first time only)
    → Mounts /dev/nvme1n1 → /var/lib/kubelet/pods/<pod-uid>/volumes/
    → Bind-mounts into container's /var/lib/mysql

STEP 8: Container starts
  App sees mounted filesystem at /var/lib/mysql
  Writes to it → data lands on EBS volume vol-0xyz789abc in us-east-1b ✓

TOTAL TIME: ~15-30 seconds
  (AWS CreateVolume ~10s + AttachVolume ~5s + format ~2s)
```

---

### Static vs Dynamic Provisioning

```
STATIC PROVISIONING:
  Admin manually creates PV with a specific EBS volume ID.
  Developer creates PVC matching the PV specs.
  Kubernetes binds them via label selector or capacity match.

  Use when:
  ✓ Pre-existing volumes that need to be imported
  ✓ Specific volume IDs already created outside Kubernetes
  ✓ Local PVs (no dynamic provisioner exists for local disks)
  ✓ Precise control over which exact storage gets used

  kubectl apply -f pv-with-specific-ebs-id.yaml
  kubectl apply -f pvc.yaml

DYNAMIC PROVISIONING:
  Developer creates PVC with storageClassName reference.
  Provisioner auto-creates PV and underlying cloud storage.
  No admin involvement after initial StorageClass setup.

  Use when:
  ✓ Normal production cloud workloads
  ✓ StatefulSet volumeClaimTemplates
  ✓ Self-service storage for dev teams
  ✓ Any EKS/GKE/AKS environment

  kubectl apply -f pvc.yaml  ← developer does this only
  # EBS volume created automatically
  # PV object created automatically
  # PVC bound automatically
```

---

### Debugging Dynamic Provisioning

```bash
# Check PVC status
kubectl get pvc mysql-data -n production
# NAME        STATUS    VOLUME  CAPACITY  ...
# mysql-data  Pending   ← problem! Not bound yet

# Get provisioning events (most useful debug tool)
kubectl describe pvc mysql-data -n production
# Events:
#   Warning  ProvisioningFailed  10s  ebs.csi.aws.com
#            failed to provision volume with StorageClass "gp3":
#            could not create volume in EC2:
#            UnauthorizedOperation: not authorized to perform ec2:CreateVolume
#   ← IAM role is missing ec2:CreateVolume permission

# Check if EBS CSI driver pods are running
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl logs -n kube-system ebs-csi-controller-xxx -c csi-provisioner

# Check VolumeAttachment objects (after binding, before pod starts)
kubectl get volumeattachment
# NAME                    ATTACHER            PV            NODE        ATTACHED
# csi-abc123...           ebs.csi.aws.com     pvc-xyz...    worker-2    false
#                                                                        ← still attaching
```

---

### 🎤 Short Crisp Interview Answer

> *"Dynamic provisioning works through a chain: developer creates a PVC referencing a StorageClass, the PV controller detects the unbound PVC and triggers the external-provisioner sidecar, which calls the CSI driver's CreateVolume — which calls the cloud API to create actual storage. The provisioner then creates a PV object in Kubernetes and the PV controller binds the PVC to it. With WaitForFirstConsumer, this is delayed until a pod claiming the PVC is scheduled — the scheduler annotates the PVC with the selected node, and the provisioner uses that to create storage in the correct AZ. This is critical for EBS on EKS because EBS volumes are AZ-specific and can only attach to instances in the same AZ."*

---

### ⚠️ Gotchas

1. **WaitForFirstConsumer is mandatory for EBS on EKS** — without it, EBS provisions in an arbitrary AZ. If the pod schedules to a different AZ, AttachVolume fails: "volume is in us-east-1a, node is in us-east-1b". This is a very common production incident.
2. **StatefulSet volumeClaimTemplates always trigger dynamic provisioning** — each pod gets its own PVC auto-provisioned from the template. 3 replicas = 3 PVCs = 3 EBS volumes created automatically.
3. **Provisioning delay affects pod startup** — EBS takes 15-30 seconds. HPA bursts create pods that wait for storage before starting. Plan for this latency in your SLO calculations.

---

### Common Interview Questions

**Q: Walk me through what happens when a developer runs kubectl apply on a PVC.**
> API Server stores the PVC in etcd with status Pending. The PV Controller in kube-controller-manager detects the PVC and checks for matching static PVs. Finding none, it signals dynamic provisioning. If WaitForFirstConsumer, it waits for a pod to be scheduled and for the scheduler to annotate the PVC with the selected node. The external-provisioner sidecar sees the PVC and calls the CSI driver's CreateVolume with the StorageClass parameters and AZ hint. The CSI driver calls the cloud API, a real volume is created, the provisioner creates a PV object, and the PV controller binds the PVC to it.

**Q: What is WaitForFirstConsumer and why is it important on EKS?**
> WaitForFirstConsumer delays PV provisioning until a pod using the PVC is actually scheduled to a node. The scheduler's node choice is annotated on the PVC, and the provisioner uses that annotation to create storage in the correct AZ. This is critical for EBS because EBS volumes are AZ-specific — a volume in us-east-1a cannot be attached to an EC2 instance in us-east-1b.

---

### Connections
- StorageClass parameters feed into this flow → **4.3**
- CSI CreateVolume is called in this flow → **4.7, 4.11**
- StatefulSet volumeClaimTemplates use dynamic provisioning → **Category 2 (2.4)**
- Pod scheduling node selection triggers WaitForFirstConsumer → **Category 1 Scheduler**

---

---

# 4.5 Access Modes — RWO, ROX, RWX, RWOP

## 🟡 Intermediate

### What They Mean

```
VOLUME ACCESS MODES — COMPLETE BREAKDOWN
═══════════════════════════════════════════════════════════════

RWO — ReadWriteOnce
  Mounted READ-WRITE by ONE NODE at a time.
  Multiple pods on the SAME NODE can all mount RWO.
  (Restriction is NODE-LEVEL, NOT pod-level)

  ⚠️  Common misconception: "only one pod can use it"
      WRONG. One NODE, but multiple pods on that node = all OK.

  Real implication: pod rescheduled to different node requires
  old node to unmount (detach), new node to attach. Causes brief
  downtime if pod evicted to different node.

  Support:  AWS EBS, GCP Persistent Disk, Azure Disk, Local PVs
            All block storage — one attachment point at a time
  Typical:  Databases, single-instance stateful applications

ROX — ReadOnlyMany
  Mounted READ-ONLY by MANY NODES simultaneously.
  No node can write to it.

  Support:  NFS, CephFS, AWS EFS (in read-only mode)
  Typical:  Shared static content, read-only config datasets,
            ML model weights served to many inference pods

RWX — ReadWriteMany
  Mounted READ-WRITE by MANY NODES simultaneously.
  Multiple pods on multiple nodes all reading and writing.

  Support:  NFS, CephFS, AWS EFS, GlusterFS
            EBS, GCP PD, Azure Disk do NOT support RWX!
  Typical:  Shared upload directories, content management,
            workloads requiring shared writable filesystem

RWOP — ReadWriteOncePod (K8s 1.27+ stable)
  Mounted READ-WRITE by EXACTLY ONE POD.
  Stricter than RWO (which allows multiple pods on same node).
  Enforced by Kubernetes admission — second pod is rejected.

  Support:  CSI drivers implementing it (AWS EBS CSI: yes)
  Typical:  Database with strict single-writer requirement
            Exclusive lock on storage at pod level
```

---

### Storage Backend Support Matrix

```
Storage              RWO    ROX    RWX    RWOP
────────────────────────────────────────────────────────────
AWS EBS              ✓      ✗      ✗      ✓
AWS EFS              ✓      ✓      ✓      ✗
GCP Persistent Disk  ✓      ✗      ✗      ✗
Azure Disk           ✓      ✗      ✗      ✗
NFS                  ✓      ✓      ✓      ✗
CephFS               ✓      ✓      ✓      ✗
CephRBD (block)      ✓      ✗      ✗      ✗
Local PV             ✓      ✗      ✗      ✓
emptyDir             ✓      ✗      ✓*     ✗
                                   (* same pod containers share)
```

---

### Real-World Scenario: RWX on EKS

```yaml
# WRONG: Using EBS (gp3) for shared storage — EBS is RWO only
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
spec:
  accessModes: [ReadWriteMany]      # ← FAILS — EBS does not support RWX
  resources:
    requests:
      storage: 100Gi
  storageClassName: gp3             # provisioner rejects with error

---
# CORRECT: Use EFS for shared storage on EKS
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
spec:
  accessModes: [ReadWriteMany]      # EFS supports RWX ✓
  resources:
    requests:
      storage: 100Gi                # EFS is elastic — size is metadata only
  storageClassName: efs-sc          # EFS CSI driver provisioner
```

---

### 🎤 Short Crisp Interview Answer

> *"Access modes define how many nodes can mount a volume and with what permissions. RWO — ReadWriteOnce — is the most common: one node read-write, which is sufficient for all databases. The critical misconception is that RWO means one pod — it actually means one node, so multiple pods on the same node can all mount it. RWX — ReadWriteMany — allows multiple nodes to mount simultaneously, needed for shared filesystems. EBS only supports RWO, EFS supports RWX. Requesting RWX from a gp3 StorageClass fails at provisioning. RWOP — ReadWriteOncePod added in 1.27 — is truly pod-exclusive, enforced by Kubernetes admission."*

---

### ⚠️ Gotchas

1. **RWO is node-level, not pod-level** — two pods on the same node can both mount RWO. Engineers assuming "one pod only" are frequently surprised.
2. **Requesting RWX from EBS StorageClass fails at provisioning** — the provisioner rejects it. Common mistake when scaling Deployments that use EBS PVCs.
3. **PVC access mode must be subset of PV access modes** — a PV advertising `[RWO, ROX]` can satisfy `[RWO]` or `[ROX]` but not `[RWX]`.

---

### Common Interview Questions

**Q: What is the difference between RWO and RWOP?**
> RWO (ReadWriteOnce) restricts to one NODE — multiple pods on the same node can all mount read-write. RWOP (ReadWriteOncePod) restricts to one POD — only a single pod cluster-wide can mount read-write, enforced by Kubernetes admission control. RWOP was added in K8s 1.27 for cases requiring true exclusive single-pod access.

**Q: A Deployment with 3 replicas uses a PVC. What access mode is needed?**
> If all 3 pods are on different nodes, you need RWX — otherwise only one pod can attach the volume. If all 3 pods happen to run on the same node, RWO works. In practice, for Deployments with multiple replicas where you want shared storage, use EFS with RWX. For stateful apps where each pod needs its own dedicated storage, use StatefulSet volumeClaimTemplates with RWO.

---

### Connections
- EFS StorageClass provides RWX — **4.3**
- StatefulSet each pod gets own RWO PVC — **Category 2 (2.4)**
- CSI driver must support the access mode — **4.7**

---

---

# 4.6 Reclaim Policies — Retain, Delete, Recycle

## 🟡 Intermediate

### What Happens When a PVC is Deleted

```
RECLAIM POLICY — THREE OUTCOMES
═══════════════════════════════════════════════════════════════

Developer runs: kubectl delete pvc mysql-data

RETAIN:
  PV status → Released
  Data:       INTACT — EBS volume still exists in AWS
  PV object:  NOT deleted — stays in Released state
  New PVC:    CANNOT bind to this PV until admin manually clears claimRef

  Admin must:
    Option A: Backup then delete
      1. Inspect data, take backup if needed
      2. kubectl delete pv pv-prod-001
      3. Delete EBS volume in AWS console (optional)

    Option B: Reuse the PV
      1. Remove .spec.claimRef from PV:
         kubectl patch pv pv-001 \
           --type='json' \
           -p='[{"op":"remove","path":"/spec/claimRef"}]'
      2. PV status → Available
      3. New PVC can now bind to it

  Use for: production databases, compliance environments
           ANY time data loss is unacceptable

DELETE (default for dynamic provisioning):
  PV object:  DELETED immediately
  EBS volume: DELETED immediately via cloud API
  Data:       GONE — no recovery possible

  Use for: dev/test environments, ephemeral workloads
           data loss on PVC deletion is intentional and acceptable
  ⚠️  This is the DEFAULT for dynamically provisioned PVs

RECYCLE (DEPRECATED — DO NOT USE):
  Action:  rm -rf /thevolume/* on the volume
  PV:      becomes Available again for new PVC
  Status:  Deprecated K8s 1.11, removed in later versions
           Replaced by dynamic provisioning (delete + provision new)
  DO NOT design new systems around Recycle.
```

---

### Setting Reclaim Policy

```yaml
# On StorageClass — applies to ALL dynamically provisioned PVs from it
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-safe
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain            # ← all PVs from this SC get Retain
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"

---
# On individual PV — for statically provisioned PVs
apiVersion: v1
kind: PersistentVolume
spec:
  persistentVolumeReclaimPolicy: Retain
```

```bash
# Change reclaim policy on an existing PV (safe to do on live PV)
kubectl patch pv pv-prod-001 \
  -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# View current reclaim policies
kubectl get pv -o custom-columns=\
  NAME:.metadata.name,\
  RECLAIM:.spec.persistentVolumeReclaimPolicy,\
  STATUS:.status.phase

# Manually release a Retained PV so it can be reused
kubectl patch pv pv-prod-001 \
  --type='json' \
  -p='[{"op":"remove","path":"/spec/claimRef"}]'
# PV status now: Available
```

---

### 🎤 Short Crisp Interview Answer

> *"Reclaim policy determines what happens to the PV and underlying storage when a PVC is deleted. Retain keeps the PV and data intact — the PV moves to Released state and requires admin intervention before it can be reused. This is the safe choice for production databases. Delete removes both the PV and the underlying cloud storage immediately — convenient for dev environments but dangerous for production data. The old Recycle policy is deprecated and removed. Production databases should always use Retain, set at the StorageClass level so all dynamically provisioned volumes inherit it automatically."*

---

### ⚠️ Gotchas

1. **Delete is the default for dynamic provisioning** — when you use a StorageClass without explicit `reclaimPolicy: Retain`, deleting the PVC permanently deletes the EBS volume and all data. Many teams discover this after a developer accidentally deletes a production PVC.
2. **Released PV cannot auto-rebind** — even with data intact, the old claimRef blocks new PVC binding. Must be manually cleared.
3. **Changing reclaimPolicy on StorageClass does not affect existing PVs** — only new PVs created after the change get the new policy. Change existing PVs explicitly with kubectl patch.

---

### Connections
- PV status transitions include Released state → **4.2**
- StorageClass reclaimPolicy setting → **4.3**
- StatefulSet PVC retention is a separate but related concept → **4.9**

---

---

# 4.7 CSI (Container Storage Interface) — How It Works

## 🟡 Intermediate

### What it is in simple terms

CSI is a **standardized interface that decouples Kubernetes from storage vendor implementations**. Before CSI, storage drivers were compiled into the Kubernetes binary — adding a new vendor required a core Kubernetes code change. CSI moved drivers out into separate containers communicating via gRPC, enabling any vendor to write a driver without touching Kubernetes code.

---

### CSI Architecture — Two Components

```
CSI DRIVER — CONTROLLER + NODE
═══════════════════════════════════════════════════════════════

CONTROLLER PLUGIN (Deployment — 1-2 replicas, leader-elected):
  Handles CLOUD-LEVEL operations:
    CreateVolume      → call AWS API: ec2.CreateVolume()
    DeleteVolume      → call AWS API: ec2.DeleteVolume()
    AttachVolume      → call AWS API: ec2.AttachVolume() to EC2 instance
    DetachVolume      → call AWS API: ec2.DetachVolume()
    CreateSnapshot    → call AWS API: ec2.CreateSnapshot()
    DeleteSnapshot    → call AWS API: ec2.DeleteSnapshot()
    ExpandVolume      → call AWS API: ec2.ModifyVolume()

  Runs on ONE node (leader-elected for HA).
  Has cloud credentials (IRSA on EKS) to call AWS APIs.
  Does NOT need to run on the specific node where volume is used.

NODE PLUGIN (DaemonSet — one per node):
  Handles NODE-LEVEL operations:
    NodeStageVolume     → format block device + mount to staging dir
    NodeUnstageVolume   → unmount from staging dir
    NodePublishVolume   → bind-mount staging dir to pod's mount path
    NodeUnpublishVolume → unmount from pod
    NodeExpandVolume    → resize filesystem (online expansion)
    NodeGetInfo         → report node topology (AZ, region)

  Runs on EVERY node.
  Needs privileged: true and host /dev access.
  Needs access to /sys, /proc, /run/udev on the node.

SIDECAR CONTAINERS (alongside CSI controller):
  external-provisioner  → watches PVCs → calls CreateVolume
  external-attacher     → watches VolumeAttachments → calls Attach/Detach
  external-snapshotter  → watches VolumeSnapshots → calls CreateSnapshot
  external-resizer      → watches PVC resize → calls ControllerExpandVolume
  node-driver-registrar → (on node) registers driver socket with kubelet
  liveness-probe        → HTTP health checks for CSI containers
```

---

### Full Volume Attach + Mount Sequence

```
COMPLETE FLOW: PVC BOUND → CONTAINER RUNNING
═══════════════════════════════════════════════════════════════

Trigger: Pod with PVC scheduled to node-3

1. AttachDetachController (in kube-controller-manager) creates:
   VolumeAttachment object in Kubernetes API

2. external-attacher sidecar watches VolumeAttachment
   Calls CSI Controller: ControllerPublishVolume(vol-0xyz789, node-3)
   CSI Controller calls: AWS ec2.AttachVolume(vol-0xyz789, i-node3-id)
   AWS attaches EBS volume → /dev/nvme1n1 appears on node-3

3. Kubelet on node-3 sees VolumeAttachment is attached
   Calls CSI Node via Unix socket: NodeStageVolume
     Input:  /dev/nvme1n1
     Action: format as ext4 if not already formatted
     Mount:  /dev/nvme1n1 → /var/lib/kubelet/plugins/ebs.csi.aws.com/
                              volume/pvc-abc123/globalmount/
     This is the STAGING directory — one per volume per node

4. Kubelet calls CSI Node: NodePublishVolume
     Input:  /var/lib/kubelet/plugins/.../globalmount/
     Action: bind-mount staging dir → pod-specific path
     Output: /var/lib/kubelet/pods/<pod-uid>/volumes/
                kubernetes.io~csi/pvc-abc123/mount

5. Kubelet bind-mounts pod path into container
     → /var/lib/mysql  (the container's volumeMount.mountPath)

6. Container starts — sees formatted filesystem at /var/lib/mysql ✓

TWO-PHASE MOUNT BENEFIT (stage then publish):
  Multiple pods on same node can each get their own bind-mount
  pointing to the same staged volume. No re-formatting needed.
  Each pod has isolated mount path. One underlying block device.
```

---

### 🎤 Short Crisp Interview Answer

> *"CSI standardized storage drivers as out-of-tree plugins communicating via gRPC, removing the need to compile drivers into Kubernetes. A CSI driver has two components: the Controller plugin which runs as a Deployment and handles cloud API operations — create, delete, attach, detach, snapshot, resize — with IRSA credentials on EKS. And the Node plugin which runs as a DaemonSet with privileged access and handles node operations — NodeStageVolume formats the block device and mounts it to a global staging directory, NodePublishVolume bind-mounts that to the pod-specific path. Kubernetes talks to the node plugin via a Unix gRPC socket registered by the node-driver-registrar sidecar."*

---

### ⚠️ Gotchas

1. **Node plugin requires privileged context** — needs `privileged: true` and host `/dev` access to format and mount block devices. This is legitimate but must be scoped to the CSI namespace via Pod Security Standards.
2. **Volume attachment is async** — the AttachVolume cloud API call is asynchronous. If the AWS API is slow, pods wait in Pending with "waiting for volume attachment". Plan for this in startup SLOs.
3. **Two-phase mount enables sharing** — the staging directory is shared across multiple containers on the same node using the same volume. Understanding Stage vs Publish helps diagnose "device busy" mount errors.

---

### Connections
- Dynamic provisioning calls CreateVolume → **4.4**
- CSI full architecture deep dive → **4.11**
- Volume snapshot calls CreateSnapshot → **4.10**
- Volume expansion calls ControllerExpandVolume → **4.8**

---

---

# 4.8 Volume Expansion — Online vs Offline

## 🟡 Intermediate

### How Volume Expansion Works

```
EXPANDING A PVC — FULL SEQUENCE
═══════════════════════════════════════════════════════════════

PREREQUISITES:
  StorageClass must have: allowVolumeExpansion: true
  CSI driver must support ControllerExpandVolume + NodeExpandVolume
  AWS EBS CSI driver: supports online expansion ✓

TRIGGER:
  kubectl patch pvc mysql-data -n production \
    -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

STEP 1: external-resizer sidecar watches PVC changes
  Sees: PVC requested size 200Gi > current 100Gi
  Calls: CSI Controller: ControllerExpandVolume(vol-0xyz789, 200Gi)
  AWS API: ec2.ModifyVolume(vol-0xyz789, size=200GiB)
  AWS:     resizes EBS volume to 200GiB (takes ~30-60 seconds)

STEP 2: PVC condition updated
  kubectl get pvc mysql-data → CONDITION: FileSystemResizePending

STEP 3a: ONLINE expansion (pod is running, volume mounted)
  Kubelet notices FileSystemResizePending condition on mounted volume
  Calls CSI Node: NodeExpandVolume(/dev/nvme1n1, 200Gi)
  CSI Node runs: resize2fs /dev/nvme1n1   (ext4 filesystem resize)
  Filesystem expanded IN PLACE — no unmounting needed
  Pod continues running — ZERO downtime ✓
  FileSystemResizePending condition cleared

STEP 3b: OFFLINE expansion (pod is not running)
  Filesystem resize deferred until next pod mount.
  NodeExpandVolume called during NodeStageVolume on next attach.
  Pod must be started to complete the filesystem resize.

STEP 4: PVC shows new size
  kubectl get pvc mysql-data
  # CAPACITY: 200Gi   STATUS: Bound   CONDITIONS: none

VERIFY INSIDE POD:
  kubectl exec -it mysql-0 -n production -- df -h /var/lib/mysql
  # Filesystem: /dev/nvme1n1  200G  45G  155G  23% /var/lib/mysql ✓
```

---

### Volume Expansion kubectl Commands

```bash
# Check allowVolumeExpansion before attempting
kubectl get storageclass gp3 \
  -o jsonpath='{.allowVolumeExpansion}'
# true ✓

# Expand via edit (interactive)
kubectl edit pvc mysql-data -n production
# Change: spec.resources.requests.storage: 100Gi → 200Gi

# Expand via patch (scripted/CI)
kubectl patch pvc mysql-data -n production \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Watch progress
kubectl get pvc mysql-data -n production -w
# NAME        STATUS  CAPACITY  CONDITIONS
# mysql-data  Bound   100Gi     Resizing
# mysql-data  Bound   100Gi     FileSystemResizePending
# mysql-data  Bound   200Gi     ← complete

# Check conditions if stuck
kubectl describe pvc mysql-data -n production | grep -A 5 Conditions:

# Verify from inside pod
kubectl exec -it mysql-0 -n production -- df -h /var/lib/mysql
```

---

### 🎤 Short Crisp Interview Answer

> *"Volume expansion requires allowVolumeExpansion: true on the StorageClass and CSI driver support. When you edit a PVC to increase its storage request, the external-resizer sidecar calls the CSI controller to resize the cloud volume via cloud API, then the node plugin runs resize2fs to expand the filesystem. Online expansion — with the pod running — is supported by the EBS CSI driver and expands the filesystem in place with zero downtime. You cannot shrink a PVC — only expansion is supported. If allowVolumeExpansion is false on the StorageClass, the PVC edit is immediately rejected by the API server's admission."*

---

### ⚠️ Gotchas

1. **Cannot shrink a PVC** — only expansion is supported. Attempting to decrease the storage request is rejected by the API server with a validation error.
2. **allowVolumeExpansion must be true on StorageClass** — if false, PVC patch is immediately rejected. Check before attempting.
3. **FileSystemResizePending requires pod restart for some CSI drivers** — if the driver only supports offline filesystem resize, the PVC shows FileSystemResizePending indefinitely until the pod restarts and remounts.

---

### Connections
- ControllerExpandVolume is a CSI controller operation → **4.7, 4.11**
- NodeExpandVolume is a CSI node operation → **4.7, 4.11**
- allowVolumeExpansion set on StorageClass → **4.3**

---

---

# 4.9 StatefulSet PVC Retention Policies

## 🔴 Advanced

### Full Configuration and Behavior

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  persistentVolumeClaimRetentionPolicy:

    whenDeleted: Retain
    # Retain (default): PVCs survive StatefulSet deletion
    # Delete:           PVCs deleted when StatefulSet is deleted

    whenScaled: Delete
    # Retain (default): PVCs survive when StatefulSet is scaled down
    # Delete:           PVCs deleted when their pod is scaled away
```

---

### Scenario Matrix — All Four Combinations

```
whenDeleted: Retain + whenScaled: Retain  (BOTH default)
  Scale 3 → 2:    PVC for pod-2 KEPT (pod gone, PVC orphaned, manual cleanup needed)
  Delete SS:      PVCs for all pods KEPT (all orphaned, manual cleanup needed)
  Risk:           Orphaned PVCs accumulate, storage costs grow

whenDeleted: Retain + whenScaled: Delete  (RECOMMENDED for most cases)
  Scale 3 → 2:    PVC for pod-2 DELETED (scale-down is intentional)
  Delete SS:      PVCs for remaining pods KEPT (SS deletion might be accidental)
  Rationale:      You explicitly chose to scale down → those pods' data expendable
                  But SS deletion could be mistake → keep data safe
  Use for:        Most production stateful applications

whenDeleted: Delete + whenScaled: Delete  (DANGER — dev/test only)
  Scale 3 → 2:    PVC for pod-2 DELETED
  Delete SS:      ALL PVCs DELETED → ALL data permanently gone
  Use for:        Non-production, truly ephemeral stateful workloads only

whenDeleted: Delete + whenScaled: Retain  (RARELY correct)
  Scale down retains PVCs, but StatefulSet deletion destroys everything
  Hard to justify — avoid in practice
```

---

### How Owner References Enable This

```
OWNER REFERENCES — THE MECHANISM
═══════════════════════════════════════════════════════════════

When Delete settings are configured, Kubernetes sets ownerReferences
on the PVC so Garbage Collector handles cascading deletion automatically.

For whenDeleted: Delete:
  PVC gets ownerReference pointing to the StatefulSet:
    PVC.ownerReferences:
    - apiVersion: apps/v1
      kind: StatefulSet
      name: mysql
      uid: <ss-uid>
    When SS is deleted → GC automatically deletes the PVC

For whenScaled: Delete:
  PVC gets ownerReference pointing to the Pod:
    PVC.ownerReferences:
    - apiVersion: v1
      kind: Pod
      name: mysql-2
      uid: <pod-2-uid>
    When pod-2 is deleted (scale-down) → GC automatically deletes PVC

No manual cleanup needed when Delete policy is used.
Kubernetes Garbage Collector handles cascading deletion.
```

---

### kubectl Commands

```bash
# View current retention policy on a StatefulSet
kubectl get statefulset mysql -n production \
  -o jsonpath='{.spec.persistentVolumeClaimRetentionPolicy}'
# {"whenDeleted":"Retain","whenScaled":"Delete"}

# Check ownerReferences on a PVC (shows who owns it)
kubectl get pvc data-mysql-2 -n production \
  -o jsonpath='{.metadata.ownerReferences}'

# PVCs after scaling down from 3 to 2 (whenScaled: Delete)
kubectl get pvc -n production -l app=mysql
# data-mysql-0  Bound  ← still exists (pod-0 running)
# data-mysql-1  Bound  ← still exists (pod-1 running)
# data-mysql-2: GONE   ← deleted because pod-2 was scaled away
```

---

### 🎤 Short Crisp Interview Answer

> *"StatefulSet PVC retention policy, stable in K8s 1.27, controls what happens to PVCs when the StatefulSet is deleted or scaled down — two knobs, whenDeleted and whenScaled, each Retain or Delete. The recommended production setting is whenDeleted: Retain because a StatefulSet deletion might be accidental, and whenScaled: Delete because scaling down is an intentional act meaning those pods' data is no longer needed. The mechanism uses Kubernetes owner references — when set to Delete, the PVC gets an ownerReference to the StatefulSet or Pod, and the Garbage Collector deletes the PVC automatically when the owner is deleted. No manual cleanup needed."*

---

### ⚠️ Gotchas

1. **Default is Retain for both** — by default, PVCs are NEVER automatically deleted. Orphaned PVCs accumulate silently as you scale down and delete StatefulSets, incurring ongoing storage costs.
2. **Feature gate requirement** — StatefulSetAutoDeletePVC feature gate must be enabled. It is enabled by default in K8s 1.27+.
3. **Owner references are set at StatefulSet creation** — changing the retention policy on an existing StatefulSet does not retroactively update ownerReferences on existing PVCs. Affects only newly created PVCs.

---

### Connections
- StatefulSet PVC lifecycle → **Category 2 (2.4, 2.10)**
- PV reclaim policy is different but related → **4.6**
- Garbage Collector uses owner references → **Category 1 (1.6 Controller Manager)**

---

---

# 4.10 Volume Snapshots & Restore

## 🔴 Advanced

### Architecture — Three Objects (Mirror of PV/PVC Pattern)

```
VOLUME SNAPSHOT CRD TRIO
═══════════════════════════════════════════════════════════════

VolumeSnapshotClass   ←→   StorageClass
  Defines which CSI driver handles snapshots
  Cluster-scoped, created by admin

VolumeSnapshot        ←→   PersistentVolumeClaim
  Developer's request to snapshot a PVC
  Namespaced, created by developer

VolumeSnapshotContent ←→   PersistentVolume
  The actual snapshot object (auto-created)
  References the cloud snapshot (AWS snap-0abc123)
  Cluster-scoped, created by the snapshotter
```

---

### Snapshot Workflow — Create, Check, Restore

```yaml
# Step 1: Admin creates VolumeSnapshotClass (once)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-vsc
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete    # Delete: remove AWS snapshot when K8s object deleted
                          # Retain: keep AWS snapshot even after K8s object deleted

---
# Step 2: Developer creates VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot-2024-01-15
  namespace: production
spec:
  volumeSnapshotClassName: ebs-vsc
  source:
    persistentVolumeClaimName: mysql-data    # PVC to snapshot

# What happens internally:
# external-snapshotter sidecar watches VolumeSnapshot objects
# Calls CSI Controller: CreateSnapshot(vol-0xyz789)
# AWS API: ec2.CreateSnapshot() → snap-0abc123def
# VolumeSnapshotContent auto-created pointing to snap-0abc123def

---
# Step 3: Restore — create new PVC from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
  storageClassName: gp3
  dataSource:                              # ← restore from snapshot
    name: mysql-snapshot-2024-01-15
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  # external-provisioner sees dataSource → calls CreateVolume from snapshot
  # AWS creates new EBS volume pre-populated with snapshot data
  # New PVC bounds to new PV — ready to use

---
# Volume Clone — create PVC directly from another PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-clone
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
  storageClassName: gp3
  dataSource:
    name: mysql-data         # source PVC (must be same namespace)
    kind: PersistentVolumeClaim
    # No apiGroup needed for PVC source
```

---

### kubectl Snapshot Commands

```bash
# Check snapshot status
kubectl get volumesnapshot -n production
# NAME                        READYTOUSE  SOURCEPVC    RESTORESIZE  AGE
# mysql-snapshot-2024-01-15   true        mysql-data   100Gi        5m
# READYTOUSE: true = snapshot is complete and restorable

# View snapshot content (cluster-scoped)
kubectl get volumesnapshotcontent
# NAME                READYTOUSE  RESTORESIZE  DELETIONPOLICY  DRIVER
# snapcontent-uuid    true        107374182400 Delete          ebs.csi.aws.com

# Describe snapshot — check for errors
kubectl describe volumesnapshot mysql-snapshot-2024-01-15 -n production
# Status:
#   Ready To Use: true
#   Snapshot Content Name: snapcontent-uuid
#   Creation Time: 2024-01-15T10:30:00Z
#   Restore Size: 100Gi

# Delete snapshot (deletes AWS snapshot if deletionPolicy: Delete)
kubectl delete volumesnapshot mysql-snapshot-2024-01-15 -n production

# List all snapshot classes
kubectl get volumesnapshotclass
```

---

### 🎤 Short Crisp Interview Answer

> *"Volume snapshots follow the same three-object pattern as PV/PVC — VolumeSnapshotClass is like StorageClass, VolumeSnapshot is the request, VolumeSnapshotContent is the actual snapshot. Creating a VolumeSnapshot triggers the external-snapshotter sidecar to call the CSI driver's CreateSnapshot, which calls the cloud API. To restore, create a new PVC with a dataSource pointing to the VolumeSnapshot — the provisioner calls CreateVolume from that snapshot, creating a pre-populated volume. This enables database backups, pre-deployment clones, and DR workflows entirely through Kubernetes APIs. You can also clone a PVC directly using another PVC as the dataSource."*

---

### ⚠️ Gotchas

1. **READYTOUSE must be true before restoring** — creating a PVC from a snapshot that is not yet ready returns an error. Always check READYTOUSE field before using snapshot as dataSource.
2. **deletionPolicy: Delete removes cloud snapshot** — deleting a VolumeSnapshot with Delete policy also deletes the underlying AWS snapshot. Use Retain if you want the cloud snapshot to outlive the K8s object.
3. **Snapshot CRDs must be installed** — volume snapshots use CRDs from the external-snapshotter project. They are NOT installed by default in a vanilla cluster. Install snapshot controller and CRDs separately.

---

### Connections
- CSI CreateSnapshot called in this flow → **4.7, 4.11**
- VolumeSnapshotClass mirrors StorageClass pattern → **4.3**
- Restore creates new PVC via dynamic provisioning → **4.4**

---

---

# 4.11 CSI Driver Architecture — Node Plugin, Controller Plugin

## 🔴 Advanced

### Deep Dive — Every Component

```
CSI DRIVER COMPLETE ARCHITECTURE
═══════════════════════════════════════════════════════════════

CONTROLLER PLUGIN POD (Deployment, ~2 replicas, leader-elected)
┌──────────────────────────────────────────────────────────────┐
│  CSI CONTROLLER CONTAINER                                    │
│    RPC interface:  CreateVolume, DeleteVolume                │
│                    ControllerPublishVolume (attach)          │
│                    ControllerUnpublishVolume (detach)        │
│                    CreateSnapshot, DeleteSnapshot            │
│                    ControllerExpandVolume (resize)           │
│    Auth:           IRSA on EKS (IAM role for SA)             │
│    Cloud calls:    AWS SDK → ec2.*, kms.*, etc.              │
│                                                              │
│  external-provisioner SIDECAR                                │
│    Watches: PVCs with storageClassName                       │
│    On new PVC: calls CreateVolume RPC                        │
│    On PVC delete: calls DeleteVolume RPC                     │
│    Creates PV objects after provisioning                     │
│                                                              │
│  external-attacher SIDECAR                                   │
│    Watches: VolumeAttachment objects                         │
│    On attach: calls ControllerPublishVolume RPC              │
│    On detach: calls ControllerUnpublishVolume RPC            │
│                                                              │
│  external-snapshotter SIDECAR                                │
│    Watches: VolumeSnapshot objects                           │
│    On create: calls CreateSnapshot RPC                       │
│    On delete: calls DeleteSnapshot RPC                       │
│                                                              │
│  external-resizer SIDECAR                                    │
│    Watches: PVC spec.resources.requests.storage changes      │
│    On increase: calls ControllerExpandVolume RPC             │
│                                                              │
│  liveness-probe SIDECAR                                      │
│    HTTP /healthz endpoint for controller container           │
└──────────────────────────────────────────────────────────────┘

NODE PLUGIN POD (DaemonSet, one per node)
┌──────────────────────────────────────────────────────────────┐
│  CSI NODE CONTAINER                                          │
│    RPC interface:  NodeStageVolume                           │
│                    NodeUnstageVolume                         │
│                    NodePublishVolume                         │
│                    NodeUnpublishVolume                       │
│                    NodeExpandVolume (online filesystem resize)│
│                    NodeGetInfo (report AZ/region topology)   │
│    Security:       privileged: true                          │
│                    hostPath: /dev, /sys, /run/udev           │
│    Syscalls:       mount, umount, mkfs, resize2fs            │
│                                                              │
│  node-driver-registrar SIDECAR                               │
│    Registers CSI driver Unix socket with kubelet             │
│    Socket path:    /var/lib/kubelet/plugins/                 │
│                    ebs.csi.aws.com/csi.sock                  │
│    Kubelet uses this socket to call NodeStageVolume etc.     │
│                                                              │
│  liveness-probe SIDECAR                                      │
│    HTTP /healthz endpoint for node container                 │
└──────────────────────────────────────────────────────────────┘

COMMUNICATION PATHS:
  kubelet                ──gRPC Unix socket──►  CSI Node plugin
  external-provisioner   ──gRPC Unix socket──►  CSI Controller plugin
  external-attacher      ──gRPC Unix socket──►  CSI Controller plugin
  CSI Controller         ──HTTPS/AWS SDK───►    AWS API endpoints
  CSI Node               ──syscalls────────►    Linux kernel
```

---

### Two-Phase Mount Detail

```
WHY TWO PHASES? NodeStageVolume + NodePublishVolume
═══════════════════════════════════════════════════════════════

PHASE 1 — NodeStageVolume (GLOBAL mount, once per volume per node):
  Input:   /dev/nvme1n1 (block device, already attached by controller)
  Action:  Format as ext4 if unformatted (first mount only)
           Mount to global staging directory:
           /var/lib/kubelet/plugins/ebs.csi.aws.com/
             volume/pvc-abc123/globalmount/
  Result:  One mounted filesystem shared at the node level

PHASE 2 — NodePublishVolume (POD-SPECIFIC bind-mount, once per pod):
  Input:   Global staging path from Phase 1
  Action:  bind-mount staging dir → pod-specific isolated path:
           /var/lib/kubelet/pods/<pod-uid>/
             volumes/kubernetes.io~csi/pvc-abc123/mount
  Result:  Pod-specific mount point, isolated namespace

KUBELET THEN:
  bind-mounts pod path → container's volumeMount.mountPath (/var/lib/mysql)

WHY THIS DESIGN:
  Multiple pods on same node can each get their own bind-mount
  to the same underlying formatted volume.
  Format happens once in Stage, not per-pod.
  If one pod is deleted, its publish-mount is removed.
  Stage-mount stays — other pods still have their bind-mounts.
  Re-publishing for a new pod doesn't re-format.
  "device busy" errors = Stage is held, prevents re-format.
```

---

### CSI Driver on EKS — IRSA Setup

```yaml
# EBS CSI Driver ServiceAccount with IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ebs-csi-controller-sa
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/EBSCSIDriverRole
    # This IAM role needs:
    # ec2:CreateVolume, ec2:DeleteVolume
    # ec2:AttachVolume, ec2:DetachVolume
    # ec2:DescribeVolumes, ec2:DescribeInstances
    # ec2:CreateSnapshot, ec2:DeleteSnapshot
    # ec2:ModifyVolume (for expansion)
    # kms:GenerateDataKeyWithoutPlaintext (for encrypted volumes)
```

```bash
# Install EBS CSI driver on EKS (recommended via EKS addon)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789:role/EBSCSIDriverRole

# Verify CSI driver pods
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-ebs-csi-driver
# ebs-csi-controller-xxx (2 replicas — Deployment)
# ebs-csi-node-xxx       (one per node — DaemonSet)

# Check CSI driver registration
kubectl get csidrivers
# NAME              ATTACHREQUIRED  PODINFOONMOUNT  STORAGECAPACITY  ...
# ebs.csi.aws.com   true            false           false
# efs.csi.aws.com   false           false           false
```

---

### 🎤 Short Crisp Interview Answer

> *"The CSI controller plugin runs as a Deployment and handles cloud API operations — create, delete, attach, detach, snapshot, resize — with sidecar containers watching Kubernetes objects to trigger those operations. external-provisioner watches PVCs, external-attacher watches VolumeAttachments, external-snapshotter watches VolumeSnapshots. The node plugin runs as a DaemonSet with privileged access and handles node operations via two phases: NodeStageVolume formats the block device and mounts it to a global staging directory shared at the node level, NodePublishVolume bind-mounts that to each pod's isolated path. Kubelet communicates with the node plugin via a Unix gRPC socket registered by the node-driver-registrar sidecar. On EKS, the controller uses IRSA for IAM permissions to call AWS APIs."*

---

### ⚠️ Gotchas

1. **IRSA permissions must be complete** — missing even one IAM permission (e.g., ec2:ModifyVolume) causes volume expansion to silently fail or EBS creation to return UnauthorizedOperation.
2. **Controller is leader-elected** — both controller replicas run but only one is active leader. Failover takes ~15 seconds. During failover, in-flight volume operations may retry.
3. **Node plugin cannot run without privileged** — if PSP or Pod Security Standards block privileged pods in kube-system, the CSI node plugin fails to start and NO volumes can mount on that node.

---

### Connections
- All CSI RPCs explained in context → **4.4, 4.7, 4.8, 4.10**
- IRSA for AWS API access → **Category 12 (EKS)**
- DaemonSet architecture → **Category 2 (2.5)**

---

---

# 4.12 Local PVs — Performance Trade-offs, Node Affinity Binding

## 🔴 Advanced

### What Local PVs Are and Why They Exist

```
LOCAL PV — DIRECT NODE DISK, NO NETWORK OVERHEAD
═══════════════════════════════════════════════════════════════

Local PV uses a disk, partition, or directory DIRECTLY on a node.
No network round-trips. Performance = raw hardware capability.

PERFORMANCE COMPARISON (approximate):
  EBS gp3 (network-attached block storage):
    Read latency:   ~1-2ms
    Write latency:  ~1-2ms
    Throughput:     125-1000 MB/s
    IOPS:           3,000-16,000

  EBS io2 (provisioned IOPS):
    Latency:        ~0.5-1ms
    IOPS:           up to 64,000

  Local NVMe SSD (directly attached):
    Latency:        ~0.05-0.1ms  (50-100 microseconds)
    Throughput:     3,500-7,000 MB/s
    IOPS:           500,000-1,000,000+

  SUMMARY:
    10-20x lower latency than EBS
    30-100x higher IOPS than EBS gp3

WHEN LOCAL PV IS THE RIGHT CHOICE:
  ✓ Kafka brokers — high throughput message log storage
    Kafka replicates across brokers. Node failure = partition
    leader re-election. Data not lost. Local NVMe = massive throughput.

  ✓ Cassandra nodes — distributed DB with tunable replication
    RF=3 means data on 3 nodes. Node failure = handled by cluster.
    Local NVMe = 100x better read/write for hot data.

  ✓ Elasticsearch hot nodes — recent index data
    Query latency = storage latency. Local NVMe speeds up search.
    Warm/cold tiers can use cheaper EBS.

  ✓ GPU training nodes — loading large datasets
    Model training bottleneck is often data loading.
    Local NVMe removes storage from the critical path.

  ✓ Redis/memcached — fast in-memory cache with persistence
    RDB/AOF snapshots benefit from local disk speed.

WHEN LOCAL PV IS WRONG:
  ✗ Single-instance PostgreSQL, MySQL standalone
    Node dies → data is inaccessible until node recovers.
    Use EBS instead — can reattach to different node.

  ✗ Any workload needing free pod scheduling
    Local PV pins the pod to a specific node permanently.

  ✗ Workloads without application-level replication
    Hardware failure = data loss. App must replicate.
```

---

### Local PV Setup YAML

```yaml
# StorageClass for local PVs
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
provisioner: kubernetes.io/no-provisioner   # no dynamic provisioning
volumeBindingMode: WaitForFirstConsumer     # bind when pod is scheduled
                                             # ensures pod goes to node with PV
reclaimPolicy: Delete

---
# PV for local NVMe on a specific node — admin creates manually
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-nvme-worker-node-1
spec:
  capacity:
    storage: 1Ti
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-nvme
  volumeMode: Filesystem      # or Block for raw device

  local:
    path: /mnt/nvme0n1/kafka  # directory on the node
    # For raw block device:
    # path: /dev/nvme0n1

  # NODE AFFINITY — MANDATORY for local PVs
  # Tells scheduler this PV only exists on worker-node-1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-1           # this PV physically exists here only

---
# PVC using local storage (StatefulSet volumeClaimTemplate)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-data-0
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 500Gi
  storageClassName: local-nvme
  volumeMode: Filesystem
```

---

### Binding Flow with WaitForFirstConsumer

```
LOCAL PV BINDING — STEP BY STEP
═══════════════════════════════════════════════════════════════

STEP 1: Admin pre-creates Local PVs on specific nodes
  local-nvme-node-1: nodeAffinity → worker-node-1  (Status: Available)
  local-nvme-node-2: nodeAffinity → worker-node-2  (Status: Available)
  local-nvme-node-3: nodeAffinity → worker-node-3  (Status: Available)

STEP 2: StatefulSet kafka created (3 replicas)
  PVCs created: kafka-data-0, kafka-data-1, kafka-data-2
  All status: Pending (WaitForFirstConsumer — waiting for pod schedule)

STEP 3: Scheduler places kafka-0 on worker-node-1
  PV Controller sees kafka-data-0 PVC annotation:
    selected-node: worker-node-1
  Binds: kafka-data-0 ←→ local-nvme-node-1
  (Only PV with nodeAffinity matching worker-node-1 qualifies)

STEP 4: kafka-0 starts on worker-node-1
  Mounts /mnt/nvme0n1/kafka directly
  Writing at ~500k IOPS to local NVMe ✓

STEP 5: kafka-1 → worker-node-2, kafka-data-1 ← local-nvme-node-2
        kafka-2 → worker-node-3, kafka-data-2 ← local-nvme-node-3

NODE FAILURE SCENARIO:
  worker-node-1 fails (hardware, OS, network issue)
  kafka-0 pod is NOT rescheduled to another node
    (local PV does not exist on any other node)
  kafka-0 stays in Pending — no matching local PV elsewhere

  Kafka handles this correctly:
    kafka-0's partitions have replicas on kafka-1 and kafka-2
    Cluster continues serving with remaining brokers
    kafka-0 rejoins when worker-node-1 recovers
  ← This is the designed trade-off for local PV
```

---

### local-static-provisioner (Automated PV Management)

```bash
# Instead of manually creating PV YAML for every disk on every node,
# deploy sig-storage/local-static-provisioner DaemonSet

# It watches a discovery directory on each node (e.g., /mnt/disks/)
# When a disk is mounted there, it automatically creates a Local PV
# When the disk is removed, it automatically deletes the Local PV

# Install via Helm
helm repo add sig-storage-local-static-provisioner \
  https://kubernetes-sigs.github.io/sig-storage-local-static-provisioner

helm install local-static-provisioner \
  sig-storage-local-static-provisioner/local-static-provisioner \
  --set "common.mountDevVolume=true" \
  --set "classes[0].name=local-nvme" \
  --set "classes[0].hostDir=/mnt/disks" \
  --set "classes[0].storageClass=true"

# Prepare disk on each node (run on node):
mkdir -p /mnt/disks/nvme0n1
mount /dev/nvme0n1 /mnt/disks/nvme0n1
# local-static-provisioner auto-discovers and creates PV
```

---

### 🎤 Short Crisp Interview Answer

> *"Local PVs use disks directly on the node — bypassing network storage for 10-20x lower latency and 30-100x higher IOPS than EBS. The trade-off is data is non-portable — if the node fails, the pod cannot reschedule to another node because the data is physically on the failed hardware. The right use case is distributed systems with their own replication — Kafka, Cassandra, Elasticsearch — where the application handles data durability across replicas, so node-local storage just needs to be fast. Local PVs require nodeAffinity telling Kubernetes exactly which PV is on which node, and volumeBindingMode: WaitForFirstConsumer to ensure the pod schedules to the node that has the PV before binding occurs."*

---

### ⚠️ Gotchas

1. **No dynamic provisioner exists** — `kubernetes.io/no-provisioner` means every Local PV must be created manually (or via local-static-provisioner DaemonSet). No self-service like EBS.
2. **Node failure = pod stuck in Pending** — unlike EBS which reattaches to a different node, local disk data is inaccessible until the original node recovers. App must tolerate this via replication.
3. **Node affinity is permanent** — once a StatefulSet pod is bound to a local PV on node-1, it always tries to schedule on node-1. To move it: manually migrate data, delete PVC, create new PVC, let StatefulSet re-provision.
4. **Disk must be pre-formatted or handled by provisioner** — Kubernetes does not format raw disks. Either mount a pre-formatted filesystem to the hostPath, or use raw Block volumeMode and let the application handle formatting.

---

### Common Interview Questions

**Q: When would you choose Local PVs over EBS on EKS?**
> For distributed stateful systems like Kafka, Cassandra, or Elasticsearch hot nodes that have their own data replication. These applications tolerate node failure via cluster-level replication, so the non-portability of local storage is acceptable — and the 10-20x latency improvement and 30-100x IOPS improvement over EBS justifies the complexity. For single-instance databases or any workload without application-level replication, EBS is always the right choice because it survives node failure.

**Q: What is nodeAffinity on a Local PV and why is it required?**
> nodeAffinity on a Local PV tells Kubernetes which specific node that PV physically exists on. Without it, the scheduler might place a pod claiming the PVC on a different node that doesn't have the disk, causing the mount to fail. The nodeAffinity constraint ensures the pod is always scheduled to the node where the local disk actually resides.

---

### Connections
- WaitForFirstConsumer binding mode → **4.3**
- StatefulSet pods pinned to nodes via local PV → **Category 2 (2.4)**
- Node affinity scheduling → **Scheduling category**

---

---

# 🏁 Category 4 — Complete Storage Map

```
STORAGE HIERARCHY AND RELATIONSHIPS
═══════════════════════════════════════════════════════════════════

POD NEEDS STORAGE
       │
       ├── EPHEMERAL (tied to pod or shorter)
       │     emptyDir          → scratch space, inter-container sharing
       │                          deleted when pod is deleted
       │     configMap volume  → non-sensitive config as files
       │                          auto-updates every ~60s
       │     secret volume     → credentials as tmpfs (never hits disk)
       │     projected volume  → combined configMap + secret + token
       │     hostPath          → node filesystem (DaemonSet use only)
       │
       └── PERSISTENT (independent of pod lifecycle)
               │
               PVC (PersistentVolumeClaim)
               ├── namespace-scoped
               ├── access mode: RWO / ROX / RWX / RWOP
               └── references StorageClass
                       │
               StorageClass
               ├── provisioner: ebs.csi.aws.com
               ├── parameters: type=gp3, iops=3000
               ├── reclaimPolicy: Delete/Retain
               └── volumeBindingMode: WaitForFirstConsumer
                       │
               DYNAMIC PROVISIONING
               (external-provisioner sidecar watches PVCs)
                       │
               CSI Controller Plugin (Deployment)
               ├── CreateVolume → AWS ec2.CreateVolume()
               ├── AttachVolume → AWS ec2.AttachVolume()
               └── CreateSnapshot → AWS ec2.CreateSnapshot()
                       │
               PV (PersistentVolume)
               ├── cluster-scoped
               ├── 1:1 bound to PVC
               └── reclaimPolicy (Retain/Delete)
                       │
               CSI Node Plugin (DaemonSet)
               ├── NodeStageVolume → format + global mount
               └── NodePublishVolume → bind-mount to pod path
                       │
               Container sees mounted filesystem ✓

STATEFULSET STORAGE:
  volumeClaimTemplates → one PVC per pod (auto-created)
  PVC retention policy:
    whenDeleted: Retain/Delete
    whenScaled:  Retain/Delete
  Mechanism: Kubernetes owner references + Garbage Collector

SNAPSHOT / BACKUP:
  VolumeSnapshot → VolumeSnapshotContent → AWS snapshot
  Restore: new PVC with dataSource → pre-populated volume

EXPANSION (online, zero downtime on EBS):
  kubectl edit PVC storage ↑
  → ControllerExpandVolume (cloud resize)
  → NodeExpandVolume (resize2fs while mounted)

LOCAL PV (high performance):
  Direct disk access, no network
  10-20x lower latency, 30-100x IOPS vs EBS
  Node affinity pins pod to node with disk
  For: Kafka, Cassandra, Elasticsearch (app handles replication)
  Not for: single-instance DBs without replication
```

---

# Quick Reference — Category 4 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **emptyDir** | Pod lifetime, shared between containers, tmpfs option, wiped on pod delete |
| **hostPath** | Node filesystem, DaemonSet use, security risk, not for stateful data |
| **configMap vol** | Config as files, auto-updates ~60s, app must re-read to pick up change |
| **secret vol** | tmpfs, never hits disk, more secure than env vars |
| **PV** | Cluster-scoped, actual storage, 1:1 with PVC |
| **PVC** | Namespaced, storage request, pod mounts PVC not PV directly |
| **StorageClass** | Provisioner config, WaitForFirstConsumer for EBS on EKS, reclaimPolicy |
| **Dynamic provisioning** | PVC → external-provisioner → CSI CreateVolume → PV → bind |
| **RWO** | One NODE read-write (not one pod!), all block storage |
| **ROX** | Many nodes read-only, NFS/EFS |
| **RWX** | Many nodes read-write, NFS/EFS only, NOT EBS |
| **RWOP** | One pod only, K8s 1.27+, EBS CSI supports it |
| **Retain** | PV and data survive PVC deletion, manual cleanup, use for production |
| **Delete** | PV + cloud storage deleted with PVC, default for dynamic, dangerous |
| **CSI controller** | Deployment, cloud API ops, external-provisioner/attacher/snapshotter sidecars |
| **CSI node** | DaemonSet, privileged, NodeStageVolume (format+stage) + NodePublishVolume (bind-mount) |
| **Volume expansion** | Edit PVC storage up only, allowVolumeExpansion: true required, online on EBS |
| **Volume snapshot** | VolumeSnapshotClass + VolumeSnapshot → VolumeSnapshotContent → cloud snapshot |
| **SS PVC retention** | whenDeleted/whenScaled: Retain/Delete, owner references mechanism |
| **Local PV** | 10-20x lower latency vs EBS, nodeAffinity required, WaitForFirstConsumer required |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| configMap volume update propagation | ~60 seconds |
| Dynamic EBS provisioning time | 15-30 seconds |
| Local NVMe latency vs EBS gp3 | 10-20x lower (~0.05ms vs ~1-2ms) |
| Local NVMe IOPS vs EBS gp3 | 30-100x higher (500k+ vs 3k-16k) |
| EBS gp3 max IOPS | 16,000 |
| EBS io2 max IOPS | 64,000 |
| PV to PVC cardinality | Always 1:1 |
| RWOP minimum K8s version | 1.27 (stable) |
| StatefulSet PVC retention minimum K8s version | 1.27 (stable) |
| Volume snapshot CRD installation | Required separately (not default) |

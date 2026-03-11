# Kubernetes Interview Mastery
# CATEGORY 10: CRDs & EXTENSIBILITY

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Gotchas.
> ⚠️ = High priority, frequently asked in interviews.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 10.1 | What is a CRD — extending the Kubernetes API ⚠️ | 🟢 Beginner |
| 10.2 | CRD anatomy — spec, validation, versions, scope | 🟢 Beginner |
| 10.3 | Custom Controllers — the reconciliation loop ⚠️ | 🟡 Intermediate |
| 10.4 | Operators — CRD + Controller pattern | 🟡 Intermediate |
| 10.5 | controller-runtime & kubebuilder basics | 🟡 Intermediate |
| 10.6 | Admission Webhooks tied to CRDs | 🟡 Intermediate |
| 10.7 | CRD versioning & conversion webhooks ⚠️ | 🟡 Intermediate |
| 10.8 | Finalizers — controlling deletion lifecycle ⚠️ | 🟡 Intermediate |
| 10.9 | Owner References & Garbage Collection | 🟡 Intermediate |
| 10.10 | Operator maturity model (levels 1–5) | 🔴 Advanced |
| 10.11 | Status subresource — conditions pattern | 🔴 Advanced |
| 10.12 | GitOps with CRDs — Flux/ArgoCD managing custom resources | 🔴 Advanced |
| 10.13 | API aggregation layer vs CRDs | 🔴 Advanced |
| 10.14 | CRD performance at scale — watch cache, pagination | 🔴 Advanced |

---

# ⚠️ 10.1 What Is a CRD — Extending the Kubernetes API

## 🟢 Beginner — HIGH PRIORITY

### What it is in simple terms

A **CustomResourceDefinition (CRD)** lets you teach Kubernetes about a new type of object. Just as Kubernetes ships with built-in types (Pod, Deployment, Service), a CRD registers a new type you define — say `Database`, `CronTab`, or `KafkaCluster` — which you can then create, read, update, delete, and watch using the exact same `kubectl` and Kubernetes API machinery that handles built-in objects.

---

### Why CRDs Exist

```
THE PROBLEM WITHOUT CRDS
═══════════════════════════════════════════════════════════════

BEFORE CRDs (Kubernetes 1.7):
  You want to model a "PostgreSQL Cluster" in Kubernetes.
  Options:
    ConfigMap hack:   store cluster config as JSON blob in ConfigMap
                      → no schema, no validation, no kubectl get cluster
    Annotations:      scatter config across Pod annotations
                      → fragile, undiscoverable
    External CMS:     manage databases outside Kubernetes entirely
                      → no GitOps, no single source of truth

AFTER CRDs:
  Register "PostgresCluster" as a native Kubernetes type.
  kubectl get postgresclusters -n production         ← first class
  kubectl describe postgrescluster my-db             ← full support
  kubectl apply -f my-cluster.yaml                   ← GitOps ready
  kubectl wait postgrescluster/my-db --for=condition=Ready
  RBAC controls who can create/delete/patch clusters ← security
  Watch: get events when cluster state changes        ← reactive

CRDs ARE THE FOUNDATION OF:
  Operator pattern      (databases, message queues, ML frameworks)
  Service Mesh          (Istio VirtualService, DestinationRule)
  GitOps tools          (Argo CD Application, Flux HelmRelease)
  Monitoring            (Prometheus ServiceMonitor, PrometheusRule)
  Certificate Manager   (Certificate, Issuer, ClusterIssuer)
  Policy engines        (OPA ConstraintTemplate, CiliumNetworkPolicy)
  Cloud controllers     (AWS ACK: RDSInstance, S3Bucket, DynamoDBTable)
```

---

### How the API Server Handles CRDs

```
CRD REGISTRATION — WHAT HAPPENS INTERNALLY
═══════════════════════════════════════════════════════════════

STEP 1: You apply a CRD YAML
  kubectl apply -f postgrescluster-crd.yaml

STEP 2: API Server stores CRD in etcd
  etcd key: /registry/apiextensions.k8s.io/customresourcedefinitions/
            postgresclusters.databases.company.com

STEP 3: API Server dynamically adds new API endpoint
  GET /apis/databases.company.com/v1/postgresclusters
  GET /apis/databases.company.com/v1/namespaces/{ns}/postgresclusters
  POST, PUT, PATCH, DELETE, WATCH — all auto-generated!

STEP 4: Custom Resources are stored just like built-in objects
  etcd key: /registry/databases.company.com/postgresclusters/{ns}/{name}
  Serialized as JSON — same storage as Pods, Deployments, etc.

STEP 5: kubectl works automatically
  kubectl api-resources | grep postgres
  # NAME                SHORTNAMES  APIVERSION                    KIND
  # postgresclusters    pgc         databases.company.com/v1      PostgresCluster

  kubectl get crd postgresclusters.databases.company.com
  kubectl explain postgrescluster.spec.replicas

NO COMPILATION NEEDED:
  The API Server adds the REST endpoint dynamically.
  No restart of API Server required.
  Available ~1 second after kubectl apply.
```

---

### Simple CRD Example

```yaml
# Minimal CRD: define a new resource type "CronTab"
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com   # MUST be: plural.group
spec:
  group: stable.example.com            # API group
  names:
    kind: CronTab                      # singular CamelCase (used in YAML kind:)
    plural: crontabs                   # plural (used in URL path and kubectl)
    singular: crontab                  # optional singular alias
    shortNames:
    - ct                               # kubectl get ct
  scope: Namespaced                    # or Cluster
  versions:
  - name: v1
    served: true                       # this version is active
    storage: true                      # this version is stored in etcd
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              cronSpec:
                type: string
                pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
              image:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 10
```

```yaml
# Custom Resource (CR) — instance of the CronTab CRD
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: my-crontab
  namespace: default
spec:
  cronSpec: "*/5 * * * *"
  image: cron-worker:v1.2
  replicas: 2
```

```bash
# Interact with the new resource exactly like built-in objects
kubectl apply -f crontab-crd.yaml        # register the type
kubectl apply -f my-crontab.yaml         # create an instance
kubectl get crontabs                     # list all
kubectl get ct                           # using shortname
kubectl describe crontab my-crontab      # full details
kubectl delete crontab my-crontab        # delete
kubectl explain crontab.spec             # schema documentation
kubectl get crontabs -o yaml             # full YAML output
kubectl watch crontabs                   # watch for changes
```

---

### 🎤 Short Crisp Interview Answer

> *"A CRD registers a new API type in Kubernetes — you're extending the API Server to understand a domain concept like 'PostgresCluster' or 'KafkaConnect'. The API Server dynamically adds REST endpoints for CRUD and watch operations, stores custom resource instances in etcd alongside built-in objects, and makes them work with kubectl, RBAC, and GitOps tools exactly like Pods or Deployments. CRDs are the foundation of the Operator pattern — databases, monitoring (Prometheus ServiceMonitor), GitOps (Argo CD Application), cert-manager, and cloud controllers like AWS ACK all build on them. The key thing: a CRD is just the schema definition. Without a controller watching for instances and taking action, a CRD is just a structured way to store data."*

---

---

# 10.2 CRD Anatomy — Spec, Validation, Versions, Scope

## 🟢 Beginner

### Complete CRD YAML Reference

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.databases.company.com
  # NAMING RULE: {plural}.{group} — must match spec.group and spec.names.plural
  annotations:
    controller-gen.kubebuilder.io/version: v0.13.0  # if generated by kubebuilder

spec:
  # ── GROUP: the API group (like apps, batch, networking.k8s.io) ──
  group: databases.company.com

  # ── NAMES ────────────────────────────────────────────────────────
  names:
    kind: PostgresCluster          # CamelCase, used in YAML kind: field
    plural: postgresclusters       # lowercase, used in URLs and kubectl
    singular: postgrescluster      # optional, used in kubectl describe
    shortNames:
    - pgc                          # kubectl get pgc
    - pg
    listKind: PostgresClusterList  # optional, auto-derived from kind
    categories:
    - databases                    # kubectl get databases → shows this type

  # ── SCOPE ────────────────────────────────────────────────────────
  scope: Namespaced                # or Cluster
  # Namespaced: exists within a namespace, kubectl get pgc -n production
  # Cluster:    global, kubectl get pgc (no namespace flag)
  # Use Cluster for: nodes, PersistentVolumes, ClusterRoles, StorageClasses

  # ── VERSIONS ─────────────────────────────────────────────────────
  versions:
  - name: v1                       # version name (v1, v1beta1, v2alpha1, etc.)
    served: true                   # API Server serves this version (GET/POST/etc.)
    storage: true                  # etcd stores objects in this version
    # ONLY ONE version can have storage: true
    # All other versions: served: true, storage: false
    # Old versions: served: false, storage: false (deprecated/removed)

    # ── SCHEMA (OpenAPI v3) ──────────────────────────────────────
    schema:
      openAPIV3Schema:
        description: "PostgresCluster is the Schema for the PostgreSQL cluster"
        type: object
        # x-kubernetes-preserve-unknown-fields: true  # allow arbitrary fields (avoid!)
        required:                   # spec required at object level
        - spec
        properties:
          spec:
            type: object
            required:
            - instances             # required within spec
            - postgresVersion
            properties:
              postgresVersion:
                type: integer
                enum: [14, 15, 16]  # only these values allowed
                description: "PostgreSQL major version"
              instances:
                type: integer
                minimum: 1
                maximum: 99
                default: 1          # default value if not specified
              resources:
                type: object
                properties:
                  requests:
                    type: object
                    properties:
                      memory:
                        type: string
                        pattern: '^[0-9]+(Mi|Gi)$'
                      cpu:
                        type: string
                  limits:
                    type: object
                    properties:
                      memory:
                        type: string
                      cpu:
                        type: string
              backups:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: true
                  schedule:
                    type: string
                  retentionDays:
                    type: integer
                    minimum: 1
                    maximum: 365
              storageSize:
                type: string
                pattern: '^[0-9]+(Gi|Ti)$'
                default: "10Gi"
          status:
            type: object
            # x-kubernetes-preserve-unknown-fields: true for status is common
            properties:
              phase:
                type: string
                enum: [Creating, Running, Updating, Failed, Deleting]
              instances:
                type: integer
              conditions:
                type: array
                items:
                  type: object
                  required: [type, status]
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                      enum: ["True", "False", "Unknown"]
                    reason:
                      type: string
                    message:
                      type: string
                    lastTransitionTime:
                      type: string
                      format: date-time

    # ── STATUS SUBRESOURCE ───────────────────────────────────────
    subresources:
      status: {}
      # Enables separate status endpoint: /apis/.../postgresclusters/{n}/status
      # Controllers use PATCH /status — doesn't trigger main spec reconcile
      # kubectl status patch works, kubectl apply can't overwrite status
      scale:
        specReplicasPath: .spec.instances
        statusReplicasPath: .status.instances
        # Enables: kubectl scale postgrescluster my-db --replicas=3

    # ── PRINTER COLUMNS (kubectl get output) ────────────────────
    additionalPrinterColumns:
    - name: Version
      type: integer
      jsonPath: .spec.postgresVersion
    - name: Instances
      type: integer
      jsonPath: .spec.instances
    - name: Status
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
    # kubectl get pgc output:
    # NAME     VERSION  INSTANCES  STATUS   AGE
    # my-db    16       3          Running  5d
```

---

### Scope: Namespaced vs Cluster

```
SCOPE DECISION GUIDE
═══════════════════════════════════════════════════════════════

NAMESPACED resources (most CRDs):
  Live within a namespace
  URL: /apis/{group}/{version}/namespaces/{ns}/{plural}
  RBAC: Role + RoleBinding in that namespace
  Use when: resource belongs to a team/application
  Examples: Database, Application, Certificate, AlertRule, HelmRelease

CLUSTER-SCOPED resources (infrastructure CRDs):
  Global to the entire cluster
  URL: /apis/{group}/{version}/{plural}
  RBAC: ClusterRole + ClusterRoleBinding required
  Namespace field ignored in YAML (not allowed)
  Use when: resource represents cluster-level infrastructure
  Examples: ClusterIssuer, StorageClass, ClusterSecretStore,
            GlobalNetworkPolicy, IngressClass, RuntimeClass

MIXED PATTERN (common for operators):
  ClusterIssuer (cluster-scoped) + Issuer (namespaced) — cert-manager
  GlobalNetworkPolicy (cluster) + NetworkPolicy (namespace) — Calico
  ClusterSecretStore (cluster) + SecretStore (namespace) — External Secrets
```

---

### Validation Rules (CEL)

```yaml
# Kubernetes 1.25+: Common Expression Language (CEL) validation rules
# More powerful than simple schema constraints
versions:
- name: v1
  schema:
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            minReplicas:
              type: integer
            maxReplicas:
              type: integer
            storageSize:
              type: string
          x-kubernetes-validations:
          # CEL rule: cross-field validation
          - rule: "self.maxReplicas >= self.minReplicas"
            message: "maxReplicas must be >= minReplicas"

          - rule: "self.minReplicas > 0"
            message: "minReplicas must be positive"

          # Immutability: prevent changing postgresVersion after creation
          - rule: "self == oldSelf || oldSelf.postgresVersion == 0"
            message: "postgresVersion is immutable after creation"
            # oldSelf: previous value (available for update rules)

          # Size validation with CEL
          - rule: >
              self.storageSize.endsWith('Gi') &&
              int(self.storageSize[0:self.storageSize.size()-2]) >= 10
            message: "storageSize must be at least 10Gi"
```

---

### 🎤 Short Crisp Interview Answer

> *"A CRD has three main sections: names (kind, plural, shortNames, categories), scope (Namespaced for team resources, Cluster for infrastructure), and versions. Each version has a schema in OpenAPI v3 format — required fields, type constraints, enum values, pattern regexes, and since K8s 1.25, CEL (Common Expression Language) rules for cross-field validation. The subresources.status section is critical in production — it creates a separate /status endpoint so controllers can update status without triggering a spec reconcile loop. The additionalPrinterColumns field controls what kubectl get shows in the output columns. Only one version can have storage: true — that's the canonical version stored in etcd, and other versions are served via conversion."*

---

---

# ⚠️ 10.3 Custom Controllers — The Reconciliation Loop

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

A **custom controller** is a program (usually running as a Pod) that watches Kubernetes objects — either built-in types or custom resources — and continuously **reconciles** the actual state toward the desired state. It's the brain of the Operator pattern. Without a controller, a CRD is just a structured data store. The controller is what makes that data *mean* something.

---

### The Reconciliation Loop Mental Model

```
RECONCILIATION LOOP — THE CORE PATTERN
═══════════════════════════════════════════════════════════════

DESIRED STATE:  what the user declared (in the CRD spec)
ACTUAL STATE:   what currently exists in the cluster / external world
RECONCILE:      the act of making actual match desired

THE LOOP:
  1. WATCH: controller subscribes to changes on resources
  2. TRIGGER: a change event puts object name in work queue
  3. RECONCILE: controller reads current desired state
              compares to actual state
              takes actions to close the gap
  4. REPEAT: continues forever, re-queuing if action fails

LEVEL-TRIGGERED vs EDGE-TRIGGERED:
  Edge-triggered (bad): "act on each event"
    If event is missed → controller doesn't know something changed
    Prone to bugs on restart or dropped events

  Level-triggered (good): "given the current state, compute desired actions"
    Controller doesn't care HOW it got here — only cares WHAT IS NOW
    Restart-safe: just re-read all objects and reconcile
    Idempotent: running reconcile twice with no change = same result
    This is the Kubernetes way.

THE DECLARATIVE CONTRACT:
  User says: "I want 3 instances of PostgreSQL v16"
  Controller reads: "There are currently 2 instances of PostgreSQL v16"
  Controller acts:  "Create 1 more StatefulSet replica"
  Controller reads: "Now there are 3 instances"
  Controller acts:  "Nothing to do. Requeue in 10 minutes to re-verify."
```

---

### Controller Architecture

```
CONTROLLER INTERNAL ARCHITECTURE
═══════════════════════════════════════════════════════════════

             API Server
                │
    ┌───────────┴───────────┐
    │        LIST+WATCH      │  ← initial sync + streaming events
    │    (Informer pattern)  │
    └───────────┬───────────┘
                │
         ┌──────▼──────┐
         │ LOCAL CACHE  │  ← in-memory copy of all watched objects
         │  (Indexer)   │  ← never query API Server directly — use cache
         └──────┬──────┘
                │ event (add/update/delete)
                │
         ┌──────▼──────┐
         │  WorkQueue   │  ← rate-limited, deduplicated queue
         │  (FIFO+rate) │  ← multiple events for same object = 1 work item
         └──────┬──────┘
                │ dequeue item: namespace/name
         ┌──────▼──────┐
         │  RECONCILER  │  ← your code lives here
         │              │  1. Get desired state from cache
         │              │  2. Get actual state from API Server/external
         │              │  3. Diff and take action
         │              │  4. Update status
         │              │  5. Return: nil (done), error (retry), requeue
         └──────┬──────┘
                │
        ┌───────▼───────┐
        │ Result actions │
        │  nil    → done (requeue after default interval)
        │  error  → requeue with exponential backoff
        │  Requeue→ requeue after specific duration
        └───────────────┘

INFORMER PATTERN:
  Informer = ListWatcher + local cache + event handlers
  - First call: LIST (get all current objects, populate cache)
  - Then: WATCH (streaming, get only changes as they happen)
  - Cache: controllers read from cache (not API Server) = no thundering herd
  - Resync: periodically re-lists to catch missed events (default 10-30min)
  - SharedInformer: multiple controllers share one informer per type
                    avoids duplicate API Server LIST calls
```

---

### Writing a Controller — Core Logic

```go
// controller-runtime Reconciler interface (simplified)
// Every controller implements this single method

package controller

import (
    "context"
    "fmt"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    databasesv1 "github.com/company/operator/api/v1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
)

type PostgresClusterReconciler struct {
    client.Client   // reads/writes to Kubernetes
    // + any other dependencies (logger, recorder, external clients)
}

// Reconcile is called whenever:
//   - A PostgresCluster object is created/updated/deleted
//   - A related object (StatefulSet, Service) changes
//   - A requeue fires (every 10 min to re-verify state)
func (r *PostgresClusterReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,   // contains: namespace + name of changed object
) (ctrl.Result, error) {

    // STEP 1: Get desired state
    // Read from LOCAL CACHE (not API Server) — fast, no rate limit
    cluster := &databasesv1.PostgresCluster{}
    if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
        if errors.IsNotFound(err) {
            // Object deleted — cleanup already done by finalizer or nothing to do
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err  // requeue with backoff
    }

    // STEP 2: Handle deletion
    if !cluster.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, cluster)
    }

    // STEP 3: Add finalizer if missing
    if !containsString(cluster.Finalizers, myFinalizer) {
        cluster.Finalizers = append(cluster.Finalizers, myFinalizer)
        if err := r.Update(ctx, cluster); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }

    // STEP 4: Compare actual vs desired — reconcile StatefulSet
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      cluster.Name,
        Namespace: cluster.Namespace,
    }, sts)

    if errors.IsNotFound(err) {
        // StatefulSet doesn't exist → CREATE it
        newSTS := r.buildStatefulSet(cluster)
        if err := r.Create(ctx, newSTS); err != nil {
            return ctrl.Result{}, fmt.Errorf("creating StatefulSet: %w", err)
        }
    } else if err != nil {
        return ctrl.Result{}, err
    } else {
        // StatefulSet exists → check if update needed
        if *sts.Spec.Replicas != int32(cluster.Spec.Instances) {
            sts.Spec.Replicas = int32Ptr(cluster.Spec.Instances)
            if err := r.Update(ctx, sts); err != nil {
                return ctrl.Result{}, err
            }
        }
    }

    // STEP 5: Update status to reflect actual state
    cluster.Status.Phase = "Running"
    cluster.Status.Instances = cluster.Spec.Instances
    // Use StatusClient.Update — goes to /status subresource
    if err := r.Status().Update(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }

    // STEP 6: Return result
    return ctrl.Result{
        RequeueAfter: 10 * time.Minute,  // re-verify in 10 minutes
    }, nil
}
```

---

### Reconciliation Error Handling

```
RETURN VALUES FROM RECONCILE
═══════════════════════════════════════════════════════════════

ctrl.Result{}, nil
  → Reconcile succeeded
  → No immediate requeue
  → Controller will watch for next change event

ctrl.Result{Requeue: true}, nil
  → Requeue immediately (next cycle of the loop)
  → Use when: you made a change and want to verify it took effect

ctrl.Result{RequeueAfter: 30 * time.Second}, nil
  → Requeue after a specific duration
  → Use when: waiting for external resource to be ready
             checking on long-running operations

ctrl.Result{}, err
  → Error occurred: requeue with EXPONENTIAL BACKOFF
  → Backoff: 5ms → 10ms → 20ms → ... → 1000s cap
  → Use when: API Server returned error, external system unavailable

IDEMPOTENCY REQUIREMENT:
  Reconcile may be called MULTIPLE TIMES for the same desired state.
  (Resync, controller restart, multiple events collapsed into one)
  Every action in Reconcile must be safe to call twice:

  GOOD (idempotent):
    CreateOrUpdate: create if not exists, update if different
    Patch (not replace): only change what you own
    Status update: always sets to current state

  BAD (not idempotent):
    Append to a list every time (duplicates on re-reconcile)
    Create unconditionally (error on second call)
    Send notification email (spam on retry)
```

---

### 🎤 Short Crisp Interview Answer

> *"A custom controller implements a reconciliation loop: watch desired state, compare to actual state, take actions to close the gap. The key architectural principle is level-triggered not edge-triggered — the controller doesn't react to individual events but continuously asks 'given what exists now, what should I do?' This makes it restart-safe and resilient to missed events. Internally, controllers use the Informer pattern — list all objects once on startup, then stream changes via watch. A local cache prevents hammering the API Server. Events get deduplicated in a rate-limited work queue before hitting the Reconcile function. Reconcile returns nil for success, error for retry-with-backoff, or RequeueAfter for time-based recheck. Every action must be idempotent — reconcile will be called multiple times."*

---

---

# 10.4 Operators — CRD + Controller Pattern

## 🟡 Intermediate

### What an Operator is

```
OPERATOR = CRD + CONTROLLER + DOMAIN KNOWLEDGE
═══════════════════════════════════════════════════════════════

A Kubernetes Operator encodes the operational knowledge of
a human operator into software.

"What would a DBA do to manage a Postgres cluster?"
  - Bootstrap: initialize cluster, set up replication, create users
  - Scale: add replicas safely, run pg_rewind for resync
  - Backup: pg_basebackup, WAL archiving to S3
  - Restore: point-in-time recovery from S3
  - Upgrade: pg_upgrade, schema migrations
  - Failover: promote standby on primary failure
  - Monitor: check lag, connections, table bloat

Operator automates all of this using Kubernetes primitives.

OPERATOR COMPONENTS:
  CRD:        "PostgresCluster" — what you declare
  Controller: watches PostgresCluster objects, manages StatefulSets,
              Services, Secrets, ConfigMaps, Backups
  Webhook:    validates PostgresCluster spec before admission
  RBAC:       ClusterRole with permissions to manage its resources

REAL-WORLD OPERATORS:
  Databases:    CrunchyData Postgres Operator, MongoDB Community Op,
                MySQL Operator, Redis Operator, TiDB Operator
  Messaging:    Strimzi Kafka Operator, RabbitMQ Cluster Operator
  Search:       Elastic Cloud on Kubernetes (ECK)
  ML:           Kubeflow, MLflow Operator, Seldon
  Storage:      Rook (Ceph), Longhorn
  Monitoring:   Prometheus Operator (PrometheusRule, ServiceMonitor)
  GitOps:       Argo CD (Application), Flux (HelmRelease, GitRepository)
  Secrets:      External Secrets Operator
  Networking:   Istio, Cilium (CiliumNetworkPolicy)
  Cloud native: AWS Controllers for K8s (ACK): RDS, S3, DynamoDB
```

---

### Operator Pattern in Practice

```yaml
# What the USER declares (high-level intent):
apiVersion: postgres.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: my-production-db
  namespace: production
spec:
  postgresVersion: 16
  instances:
  - name: primary
    replicas: 1
    dataVolumeClaimSpec:
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi
  - name: replica
    replicas: 2
    dataVolumeClaimSpec:
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi
  backups:
    pgbackrest:
      repos:
      - name: repo1
        s3:
          bucket: my-pg-backups
          region: us-east-1
          endpoint: s3.amazonaws.com
  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          max_connections: "300"
          shared_buffers: "1GB"
```

```
WHAT THE OPERATOR CREATES IN RESPONSE:
  StatefulSet (primary):   1 replica, postgres:16, 100Gi EBS volume
  StatefulSet (replicas):  2 replicas, streaming replication configured
  Services:                primary-rw (read-write), replica-ro (read-only)
  ConfigMap:               patroni config, pg_hba.conf
  Secret:                  postgres passwords, replication credentials
  CronJob:                 backup schedule via pgBackRest
  ServiceAccount + RBAC:   operator permissions within namespace

USER GETS:
  Single YAML describing intent → operator handles ALL implementation details
  Day-2 operations: failover, backup, restore via CR updates
  Status reflects actual cluster state:
    kubectl describe postgrescluster my-production-db
    # Status:
    #   Phase: Running
    #   Postgres Version: 16
    #   Primary: my-production-db-primary-0
    #   Replicas: 2/2 Ready
    #   Last Backup: 2024-01-15T02:00:00Z
    #   Conditions:
    #     - type: ProxyAvailable, status: True
    #     - type: Healthy, status: True
```

---

### 🎤 Short Crisp Interview Answer

> *"An Operator is a CRD plus a controller that encodes domain-specific operational knowledge. The user declares high-level intent in a custom resource — 'I want a 3-node Postgres cluster with daily backups' — and the Operator translates that into the correct Kubernetes primitives: StatefulSets, Services, Secrets, CronJobs, all properly configured for that specific workload. The Operator also handles day-2 operations: if the primary fails, it promotes a replica. If you update the CR to 5 nodes, it adds replicas safely. Prometheus Operator is a good simple example — you declare a ServiceMonitor CR saying 'scrape this Service on port 9090' and it automatically updates Prometheus config without restarting Prometheus or manually editing YAML."*

---

---

# 10.5 controller-runtime & kubebuilder Basics

## 🟡 Intermediate

### controller-runtime

```
CONTROLLER-RUNTIME — THE FRAMEWORK
═══════════════════════════════════════════════════════════════

controller-runtime is the Go library that provides:
  Manager:    lifecycle management for multiple controllers
  Client:     read/write to Kubernetes API with caching
  Reconciler: interface every controller implements
  Builder:    fluent API to configure watches and predicates
  Webhook:    HTTP server for admission/conversion webhooks
  Metrics:    Prometheus metrics endpoint for controller health
  Leader Election: ensure only one controller instance active

All Kubernetes controllers use it:
  Kubebuilder-generated operators
  Operator SDK operators
  Many production operators (cert-manager, external-secrets, etc.)

MANAGER SETUP:
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,        // registered types
    MetricsBindAddress:     ":8080",       // prometheus metrics
    HealthProbeBindAddress: ":8081",       // /healthz + /readyz
    LeaderElection:         true,          // HA: only one active reconciler
    LeaderElectionID:       "my-operator.company.com",
    // Cache options (filter what types to watch):
    Cache: cache.Options{
        DefaultNamespaces: map[string]cache.Config{
            "production": {},  // only watch production namespace
        },
    },
})
```

---

### kubebuilder — Scaffolding Tool

```bash
# KUBEBUILDER WORKFLOW
# 1. Initialize a new operator project
kubebuilder init \
  --domain company.com \
  --repo github.com/company/postgres-operator

# 2. Create API (CRD + controller stub)
kubebuilder create api \
  --group databases \
  --version v1 \
  --kind PostgresCluster

# Creates:
# api/v1/postgrescluster_types.go    ← CRD Go struct (you fill in)
# controllers/postgrescluster_controller.go ← Reconciler (you fill in)
# config/crd/bases/...yaml           ← generated CRD YAML
# config/rbac/...yaml                ← generated RBAC

# 3. Add validation webhook
kubebuilder create webhook \
  --group databases \
  --version v1 \
  --kind PostgresCluster \
  --defaulting \     # MutatingWebhook (set defaults)
  --programmatic-validation  # ValidatingWebhook (validate)

# 4. Generate CRD YAML from Go struct
make manifests        # runs controller-gen to generate CRD YAML

# 5. Generate DeepCopy methods (required for Kubernetes objects)
make generate

# 6. Run locally (against current kubectl context)
make run              # builds and runs controller locally

# 7. Deploy to cluster
make docker-build IMG=my-operator:v1
make docker-push IMG=my-operator:v1
make deploy IMG=my-operator:v1
```

---

### Go Types to CRD Schema

```go
// api/v1/postgrescluster_types.go
// controller-gen reads these struct tags → generates CRD YAML

// +kubebuilder:object:root=true         ← register as K8s object type
// +kubebuilder:subresource:status       ← enable status subresource
// +kubebuilder:printcolumn:name="Version",type=integer,JSONPath=.spec.postgresVersion
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=.status.phase
// +kubebuilder:resource:scope=Namespaced,shortName=pgc,categories=databases
type PostgresCluster struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   PostgresClusterSpec   `json:"spec,omitempty"`
    Status PostgresClusterStatus `json:"status,omitempty"`
}

type PostgresClusterSpec struct {
    // +kubebuilder:validation:Minimum=14
    // +kubebuilder:validation:Maximum=16
    // +kubebuilder:validation:Required
    PostgresVersion int `json:"postgresVersion"`

    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=99
    // +kubebuilder:default=1
    Instances int `json:"instances,omitempty"`

    // +kubebuilder:validation:Optional
    Resources *corev1.ResourceRequirements `json:"resources,omitempty"`

    // +kubebuilder:validation:Pattern=`^\d+Gi$`
    StorageSize string `json:"storageSize,omitempty"`
}

type PostgresClusterStatus struct {
    // +kubebuilder:validation:Enum=Creating;Running;Updating;Failed;Deleting
    Phase string `json:"phase,omitempty"`

    Instances  int                  `json:"instances,omitempty"`
    Conditions []metav1.Condition   `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
type PostgresClusterList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []PostgresCluster `json:"items"`
}
```

---

### Controller Watch Configuration

```go
// In SetupWithManager — configure what the controller watches
func (r *PostgresClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        // PRIMARY RESOURCE: watch PostgresCluster objects
        For(&databasesv1.PostgresCluster{}).

        // OWNED RESOURCES: watch StatefulSets owned by PostgresCluster
        // If a StatefulSet changes → trigger reconcile of its owner
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Owns(&corev1.ConfigMap{}).
        Owns(&corev1.Secret{}).

        // WATCHES with custom handler: watch objects not owned by us
        Watches(
            &corev1.Pod{},
            handler.EnqueueRequestsFromMapFunc(r.findClustersForPod),
            builder.WithPredicates(predicate.ResourceVersionChangedPredicate{}),
        ).

        // PREDICATES: filter which events trigger reconcile
        WithEventFilter(predicate.GenerationChangedPredicate{}).
        // GenerationChangedPredicate: only trigger on spec changes
        // (not status updates or annotation changes — avoids infinite loops)

        // CONCURRENCY: how many parallel reconcile goroutines
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 5,  // 5 clusters reconciled in parallel
        }).

        Complete(r)
}

// Map a Pod to the PostgresCluster that owns it
func (r *PostgresClusterReconciler) findClustersForPod(
    ctx context.Context, pod client.Object,
) []reconcile.Request {
    // Find which PostgresCluster this pod belongs to
    // Return the cluster's namespace/name → triggers its reconcile
    clusterName, ok := pod.GetLabels()["postgres-cluster"]
    if !ok {
        return nil
    }
    return []reconcile.Request{
        {NamespacedName: types.NamespacedName{
            Name:      clusterName,
            Namespace: pod.GetNamespace(),
        }},
    }
}
```

---

### 🎤 Short Crisp Interview Answer

> *"controller-runtime provides the framework every Go-based Kubernetes controller uses: Manager (lifecycle of multiple controllers), Client (cached reads + writes to API), and the Reconciler interface with the single Reconcile method. Kubebuilder is the scaffolding tool on top — you run kubebuilder init and kubebuilder create api, fill in your Go types and reconciler logic, then make manifests generates the CRD YAML from struct tags and markers like +kubebuilder:validation:Minimum=1. The controller watches its primary resource (the CRD) plus any Owned resources — if a StatefulSet that a PostgresCluster created gets deleted externally, the controller gets notified and recreates it. GenerationChangedPredicate is critical to add — without it, every status update triggers another reconcile, which updates status, which triggers again — an infinite loop."*

---

---

# 10.6 Admission Webhooks Tied to CRDs

## 🟡 Intermediate

### Why CRDs Need Webhooks

```
WEBHOOKS FOR CRD VALIDATION AND DEFAULTING
═══════════════════════════════════════════════════════════════

SCHEMA VALIDATION (OpenAPI) CAN:
  Type checking (string, integer, boolean)
  Range constraints (minimum, maximum)
  Pattern matching (regex)
  Enum values
  Required fields
  CEL rules (cross-field within same object)

SCHEMA VALIDATION CANNOT:
  ✗ Check if a referenced object exists (does Secret "db-pass" exist?)
  ✗ Validate against external state (is this IP in our VPC CIDR?)
  ✗ Cross-object constraints (total replicas across all clusters < 100)
  ✗ Complex business rules (PostgreSQL version must be >= existing version)
  ✗ Default values based on other fields or cluster state

WEBHOOKS SOLVE THESE:
  ValidatingWebhook: reject bad requests with helpful messages
  MutatingWebhook:   set defaults, inject required fields

COMMON WEBHOOK USE CASES WITH CRDs:
  - Validate Secret reference exists
  - Inject default StorageClass if not specified
  - Prevent downgrades (can't go from Postgres 16 → Postgres 14)
  - Validate resource requests within cluster policy limits
  - Add required labels/annotations for cost tracking
  - Immutability enforcement (can't change storageSize after creation)
```

---

### Implementing Webhooks with kubebuilder

```go
// api/v1/postgrescluster_webhook.go
// Generated by: kubebuilder create webhook --defaulting --programmatic-validation

// +kubebuilder:webhook:path=/mutate-databases-company-com-v1-postgrescluster,
//   mutating=true,failurePolicy=fail,sideEffects=None,
//   groups=databases.company.com,resources=postgresclusters,
//   verbs=create;update,versions=v1,
//   name=mpostgrescluster.kb.io,admissionReviewVersions=v1

type PostgresClusterCustomDefaulter struct{}

// Default — called by MutatingWebhook
func (d *PostgresClusterCustomDefaulter) Default(
    ctx context.Context, obj runtime.Object,
) error {
    cluster := obj.(*PostgresCluster)

    // Set defaults that can't be expressed in schema
    if cluster.Spec.StorageSize == "" {
        cluster.Spec.StorageSize = "10Gi"
    }
    if cluster.Spec.Instances == 0 {
        cluster.Spec.Instances = 1
    }
    // Add required labels for cost tracking
    if cluster.Labels == nil {
        cluster.Labels = map[string]string{}
    }
    if _, ok := cluster.Labels["team"]; !ok {
        // Look up team from namespace labels
        ns := &corev1.Namespace{}
        // ... fetch namespace, copy team label
        cluster.Labels["team"] = ns.Labels["team"]
    }
    return nil
}

// +kubebuilder:webhook:path=/validate-databases-company-com-v1-postgrescluster,
//   mutating=false,failurePolicy=fail,sideEffects=None,
//   groups=databases.company.com,resources=postgresclusters,
//   verbs=create;update;delete,versions=v1,
//   name=vpostgrescluster.kb.io,admissionReviewVersions=v1

type PostgresClusterCustomValidator struct {
    Client client.Client
}

// ValidateCreate — called on POST (new object)
func (v *PostgresClusterCustomValidator) ValidateCreate(
    ctx context.Context, obj runtime.Object,
) (admission.Warnings, error) {
    cluster := obj.(*PostgresCluster)

    // Validate Secret exists
    secret := &corev1.Secret{}
    if err := v.Client.Get(ctx, client.ObjectKey{
        Name:      cluster.Spec.AdminSecretName,
        Namespace: cluster.Namespace,
    }, secret); err != nil {
        return nil, field.Invalid(
            field.NewPath("spec", "adminSecretName"),
            cluster.Spec.AdminSecretName,
            "referenced secret does not exist",
        )
    }
    return nil, nil
}

// ValidateUpdate — called on PUT/PATCH (existing object)
func (v *PostgresClusterCustomValidator) ValidateUpdate(
    ctx context.Context, oldObj, newObj runtime.Object,
) (admission.Warnings, error) {
    oldCluster := oldObj.(*PostgresCluster)
    newCluster := newObj.(*PostgresCluster)

    // Prevent PostgreSQL version downgrade
    if newCluster.Spec.PostgresVersion < oldCluster.Spec.PostgresVersion {
        return nil, field.Invalid(
            field.NewPath("spec", "postgresVersion"),
            newCluster.Spec.PostgresVersion,
            fmt.Sprintf("cannot downgrade from version %d to %d",
                oldCluster.Spec.PostgresVersion,
                newCluster.Spec.PostgresVersion),
        )
    }

    // Prevent storage size reduction (can't shrink PVCs)
    if newCluster.Spec.StorageSize < oldCluster.Spec.StorageSize {
        return nil, field.Invalid(
            field.NewPath("spec", "storageSize"),
            newCluster.Spec.StorageSize,
            "storageSize cannot be reduced",
        )
    }
    return nil, nil
}
```

---

### ⚠️ Critical: Webhook Failure and Safety

```yaml
# The failurePolicy is critical for CRD webhooks
# Misconfigured: can block ALL object creation

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: postgres-operator-validating
webhooks:
- name: vpostgrescluster.kb.io
  failurePolicy: Fail     # ← if webhook is down, all creates BLOCKED
  # failurePolicy: Ignore # ← if webhook is down, validation bypassed

  # ALWAYS exclude the operator's own namespace
  # (so operator can restart even if its webhook is down)
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values:
      - postgres-operator-system   # operator's namespace
      - kube-system

  # ALWAYS exclude kube-system
  # (cluster system pods must always be creatable)

  rules:
  - apiGroups: ["databases.company.com"]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["postgresclusters"]

  sideEffects: None        # required for dry-run support
  timeoutSeconds: 10       # fail fast if webhook is slow
  admissionReviewVersions: ["v1"]
```

---

# ⚠️ 10.7 CRD Versioning & Conversion Webhooks

## 🟡 Intermediate — HIGH PRIORITY

### Why CRD Versioning Exists

```
CRD VERSION LIFECYCLE
═══════════════════════════════════════════════════════════════

v1alpha1 → v1beta1 → v1
  Early experimental   Near stable   Stable/GA

AS A CRD EVOLVES, FIELDS CHANGE:
  v1alpha1: spec.size (integer: 1-10)
  v1beta1:  spec.instances (renamed + integer: 1-99)
  v1:       spec.instances (same) + spec.resources (new required field)

PROBLEM: Users have existing CRs in v1alpha1.
  Their YAML files reference v1alpha1 schema.
  Kubernetes still needs to serve the old API version.
  But internally you want to store in v1 format.

SOLUTION: Conversion Webhooks
  Multiple versions served to clients
  One canonical storage version in etcd (storage: true)
  Conversion webhook translates between versions on-the-fly

SERVED vs STORAGE:
  served:  true  → clients can use this version (kubectl, controllers)
  storage: true  → one version only, this is what's in etcd
  
  User sends v1alpha1 object → API Server → conversion webhook → v1 → etcd
  User reads v1alpha1 object ← API Server ← conversion webhook ← v1 ← etcd
```

---

### Multi-Version CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.databases.company.com
spec:
  group: databases.company.com
  names:
    kind: PostgresCluster
    plural: postgresclusters
  scope: Namespaced

  # CONVERSION CONFIG
  conversion:
    strategy: Webhook            # None (trivial) or Webhook (custom logic)
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          name: postgres-operator-webhook-service
          namespace: postgres-operator-system
          path: /convert           # separate from validation/mutation paths
        caBundle: <base64-ca-cert>

  versions:
  # OLD VERSION — still served but not stored
  - name: v1alpha1
    served: true               # old clients still work
    storage: false             # NOT stored in etcd (v1 is stored)
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:              # old field name
                type: integer

  # INTERMEDIATE VERSION
  - name: v1beta1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              instances:         # renamed from size
                type: integer

  # CURRENT STABLE VERSION — canonical storage
  - name: v1
    served: true
    storage: true              # ← ONLY ONE can be true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [instances, postgresVersion]
            properties:
              instances:
                type: integer
              postgresVersion:
                type: integer
              resources:         # new field added in v1
                type: object
```

---

### Conversion Webhook Implementation

```go
// Conversion webhook handler
// Called when: user requests v1alpha1, object stored as v1

func (h *ConversionHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)

    var convReview apiextensionsv1.ConversionReview
    json.Unmarshal(body, &convReview)

    convReview.Response = h.convert(convReview.Request)

    json.NewEncoder(w).Encode(convReview)
}

func (h *ConversionHandler) convert(
    req *apiextensionsv1.ConversionRequest,
) *apiextensionsv1.ConversionResponse {

    desiredVersion := req.DesiredAPIVersion  // e.g., "databases.company.com/v1"
    convertedObjects := make([]runtime.RawExtension, 0)

    for _, rawObj := range req.Objects {
        // Determine source version from object
        var objMeta metav1.TypeMeta
        json.Unmarshal(rawObj.Raw, &objMeta)

        switch {
        case objMeta.APIVersion == "databases.company.com/v1alpha1" &&
             desiredVersion == "databases.company.com/v1":
            // Convert v1alpha1 → v1
            converted := convertV1alpha1ToV1(rawObj.Raw)
            convertedObjects = append(convertedObjects,
                runtime.RawExtension{Raw: converted})

        case objMeta.APIVersion == "databases.company.com/v1" &&
             desiredVersion == "databases.company.com/v1alpha1":
            // Convert v1 → v1alpha1 (for old clients)
            converted := convertV1ToV1alpha1(rawObj.Raw)
            convertedObjects = append(convertedObjects,
                runtime.RawExtension{Raw: converted})
        }
    }

    return &apiextensionsv1.ConversionResponse{
        UID:              req.UID,
        ConvertedObjects: convertedObjects,
        Result:           metav1.Status{Status: "Success"},
    }
}

func convertV1alpha1ToV1(raw []byte) []byte {
    // Parse old format
    var old struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata"`
        Spec struct {
            Size int `json:"size"`  // old field
        } `json:"spec"`
    }
    json.Unmarshal(raw, &old)

    // Build new format
    newObj := map[string]interface{}{
        "apiVersion": "databases.company.com/v1",
        "kind":       "PostgresCluster",
        "metadata":   old.ObjectMeta,
        "spec": map[string]interface{}{
            "instances":       old.Spec.Size,  // renamed
            "postgresVersion": 14,              // default for old objects
            "resources": map[string]interface{}{  // new required field
                "requests": map[string]string{
                    "memory": "256Mi",
                    "cpu":    "250m",
                },
            },
        },
    }

    result, _ := json.Marshal(newObj)
    return result
}
```

---

### Version Deprecation Lifecycle

```
HOW TO DEPRECATE A CRD VERSION
═══════════════════════════════════════════════════════════════

PHASE 1: Introduce new version alongside old
  v1alpha1: served=true, storage=true
  v1:       served=true, storage=false
  → Users can start using v1, old clients still work

PHASE 2: Make new version canonical storage
  v1alpha1: served=true, storage=false
  v1:       served=true, storage=true
  → Kubernetes migrates existing etcd entries to v1 on read/write
  → Existing v1alpha1 CRs converted transparently

PHASE 3: Deprecation warning (K8s 1.19+)
  v1alpha1:
    served: true
    storage: false
    deprecated: true              ← sends 299 warning header
    deprecationWarning: "v1alpha1 deprecated, use v1"

PHASE 4: Disable old version
  v1alpha1:
    served: false    ← clients using v1alpha1 now get 404
    storage: false

PHASE 5: Remove old version from CRD
  Remove v1alpha1 from versions list entirely

TOOLS:
  kubectl convert: convert manifests between API versions
  kubectl convert -f my-cluster.yaml --output-version databases.company.com/v1
```

---

### 🎤 Short Crisp Interview Answer

> *"CRD versioning handles the reality that APIs evolve — fields get renamed, new required fields added, old fields removed. Multiple versions can be served simultaneously: old clients continue to use v1alpha1 while new code uses v1. Only one version is stored in etcd (storage: true). When a client requests a different version than what's stored, the API Server calls the conversion webhook to translate. The webhook receives the stored object and the desired version, and returns a converted object. For simple additive changes, hub-and-spoke conversion (all versions convert through the hub version, usually latest) keeps the code manageable. The lifecycle: add new version → move storage to it → mark old as deprecated → mark as not served → remove."*

---

---

# ⚠️ 10.8 Finalizers — Controlling Deletion Lifecycle

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

A **finalizer** is a string on a Kubernetes object's metadata that **prevents deletion** until some cleanup task completes. When you delete an object with finalizers, Kubernetes doesn't delete it immediately — instead it sets `deletionTimestamp` and waits. Your controller sees the timestamp, performs cleanup, then removes the finalizer. Only when all finalizers are removed does Kubernetes actually delete the object from etcd.

---

### Why Finalizers Exist

```
THE PROBLEM WITHOUT FINALIZERS
═══════════════════════════════════════════════════════════════

SCENARIO: PostgresCluster has:
  - A StatefulSet (owned — garbage collected automatically)
  - An AWS RDS read replica (external — NOT owned by K8s)
  - S3 backup data (external — NOT owned by K8s)
  - DNS records in Route53 (external — NOT owned by K8s)

WITHOUT FINALIZER:
  kubectl delete postgrescluster my-db
  → Object deleted from etcd immediately
  → StatefulSet orphaned or garbage collected
  → AWS RDS replica: still running, charging money!
  → S3 backup data: leaked!
  → DNS records: stale!

WITH FINALIZER:
  kubectl delete postgrescluster my-db
  → DeletionTimestamp set: "2024-01-15T10:00:00Z"
  → Object NOT deleted (finalizer still present)
  → Controller sees DeletionTimestamp → runs cleanup:
       Deletes AWS RDS replica via API
       Removes DNS records
       (S3 backup optionally kept or deleted per policy)
  → Controller removes finalizer
  → K8s deletes object from etcd
  → No resource leak!
```

---

### Finalizer Lifecycle

```
FINALIZER LIFECYCLE — STATE MACHINE
═══════════════════════════════════════════════════════════════

NORMAL OBJECT:
  metadata:
    finalizers:
    - databases.company.com/cleanup   ← finalizer present
    deletionTimestamp: null           ← not being deleted

KUBECTL DELETE CALLED:
  metadata:
    finalizers:
    - databases.company.com/cleanup   ← still present
    deletionTimestamp: "2024-01-15T10:00:00Z"  ← set!
    # Object is in "terminating" state
    # Object stays in etcd, visible via kubectl get (STATUS: Terminating)
    # No new resources can be created in this namespace if namespace is deleting

CONTROLLER RUNS CLEANUP:
  Sees DeletionTimestamp != nil
  Performs all cleanup tasks
  Removes finalizer:
  metadata:
    finalizers: []                    ← empty!
    deletionTimestamp: "2024-01-15T10:00:00Z"

KUBERNETES GARBAGE COLLECTS:
  No more finalizers → etcd deletes the object
  Object gone from kubectl get

MULTIPLE FINALIZERS:
  metadata:
    finalizers:
    - databases.company.com/cleanup    ← your controller removes this
    - metrics.company.com/deregister   ← metrics controller removes this
  ALL must be removed before object is deleted.
  Order of removal is up to controllers, not sequential.

STUCK TERMINATING OBJECT (production emergency):
  If controller crashed and can't remove finalizer:
  kubectl patch postgrescluster my-db \
    -p '{"metadata":{"finalizers":[]}}' \
    --type=merge
  → DANGEROUS: bypasses cleanup, may leak resources
  → Use only when you're sure cleanup is safe to skip
```

---

### Finalizer Implementation

```go
const (
    myFinalizer = "databases.company.com/cleanup"
)

func (r *PostgresClusterReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {

    cluster := &databasesv1.PostgresCluster{}
    if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // ── DELETION PATH ─────────────────────────────────────────────
    if !cluster.DeletionTimestamp.IsZero() {
        // Object is being deleted
        if controllerutil.ContainsFinalizer(cluster, myFinalizer) {
            // Run cleanup
            if err := r.cleanup(ctx, cluster); err != nil {
                // Cleanup failed → keep finalizer, retry
                return ctrl.Result{}, fmt.Errorf("cleanup failed: %w", err)
            }
            // Cleanup succeeded → remove finalizer
            controllerutil.RemoveFinalizer(cluster, myFinalizer)
            if err := r.Update(ctx, cluster); err != nil {
                return ctrl.Result{}, err
            }
        }
        // Finalizer removed → stop reconciling, K8s will delete object
        return ctrl.Result{}, nil
    }

    // ── CREATION/UPDATE PATH ──────────────────────────────────────
    // Add finalizer if not present (on first reconcile)
    if !controllerutil.ContainsFinalizer(cluster, myFinalizer) {
        controllerutil.AddFinalizer(cluster, myFinalizer)
        if err := r.Update(ctx, cluster); err != nil {
            return ctrl.Result{}, err
        }
        // Requeue to proceed with normal reconcile
        return ctrl.Result{Requeue: true}, nil
    }

    // Normal reconcile logic here...
    return ctrl.Result{RequeueAfter: 10 * time.Minute}, nil
}

func (r *PostgresClusterReconciler) cleanup(
    ctx context.Context,
    cluster *databasesv1.PostgresCluster,
) error {
    log := log.FromContext(ctx)
    log.Info("Running cleanup for", "cluster", cluster.Name)

    // 1. Delete AWS RDS read replica
    if err := r.awsClient.DeleteRDSReplica(ctx, cluster.Status.RDSReplicaID); err != nil {
        if !isNotFoundError(err) {
            return fmt.Errorf("deleting RDS replica: %w", err)
        }
    }

    // 2. Deregister DNS
    if err := r.route53Client.DeleteRecord(ctx, cluster.Status.DNSName); err != nil {
        if !isNotFoundError(err) {
            return fmt.Errorf("deleting DNS record: %w", err)
        }
    }

    // 3. Update status during deletion (shows deletion progress)
    cluster.Status.Phase = "Deleting"
    r.Status().Update(ctx, cluster)

    log.Info("Cleanup complete for", "cluster", cluster.Name)
    return nil
}
```

---

### Foreground Deletion (Kubernetes built-in cascading)

```
KUBERNETES DELETION PROPAGATION POLICIES
═══════════════════════════════════════════════════════════════

Background (default):
  kubectl delete deployment my-app
  → Deployment deleted immediately from etcd
  → Garbage collector deletes owned ReplicaSets and Pods in background
  → kubectl get pods briefly shows pods before GC removes them

Foreground:
  kubectl delete deployment my-app --cascade=foreground
  → Deployment gets "foreground deletion" finalizer added automatically
  → Kubernetes waits for ALL owned objects (ReplicaSets, Pods) to be deleted
  → Only then deletes the Deployment
  → Useful when you need to be sure all children are gone before parent

Orphan:
  kubectl delete deployment my-app --cascade=orphan
  → Deployment deleted, but ReplicaSets and Pods are NOT deleted
  → Their ownerReferences still point to the deleted Deployment
  → Use for: migrating ownership from one controller to another

FINALIZER vs FOREGROUND DELETION:
  Finalizer:           YOUR CODE runs cleanup (external resources)
  Foreground deletion: KUBERNETES handles owned K8s object cascade
  Both can coexist on the same object
```

---

### 🎤 Short Crisp Interview Answer

> *"Finalizers are strings on an object's metadata that block deletion. When you kubectl delete an object with a finalizer, Kubernetes sets deletionTimestamp but doesn't delete it — the object stays in etcd in 'Terminating' state. Your controller detects the non-zero deletionTimestamp, runs cleanup (delete external resources, deregister DNS, release cloud resources), then removes the finalizer. Only when all finalizers are gone does Kubernetes delete the object. The critical gotcha is stuck Terminating objects when a controller crashes — the finalizer never gets removed. In a production emergency you can patch the finalizers to empty, but this bypasses cleanup and may leak external resources. Always write cleanup to be idempotent (safe to retry) and to gracefully handle 'not found' errors from external APIs."*

---

---

# 10.9 Owner References & Garbage Collection

## 🟡 Intermediate

### What Owner References Are

```
OWNER REFERENCES — KUBERNETES PARENT-CHILD RELATIONSHIPS
═══════════════════════════════════════════════════════════════

When a controller creates a child object, it sets an ownerReference
on the child pointing back to the parent.

EXAMPLE: Deployment creates ReplicaSet creates Pods
  Deployment my-app:
    uid: abc-123

  ReplicaSet my-app-7d9f8:
    ownerReferences:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      uid: abc-123
      controller: true        ← this is the managing controller
      blockOwnerDeletion: true ← block parent deletion until child is gone

  Pod my-app-7d9f8-xyz:
    ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: my-app-7d9f8
      uid: def-456
      controller: true

GARBAGE COLLECTION:
  Parent deleted → Kubernetes GC scans for objects with ownerRef to deleted parent
  → Deletes all owned objects (cascade)
  → Recursive: ReplicaSet deletion → Pod deletion

CONTROLLER RULE:
  Any object your controller creates should have ownerReference set to the CR.
  If the CR is deleted → all created objects (StatefulSets, Services, etc.)
  are automatically garbage collected without any finalizer logic needed.
  (For K8s-native children — not for external resources like RDS/S3)
```

---

### Setting Owner References in Controllers

```go
// In your reconciler, when creating child objects:

import (
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
)

func (r *PostgresClusterReconciler) reconcileStatefulSet(
    ctx context.Context,
    cluster *databasesv1.PostgresCluster,
) error {

    sts := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cluster.Name,
            Namespace: cluster.Namespace,
        },
    }

    // Create or Update the StatefulSet
    result, err := controllerutil.CreateOrUpdate(ctx, r.Client, sts, func() error {
        // This func is called with the current state of the object
        // Set desired state here:
        sts.Spec = buildStatefulSetSpec(cluster)

        // SET OWNER REFERENCE ← critical!
        // Links sts to cluster (parent → child)
        return ctrl.SetControllerReference(cluster, sts, r.Scheme)
    })

    if err != nil {
        return fmt.Errorf("reconciling StatefulSet: %w", err)
    }

    log.Info("StatefulSet reconciled", "result", result)
    return nil
}

// What SetControllerReference does:
// Adds to sts.OwnerReferences:
// - apiVersion: databases.company.com/v1
//   kind: PostgresCluster
//   name: my-db
//   uid: <cluster-uid>
//   controller: true
//   blockOwnerDeletion: true
```

---

### Cross-Namespace Owner References

```
OWNER REFERENCE CONSTRAINTS
═══════════════════════════════════════════════════════════════

RULE: Owner and owned object MUST be in the same namespace
  OR owner must be cluster-scoped.

CAN DO:
  Namespaced owner → Namespaced child (same namespace)
  Cluster-scoped owner → Namespaced child (any namespace)
  Cluster-scoped owner → Cluster-scoped child

CANNOT DO:
  Namespaced owner → child in different namespace
    → Kubernetes rejects this
  Namespaced owner → cluster-scoped child
    → Kubernetes rejects this (who does the cluster object "belong" to?)

FOR CROSS-NAMESPACE CLEANUP:
  Use finalizers instead of ownerReferences
  OR: make the parent cluster-scoped

EXAMPLE: ClusterIssuer (cluster-scoped) creates Certificates (namespaced)
  This works because ClusterIssuer is cluster-scoped
  Certificates have ownerRef to ClusterIssuer ✓

EXAMPLE: Application (namespaced) creates resources in other namespaces
  Cannot use ownerRef across namespaces
  Must use finalizer to clean up cross-namespace resources
```

---

```bash
# Inspect owner references
kubectl get pods -n production -o yaml | \
  grep -A 8 "ownerReferences:"
# ownerReferences:
# - apiVersion: apps/v1
#   blockOwnerDeletion: true
#   controller: true
#   kind: ReplicaSet
#   name: my-app-7d9f8
#   uid: def-456

# Find all objects owned by a specific UID
kubectl get all -n production \
  -o json | \
  jq '.items[] |
    select(.metadata.ownerReferences[]?.uid == "abc-123") |
    .metadata.name'

# Check garbage collection in action
kubectl delete deployment my-app -n production
kubectl get replicasets -n production  # should disappear shortly
kubectl get pods -n production          # should disappear shortly

# Orphan children (don't delete children when parent is deleted)
kubectl delete deployment my-app \
  --cascade=orphan \
  -n production
# Deployment gone, ReplicaSets and Pods still exist
```

---

---

# 10.10 Operator Maturity Model (Levels 1–5)

## 🔴 Advanced

### The Five Levels

```
OPERATOR MATURITY MODEL — CNCF DEFINED
═══════════════════════════════════════════════════════════════

LEVEL 1 — BASIC INSTALL
  Automates: initial deployment of the application
  Capabilities:
    - Deploy application via a single CRD
    - Configurable via CRD spec
    - Create required Kubernetes resources (Deployment, Service, ConfigMap)
  What it doesn't do: handle failures, backups, upgrades
  Example: "create a Redis instance with these settings"

LEVEL 2 — SEAMLESS UPGRADES
  Automates: application and operator upgrades
  Capabilities:
    - Upgrade app to new version with spec.version change
    - Rolling update of application pods
    - Upgrade the operator itself without downtime
    - Patch upgrades applied automatically
  What it doesn't do: data backup, failure recovery
  Example: bump spec.postgresVersion from 15 to 16 → operator handles pg_upgrade

LEVEL 3 — FULL LIFECYCLE
  Automates: backup and restore
  Capabilities:
    - Scheduled backups (automated, to S3/GCS)
    - Point-in-time recovery (restore to specific timestamp)
    - Backup verification (test restores)
    - Storage scaling (PVC expansion)
    - Minor configuration changes without restart
  Example: CrunchyData Postgres Operator at this level

LEVEL 4 — DEEP INSIGHTS
  Automates: metrics, alerting, log management
  Capabilities:
    - Rich status conditions on the CRD
    - Prometheus metrics exposed (replication lag, connection pool, etc.)
    - PrometheusRule / ServiceMonitor created automatically
    - Alerting rules for critical conditions
    - Workload analysis (detect slow queries, table bloat)
  Example: Elastic Cloud on Kubernetes (ECK) at this level

LEVEL 5 — AUTOPILOT
  Automates: autonomous operation, self-healing
  Capabilities:
    - Automatic failover (detect primary failure → promote replica)
    - Automatic scaling based on load metrics
    - Automatic performance tuning (adjust PostgreSQL parameters)
    - Anomaly detection and remediation
    - Cross-cluster failover (DR automation)
  Example: AWS RDS-like behavior in Kubernetes
           Cockroach Operator at this level for some features
```

---

### Level Assessment for Common Operators

```
OPERATOR LEVEL ASSESSMENT (approximate)
═══════════════════════════════════════════════════════════════

Operator                        Level  Key Capability
─────────────────────────────────────────────────────────────
cert-manager                      3    Auto-renewal + ACME
Prometheus Operator               3    Full metrics lifecycle
CrunchyData Postgres Operator     4    Deep Postgres insights
Elastic ECK                       4    Elasticsearch metrics
Strimzi Kafka Operator            4    Kafka operations + metrics
MongoDB Community Operator        3    Backups + replication
Redis Operator (spotahome)        2    Basic Redis HA
External Secrets Operator         2    Secret sync lifecycle
Argo CD                          4    GitOps reconciliation
Kubeflow Pipelines               3    ML workflow lifecycle
Rook/Ceph                        4    Ceph management

REACHING LEVEL 5 requires deep domain knowledge:
  - Understanding failure modes specific to the application
  - Metrics that indicate scaling need (queue depth, connection pool saturation)
  - Safe automation criteria (when is it safe to auto-scale without data loss?)
  Only mature, production-tested operators reach this level
```

---

---

# 10.11 Status Subresource — Conditions Pattern

## 🔴 Advanced

### Status Subresource

```
STATUS SUBRESOURCE — WHY IT EXISTS
═══════════════════════════════════════════════════════════════

WITHOUT STATUS SUBRESOURCE:
  Controller updates status via normal Update(obj)
  → This updates the entire object (spec + status)
  → Sends full object to etcd, triggers spec watches
  → Other controllers see a "spec changed" event → infinite loop!

WITH STATUS SUBRESOURCE:
  Status lives at a separate API endpoint:
  PATCH /apis/databases.company.com/v1/ns/production/postgresclusters/my-db/status
  → Only status section updated
  → Spec unchanged → no spec-change events triggered
  → Users cannot modify status via kubectl apply (spec and status separated)
  → Controllers: r.Status().Update(ctx, cluster)
  → Users: kubectl apply changes spec only (status ignored)

ENABLING IN CRD:
  subresources:
    status: {}   ← enables /status subresource

SCALE SUBRESOURCE:
  subresources:
    scale:
      specReplicasPath: .spec.instances
      statusReplicasPath: .status.instances
  → Enables: kubectl scale postgrescluster my-db --replicas=5
  → HPA can target this CRD via scale subresource
```

---

### The Conditions Pattern

```
CONDITIONS — KUBERNETES STANDARD STATUS FORMAT
═══════════════════════════════════════════════════════════════

KUBERNETES CONVENTION for status.conditions:
  Array of Condition objects
  Each condition type is INDEPENDENTLY observable
  Allows: multiple things to be True/False simultaneously

CONDITION STRUCTURE:
  type:               string  — unique name for this condition (CamelCase)
  status:             string  — "True", "False", or "Unknown"
  reason:             string  — CamelCase machine-readable reason
  message:            string  — human-readable explanation
  lastTransitionTime: string  — when status last changed (ISO 8601)
  observedGeneration: int     — generation when this condition was set

GOOD CONDITION NAMING:
  Type       Status  Meaning
  ─────────────────────────────────────────────────────────────
  Ready      True    Object is fully operational
  Ready      False   Object is not operational (see reason)
  Ready      Unknown Object state not yet determined (pending)
  Available  True    At least minimum capacity available
  Progressing True   Change in progress (rolling update)
  Degraded   True    Operating but below desired state (some replicas down)
  Synced     True    CRD synced to external system
  Stalled    True    Cannot make progress (needs human intervention)

ANTIPATTERNS:
  ✗ Phase enum: "Creating" | "Running" | "Failed"
    Hard to extend, two things can't be true simultaneously
  ✓ Conditions array: Ready=True, Synced=True, Degraded=False
    Each independently observable, easy to add new conditions

BUILTIN EXAMPLES:
  Pod conditions:
    - type: Ready, status: True/False/Unknown
    - type: ContainersReady
    - type: Initialized
    - type: PodScheduled
  Node conditions:
    - type: Ready
    - type: MemoryPressure
    - type: DiskPressure
```

---

### Implementing Conditions

```go
// Standard conditions management in a controller
import (
    apimeta "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func (r *PostgresClusterReconciler) setCondition(
    cluster *databasesv1.PostgresCluster,
    condType string,
    status metav1.ConditionStatus,
    reason, message string,
) {
    condition := metav1.Condition{
        Type:               condType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        ObservedGeneration: cluster.Generation,  // tie condition to spec version
        LastTransitionTime: metav1.NewTime(time.Now()),
    }

    // apimeta.SetStatusCondition handles:
    // - Adding new condition if type doesn't exist
    // - Updating existing condition if status/reason/message changed
    // - NOT updating LastTransitionTime if status unchanged (stable timestamps)
    apimeta.SetStatusCondition(&cluster.Status.Conditions, condition)
}

// Usage in reconcile:
func (r *PostgresClusterReconciler) Reconcile(...) {
    // ...

    // On success:
    r.setCondition(cluster, "Ready", metav1.ConditionTrue,
        "ClusterRunning", "All instances are running")
    r.setCondition(cluster, "Progressing", metav1.ConditionFalse,
        "ReconcileComplete", "No changes pending")

    // On error (DB connection failed):
    r.setCondition(cluster, "Ready", metav1.ConditionFalse,
        "PrimaryUnavailable", fmt.Sprintf("Cannot connect to primary: %v", err))

    // During rolling update:
    r.setCondition(cluster, "Progressing", metav1.ConditionTrue,
        "RollingUpdate",
        fmt.Sprintf("Updating from %d to %d instances", current, desired))
    r.setCondition(cluster, "Available", metav1.ConditionTrue,
        "MinimumReplicasAvailable",
        "At least 1 instance is serving traffic")

    // Update status
    if err := r.Status().Update(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
}
```

---

```bash
# Checking conditions from kubectl
kubectl get postgrescluster my-db -n production \
  -o jsonpath='{.status.conditions[*]}' | python3 -m json.tool

# Better: jsonpath table
kubectl get postgrescluster my-db -n production \
  -o jsonpath='{range .status.conditions[*]}
  {.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
# Ready         True    ClusterRunning      All instances are running
# Available     True    QuorumMet           3/3 instances ready
# Progressing   False   ReconcileComplete   No changes pending
# Synced        True    BackupComplete      Last backup 2h ago

# Wait for condition (scripting / CI)
kubectl wait postgrescluster/my-db \
  --for=condition=Ready \
  --timeout=300s \
  -n production

# Watch conditions change during update
watch -n 2 "kubectl get postgrescluster my-db -n production \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status}{\"\\n\"}{end}'"
```

---

---

# 10.12 GitOps with CRDs — Flux/ArgoCD Managing Custom Resources

## 🔴 Advanced

### CRDs in GitOps Workflows

```
GITOPS WITH CRDS — KEY CHALLENGES
═══════════════════════════════════════════════════════════════

CHALLENGE 1: CRD must exist BEFORE custom resources
  If Argo CD applies CRs before the CRD is registered:
  → "no matches for kind 'PostgresCluster' in version 'databases.company.com/v1'"
  → Application fails to sync
  Solution: CRD in separate App with Sync Wave 0, CRs in Wave 1

CHALLENGE 2: Status fields in git create noise
  GitOps tools watch for drift between git and cluster
  Status fields change constantly (updated by controller)
  If status is in git: perpetual "out of sync" state
  Solution: Argo CD ignores status by default
            Flux: don't commit status to git

CHALLENGE 3: Operator installation before CRD instances
  Installing cert-manager Helm chart creates CRDs
  But CRD registration takes ~1 second
  If CertificateIssuer CR applied too fast → validation error
  Solution: Flux health checks, Argo CD sync waves, retry logic

CHALLENGE 4: Webhook dependency
  CRDs with validation webhooks: webhook must be running
  Argo CD applies CR → webhook not up yet → rejection
  Solution: Argo CD resource hook to wait for webhook endpoint
```

---

### Argo CD Sync Waves and Hooks

```yaml
# PATTERN: CRD in Wave -1, Operator in Wave 0, CRs in Wave 1

# App 1: CRDs only (wave -1 = first)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: postgres-operator-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # apply first
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-config
    path: operators/postgres/crds
  destination:
    server: https://kubernetes.default.svc
    namespace: postgres-operator-system
  syncPolicy:
    automated: {}

---
# App 2: Operator deployment (wave 0)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: postgres-operator
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  source:
    repoURL: https://registry.example.com
    chart: postgres-operator
    targetRevision: 5.4.0
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true

---
# Postgres CR (wave 1) — with pre-sync hook to wait for operator
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-databases
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # apply after operator
spec:
  source:
    path: environments/production/databases
  syncPolicy:
    automated:
      prune: true      # delete CRs removed from git
      selfHeal: true   # reconcile drift

---
# In environments/production/databases/postgres-cluster.yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: production-db
  namespace: production
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  postgresVersion: 16
  instances:
  - name: main
    replicas: 3
```

---

### Argo CD Resource Customizations

```yaml
# argocd-cm ConfigMap — ignore specific fields to prevent drift detection
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations: |
    # Ignore status on ALL custom resources (most common)
    "*.company.com/*":
      ignoreDifferences: |
        - group: "*.company.com"
          jsonPointers:
          - /status

    # Ignore operator-managed annotations on specific CRD
    "databases.company.com/PostgresCluster":
      ignoreDifferences: |
        - jsonPointers:
          - /metadata/annotations/postgres-operator.crunchydata.com~1pgbackrest-config
          - /status

    # Health check for custom CRD
    # Argo CD uses this to determine if the resource is healthy
    "databases.company.com/PostgresCluster":
      health.lua: |
        health_status = {}
        if obj.status ~= nil then
          if obj.status.conditions ~= nil then
            for _, condition in ipairs(obj.status.conditions) do
              if condition.type == "Ready" then
                if condition.status == "True" then
                  health_status.status = "Healthy"
                  health_status.message = condition.message
                elseif condition.status == "False" then
                  health_status.status = "Degraded"
                  health_status.message = condition.message
                else
                  health_status.status = "Progressing"
                  health_status.message = condition.message
                end
              end
            end
          end
        end
        if health_status.status == nil then
          health_status.status = "Progressing"
          health_status.message = "Waiting for Ready condition"
        end
        return health_status
```

---

### Flux CRD Management

```yaml
# Flux: HelmRelease for operator (installs CRDs + controller)
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: postgres-operator
  namespace: flux-system
spec:
  interval: 1h
  chart:
    spec:
      chart: pgo
      version: 5.x
      sourceRef:
        kind: HelmRepository
        name: crunchy-data
  install:
    crds: CreateReplace    # install CRDs on first install, replace on upgrade
  upgrade:
    crds: CreateReplace
  # Health checks: Flux waits for these before marking HelmRelease as Ready
  # This prevents dependent resources from being applied too early
  dependsOn:
  - name: cert-manager    # wait for cert-manager before applying (webhook dep)

---
# Flux Kustomization with dependency ordering
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production-databases
  namespace: flux-system
spec:
  interval: 10m
  path: ./environments/production/databases
  sourceRef:
    kind: GitRepository
    name: fleet-repo
  # CRITICAL: wait for operator to be healthy before applying CRs
  dependsOn:
  - name: postgres-operator
  # Wait for CRDs to be established before reconciling
  wait: true
  timeout: 5m
  # Health check: Kustomization is healthy when these conditions are met
  healthChecks:
  - apiVersion: postgres-operator.crunchydata.com/v1beta1
    kind: PostgresCluster
    name: production-db
    namespace: production
```

---

---

# 10.13 API Aggregation Layer vs CRDs — When to Use Which

## 🔴 Advanced

### Two Ways to Extend the Kubernetes API

```
CRD vs API AGGREGATION — COMPARISON
═══════════════════════════════════════════════════════════════

CRDs (CustomResourceDefinitions):
  Mechanism: register new types in existing API Server
  Storage:   etcd (same database as built-in objects)
  Code:      no code for the API itself (auto-generated)
  Pros:      simple, fast to create, full kubectl support,
             RBAC works automatically, GitOps-friendly
  Cons:      limited to CRUD+Watch pattern,
             no custom request handling logic,
             all objects stored in etcd (potentially large),
             limited validation expressiveness
  Use for:   95%+ of extensibility use cases

API AGGREGATION LAYER (AA):
  Mechanism: separate API Server that the main API Server proxies to
  Storage:   you decide (not necessarily etcd)
  Code:      you write a full HTTP server implementing the API
  Pros:      full control over API behavior (any HTTP logic)
             custom storage backends (Prometheus: in-memory time series)
             non-CRUD operations (e.g., /exec, /log, /portforward)
             subresource patterns that don't fit CRD model
  Cons:      complex: write and operate a full API Server
             must handle: auth, RBAC delegation, certificates, etcd (or own storage)
             high operational burden
  Use for:   1-5% of cases needing special behavior

BUILT-IN API AGGREGATION EXAMPLES:
  metrics.k8s.io/v1beta1:   metrics-server (in-memory, not etcd)
  custom.metrics.k8s.io:    Prometheus Adapter
  external.metrics.k8s.io:  Datadog, Wavefront metrics
  These aren't CRDs — they're separate API servers registered via APIService
```

---

### APIService Registration

```yaml
# Register an aggregated API Server
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  # The backing service that handles requests for this API group/version
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: false
  caBundle: <base64-ca-bundle>
  groupPriorityMinimum: 100
  versionPriority: 100

# When a client requests: GET /apis/metrics.k8s.io/v1beta1/nodes
# API Server: "metrics.k8s.io is registered with APIService"
# → Proxies request to metrics-server:443
# → metrics-server responds with in-memory metrics
# → API Server returns response to client

# Client sees the metrics as a native Kubernetes API:
kubectl top nodes           # hits metrics.k8s.io/v1beta1/nodes
kubectl top pods            # hits metrics.k8s.io/v1beta1/pods/...

# These look like normal kubectl commands but
# the API is served by metrics-server, not the main API Server
```

---

### Decision Guide

```
WHEN TO USE WHAT
═══════════════════════════════════════════════════════════════

USE CRD WHEN:
  ✓ You want to store configuration or state in Kubernetes
  ✓ Standard CRUD+Watch operations are sufficient
  ✓ You want GitOps (kubectl apply your state)
  ✓ You want RBAC to control access
  ✓ Objects have finite, manageable size
  ✓ Object count < few million
  Examples: Database declarations, application configs,
            certificate requests, network policies

USE API AGGREGATION WHEN:
  ✓ API behavior can't be expressed as CRUD
    - Custom subresources (/exec, /portforward, /log)
    - Server-side computation returning derived data
    - Streaming responses
  ✓ Data doesn't belong in etcd
    - Metrics (high-cardinality time-series data)
    - Log queries (streaming, not stored)
    - Resource metrics (computed on the fly)
  ✓ Non-Kubernetes storage needed
    - External database as source of truth
    - Cache or in-memory backend
  ✓ Object counts too large for etcd (millions of objects)
  Examples: metrics.k8s.io, kubectl top (metrics-server),
            HPA metrics (prometheus-adapter),
            Service Catalog (deprecated)

THE INTERMEDIATE OPTION — CRD WITH OPERATOR WEBHOOK:
  Custom admission behavior + CEL validation covers
  many "custom API logic" needs without full aggregation.
  CRD + ValidatingWebhook can reject invalid cross-object states.
  CRD + MutatingWebhook can set complex defaults.
  This covers ~95% of "I need custom API logic" cases.
```

---

---

# 10.14 CRD Performance at Scale — Watch Cache, Pagination

## 🔴 Advanced

### Informer and Watch Cache

```
WATCH CACHE — SCALING CONTROLLER READS
═══════════════════════════════════════════════════════════════

PROBLEM WITHOUT CACHING:
  Controller has 1,000 PostgresCluster objects to watch.
  On every reconcile: GET /apis/.../postgresclusters/my-db
  → Direct API Server query → direct etcd read
  1,000 controllers reconciling every minute = 1,000 etcd reads/min
  → API Server and etcd overloaded

API SERVER WATCH CACHE:
  API Server maintains an in-memory cache of all objects per resource type
  Controllers use WATCH (streaming) not repeated GET
  Initial LIST: populates controller's local cache from API Server cache
  Subsequent events: streaming watch (no polling)
  → API Server cache hit: no etcd read for most reads
  → Controller cache: no API Server call for most reads

CONTROLLER CACHE PATTERN (controller-runtime):
  Manager starts shared informer per resource type
  SharedInformer: multiple controllers share one cache per resource
  Reconcile reads from cache, not API Server:
  r.Get(ctx, key, obj)  // reads from cache!
  // Only misses go to API Server

CACHE LIMITATIONS:
  Cache is eventually consistent (small lag after etcd write)
  For status updates: always use r.Status().Update() (server-side)
  For conflict detection: use resource version (optimistic locking)
  For guaranteed fresh read: use --disable-deepcopy on specific Gets

WATCH CACHE SIZE:
  API Server flag: --watch-cache-sizes=pods#1000,postgresclusters#500
  Larger = more memory but fewer etcd reads
  Default: 100 most recent watch events per resource type
```

---

### Pagination for Large Lists

```go
// NEVER list all objects at once for large CRs
// This can OOM your controller or the API Server

// BAD — loads all objects into memory at once
allClusters := &databasesv1.PostgresClusterList{}
if err := r.List(ctx, allClusters); err != nil {  // could be 10,000 objects!
    return err
}

// GOOD — paginate with continue tokens
// API Server returns a page at a time
limit := int64(100)  // 100 objects per page
continueToken := ""

for {
    clusters := &databasesv1.PostgresClusterList{}
    listOpts := &client.ListOptions{
        Limit:    limit,
        Continue: continueToken,
    }
    if err := r.List(ctx, clusters, listOpts); err != nil {
        return err
    }

    // Process this page
    for _, cluster := range clusters.Items {
        // process cluster
    }

    // Check for next page
    if clusters.Continue == "" {
        break  // no more pages
    }
    continueToken = clusters.Continue
}

// FIELD SELECTORS — filter at API Server before sending to controller
clusters := &databasesv1.PostgresClusterList{}
r.List(ctx, clusters,
    client.MatchingFields{"spec.postgresVersion": "16"},
    client.InNamespace("production"),
)
// Requires: index registered with controller manager:
// mgr.GetFieldIndexer().IndexField(ctx, &PostgresCluster{},
//     "spec.postgresVersion",
//     func(obj client.Object) []string {
//         return []string{strconv.Itoa(obj.(*PostgresCluster).Spec.PostgresVersion)}
//     })
```

---

### CRD Performance Considerations at Scale

```
PERFORMANCE CONSIDERATIONS FOR LARGE CRD DEPLOYMENTS
═══════════════════════════════════════════════════════════════

ETCD SIZE:
  etcd recommended max: 8GB (hard limit: 8GB)
  etcd works best: < 4GB
  A single object max: 1.5MB (etcd hard limit per value)
  Planning: 10,000 PostgresCluster objects × 10KB each = 100MB → fine
           1,000,000 objects × 10KB each = 10GB → etcd problem!
  Solution for many objects: use API aggregation with own storage

CRD SCHEMA IMPACT:
  Large, complex schemas (1000+ lines) slow CRD validation
  Each admission request: schema validation of the full object
  Solution: split into multiple CRDs or use CEL for complex validation

WATCH MULTIPLIER PROBLEM:
  N controllers watching resource type = N watch connections to API Server
  SharedInformer: reduces N to 1 per process (shared within same binary)
  Multiple operator replicas: each replica has separate watch connections
  → HA setup: 2-3 replicas = 2-3x watch connections
  Mitigation: leader election (only leader reconciles, followers standby)

OBJECT SIZE GROWTH:
  Conditions array grows over time if not pruned
  Annotations accumulate from multiple operators
  Large status.observedGeneration fields
  Best practice: limit conditions to last N, prune old events from status

GENERATION vs ResourceVersion:
  Generation: increments only on spec changes (metadata.generation)
  ResourceVersion: increments on ANY change (spec or status)
  Use generation-based predicates:
    predicate.GenerationChangedPredicate{}
    → Only trigger reconcile on spec changes, not status updates
    → Avoids reconcile loop from status updates
```

---

```bash
# ── OPERATIONAL COMMANDS FOR CRD SCALE ───────────────────────────

# Count CRD instances
kubectl get postgresclusters -A --no-headers | wc -l
# 847 PostgresCluster instances across cluster

# Find large CRD objects (may cause etcd pressure)
kubectl get postgresclusters -A -o json | \
  jq '.items[] | {name: .metadata.name, ns: .metadata.namespace,
      size: (. | tojson | length)}' | \
  jq -s 'sort_by(.size) | reverse | .[0:10]'
# Shows top 10 largest objects by JSON size

# Check etcd DB size
etcdctl endpoint status -w table
# ENDPOINT                DBSIZE   IS LEADER
# https://10.0.1.10:2379  512 MB   true
# 512MB total etcd DB — all objects across cluster

# Watch controller reconcile rate (metrics)
kubectl port-forward -n my-operator deployment/my-operator 8080
curl http://localhost:8080/metrics | grep controller_runtime_reconcile
# controller_runtime_reconcile_total{controller="postgrescluster",result="success"} 12847
# controller_runtime_reconcile_total{controller="postgrescluster",result="error"} 23
# controller_runtime_reconcile_duration_seconds{controller="postgrescluster"} histogram

# Check API Server cache hit ratio (proxy to API Server metrics)
# High miss ratio → controllers not using cache properly
curl https://api-server:6443/metrics | grep watchcache
# apiserver_watch_cache_events_dispatched_total

# Controller work queue depth (should be near 0 in steady state)
curl http://localhost:8080/metrics | grep workqueue_depth
# workqueue_depth{name="postgrescluster"} 3  ← 3 items queued
# workqueue_depth > 100 → controller falling behind
```

---

# Quick Reference — Category 10 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **CRD purpose** | Register new API types. Stored in etcd. Full kubectl + RBAC + GitOps support |
| **CRD naming** | `{plural}.{group}` — e.g., `postgresclusters.databases.company.com` |
| **Scope** | Namespaced (team resources) vs Cluster (infrastructure resources) |
| **Storage: true** | Only ONE version can be storage: true. This version lives in etcd |
| **Status subresource** | `subresources: status: {}`. Separate endpoint. Controllers use r.Status().Update() |
| **Scale subresource** | Enables `kubectl scale`. HPA can target it |
| **AdditionalPrinterColumns** | Controls `kubectl get` output columns via JSONPath |
| **CEL validation** | K8s 1.25+. Cross-field rules. `x-kubernetes-validations`. `oldSelf` for immutability |
| **Controller loop** | Level-triggered. Watch → queue → reconcile → return Result |
| **Informer pattern** | LIST once, WATCH forever. Local cache. Shared across controllers in same process |
| **Reconcile return** | nil=done, error=backoff retry, Requeue=immediate, RequeueAfter=timed |
| **Idempotency** | Reconcile must be safe to call multiple times with same desired state |
| **Operator = CRD + Controller** | CRD = schema. Controller = brain. Together = Operator |
| **Finalizer** | String blocking deletion. DeletionTimestamp set → controller runs cleanup → removes finalizer |
| **Stuck Terminating** | Patch finalizers to [] — dangerous, bypasses cleanup |
| **Owner References** | Same-namespace only. Garbage collection when parent deleted |
| **Foreground deletion** | Wait for all children before deleting parent |
| **CRD versioning** | Multiple served, one stored. Conversion webhook translates between versions |
| **Hub-and-spoke** | All versions convert through hub (latest) — keeps N conversions manageable |
| **Conditions pattern** | Array of {type, status, reason, message, lastTransitionTime}. Independent observability |
| **Ready/Available/Progressing** | Standard condition types. Avoid Phase enum antipattern |
| **Operator Level 1-5** | Basic install → Upgrades → Lifecycle → Insights → Autopilot |
| **GitOps: CRD before CR** | Sync Wave -1 for CRDs, Wave 0 for operator, Wave 1 for CRs |
| **Argo CD status ignore** | Add ignoreDifferences for /status to prevent perpetual drift |
| **API Aggregation** | Separate API Server. Needed for non-CRUD APIs (metrics, streaming). Rare |
| **CRD vs AA** | CRDs: 95%+ of use cases. AA: metrics, custom subresources, non-etcd storage |
| **Watch cache** | API Server in-memory. Controllers use local cache. Most reads never hit etcd |
| **Pagination** | List with Limit + Continue token. Never list all at once for large resource sets |
| **GenerationChangedPredicate** | Only trigger on spec changes. Prevents infinite loop from status updates |
| **Field indexer** | Register indexed fields for efficient filtered List calls from controller |
| **etcd DB limit** | 8GB hard, 4GB recommended. Plan CRD object count × size accordingly |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| CRD available after apply | ~1 second |
| Max object size in etcd | 1.5MB |
| Recommended etcd DB max | 4GB (hard limit 8GB) |
| Default watch cache events | 100 per resource type |
| Reconcile backoff cap | ~1000 seconds |
| Operator Maturity Levels | 1–5 (Basic Install → Autopilot) |
| CEL validation GA | Kubernetes 1.25 |
| Status subresource since | Kubernetes 1.13 |
| Conversion webhook timeout | 30 seconds (configurable) |
| Default manager resync period | 10 minutes (SharedInformer full re-list) |
| Recommended MaxConcurrentReconciles | 1 (safe default), increase for I/O-bound |
| controller-gen marker prefix | `// +kubebuilder:` |
| Finalizer stuck: emergency fix | `kubectl patch --type=merge -p '{"metadata":{"finalizers":[]}}'` |

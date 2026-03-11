# Kubernetes Interview Mastery
# CATEGORY 9: CLUSTER OPERATIONS

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Deep Answer → Gotchas → Interview Q&A → Connections.
> ⚠️ = High priority, frequently asked, commonly misunderstood.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 9.1 | kubeconfig — contexts, users, clusters | 🟢 Beginner |
| 9.2 | kubectl essentials — get, describe, logs, exec, port-forward | 🟢 Beginner |
| 9.3 | Namespaces in practice | 🟢 Beginner |
| 9.4 | Node lifecycle — cordon, drain, uncordon ⚠️ | 🟡 Intermediate |
| 9.5 | Cluster upgrades — control plane & node upgrade strategy ⚠️ | 🟡 Intermediate |
| 9.6 | etcd backup & restore ⚠️ | 🟡 Intermediate |
| 9.7 | Helm — charts, values, releases, hooks | 🟡 Intermediate |
| 9.8 | Kustomize — overlays, patches | 🟡 Intermediate |
| 9.9 | Cluster autoscaler — how it works, scale-down safety ⚠️ | 🔴 Advanced |
| 9.10 | Node pools & machine sets (Cluster API) | 🔴 Advanced |
| 9.11 | Multi-tenancy patterns — namespace, vcluster, separate clusters | 🔴 Advanced |
| 9.12 | Disaster recovery planning for Kubernetes | 🔴 Advanced |

---

# 9.1 kubeconfig — Contexts, Users, Clusters

## 🟢 Beginner

### What it is in simple terms

A kubeconfig file is a **YAML file that stores cluster connection information** — API server address, authentication credentials, and a "context" that pairs a cluster with a user identity. `kubectl` reads this file to know which cluster to talk to and how to authenticate. Most engineers manage multiple kubeconfig files daily when working across dev, staging, and production clusters.

---

### kubeconfig Structure

```
KUBECONFIG FILE ANATOMY
═══════════════════════════════════════════════════════════════

~/.kube/config (default location)

THREE SECTIONS:
  clusters:  defines clusters (API server URL + CA cert)
  users:     defines authentication methods (cert, token, OIDC exec)
  contexts:  pairs a cluster + user + optional default namespace

CONTEXT = cluster + user + namespace
  Switching context = switching which cluster + who you are + which namespace

DEFAULT KUBECONFIG LOCATION:
  ~/.kube/config
  OR: KUBECONFIG environment variable (colon-separated paths)
  kubectl --kubeconfig=/path/to/file get pods
```

---

### kubeconfig YAML — Full Example

```yaml
apiVersion: v1
kind: Config
preferences: {}

# SECTION 1: Clusters — where to connect
clusters:
- name: production-cluster
  cluster:
    server: https://api.production.company.com:6443
    certificate-authority-data: <base64-encoded-CA-cert>
    # OR: certificate-authority: /path/to/ca.crt
    # OR (insecure, dev only): insecure-skip-tls-verify: true

- name: staging-cluster
  cluster:
    server: https://ABCDEF1234567890.gr7.us-east-1.eks.amazonaws.com
    certificate-authority-data: <base64-encoded-CA-cert>

# SECTION 2: Users — how to authenticate
users:
- name: production-admin
  user:
    # Option A: Certificate-based auth (self-managed clusters)
    client-certificate-data: <base64-encoded-client-cert>
    client-key-data: <base64-encoded-private-key>

- name: staging-oidc-user
  user:
    # Option B: OIDC exec plugin (kubelogin)
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://login.microsoftonline.com/<tenant>/v2.0
      - --oidc-client-id=kubernetes
      interactiveMode: IfAvailable

- name: eks-aws-user
  user:
    # Option C: AWS exec plugin (EKS)
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args:
      - eks
      - get-token
      - --cluster-name
      - my-eks-cluster
      - --region
      - us-east-1
      env:
      - name: AWS_PROFILE
        value: production

# SECTION 3: Contexts — pairs cluster + user + namespace
contexts:
- name: production
  context:
    cluster: production-cluster
    user: production-admin
    namespace: production          # default namespace for this context

- name: staging
  context:
    cluster: staging-cluster
    user: staging-oidc-user
    namespace: staging

- name: eks-production
  context:
    cluster: staging-cluster       # reuse cluster definition
    user: eks-aws-user
    namespace: production

# WHICH CONTEXT IS ACTIVE
current-context: eks-production
```

---

### kubectl kubeconfig Commands

```bash
# View current context
kubectl config current-context
# eks-production

# List all contexts
kubectl config get-contexts
# CURRENT   NAME              CLUSTER              AUTHINFO        NAMESPACE
# *         eks-production    staging-cluster      eks-aws-user    production
#           production        production-cluster   prod-admin      production
#           staging           staging-cluster      staging-oidc    staging

# Switch context
kubectl config use-context staging
kubectl config use-context production

# View kubeconfig
kubectl config view
kubectl config view --minify               # only current context
kubectl config view --raw                  # include credentials (careful!)

# Temporary context switch (per command)
kubectl get pods --context=staging
kubectl get pods --namespace=monitoring --context=production

# Merge multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/eks-production:~/.kube/eks-staging
kubectl config view --flatten > ~/.kube/merged-config
# Then: export KUBECONFIG=~/.kube/merged-config

# Add new cluster from EKS
aws eks update-kubeconfig \
  --name my-cluster \
  --region us-east-1 \
  --alias my-cluster-prod            # optional alias for context name

# Add new cluster from kubeadm (manually)
kubectl config set-cluster my-cluster \
  --server=https://api.my-cluster.com:6443 \
  --certificate-authority=/path/to/ca.crt

kubectl config set-credentials my-admin \
  --client-certificate=/path/to/admin.crt \
  --client-key=/path/to/admin.key

kubectl config set-context my-context \
  --cluster=my-cluster \
  --user=my-admin \
  --namespace=default

kubectl config use-context my-context

# Delete context/cluster/user
kubectl config delete-context staging
kubectl config delete-cluster staging-cluster
kubectl config unset users.eks-aws-user

# Set default namespace for current context
kubectl config set-context --current --namespace=production
# Now: kubectl get pods → looks in production namespace (no -n needed)

# KUBECTX / KUBENS (popular tools for faster switching)
# brew install kubectx kubens
# kubectx                    → list and switch contexts
# kubens                     → list and switch namespaces
# kubectx staging            → switch to staging
# kubens monitoring          → switch to monitoring namespace
```

---

### Security Considerations

```
KUBECONFIG SECURITY
═══════════════════════════════════════════════════════════════

RISKS:
  kubeconfig contains private keys and tokens — treat like a password
  ~/.kube/config permissions should be: chmod 600 ~/.kube/config
  Never commit kubeconfig to git

BEST PRACTICES:
  Use exec plugins (OIDC, AWS) instead of static credentials
    → Credentials are short-lived, fetched at use time
    → No long-lived secrets in the file

  Use separate kubeconfig files per cluster (not one merged file)
    → Reduces blast radius if one file is compromised

  Production kubeconfig: require MFA via OIDC before token issue
    → kubelogin redirects to browser → MFA → short-lived token

  Rotate certificates regularly (kubeadm: every 1 year default)
  Monitor API Server access logs for unexpected users

KUBECONFIG IN CI/CD:
  Store as encrypted secret (GitHub Secrets, Vault, AWS Secrets Manager)
  Use short-lived credentials:
    EKS: aws eks get-token → 15-minute expiry
    Service Accounts: create short-lived token with kubectl create token
  Prefer IRSA/Workload Identity over static kubeconfig for CI pods
```

---

### 🎤 Short Crisp Interview Answer

> *"A kubeconfig file has three sections: clusters (API server URL and CA cert), users (authentication method — certificates, tokens, or exec plugins for OIDC/AWS), and contexts which pair a cluster with a user and an optional default namespace. The current-context field sets which cluster you're talking to. kubectl config use-context switches between them. For EKS, aws eks update-kubeconfig populates the file automatically using an exec plugin that runs aws eks get-token, producing 15-minute tokens. Security-wise, kubeconfig should be chmod 600 and never committed to git — treat it like a password file."*

---

### ⚠️ Gotchas

1. **KUBECONFIG env var overrides ~/.kube/config completely** — if KUBECONFIG is set, the default file is ignored unless explicitly included in the colon-separated list.
2. **Merging kubeconfigs can overwrite entries with same name** — if two files both have a context named "production", the last one wins in `--flatten`. Rename before merging.
3. **exec plugin credentials are not refreshed mid-command** — if a token expires during a very long kubectl operation, the operation fails. This is rare but happens with large bulk operations.

---

---

# 9.2 kubectl Essentials — get, describe, logs, exec, port-forward

## 🟢 Beginner

### What it is in simple terms

`kubectl` is the **primary CLI tool for interacting with Kubernetes clusters**. Mastery of its core commands is mandatory for any Kubernetes operator. This topic covers the commands used daily in production operations.

---

### kubectl get — Viewing Resources

```bash
# BASIC GET PATTERNS
kubectl get pods                          # pods in current namespace
kubectl get pods -n production            # specific namespace
kubectl get pods -A                       # ALL namespaces
kubectl get pods --all-namespaces         # same as -A

# GET WITH OUTPUT FORMATS
kubectl get pods -o wide                  # adds node, IP columns
kubectl get pods -o yaml                  # full YAML spec
kubectl get pods -o json                  # full JSON
kubectl get pods -o name                  # just "pod/my-pod" names
kubectl get pod my-pod -o jsonpath='{.status.podIP}'    # specific field
kubectl get pods -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName"

# GET MULTIPLE RESOURCE TYPES
kubectl get pods,services,deployments -n production
kubectl get all -n production             # pods, services, deployments, replicasets

# LABEL SELECTORS
kubectl get pods -l app=my-api            # pods with label app=my-api
kubectl get pods -l app=my-api,env=prod   # AND logic
kubectl get pods -l 'env in (prod,staging)' # IN selector
kubectl get pods -l 'env!=dev'            # NOT EQUAL

# FIELD SELECTORS
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=spec.nodeName=worker-2
kubectl get pods --field-selector=status.phase!=Running -A

# SORTING
kubectl get pods --sort-by='.status.startTime'
kubectl get pods --sort-by='.metadata.name'
kubectl get pods -A --sort-by='.metadata.creationTimestamp'

# WATCHING (live update)
kubectl get pods -w                       # watch for changes
kubectl get pods -w --no-headers          # cleaner watch output

# RESOURCE SHORTNAMES
# pod = po, service = svc, deployment = deploy, namespace = ns
# replicaset = rs, statefulset = sts, daemonset = ds
# configmap = cm, persistentvolumeclaim = pvc, persistentvolume = pv
# serviceaccount = sa, horizontalpodautoscaler = hpa
kubectl get po,svc,deploy -n production   # shortnames

# JSONPATH EXAMPLES
kubectl get pod my-pod -o jsonpath='{.metadata.labels}'
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

---

### kubectl describe — Detailed Resource Info

```bash
# Describe shows full spec + status + events
kubectl describe pod my-pod -n production
kubectl describe node worker-2
kubectl describe deployment my-api -n production
kubectl describe service my-svc -n production
kubectl describe pvc my-pvc -n production
kubectl describe storageclass gp3

# Most useful for debugging:
kubectl describe pod my-pod         # → Events, Conditions, probe config
kubectl describe node worker-2      # → Taints, conditions, resource usage
kubectl describe pvc my-pvc         # → binding status, events if stuck
kubectl describe hpa my-api         # → current vs desired replicas, metrics
kubectl describe ingress my-ingress # → rules, backend services

# Shorthand with label selector
kubectl describe pods -l app=my-api -n production
```

---

### kubectl exec — Running Commands in Containers

```bash
# Run command in container
kubectl exec my-pod -- ls /app
kubectl exec my-pod -n production -- cat /etc/config/app.yaml
kubectl exec my-pod -c sidecar -- ps aux           # specific container

# Interactive shell
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -- /bin/sh                 # if no bash
kubectl exec -it my-pod -c app -- /bin/sh

# Common debugging tasks via exec:
# Check environment variables
kubectl exec my-pod -- env | sort

# Test DNS resolution
kubectl exec my-pod -- nslookup postgres.production.svc.cluster.local

# Test connectivity
kubectl exec my-pod -- wget -qO- http://other-service:8080/health
kubectl exec my-pod -- nc -zv postgres-svc 5432

# Check mounted secrets/configs
kubectl exec my-pod -- cat /etc/secrets/db-password
kubectl exec my-pod -- ls -la /etc/config/

# Check process list
kubectl exec my-pod -- ps aux

# Check network connections
kubectl exec my-pod -- ss -tnp    # or netstat -tnp

# EPHEMERAL CONTAINERS (K8s 1.23+ for debugging distroless/scratch images)
kubectl debug -it my-pod --image=busybox --target=app
# Adds a temporary debug container that shares PID/network namespace
# Useful when main container has no shell (distroless images)
```

---

### kubectl port-forward — Local Access to Cluster Services

```bash
# Forward local port → pod port
kubectl port-forward pod/my-pod 8080:8080
kubectl port-forward pod/my-pod 8080:8080 -n production

# Forward local port → service port (load balanced to any pod)
kubectl port-forward service/my-api 8080:80 -n production

# Forward local port → deployment (picks one pod)
kubectl port-forward deployment/my-api 8080:8080 -n production

# Common use cases:
# Access Prometheus UI locally
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Then: open http://localhost:9090

# Access Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80
# Then: open http://localhost:3000

# Access database for local debugging
kubectl port-forward -n production svc/postgres 5432:5432
# Then: psql -h localhost -U myuser mydb

# Access Kubernetes Dashboard
kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard 8443:443

# Run in background
kubectl port-forward svc/my-api 8080:80 -n production &
# Kill: kill %1  (or fg then Ctrl-C)

# ADDRESS flag: expose to other hosts (not just localhost)
kubectl port-forward --address 0.0.0.0 svc/my-api 8080:80 -n production
```

---

### kubectl apply, patch, delete — Modifying Resources

```bash
# Apply manifest (create or update)
kubectl apply -f pod.yaml
kubectl apply -f ./manifests/                    # all files in directory
kubectl apply -f ./manifests/ -R                 # recursive subdirectories
kubectl apply -f https://example.com/deploy.yaml # from URL

# Dry run — what would happen without applying
kubectl apply -f pod.yaml --dry-run=client       # client-side validation only
kubectl apply -f pod.yaml --dry-run=server       # server validates + admission webhooks

# Diff — show what would change
kubectl diff -f deployment.yaml
kubectl diff -f ./manifests/

# Patch resources (without full YAML)
kubectl patch deployment my-api \
  --patch '{"spec":{"replicas":5}}' -n production
kubectl patch deployment my-api \
  --type=json \
  --patch='[{"op":"replace","path":"/spec/replicas","value":5}]'

# Edit resource in-place (opens $EDITOR)
kubectl edit deployment my-api -n production
kubectl edit configmap app-config -n production

# Scale
kubectl scale deployment my-api --replicas=10 -n production
kubectl scale statefulset postgres --replicas=3 -n production

# Set image (trigger rolling update)
kubectl set image deployment/my-api app=myapp:v1.2.3 -n production

# Delete resources
kubectl delete pod my-pod -n production
kubectl delete -f pod.yaml
kubectl delete pods -l app=my-api -n production
kubectl delete pods --all -n production           # delete ALL pods in namespace
kubectl delete pod my-pod --grace-period=0 --force  # force delete (use sparingly)

# Rollout management
kubectl rollout status deployment/my-api -n production
kubectl rollout history deployment/my-api -n production
kubectl rollout undo deployment/my-api -n production
kubectl rollout undo deployment/my-api --to-revision=3 -n production
kubectl rollout restart deployment/my-api -n production  # rolling restart
```

---

### kubectl Power Commands

```bash
# Copy files between pod and local
kubectl cp my-pod:/app/logs/app.log ./app.log -n production
kubectl cp ./config.yaml my-pod:/etc/config/ -n production -c app

# Run a temporary debug pod
kubectl run debug --image=nicolaka/netshoot \
  --rm -it --restart=Never \
  -n production -- /bin/bash
# --rm: delete pod when you exit
# --restart=Never: run as bare pod (not deployment)

# Top (resource usage)
kubectl top pods -n production --containers
kubectl top nodes
kubectl top pods -A --sort-by=memory

# API resource discovery
kubectl api-resources                            # all resource types
kubectl api-resources --namespaced=true          # namespaced only
kubectl api-resources --api-group=apps           # specific API group
kubectl explain pod                              # field documentation
kubectl explain pod.spec.containers.resources    # nested field docs

# Check cluster info
kubectl cluster-info
kubectl version --short
kubectl get componentstatuses                    # API Server, etcd, scheduler health

# Generate YAML without applying (--dry-run=client -o yaml)
kubectl create deployment my-api --image=myapp:v1 \
  --replicas=3 --dry-run=client -o yaml > deployment.yaml
kubectl create secret generic my-secret \
  --from-literal=password=abc123 \
  --dry-run=client -o yaml > secret.yaml

# Verbose output for debugging API calls
kubectl get pods -v=6    # shows HTTP requests
kubectl get pods -v=8    # shows request + response headers
kubectl get pods -v=10   # maximum verbosity (full body)

# Auth inspection
kubectl auth can-i create pods -n production
kubectl auth can-i '*' '*' --all-namespaces     # check if cluster-admin
kubectl auth whoami                              # show current identity
```

---

### 🎤 Short Crisp Interview Answer

> *"The core kubectl workflow: get for listing resources with -o wide/yaml/json/jsonpath output, describe for full spec+status+events, logs for container output with --previous for crashes and -f for streaming, exec -it for interactive debugging, and port-forward for local access to cluster services without exposing them. The most useful debugging sequence for a failing pod: kubectl get pod (status), kubectl describe pod (events and conditions), kubectl logs --previous (last crash output), kubectl exec -it (interactive investigation). For distroless images with no shell, kubectl debug --image=busybox adds an ephemeral container."*

---

---

# 9.3 Namespaces in Practice

## 🟢 Beginner

### (Covered extensively in Category 6 topic 6.2)

This topic focuses on **operational patterns** for namespace management.

---

### Namespace Operations

```bash
# Create namespace (declarative)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: feature-xyz
  labels:
    team: payments
    environment: development
    pod-security.kubernetes.io/enforce: restricted
EOF

# Create namespace (imperative)
kubectl create namespace feature-xyz

# List all namespaces with labels
kubectl get namespaces --show-labels

# Set default namespace for session
kubectl config set-context --current --namespace=production

# Move a resource to another namespace (copy + delete)
# There is NO kubectl move — namespaces cannot be changed on existing resources
kubectl get deployment my-api -n old-ns -o yaml | \
  sed 's/namespace: old-ns/namespace: new-ns/' | \
  kubectl apply -f -
kubectl delete deployment my-api -n old-ns

# Delete namespace (cascade deletes EVERYTHING inside)
kubectl delete namespace feature-xyz
# ⚠️ This deletes ALL resources: pods, pvcs, secrets, services, etc.

# Namespace stuck in Terminating?
kubectl get namespace feature-xyz -o json | \
  python3 -c "import json,sys; d=json.load(sys.stdin); d['spec']['finalizers']=[]; print(json.dumps(d))" | \
  kubectl replace --raw /api/v1/namespaces/feature-xyz/finalize -f -
# Removes finalizers — forces namespace deletion

# Namespace resource summary
kubectl get all -n production
kubectl get events -n production --sort-by='.lastTimestamp' | tail -20

# Cross-namespace access pattern (services)
# From production namespace, reach staging service:
# curl http://my-svc.staging.svc.cluster.local:8080
# Full DNS: <service>.<namespace>.svc.<cluster-domain>
```

---

### Namespace Quotas and Limits (Production Pattern)

```yaml
# Complete namespace setup: namespace + quota + limitrange
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments-prod
  labels:
    team: payments
    environment: production
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-payments-quota
  namespace: team-payments-prod
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    services: "20"
    persistentvolumeclaims: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-payments-limits
  namespace: team-payments-prod
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "4"
      memory: "4Gi"
```

---

---

# ⚠️ 9.4 Node Lifecycle — Cordon, Drain, Uncordon

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

**Cordon, drain, and uncordon** are the three operations for safely removing a node from active use — for maintenance, upgrade, or decommissioning — without disrupting running workloads.

---

### The Three Operations Explained

```
NODE LIFECYCLE OPERATIONS
═══════════════════════════════════════════════════════════════

CORDON (kubectl cordon <node>):
  Marks the node as Unschedulable.
  Effect: No NEW pods scheduled on this node.
  Effect: EXISTING pods continue running (not affected).
  Mechanism: Sets node.spec.unschedulable: true
             Automatically adds taint: node.kubernetes.io/unschedulable:NoSchedule
  Use: First step before any node work. Stop new traffic arriving.
  Analogy: Put up a "no new guests" sign on a hotel room.

DRAIN (kubectl drain <node>):
  Evicts ALL pods from the node gracefully.
  Prerequisite: Node should be cordoned first (drain cordons automatically).
  Effect: Each pod gets SIGTERM → graceful shutdown period → SIGKILL.
  Effect: ReplicaSet/Deployment pods reschedule on other nodes.
  Effect: DaemonSet pods: skipped by default (use --ignore-daemonsets).
  Effect: Pods with local storage: blocked by default (use --delete-emptydir-data).
  Effect: Pods without a controller: blocked (use --force).
  Respects: PodDisruptionBudgets — waits if eviction would breach PDB.
  Use: After cordon, before maintenance.

UNCORDON (kubectl uncordon <node>):
  Marks the node as Schedulable again.
  Effect: New pods can schedule on this node again.
  Effect: Does NOT automatically reschedule existing pods (they stay where they are).
  Use: After maintenance is complete, node is healthy.

COMPLETE MAINTENANCE WORKFLOW:
  1. kubectl cordon worker-2          # stop new pods
  2. kubectl drain worker-2 \
       --ignore-daemonsets \
       --delete-emptydir-data \
       --grace-period=60             # evict pods gracefully
  3. <perform maintenance>           # OS upgrade, kernel patch, hardware
  4. kubectl uncordon worker-2       # re-enable for scheduling
```

---

### kubectl drain — Flags and Behavior

```bash
# Basic drain (safest — respects all protections)
kubectl drain worker-2

# Most common production drain command:
kubectl drain worker-2 \
  --ignore-daemonsets \          # skip DaemonSet pods (they're always present)
  --delete-emptydir-data \       # allow evicting pods with emptyDir volumes
  --grace-period=60 \            # give pods 60s graceful shutdown
  --timeout=300s \               # fail if drain takes > 5 minutes
  --force                        # evict pods without a controller (bare pods)
  # ⚠️ --force deletes pods with no owner (they won't reschedule!)

# Dry-run to see what would be evicted
kubectl drain worker-2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --dry-run

# What drain evicts:
#   Evicted:   Regular pods (Deployment, ReplicaSet, StatefulSet)
#   Skipped:   DaemonSet pods (use --ignore-daemonsets to proceed)
#   Blocked:   Pods with local storage (use --delete-emptydir-data)
#   Blocked:   Pods with no controller (use --force)
#   Blocked:   Pods where eviction would breach PDB (WAITS until safe)

# If drain is stuck (waiting for PDB):
# Check which PDB is blocking
kubectl get pdb -A
kubectl describe pdb -n production my-api-pdb
# Shows: minAvailable: 2, current: 2, allowed disruptions: 0
# → Must scale up before draining, OR temporarily reduce minAvailable

# Monitor drain progress
watch kubectl get pods -n production -o wide

# Force-complete a stuck drain (dangerous — data loss risk)
kubectl delete pod <stuck-pod> --grace-period=0 --force -n production
# Use only as last resort after diagnosing why pod won't terminate

# After maintenance:
kubectl uncordon worker-2
# Verify node is schedulable:
kubectl get nodes
# NAME       STATUS   ROLES    AGE   VERSION
# worker-2   Ready    <none>   30d   v1.28.5   ← Ready (not SchedulingDisabled)
```

---

### PodDisruptionBudget Interaction

```yaml
# PDB protects pods during drain
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-api-pdb
  namespace: production
spec:
  minAvailable: 2              # at least 2 pods must remain Available
  # OR: maxUnavailable: 1     # at most 1 pod may be unavailable
  selector:
    matchLabels:
      app: my-api

# DRAIN BEHAVIOR WITH PDB:
# Scenario: 3 replicas, PDB minAvailable: 2, draining worker-2
#
# Initial: worker-1=pod-A, worker-2=pod-B, worker-3=pod-C  → 3 Available
#
# kubectl drain worker-2:
#   Try to evict pod-B
#   After eviction: worker-1=pod-A, worker-3=pod-C → 2 Available
#   2 >= minAvailable(2) → eviction ALLOWED ✓
#
# Result: drain succeeds, pod-B evicted, reschedules elsewhere

# BLOCKING SCENARIO:
# Scenario: 2 replicas, PDB minAvailable: 2, draining worker-2
#
# Initial: worker-2=pod-A, worker-3=pod-B → 2 Available
#
# kubectl drain worker-2:
#   Try to evict pod-A
#   After eviction: only pod-B → 1 Available
#   1 < minAvailable(2) → eviction BLOCKED
#   drain WAITS INDEFINITELY
#
# Fix: kubectl scale deployment my-api --replicas=3
#      Then drain proceeds (2 Available minimum maintained)
```

---

### Node Conditions and Taints

```bash
# Check node status and conditions
kubectl describe node worker-2
# Conditions:
#   Type              Status  Reason
#   MemoryPressure    False   KubeletHasSufficientMemory
#   DiskPressure      False   KubeletHasSufficientDisk
#   PIDPressure       False   KubeletHasSufficientPID
#   Ready             True    KubeletReady
#
# Taints:
#   node.kubernetes.io/unschedulable:NoSchedule   ← added by cordon

# View all nodes with their status
kubectl get nodes -o wide
# NAME       STATUS                     ROLES    AGE   VERSION
# worker-1   Ready                      <none>   30d   v1.28.5
# worker-2   Ready,SchedulingDisabled   <none>   30d   v1.28.5  ← cordoned

# Check which pods are on a specific node
kubectl get pods -A -o wide | grep worker-2
# OR:
kubectl get pods -A --field-selector=spec.nodeName=worker-2

# Manually taint a node (for other purposes)
kubectl taint nodes worker-2 maintenance=true:NoExecute
# NoExecute: evicts existing pods without toleration
# This is what kubectl drain does internally + the eviction API

# Remove taint manually
kubectl taint nodes worker-2 maintenance=true:NoExecute-
```

---

### 🎤 Short Crisp Interview Answer

> *"The node maintenance workflow is cordon → drain → maintain → uncordon. Cordon marks the node Unschedulable, preventing new pods but keeping existing ones running. Drain evicts all pods gracefully: it sends SIGTERM, waits the grace period, then SIGKILL. Critically, drain respects PodDisruptionBudgets — if evicting a pod would breach a PDB's minAvailable, drain WAITS until the eviction is safe. DaemonSet pods are skipped by default (--ignore-daemonsets flag). If drain hangs, the most common cause is a PDB blocking eviction — check with kubectl get pdb and scale up the deployment before draining. Uncordon re-enables scheduling but doesn't automatically move pods back to the node."*

---

### ⚠️ Gotchas

1. **drain --force deletes uncontrolled pods permanently** — pods without an owner (ReplicaSet, Deployment) will be deleted and NOT rescheduled. Use --force only if you understand what you're deleting.
2. **PDB with minAvailable = replicas blocks all drains** — if `minAvailable` equals the current replica count, NO pods can be evicted. Always leave at least one unit of slack (minAvailable = replicas - 1).
3. **StatefulSet pods don't reschedule to the same storage** — draining a node with a StatefulSet pod using local PVs will leave the pod Pending until the original node returns (local PV can't follow the pod).
4. **Uncordon doesn't rebalance** — after uncordon, pods don't automatically move back. The Descheduler is needed for rebalancing (Category 6 topic 6.13).

---

---

# ⚠️ 9.5 Cluster Upgrades — Control Plane & Node Upgrade Strategy

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Kubernetes clusters require regular version upgrades for security patches and new features. The standard process is to upgrade the **control plane first**, then upgrade **nodes one by one** (rolling node upgrade). Understanding version skew policy — which component versions can coexist — is critical for planning safe upgrades.

---

### Version Skew Policy

```
KUBERNETES VERSION SKEW POLICY
═══════════════════════════════════════════════════════════════

Kubernetes releases: 3 minor versions per year (e.g., 1.26, 1.27, 1.28)
Each version: supported for ~14 months (1 year + ~2 months)

SUPPORTED VERSION SKEW:
  kube-apiserver:     latest version (e.g., 1.28)
  kube-controller-manager: must be same or one minor OLDER (1.27 or 1.28)
  kube-scheduler:     must be same or one minor OLDER (1.27 or 1.28)
  kubelet:            must be same or up to TWO minor OLDER (1.26, 1.27, 1.28)
  kubectl:            must be within one minor EITHER direction (1.27-1.29)

RULE: API Server is ALWAYS the first to upgrade.
RULE: Nodes (kubelet) can be up to 2 minor versions behind API Server.
  This allows nodes to be upgraded gradually without cluster downtime.
  You can have some nodes on 1.26, some on 1.27, API Server on 1.28.

UPGRADE ORDER (never skip versions):
  1.25 → 1.26 → 1.27 → 1.28
  CANNOT: 1.25 → 1.28 (must do each minor version)

UPGRADE PATH:
  1. Control plane (API Server, Controller Manager, Scheduler, etcd)
  2. Nodes (kubelet, kube-proxy) — one at a time or in batches
```

---

### EKS Cluster Upgrade Process

```bash
# EKS: AWS manages the control plane upgrade — you just initiate it

# STEP 0: Pre-upgrade checks
# Check current version
kubectl version --short
aws eks describe-cluster --name my-cluster --query 'cluster.version'
# 1.27

# Check deprecated API usage (will break after upgrade)
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
# Or use: kubent (Kubernetes No Trouble) tool
kubent
# 🔍 Checking for deprecated APIs in 1.28...
# [WARNING] Found deprecated API: flowcontrol.apiserver.k8s.io/v1beta2

# Check add-on compatibility
aws eks describe-addon-versions --kubernetes-version 1.28
# Shows which addon versions support K8s 1.28

# Check node group versions
aws eks list-nodegroups --cluster-name my-cluster
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name general-workers

# STEP 1: Upgrade control plane (EKS manages API Server, etcd, etc.)
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.28

# Monitor upgrade progress (~15-30 minutes)
aws eks describe-cluster --name my-cluster \
  --query 'cluster.status'
# UPDATING → ACTIVE

# Watch API Server accessibility during upgrade
while true; do
  kubectl get nodes 2>&1 | head -1
  sleep 10
done
# EKS control plane upgrade has <5 minutes of API Server unavailability

# STEP 2: Upgrade EKS add-ons (before node upgrade)
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.15.3-eksbuild.1

aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name coredns \
  --addon-version v1.10.1-eksbuild.6

aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name kube-proxy \
  --addon-version v1.28.4-eksbuild.4

# STEP 3: Upgrade node groups
# Option A: Rolling update of managed node group
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name general-workers \
  --kubernetes-version 1.28
# EKS cordons+drains old nodes, launches new nodes, one at a time

# Option B: Karpenter nodes — just update EC2 AMI and Karpenter drift handles it
# Karpenter sees nodes are on old AMI → terminates and replaces

# STEP 4: Verify
kubectl get nodes
# All nodes should show v1.28.x
kubectl get pods -A | grep -v Running | grep -v Completed
# All pods should be Running
```

---

### Self-Managed Cluster Upgrade (kubeadm)

```bash
# kubeadm upgrade workflow (control plane node)

# STEP 1: Upgrade kubeadm on control plane node
apt-get update
apt-get install -y kubeadm=1.28.0-00

# STEP 2: Verify upgrade plan
kubeadm upgrade plan
# [upgrade/versions] Cluster version: v1.27.5
# [upgrade/versions] kubeadm version: v1.28.0
# Components that must be upgraded manually after you upgrade the API server:
# COMPONENT   CURRENT   TARGET
# kubelet     v1.27.5   v1.28.0
# ...
# Upgrade to the latest stable version:
# COMPONENT            CURRENT   TARGET
# kube-apiserver       v1.27.5   v1.28.0   ← will upgrade
# kube-controller-manager v1.27.5 v1.28.0
# kube-scheduler       v1.27.5   v1.28.0
# kube-proxy           v1.27.5   v1.28.0
# CoreDNS              v1.10.0   v1.10.1
# etcd                 3.5.6     3.5.9

# STEP 3: Apply control plane upgrade
kubeadm upgrade apply v1.28.0
# Drains the node (if single control plane) → upgrades → uncordons

# STEP 4: Upgrade kubelet and kubectl on control plane
apt-get install -y kubelet=1.28.0-00 kubectl=1.28.0-00
systemctl daemon-reload
systemctl restart kubelet

# Verify control plane is upgraded
kubectl get nodes
# NAME          STATUS   VERSION
# control-1     Ready    v1.28.0   ← upgraded

# STEP 5: For each worker node (repeat per node):
# On control plane: cordon the worker
kubectl cordon worker-1

# On control plane: drain the worker
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# On the worker node:
ssh worker-1
apt-get update
apt-get install -y kubeadm=1.28.0-00
kubeadm upgrade node                   # upgrade node configuration
apt-get install -y kubelet=1.28.0-00
systemctl daemon-reload
systemctl restart kubelet

# On control plane: uncordon
kubectl uncordon worker-1

# Verify worker upgraded
kubectl get nodes
# worker-1    Ready    v1.28.0

# STEP 6: Repeat for all workers
```

---

### Upgrade Strategy for Zero-Downtime

```
ZERO-DOWNTIME UPGRADE REQUIREMENTS
═══════════════════════════════════════════════════════════════

BEFORE UPGRADING:
  ✓ All Deployments have replicas > 1 (no single points of failure)
  ✓ PodDisruptionBudgets defined for all critical services
    (ensures at least 1 replica stays running during drain)
  ✓ Pod resource requests set (scheduler can find new nodes)
  ✓ Liveness/readiness probes configured (traffic only to healthy pods)
  ✓ Deprecation check: no deprecated API versions in use
  ✓ Helm/Kustomize manifests updated for new API versions
  ✓ Add-on compatibility verified

UPGRADE NODE STRATEGY:
  Option A: One node at a time (safest, slowest)
    Cordon → Drain → Upgrade → Uncordon → Verify → Next node
    Downtime: 0 (if replicas > 1 and PDB in place)
    Time: N nodes × ~10 minutes per node

  Option B: Replace nodes (blue-green node groups)
    Create new node group with v1.28 nodes
    Cordon ALL old nodes
    Drain old nodes progressively
    Verify all pods on new nodes
    Delete old node group
    Downtime: 0 (pods move to new nodes)
    Time: ~30 minutes regardless of node count (parallel)
    RECOMMENDED for large clusters

  Option C: In-place parallel upgrade (fastest, higher risk)
    Upgrade 20-30% of nodes simultaneously
    Monitor error rate during upgrade
    Faster but brief service degradation if too many pods evicted

POST-UPGRADE VERIFICATION:
  kubectl get nodes                              # all nodes new version
  kubectl get pods -A | grep -v Running          # no stuck pods
  kubectl get events -A | grep Warning           # no warning storms
  kubectl get componentstatuses                  # API Server, etcd healthy
  curl https://api.production.com/health         # external health check
  Check Grafana dashboards: error rate, latency  # SLOs not breached
```

---

### 🎤 Short Crisp Interview Answer

> *"Cluster upgrades follow a strict order: control plane first, then nodes one minor version at a time — you cannot skip versions. The version skew policy allows kubelets up to two minor versions behind the API Server, so you can upgrade nodes gradually without cluster downtime. On EKS, the control plane upgrade is a managed operation — you call aws eks update-cluster-version and AWS handles the API Server, etcd, and scheduler upgrade with minimal API downtime. Then upgrade add-ons (VPC CNI, CoreDNS, kube-proxy) and finally node groups. Before any upgrade: check deprecated API usage with kubent, ensure PDBs are in place, and verify add-on compatibility. For large clusters, the blue-green node group strategy — creating a new node group with the target version and migrating pods — is faster and safer than upgrading nodes in-place."*

---

### ⚠️ Gotchas

1. **Deprecated API removal breaks existing manifests** — Kubernetes removes deprecated APIs at specific versions. A Deployment using apps/v1beta1 would fail after that API version is removed. Run kubent before every upgrade.
2. **etcd version must be upgraded alongside API Server** — kubeadm handles this automatically, but manual etcd upgrades must happen in concert with the API Server.
3. **EKS control plane upgrade takes ~15-30 minutes** — during this time, the API Server may be briefly unavailable (< 5 minutes typically). Ensure your monitoring doesn't alert on expected API server unavailability during upgrades.
4. **Add-ons must be upgraded BEFORE node groups** — VPC CNI and kube-proxy versions must be compatible with the new K8s version. Upgrade add-ons first, then node groups.

---

---

# ⚠️ 9.6 etcd Backup & Restore

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

etcd is **the single source of truth for all cluster state** — every Deployment, ConfigMap, Secret, Service exists as bytes in etcd. Losing etcd without a backup means losing the entire cluster configuration permanently. Regular backups and tested restores are mandatory for production clusters.

---

### What's in etcd

```
WHAT ETCD STORES (EVERYTHING):
═══════════════════════════════════════════════════════════════

/registry/pods/<namespace>/<pod-name>
/registry/deployments/<namespace>/<deployment-name>
/registry/services/specs/<namespace>/<service-name>
/registry/secrets/<namespace>/<secret-name>
/registry/configmaps/<namespace>/<configmap-name>
/registry/namespaces/<namespace-name>
/registry/nodes/<node-name>
/registry/persistentvolumes/<pv-name>
/registry/persistentvolumeclaims/<namespace>/<pvc-name>
/registry/clusterroles/<name>
/registry/storageclasses/<name>
... and every CRD instance, every ServiceAccount, etc.

WHAT ETCD DOES NOT STORE:
  ✗ Container images (stored in registry)
  ✗ Actual application data (stored in PVs/databases)
  ✗ Node local state (host filesystem, logs)
  ✗ Secrets values if using external Secret Manager (ESO)

ETCD BACKUP = CLUSTER CONFIGURATION BACKUP
  Restoring from backup restores the desired state (YAML config).
  Running applications on nodes will reconcile to desired state.
  PV data is separate — must be backed up independently.
```

---

### etcd Backup

```bash
# Method 1: etcdctl snapshot (self-managed clusters — access to etcd)

# Environment setup
export ETCDCTL_API=3
export ETCD_ENDPOINTS=https://127.0.0.1:2379
export ETCD_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCD_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCD_KEY=/etc/kubernetes/pki/etcd/server.key

# Take snapshot
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Verify snapshot integrity
etcdctl snapshot status /backup/etcd-snapshot-2024-01-15-103000.db \
  --write-out=table
# +---------+----------+------------+------------+
# |   HASH  | REVISION | TOTAL KEYS | TOTAL SIZE |
# +---------+----------+------------+------------+
# | 5b4e8f9 |   123456 |       2847 |    45.2 MB |
# +---------+----------+------------+------------+

# List etcd keys to verify content
etcdctl get /registry/namespaces --prefix --keys-only \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY
# /registry/namespaces/default
# /registry/namespaces/kube-system
# /registry/namespaces/production
# ...

# Automated backup CronJob (self-managed)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"        # every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true        # need access to etcd on host network
          tolerations:
          - operator: Exists
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          containers:
          - name: backup
            image: bitnami/etcd:3.5.9
            command:
            - /bin/sh
            - -c
            - |
              SNAPSHOT_FILE="/backup/etcd-$(date +%Y%m%d-%H%M%S).db"
              etcdctl snapshot save $SNAPSHOT_FILE \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              # Upload to S3
              aws s3 cp $SNAPSHOT_FILE s3://my-cluster-etcd-backups/
              # Cleanup local files older than 3 days
              find /backup -name "*.db" -mtime +3 -delete
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-storage
            emptyDir: {}

# EKS: etcd is managed by AWS — backup strategy is different
# EKS does NOT expose etcd → cannot use etcdctl
# EKS backup alternatives:
#   1. Velero: backs up Kubernetes resources (not etcd bytes, but K8s objects)
#   2. cluster-backup-operator: exports all K8s resources to S3
#   3. AWS EKS backup policy: AWS maintains etcd, cluster infra in IaC (Terraform)
```

---

### etcd Restore

```bash
# Method 1: Restore from snapshot (self-managed clusters)
# WARNING: This replaces ALL cluster state with snapshot state

# STEP 1: Stop API Server on all control plane nodes
# (move kube-apiserver.yaml out of /etc/kubernetes/manifests/)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# STEP 2: Restore snapshot to new data directory
etcdctl snapshot restore /backup/etcd-snapshot-2024-01-15-103000.db \
  --data-dir=/var/lib/etcd-restore \
  --name=master-1 \
  --initial-cluster="master-1=https://10.0.1.5:2380" \
  --initial-cluster-token=etcd-cluster-restore \
  --initial-advertise-peer-urls=https://10.0.1.5:2380
# Creates /var/lib/etcd-restore with restored data

# STEP 3: Update etcd manifest to use restored data directory
# Edit /etc/kubernetes/manifests/etcd.yaml:
# Change: --data-dir=/var/lib/etcd
# To:     --data-dir=/var/lib/etcd-restore

# STEP 4: Move API Server manifest back
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# STEP 5: Wait for API Server to come up
kubectl get nodes
# Should return node list from snapshot state

# STEP 6: Force re-sync of running state
# Deployments/ReplicaSets will reconcile automatically
# Controllers detect difference between etcd state and reality

# HA etcd restore (3 etcd nodes):
# Restore must be performed on ALL etcd nodes simultaneously
# Each gets the same snapshot but different --name and --initial-cluster

# FOR EKS: etcd restore = cluster replacement
# You cannot restore etcd on EKS.
# Disaster recovery = recreate cluster from IaC + redeploy from GitOps.
# RECOMMENDATION for EKS: Velero for K8s resource backup
```

---

### Velero — Kubernetes Resource Backup

```bash
# Velero: backs up Kubernetes objects and PV data
# Works on EKS and self-managed clusters
# Stores backups in S3 (or GCS, Azure Blob)

# Install Velero (EKS with S3)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./velero-credentials
  # --use-node-agent     # enable for PV backup with restic

# Create backup
velero backup create cluster-backup-2024-01-15 \
  --include-namespaces production,staging \
  --include-cluster-resources=true    # include ClusterRoles, etc.

# Create scheduled backup
velero schedule create daily-backup \
  --schedule="0 2 * * *" \            # 2 AM daily
  --include-namespaces production,staging,monitoring \
  --ttl 720h                          # keep for 30 days

# List backups
velero backup get
# NAME                          STATUS     ERRORS   WARNINGS   CREATED              TTL
# cluster-backup-2024-01-15    Completed  0        0          2024-01-15 10:30:00   720h

# Restore from backup
velero restore create --from-backup cluster-backup-2024-01-15
velero restore create my-restore \
  --from-backup cluster-backup-2024-01-15 \
  --include-namespaces production         # restore just production
velero restore get                        # check restore status
```

---

### 🎤 Short Crisp Interview Answer

> *"etcd is the single source of truth for all cluster state — every K8s object exists as bytes in etcd. The backup tool is etcdctl snapshot save, which takes an atomic point-in-time snapshot. Verify snapshots with etcdctl snapshot status. Restores require stopping the API Server, running etcdctl snapshot restore to a new data directory, updating the etcd manifest to point to it, and restarting. The key operational points: back up every 6 hours minimum, verify snapshot integrity, store backups off-cluster (S3), and TEST restores regularly — an untested backup is not a backup. On EKS, etcd is fully managed by AWS and inaccessible — use Velero to back up Kubernetes resources and PV data to S3."*

---

### ⚠️ Gotchas

1. **etcd snapshot is a point-in-time — any changes after snapshot are lost** — restoring from a 6-hour-old snapshot loses 6 hours of configuration changes. Shorter backup intervals reduce this window.
2. **HA etcd restore must be simultaneous on all nodes** — restoring etcd on only one of three nodes causes the restored node to be rejected by the other two (Raft quorum conflict). All nodes must be restored from the same snapshot simultaneously.
3. **PV data is NOT in etcd** — etcd backup restores PVC objects (the claims) but not the data inside the volumes. PV data requires separate backup (Velero with restic, or cloud snapshot).
4. **EKS cannot use etcdctl** — AWS doesn't expose etcd endpoints on managed EKS clusters. Velero is the backup solution for EKS.

---

---

# 9.7 Helm — Charts, Values, Releases, Hooks

## 🟡 Intermediate

### What it is in simple terms

**Helm** is the package manager for Kubernetes. A **chart** is a package of pre-configured Kubernetes resources. Helm lets you deploy complex applications (Prometheus, PostgreSQL, Ingress controllers) with a single command, manage configuration through **values**, upgrade and rollback **releases**, and orchestrate pre/post install actions via **hooks**.

---

### Helm Core Concepts

```
HELM COMPONENTS
═══════════════════════════════════════════════════════════════

CHART:
  A collection of YAML templates + default values
  Structure:
    my-chart/
    ├── Chart.yaml        # chart metadata (name, version, appVersion)
    ├── values.yaml       # default values
    ├── templates/        # Kubernetes YAML templates (Go template syntax)
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yaml
    │   ├── _helpers.tpl  # shared template functions
    │   └── NOTES.txt     # post-install notes
    ├── charts/           # sub-charts (dependencies)
    └── .helmignore

RELEASE:
  A deployed instance of a chart.
  Same chart deployed twice = two releases.
  Each release has: name, namespace, version history.
  helm install my-release ./my-chart
  → Release "my-release" created with all chart resources

VALUES:
  Configuration that overrides chart defaults.
  Hierarchy: chart defaults → -f values-file.yaml → --set key=value
  Later overrides earlier.

REPOSITORY:
  HTTP server hosting packaged charts (.tgz files) + index.yaml
  Examples: hub.helm.sh, artifacthub.io, custom company repos
```

---

### Helm Chart Structure

```yaml
# Chart.yaml
apiVersion: v2
name: my-api
description: My API service
type: application       # or library
version: 1.2.3          # chart version (you change this)
appVersion: "v2.5.1"    # application version (informational)
dependencies:
- name: postgresql
  version: "12.5.6"
  repository: "https://charts.bitnami.com/bitnami"
  condition: postgresql.enabled    # only install if values.postgresql.enabled=true

---
# values.yaml (default values)
replicaCount: 2

image:
  repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-api
  tag: "latest"             # override with specific tag in production
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true            # install postgresql sub-chart
  auth:
    username: myapp
    database: myappdb

ingress:
  enabled: false
  className: "nginx"
  hosts: []

---
# templates/deployment.yaml (Go template syntax)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-api.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-api.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        {{- if .Values.postgresql.enabled }}
        env:
        - name: DATABASE_URL
          value: "postgres://{{ .Values.postgresql.auth.username }}@{{ include "my-api.fullname" . }}-postgresql:5432/{{ .Values.postgresql.auth.database }}"
        {{- end }}
  {{- if .Values.autoscaling.enabled }}
  # HPA template only rendered when autoscaling.enabled=true
  {{- end }}
```

---

### Helm Commands

```bash
# Repository management
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                              # fetch latest charts

# Search charts
helm search repo prometheus                   # search local repos
helm search hub nginx                         # search Artifact Hub

# Install a release
helm install my-api ./my-chart                              # from local chart
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f my-prometheus-values.yaml \             # custom values file
  --set alertmanager.enabled=false \         # inline override
  --version 55.5.0                           # specific chart version

# Dry-run (see what would be installed without applying)
helm install my-api ./my-chart --dry-run --debug

# Upgrade (update existing release)
helm upgrade my-api ./my-chart \
  -f production-values.yaml \
  --set image.tag=v1.2.3 \
  --atomic \                                 # rollback automatically if upgrade fails
  --timeout 5m \                             # timeout for upgrade
  --cleanup-on-fail                          # delete new resources on failure

# Upgrade or install if not exists
helm upgrade --install my-api ./my-chart \
  -f production-values.yaml \
  --set image.tag=v1.2.3

# Rollback
helm rollback my-api                          # rollback to previous revision
helm rollback my-api 3                        # rollback to revision 3
helm history my-api                           # view all revisions

# List releases
helm list                                     # current namespace
helm list -A                                  # all namespaces
helm list -n monitoring

# Get release info
helm get values my-api                        # values used in current release
helm get values my-api --all                  # including defaults
helm get manifest my-api                      # rendered YAML

# Uninstall
helm uninstall my-api                         # removes all release resources
helm uninstall my-api --keep-history          # keep history for rollback

# Template (render locally without installing)
helm template my-api ./my-chart -f values.yaml
helm template my-api ./my-chart -f values.yaml > rendered.yaml

# Lint chart
helm lint ./my-chart -f values.yaml

# Package chart
helm package ./my-chart                       # creates my-chart-1.2.3.tgz
helm package ./my-chart --destination ./dist
```

---

### Helm Hooks

```yaml
# Hooks run at specific points in release lifecycle
# Common hooks: pre-install, post-install, pre-upgrade, post-upgrade,
#               pre-delete, post-delete, pre-rollback, post-rollback, test

# Example: database migration job (pre-upgrade hook)
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install   # run before install and upgrade
    "helm.sh/hook-weight": "-5"               # lower = runs first (negative before positive)
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
    # before-hook-creation: delete old job before creating new one
    # hook-succeeded: delete job after it succeeds
    # hook-failed: delete job after it fails
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["./migrate", "--up"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ .Release.Name }}-db-secret
              key: connection-string

# Helm test hook
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-test-connection
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "my-api.fullname" . }}:{{ .Values.service.port }}']

# Run tests:
helm test my-api
# TEST SUITE: my-api-test-connection
# Phase:    Succeeded
```

---

### 🎤 Short Crisp Interview Answer

> *"Helm is the package manager for Kubernetes. A chart is a package of templated Kubernetes YAML with default values. A release is a deployed instance of a chart in a namespace. Values override defaults via -f values.yaml files or --set flags, with later values taking precedence. Core commands: helm install (deploy), helm upgrade (update release), helm rollback (revert to previous revision), helm list (view releases), helm history (view revisions). Hooks run jobs at specific lifecycle points — pre-upgrade hooks for database migrations are the classic use case. The --atomic flag automatically rolls back if an upgrade fails, which is essential for production pipelines."*

---

### ⚠️ Gotchas

1. **helm upgrade --atomic rollback doesn't roll back PVs** — if your upgrade creates new PVCs and then fails, --atomic rolls back the Helm release but PVCs already created might remain.
2. **Values precedence: --set overrides -f values.yaml** — if the same key appears in both a values file and a --set flag, --set wins. Order within -f files: last file wins for duplicate keys.
3. **Helm secret storage** — Helm stores release state as Kubernetes Secrets in the release namespace. `kubectl get secrets -n production | grep helm` shows all Helm release metadata. Deleting these Secrets orphans resources (Helm loses track of the release).

---

---

# 9.8 Kustomize — Overlays, Patches

## 🟡 Intermediate

### What it is in simple terms

**Kustomize** is a **template-free customization tool** built into kubectl (`kubectl apply -k`). Instead of templating YAML with Go templates (like Helm), Kustomize transforms base Kubernetes YAML through patches, field overrides, and generators — keeping base YAML clean and environment-specific changes isolated in overlay directories.

---

### Kustomize Directory Structure

```
KUSTOMIZE DIRECTORY LAYOUT
═══════════════════════════════════════════════════════════════

base/                          # shared, environment-agnostic config
  kustomization.yaml
  deployment.yaml
  service.yaml
  configmap.yaml

overlays/
  dev/                         # development overrides
    kustomization.yaml
  staging/
    kustomization.yaml
  production/                  # production overrides
    kustomization.yaml
    hpa.yaml                   # production-specific resources
    replica-patch.yaml         # override replicas

FLOW:
  kubectl apply -k overlays/production/
  → Kustomize reads overlays/production/kustomization.yaml
  → Applies base resources
  → Applies all patches specified in overlay
  → Sends final YAML to API Server

KEY ADVANTAGE OVER HELM:
  Base YAML is valid Kubernetes YAML — readable and understandable
  No {{ .Values.xxx }} template syntax to parse mentally
  Patches are targeted and explicit
  Built into kubectl — no separate tool installation (kubectl apply -k)
```

---

### Kustomize YAML — Base and Overlays

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:                    # added to ALL resources
  app: my-api
  managed-by: kustomize

commonAnnotations:
  company.com/team: platform

---
# base/deployment.yaml (clean YAML — no templates)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
      - name: app
        image: my-api:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production            # override namespace for ALL resources

bases:
- ../../base                     # reference to base directory

resources:                       # production-only resources
- hpa.yaml

# IMAGE: override image tag for production
images:
- name: my-api                   # matches image name in deployment
  newName: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-api
  newTag: v1.2.3                 # specific tag for production

# PATCHES: modify base resources
patches:
# Strategic merge patch (merge-style)
- path: replica-patch.yaml

# Inline JSON6902 patch (surgical)
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-api
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: 500m
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: 1Gi

# CONFIGMAP GENERATOR: create ConfigMap from files
configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=INFO
  - DATABASE_POOL_SIZE=20
  - ENVIRONMENT=production

# SECRET GENERATOR: create Secret from files
secretGenerator:
- name: app-secrets
  files:
  - tls.crt
  - tls.key
  type: kubernetes.io/tls

# REPLICA COUNT OVERRIDE
replicas:
- name: my-api
  count: 5

---
# overlays/production/replica-patch.yaml (strategic merge patch)
# Strategic merge: existing fields merged, arrays merged by strategy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "1Gi"
        env:
        - name: LOG_LEVEL
          value: "INFO"
        - name: METRICS_ENABLED
          value: "true"

---
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

bases:
- ../../base

images:
- name: my-api
  newTag: dev-latest

patches:
- target:
    kind: Deployment
    name: my-api
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1               # single replica in dev to save resources

configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=DEBUG
  - DATABASE_POOL_SIZE=2
  - ENVIRONMENT=development
```

---

### kubectl Kustomize Commands

```bash
# Apply a kustomization
kubectl apply -k overlays/production/
kubectl apply -k overlays/dev/

# Preview output (render without applying)
kubectl kustomize overlays/production/        # render to stdout
kubectl kustomize overlays/production/ > rendered.yaml

# Standalone kustomize CLI
kustomize build overlays/production/
kustomize build overlays/production/ | kubectl apply -f -
kustomize build overlays/production/ | kubectl diff -f -

# Edit base (helper for quick changes)
kustomize edit set image my-api:v1.2.3 -n overlays/production
kustomize edit set replicas my-api=5 -n overlays/production

# Build and apply in CI/CD pipeline
kustomize build overlays/production/ | kubectl apply --server-side -f -
# --server-side: server-side apply (better for large manifests)

# Diff before applying
kustomize build overlays/production/ | kubectl diff -f -
```

---

### Helm vs Kustomize — When to Use Which

```
HELM vs KUSTOMIZE DECISION GUIDE
═══════════════════════════════════════════════════════════════

USE HELM WHEN:
  ✓ Deploying third-party software (Prometheus, Nginx, PostgreSQL)
    → Chart exists → just install
  ✓ Distributing your own software to external customers
    → Charts are the standard packaging format
  ✓ Complex conditional logic in manifests
    → {{- if .Values.autoscaling.enabled }} blocks are powerful
  ✓ Strong release management needed
    → helm rollback, helm history are excellent

USE KUSTOMIZE WHEN:
  ✓ Managing your own internal microservices across environments
    → Overlays are clean and Git-diff-friendly
  ✓ GitOps pipelines (Argo CD native support)
    → argocd app create with --kustomize flag
  ✓ Simple environment differences (different image tags, replicas)
    → Kustomize does this cleanly without templates
  ✓ Team prefers reading plain YAML (no template syntax)
    → Base YAML is readable without Helm knowledge

USE BOTH (common in practice):
  Helm: install third-party charts (infra dependencies)
  Kustomize: manage your own application manifests
  Argo CD: supports both natively
```

---

### 🎤 Short Crisp Interview Answer

> *"Kustomize is a template-free way to manage Kubernetes YAML across environments. The base directory contains clean, valid Kubernetes YAML. Overlays reference the base and apply patches — strategic merge patches for field overrides and JSON6902 patches for surgical changes. The images field overrides image tags for a specific environment, configMapGenerator creates ConfigMaps from literals or files. Built into kubectl as kubectl apply -k. Compared to Helm: Kustomize keeps base YAML readable without template syntax, is better for managing your own services across environments, and integrates natively with Argo CD. Helm is better for packaging third-party software and complex conditional manifests."*

---

---

# ⚠️ 9.9 Cluster Autoscaler — How It Works, Scale-Down Safety

## 🔴 Advanced — HIGH PRIORITY

### What it is in simple terms

The **Cluster Autoscaler (CA)** automatically adjusts the **number of nodes** in a cluster based on pending pods and node utilization. When pods can't schedule because nodes are full, CA adds nodes. When nodes are underutilized for a sustained period, CA removes them. Scale-down is gated by strict safety checks to prevent disrupting running workloads.

---

### How Cluster Autoscaler Works

```
CLUSTER AUTOSCALER CONTROL LOOP
═══════════════════════════════════════════════════════════════

RUNS EVERY 10 SECONDS (configurable):

SCALE-UP TRIGGER:
  1. CA scans for pods in Pending state
  2. For each pending pod, CA simulates: "would adding a node from
     nodegroup X allow this pod to schedule?"
  3. If YES: CA calls cloud provider API to add a node
     AWS: increases ASG desired count
     On AWS/EKS: new EC2 instance launches (~2-3 min)
     Kubelet registers → node becomes Ready → pod schedules

SCALE-DOWN TRIGGER (more complex — many safety checks):
  1. Every 10 seconds, CA finds nodes where:
     a. All pods can fit on other nodes
     b. Node has been underutilized for scale-down-delay (default 10 min)
     c. node CPU requested < 50% AND memory requested < 50%
        (customizable via --scale-down-utilization-threshold=0.5)
  2. CA evaluates scale-down-safety checks (see below)
  3. If SAFE: node is cordoned, pods evicted, node terminated

SCALE-DOWN SAFETY CHECKS — ALL MUST PASS:
  ✓ No pods with local storage (emptyDir, hostPath) — unless annotated
  ✓ No pods from kube-system — unless annotated safe-to-evict
  ✓ No pods without a controller (bare pods)
  ✓ No pods that violate PodDisruptionBudget
  ✓ No pods with restrictive PodAntiAffinity that block rescheduling
  ✓ Node has been underutilized for scale-down-unneeded-time (10 min)
  ✓ No active scale-up is happening (not during scale-up events)

IMPORTANT TIMING:
  Scale-up:   Triggers after first unschedulable event → ~10s
  New node:   EC2 launch + register → 2-4 minutes total
  Scale-down: Node must be low-util for 10 minutes (default)
              After decision: drain + terminate → 2-3 more minutes
  Total scale-down: ~13-15 minutes from low-util to terminated
```

---

### Cluster Autoscaler on EKS — Installation

```yaml
# PREREQUISITE: Node group must have --balance-similar-node-groups
# and appropriate IAM permissions

# IAM policy for Cluster Autoscaler (attach to node group IAM role)
# OR use IRSA for the CA pod itself (recommended)
# {
#   "Statement": [
#     {
#       "Effect": "Allow",
#       "Action": [
#         "autoscaling:DescribeAutoScalingGroups",
#         "autoscaling:DescribeAutoScalingInstances",
#         "autoscaling:DescribeLaunchConfigurations",
#         "autoscaling:DescribeScalingActivities",
#         "autoscaling:SetDesiredCapacity",
#         "autoscaling:TerminateInstanceInAutoScalingGroup",
#         "ec2:DescribeLaunchTemplateVersions",
#         "ec2:DescribeInstanceTypes"
#       ],
#       "Resource": "*"
#     }
#   ]
# }

# Node group ASG tags (required for CA to discover node groups):
#   k8s.io/cluster-autoscaler/enabled = true
#   k8s.io/cluster-autoscaler/<cluster-name> = owned

---
# Cluster Autoscaler Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1                      # HA: 1 replica (uses leader election)
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    spec:
      serviceAccountName: cluster-autoscaler
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.2
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false  # allow scaling down nodes with emptyDir
        - --expander=least-waste               # expansion strategy (see below)
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled=true,k8s.io/cluster-autoscaler/my-cluster=owned
        - --balance-similar-node-groups        # balance across AZs
        - --scale-down-enabled=true
        - --scale-down-delay-after-add=10m     # wait after scale-up before scale-down
        - --scale-down-unneeded-time=10m       # how long must node be underutilized
        - --scale-down-utilization-threshold=0.5  # < 50% = underutilized
        - --max-node-provision-time=15m        # fail if node doesn't register in 15m
        - --max-graceful-termination-sec=600   # 10 min for pods to terminate
        resources:
          requests:
            cpu: 100m
            memory: 600Mi
          limits:
            cpu: 100m
            memory: 600Mi
        env:
        - name: AWS_REGION
          value: us-east-1
```

---

### Scale-Down Safety Annotations

```yaml
# Pods that block scale-down by default:
# 1. kube-system pods
# 2. Pods with local storage
# 3. Pods without a controller

# ANNOTATION: allow CA to evict kube-system pods during scale-down
# (applied to pods you control in kube-system)
apiVersion: v1
kind: Pod
metadata:
  namespace: kube-system
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    # Tells CA: it's OK to evict this pod for scale-down

# ANNOTATION: prevent CA from evicting a pod (protect critical workloads)
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    # CA will NEVER remove the node this pod is on

# NODE ANNOTATION: opt node out of scale-down permanently
kubectl annotate node worker-1 \
  cluster-autoscaler.kubernetes.io/scale-down-disabled=true

# Check CA activity
kubectl -n kube-system logs -f deployment/cluster-autoscaler | \
  grep -E "Scale|scale|expander|Node"
```

---

### Expander Strategies

```
EXPANDER STRATEGIES — HOW CA PICKS WHICH NODE GROUP TO EXPAND
═══════════════════════════════════════════════════════════════

least-waste (RECOMMENDED):
  Picks the node group that would have least CPU/memory waste
  after placing the pending pod.
  Most cost-efficient: doesn't over-provision.
  Use for: production clusters optimizing for cost.

random:
  Picks a random eligible node group.
  Use for: testing, when all node groups are equivalent.

most-pods:
  Picks the node group that allows the most pods to schedule.
  Use for: batch workloads with many small pending pods.

priority:
  Assigns priorities to node groups via ConfigMap.
  Highest priority node group expanded first.
  Use for: prefer spot over on-demand with fallback.

price:
  Uses cloud provider pricing API.
  Available for GKE, not fully supported on EKS.

---
# Priority expander ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - spot-.*           # try spot node groups first (lower priority number = lower priority... wait:)
    # CONFUSING: HIGHER number = HIGHER priority
    50:
      - on-demand-.*      # on-demand as fallback
    # Actually: 50 > 10 → on-demand preferred over spot
    # For spot-first: set spot=50, on-demand=10
```

---

### Karpenter — CA Alternative on EKS

```
KARPENTER vs CLUSTER AUTOSCALER
═══════════════════════════════════════════════════════════════

CLUSTER AUTOSCALER:
  Works with: pre-defined node groups (ASGs)
  Picks from: fixed set of instance types per node group
  Speed: 2-4 minutes (ASG launch + registration)
  Bin packing: No — adds whole nodes from ASG
  Multi-cloud: Yes (AWS, GCP, Azure, etc.)

KARPENTER (EKS-specific, AWS):
  Works with: EC2 directly (no ASGs needed)
  Picks from: ANY instance type that fits the pending pod
  Speed: 30-60 seconds (direct EC2 API)
  Bin packing: YES — consolidation moves pods to fewer nodes
  Spot fallback: Native fallback to on-demand if spot unavailable
  Drift detection: Replaces nodes on new AMI versions automatically

KARPENTER CONSOLIDATION (key differentiator):
  Periodically checks if pods can fit on fewer nodes
  If yes: cordons + drains underutilized node → terminates it
  Reprovisions: may replace with a smaller, cheaper instance type
  Result: significant cost savings vs CA (20-40% typical)

WHEN TO USE EACH:
  EKS new cluster:        Karpenter (faster, cheaper, more flexible)
  Multi-cloud:            Cluster Autoscaler
  Existing CA clusters:   Migrate to Karpenter gradually (they can coexist)
  Need strict node types: Either (both support node selectors)
```

---

### 🎤 Short Crisp Interview Answer

> *"The Cluster Autoscaler runs every 10 seconds. For scale-up, it detects Pending pods and simulates whether adding a node from any configured node group would allow the pod to schedule — if yes, it calls the cloud API to add a node. Scale-down is much more conservative: a node must be below the utilization threshold (50% CPU and memory requested) for 10 minutes, AND all its pods must be safely evictable — no PDB violations, no bare pods, no pods with safe-to-evict: false annotation. Scale-down safety blocks are the most common operational pain point: a single kube-system pod without the safe-to-evict annotation will prevent a node from ever being removed. On EKS, Karpenter is the modern alternative — it launches EC2 instances directly in 30-60 seconds instead of 2-4 minutes, supports any instance type, and actively consolidates pods to fewer nodes for cost savings."*

---

### ⚠️ Gotchas

1. **CA doesn't know about actual resource usage** — CA decisions are based on requested resources, not actual usage. A node where all pods request 10% CPU but use 80% might be marked for scale-down (requests < 50%) while actually being busy.
2. **PodDisruptionBudget blocking scale-down indefinitely** — if a PDB's minAvailable equals the number of replicas and those replicas are spread across nodes, CA can never drain any node. Design PDBs with one unit of headroom.
3. **Scale-down-delay-after-add prevents thrashing** — by default, CA won't scale down for 10 minutes after a scale-up. This prevents adding a node, immediately deciding it's unneeded, and removing it in a rapid cycle.
4. **CA and HPA can conflict** — HPA scales pods up → pods need more nodes → CA adds nodes → HPA scales pods down → nodes are empty → CA removes nodes → repeat. Use `--scale-down-delay-after-add` and `--scale-down-unneeded-time` tuning to dampen oscillation.

---

---

# 9.10 Node Pools & Machine Sets (Cluster API)

## 🔴 Advanced

### What it is in simple terms

**Node pools** are groups of nodes with identical configuration (instance type, disk, labels, taints). Running multiple node pools lets you have heterogeneous infrastructure — CPU-optimized nodes for web servers, GPU nodes for ML workloads, spot nodes for batch jobs. **Cluster API (CAPI)** is the Kubernetes-native framework for declaratively managing cluster infrastructure — nodes and even entire clusters as Kubernetes objects.

---

### Node Pools on EKS

```
EKS NODE GROUP TYPES
═══════════════════════════════════════════════════════════════

MANAGED NODE GROUPS:
  AWS manages: EC2 launch template, AMI updates, drain-on-upgrade
  You configure: instance type, size range, labels, taints
  Upgrade: aws eks update-nodegroup-version → rolling node replacement
  Use for: general workloads, most production use cases

SELF-MANAGED NODE GROUPS:
  You manage: ASG, launch template, AMI selection, kubelet config
  Full control: any AMI, any kubelet flags, any custom user data
  Use for: custom AMI requirements, specific GPU drivers, Bottlerocket

FARGATE PROFILES:
  Serverless: no nodes to manage, pods run on AWS-managed infrastructure
  Limitations: no daemonsets, no stateful workloads, limited node selection
  Use for: infrequent batch jobs, burst workloads

COMMON MULTI-POOL PATTERN:
  system pool:       m5.large  × 3, on-demand, reserved for kube-system
  application pool:  m5.xlarge × auto, on-demand, general app workloads
  spot pool:         m5.2xlarge × auto, spot, batch/ML training
  gpu pool:          g4dn.xlarge × 0-10, on-demand, ML inference
```

---

### Node Pool YAML (EKS Managed Node Group via eksctl)

```yaml
# eksctl cluster with multiple node groups
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
  region: us-east-1
  version: "1.28"

managedNodeGroups:
# System node group — dedicated for system pods
- name: system-nodes
  instanceType: m5.large
  minSize: 3
  maxSize: 3                      # fixed size — don't autoscale system nodes
  desiredCapacity: 3
  availabilityZones: [us-east-1a, us-east-1b, us-east-1c]
  labels:
    role: system
    node-type: system
  taints:
  - key: CriticalAddonsOnly
    value: "true"
    effect: NoSchedule            # only critical pods tolerate this taint
  iam:
    attachPolicyARNs:
    - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
    - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# General application node group — autoscales
- name: app-nodes
  instanceType: m5.xlarge
  minSize: 3
  maxSize: 50
  desiredCapacity: 5
  availabilityZones: [us-east-1a, us-east-1b, us-east-1c]
  labels:
    role: application
    node-type: on-demand
  asgMetricsCollection:
  - granularity: 1Minute          # CloudWatch ASG metrics for CA

# Spot node group — for fault-tolerant batch workloads
- name: spot-nodes
  instanceTypes:                  # multiple types = better spot availability
  - m5.2xlarge
  - m5a.2xlarge
  - m4.2xlarge
  - m5d.2xlarge
  spot: true
  minSize: 0
  maxSize: 100
  desiredCapacity: 0
  labels:
    role: spot
    node-type: spot
  taints:
  - key: spot
    value: "true"
    effect: NoSchedule            # only spot-tolerant pods schedule here
  tags:
    k8s.io/cluster-autoscaler/enabled: "true"
    k8s.io/cluster-autoscaler/production-cluster: owned

# GPU node group — for ML inference
- name: gpu-nodes
  instanceType: g4dn.xlarge
  minSize: 0
  maxSize: 10
  desiredCapacity: 0
  labels:
    role: gpu
    accelerator: nvidia-tesla-t4
  taints:
  - key: nvidia.com/gpu
    value: "present"
    effect: NoSchedule
```

---

### Targeting Specific Node Pools

```yaml
# Pods must target the right node pool

# Target spot nodes (tolerate the taint + use node selector or affinity)
spec:
  tolerations:
  - key: spot
    value: "true"
    effect: NoSchedule
  nodeSelector:
    node-type: spot               # simple selector — must have this label

# OR: node affinity (more expressive)
spec:
  tolerations:
  - key: spot
    value: "true"
    effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values: [spot]

# Target GPU nodes
spec:
  tolerations:
  - key: nvidia.com/gpu
    value: present
    effect: NoSchedule
  resources:
    limits:
      nvidia.com/gpu: 1           # request 1 GPU

# Prefer on-demand, fall back to spot (weighted affinity)
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values: [on-demand]
      - weight: 10
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values: [spot]
```

---

### Cluster API (CAPI) — Declarative Infrastructure

```
CLUSTER API CONCEPTS
═══════════════════════════════════════════════════════════════

Cluster API brings Kubernetes-style declarative management to
cluster infrastructure itself.

MANAGEMENT CLUSTER:
  A running Kubernetes cluster that hosts CAPI controllers.
  Usually: a small, stable cluster (or kind for local dev).

WORKLOAD CLUSTERS:
  Clusters created and managed by CAPI from the management cluster.
  Defined as Kubernetes objects: Cluster, Machine, MachineDeployment.

KEY CRDs:
  Cluster:              Defines a cluster (network, control plane ref)
  Machine:              Defines a single node (EC2 instance equivalent)
  MachineDeployment:    Like Deployment for Machines (rolling updates)
  MachineSet:           Like ReplicaSet for Machines (N identical nodes)
  MachineHealthCheck:   Replaces unhealthy nodes automatically

INFRASTRUCTURE PROVIDERS (plugins for cloud-specific resources):
  CAPA: Cluster API Provider AWS
  CAPZ: Cluster API Provider Azure
  CAPG: Cluster API Provider GCP
  CAPH: Cluster API Provider Hetzner
  CAPV: Cluster API Provider vSphere
```

---

```yaml
# Cluster API MachineDeployment (worker node pool)
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: worker-pool-1
  namespace: default
spec:
  clusterName: production-cluster
  replicas: 5
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
      cluster.x-k8s.io/deployment-name: worker-pool-1
  template:
    spec:
      clusterName: production-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: worker-kubeadmconfig
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate
        name: worker-awsmachinetemplate

---
# MachineHealthCheck — auto-replace unhealthy nodes
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineHealthCheck
metadata:
  name: worker-health-check
  namespace: default
spec:
  clusterName: production-cluster
  selector:
    matchLabels:
      cluster.x-k8s.io/deployment-name: worker-pool-1
  unhealthyConditions:
  - type: Ready
    status: Unknown
    timeout: 300s               # node NotReady for 5m → replace
  - type: Ready
    status: "False"
    timeout: 300s
  maxUnhealthy: 33%             # don't replace more than 33% simultaneously
```

---

### 🎤 Short Crisp Interview Answer

> *"Node pools segment cluster infrastructure by workload type — you typically have a system pool (fixed size, for kube-system and critical add-ons), an application pool (autoscaled, on-demand), a spot pool (for fault-tolerant batch workloads with spot instance taint), and optionally a GPU pool. Pods target pools via tolerations for the taint plus node selectors or affinity rules. On EKS, managed node groups handle OS patching and drain-on-upgrade automatically. Cluster API is the Kubernetes-native framework for managing infrastructure declaratively — you define clusters and node groups as CRDs in a management cluster, and CAPI controllers create the actual cloud resources. MachineHealthCheck automatically replaces nodes that stay NotReady for over 5 minutes — it's the cluster-level equivalent of a liveness probe."*

---

---

# 9.11 Multi-Tenancy Patterns — Namespace, vcluster, Separate Clusters

## 🔴 Advanced

### What it is in simple terms

Multi-tenancy means **sharing Kubernetes infrastructure among multiple teams or customers** while ensuring isolation — teams can't interfere with each other's workloads. The right isolation level depends on the trust boundary: internal teams sharing hardware need namespace isolation, untrusted external tenants need stronger separation.

---

### Multi-Tenancy Spectrum

```
ISOLATION LEVELS — FROM WEAKEST TO STRONGEST
═══════════════════════════════════════════════════════════════

LEVEL 1: NAMESPACE ISOLATION (soft multi-tenancy)
  What it is: separate namespaces per team on shared cluster
  Isolation: RBAC (API access), ResourceQuota (resource caps),
             NetworkPolicy (network isolation), PSS (pod security)
  NOT isolated: node sharing (pods co-located on same nodes),
                cluster-level resources (Nodes, PVs, StorageClasses),
                kernel (syscall table, CPU scheduling)
  Blast radius: misconfigured node → affects ALL namespaces
  Use for: internal teams with mutual trust, shared infra teams
  Cost: very low (shared cluster)
  Operational overhead: low

LEVEL 2: VIRTUAL CLUSTERS (vcluster)
  What it is: a full K8s API Server running inside a pod
  Isolation: each tenant gets their own API Server, etcd, scheduler
             but shares the underlying node OS and kernel
  What they can do: install CRDs, manage RBAC, use any K8s features
  What they can't do: access other vclusters' resources
  Use for: dev/staging environments, tenant isolation for SaaS platforms
           CI/CD pipelines (each PR gets its own cluster)
  Cost: low-medium (shares nodes, each vcluster ~3 pods overhead)
  Operational overhead: medium

LEVEL 3: SEPARATE CLUSTERS (hard multi-tenancy)
  What it is: completely separate Kubernetes clusters per tenant
  Isolation: full — separate control plane, nodes, networking, IAM
  Use for: customers with compliance requirements (PCI, HIPAA),
           untrusted external tenants, strict blast radius requirements
  Cost: high (multiplied control plane + node costs)
  Operational overhead: high (N clusters to manage)

CHOOSING THE RIGHT LEVEL:
  Internal dev teams          → Namespace isolation
  Internal teams with secrets → Namespace + strict PSS + NetworkPolicy
  SaaS platform tenants       → vcluster (API isolation without cost of full clusters)
  Financial/healthcare        → Separate clusters per environment
  CI/CD isolation             → vcluster per pipeline run
```

---

### Namespace Multi-Tenancy (Complete Setup)

```yaml
# Complete namespace multi-tenancy for "team-payments"
# 1. Namespace with security labels
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    team: payments
    environment: production
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28

---
# 2. ResourceQuota — prevent one team from consuming all cluster resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services.loadbalancers: "2"        # limit expensive load balancers
    persistentvolumeclaims: "20"

---
# 3. LimitRange — inject defaults and prevent resource abuse
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-limits
  namespace: team-payments
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 500m
      memory: 512Mi
    max:
      cpu: "4"
      memory: 4Gi
    min:
      cpu: 50m
      memory: 64Mi

---
# 4. NetworkPolicy — default deny + allow only team-internal + ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-payments
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-namespace
  namespace: team-payments
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: {}              # from same namespace
  egress:
  - to:
    - podSelector: {}              # to same namespace
  - ports:                        # DNS
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

---
# 5. RBAC — team gets admin in their namespace, nothing else
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-team-admin
  namespace: team-payments
subjects:
- kind: Group
  name: "oidc:payments-team"      # OIDC group from IdP
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin                     # full namespace admin (no cluster resources)
  apiGroup: rbac.authorization.k8s.io

---
# 6. OPA/Gatekeeper — enforce team policies
# (see Category 7 — require labels, no latest tag, etc.)
```

---

### vcluster — Virtual Clusters

```bash
# vcluster: a full Kubernetes API Server running inside a namespace
# Each vcluster gets: kube-apiserver, kube-controller-manager, etcd (or SQLite)
# Runs as 1-3 pods in the host cluster namespace
# Syncs objects to host cluster for actual workload execution

# Install vcluster CLI
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64"
chmod +x vcluster && mv vcluster /usr/local/bin

# Create a vcluster (single-node K3s by default)
vcluster create tenant-a \
  --namespace vcluster-tenant-a \
  --connect=false \
  --chart-values - <<EOF
vcluster:
  image: rancher/k3s:v1.28.2-k3s1
sync:
  ingresses:
    enabled: true              # sync ingresses to host cluster
  storageClasses:
    enabled: true
resources:
  requests:
    cpu: 200m
    memory: 256Mi
EOF

# Connect to vcluster (gets a kubeconfig for the virtual cluster)
vcluster connect tenant-a --namespace vcluster-tenant-a
# Updates ~/.kube/config with vcluster context

# Inside the vcluster: full K8s API
kubectl get nodes                # shows virtual nodes
kubectl apply -f deployment.yaml # creates deployment in vcluster
# vcluster syncs pods down to host cluster actual nodes
# tenant sees "virtual nodes" but pods run on real host nodes

# Tenant has full control inside vcluster:
kubectl create clusterrole my-role ...   # CAN create ClusterRoles (own API server)
kubectl install helm charts that add CRDs  # CAN add CRDs
kubectl get nodes                          # sees virtual topology only

# Host cluster perspective:
kubectl get pods -n vcluster-tenant-a
# NAME                          READY   STATUS    RESTARTS   AGE
# vcluster-tenant-a-0           2/2     Running   0          5m   ← vcluster pod
# tenant-a-web-abc123           1/1     Running   0          2m   ← synced workload pod

# List all vclusters
vcluster list

# Disconnect from vcluster
vcluster disconnect

# Delete vcluster (and all its workloads)
vcluster delete tenant-a --namespace vcluster-tenant-a
```

---

### vcluster Architecture

```
VCLUSTER INTERNALS
═══════════════════════════════════════════════════════════════

HOST CLUSTER:
  namespace: vcluster-tenant-a
    StatefulSet: vcluster-tenant-a (control plane pod)
      └── container: vcluster    (K3s/K8s API Server + ETCD + Controller)
      └── container: syncer      (syncs objects between virtual + host)

TENANT INTERACTION:
  Tenant → tenant's kubeconfig → vcluster API Server (port-forwarded)
  Tenant creates: Deployment in virtual cluster
  Syncer translates: virtual Deployment → host Deployment in namespace

SYNCED OBJECTS (host ← → virtual):
  Pods:            Yes (runs on host nodes, tenant sees virtual names)
  PVCs:            Yes (creates real PVCs in host namespace)
  Services:        Yes (creates real Services in host namespace)
  ConfigMaps:      Optionally
  Secrets:         Optionally (only synced ones)
  Ingresses:       Optionally

NOT SYNCED (virtual only):
  Namespaces:      Virtual namespaces don't appear in host
  CRDs:            Custom resources stay in vcluster etcd
  RBAC:            Virtual RBAC independent of host
  ClusterRoles:    Tenant can create any ClusterRole (isolated)

ISOLATION PROPERTIES:
  ✓ Tenant can install any CRD (doesn't affect host or other vclusters)
  ✓ Tenant can create ClusterRoles (scoped to virtual cluster)
  ✓ Tenant cannot see other tenants' pods/services/secrets
  ✓ Tenant cannot escape to host cluster namespace directly
  ✗ Kernel shared (same nodes = shared kernel)
  ✗ Node failure affects all vclusters on that node
```

---

### Separate Clusters Pattern

```
WHEN SEPARATE CLUSTERS ARE REQUIRED
═══════════════════════════════════════════════════════════════

COMPLIANCE REQUIREMENTS:
  PCI-DSS:   cardholder data environment must be isolated
  HIPAA:     PHI (Protected Health Information) isolation
  SOC 2:     audit trail per environment
  GDPR:      data residency (different AWS regions per country)

OPERATIONAL SEPARATE CLUSTERS:
  per-environment: dev / staging / production (most common)
  per-region:      us-east-1 / eu-west-1 / ap-southeast-1
  per-customer:    enterprise SaaS with dedicated infrastructure

FLEET MANAGEMENT PATTERNS:
  Hub-and-spoke: central management cluster + workload clusters
    Hub: Argo CD, policy engine, Vault, observability
    Spokes: workload clusters register to hub
    Argo CD deploys to spoke clusters from hub

  GitOps fleet: all cluster configs in Git
    Cluster API (CAPI): provisions clusters declaratively
    Argo CD / Flux:     deploys workloads to each cluster
    Single Git repo:    source of truth for all clusters

MULTI-CLUSTER CHALLENGES:
  Service discovery:   clusters can't DNS-resolve each other by default
    Solutions: Istio multi-cluster, Cilium Cluster Mesh, Submariner
  Secret management:   each cluster needs its own secrets
    Solutions: External Secrets Operator pointing to central Vault/AWS SM
  Observability:       metrics/logs from N clusters → central backend
    Solutions: Thanos (Prometheus), Loki labels, central Grafana
  GitOps:              Argo CD ApplicationSets for fleet management
```

---

### 🎤 Short Crisp Interview Answer

> *"Multi-tenancy has three levels. Namespace isolation uses RBAC, ResourceQuota, NetworkPolicy, and PSS to share a cluster among trusted internal teams — it's cost-efficient but provides soft isolation (same nodes, same kernel). vcluster provides virtual clusters running as pods in the host cluster — each tenant gets a dedicated API Server and can install CRDs and manage ClusterRoles freely, while actual workloads still run on shared host nodes. This is the sweet spot for SaaS platforms or CI/CD isolation without the cost of separate clusters. Hard multi-tenancy with separate clusters is required for compliance (PCI, HIPAA) or genuinely untrusted tenants. In practice, most organizations run dev/staging/production as separate clusters and use namespace isolation within each cluster for team separation."*

---

---

# 9.12 Disaster Recovery Planning for Kubernetes

## 🔴 Advanced

### What it is in simple terms

Kubernetes DR planning ensures you can **recover cluster functionality after catastrophic failure** — a region outage, accidental deletion of namespaces, etcd corruption, or control plane failure. DR planning requires defining RPO (Recovery Point Objective — how much data loss is acceptable) and RTO (Recovery Time Objective — how long recovery can take), then engineering backup and restore procedures to meet those targets.

---

### DR Failure Scenarios

```
KUBERNETES FAILURE SCENARIOS AND SEVERITY
═══════════════════════════════════════════════════════════════

SCENARIO 1: Pod/Deployment failure (low severity)
  Impact: One application component down
  Recovery: Kubernetes self-healing (restart, reschedule)
  RTO: seconds to minutes (automatic)
  DR tool: Liveness probes, ReplicaSets, PodDisruptionBudgets

SCENARIO 2: Node failure (medium severity)
  Impact: Pods on the node need rescheduling
  Recovery: Cluster Autoscaler adds node, pods reschedule (~5 min)
  RTO: 5-15 minutes
  DR tool: Multiple replicas, CA, MachineHealthCheck

SCENARIO 3: AZ failure (high severity)
  Impact: All nodes in one AZ lost
  Recovery: Pods reschedule to surviving AZs
  RTO: 5-30 minutes (depends on cluster capacity in other AZs)
  DR tool: Multi-AZ node groups, TopologySpreadConstraints,
           headroom capacity in each AZ

SCENARIO 4: Control plane failure (critical)
  Self-managed: etcd quorum loss, API Server crash
    Recovery: restore etcd from backup, restart components
    RTO: 15 min - 2 hours
  EKS: AWS manages control plane HA
    Recovery: usually automatic, AWS SLA 99.95%
    RTO: 0-15 minutes

SCENARIO 5: Accidental deletion (critical)
  "kubectl delete namespace production" by mistake
  Impact: all resources in namespace gone
  Recovery: restore from Velero backup OR recreate from GitOps
  RTO: 15 min - 4 hours depending on complexity
  DR tool: Velero, GitOps (Argo CD), admission webhook to block

SCENARIO 6: Region/cloud failure (catastrophic)
  Impact: entire cluster unavailable
  Recovery: failover to secondary region
  RTO: 15 min - 2 hours (if active-passive setup)
       < 5 minutes (if active-active with traffic management)
  DR tool: Multi-region clusters, global load balancer (Route53),
           cross-region GitOps

SCENARIO 7: etcd corruption (catastrophic for self-managed)
  Impact: all cluster state lost
  Recovery: restore from etcd snapshot
  RTO: 30 min - 2 hours
  DR tool: etcd snapshots (6-hourly), off-cluster storage
```

---

### DR Tooling Stack

```
COMPLETE DR TOOLING STACK
═══════════════════════════════════════════════════════════════

LAYER 1: CLUSTER CONFIGURATION (Infrastructure as Code)
  Tool: Terraform / Pulumi / AWS CloudFormation
  What: cluster creation, node groups, VPC, IAM roles
  Recovery: re-provision entire cluster from code
  Storage: Git repository (source of truth)
  RTO contribution: 10-30 minutes to provision new cluster

LAYER 2: APPLICATION WORKLOADS (GitOps)
  Tool: Argo CD / Flux CD
  What: all Kubernetes manifests (Deployments, Services, ConfigMaps)
  Recovery: new cluster → Argo CD syncs → all apps deployed
  Storage: Git repository
  RTO contribution: 5-30 minutes for full sync

LAYER 3: KUBERNETES RESOURCES (Object backup)
  Tool: Velero
  What: snapshots of K8s objects (Deployments, PVCs, Secrets, ConfigMaps)
  Recovery: velero restore → all K8s objects restored
  Storage: S3 bucket (cross-region replication recommended)
  RPO: schedule = backup frequency (15min, 1hr, 6hr)
  RTO contribution: 5-20 minutes for restore

LAYER 4: PERSISTENT DATA (PV backup)
  Tool: Velero + Restic/Kopia (file-level) OR AWS EBS snapshots
  What: actual data inside PersistentVolumes
  Recovery: EBS snapshot restored + PVC rebound
  Storage: S3 (Velero/Restic) or EBS snapshot (AWS native)
  RPO: depends on snapshot schedule
  RTO contribution: 5-60 minutes depending on data size

LAYER 5: ETCD STATE (for self-managed clusters only)
  Tool: etcdctl snapshot, CronJob
  What: etcd database (entire cluster state)
  Recovery: etcdctl snapshot restore
  Storage: S3, encrypted
  RPO: 6 hours maximum recommended
  RTO contribution: 15-30 minutes for restore

LAYER 6: SECRETS (external secret management)
  Tool: AWS Secrets Manager / HashiCorp Vault
  What: actual secret values (not stored in etcd)
  Recovery: secrets auto-populated from ESO on cluster restore
  Storage: cloud secret manager (AWS SM cross-region replication)
  RTO contribution: 0 (auto-populated on pod start)
```

---

### Velero DR Setup (Production)

```bash
# VELERO SETUP FOR CROSS-REGION DR
# Primary: us-east-1
# DR: us-west-2

# Create S3 bucket in DR region with CRR (Cross-Region Replication)
aws s3api create-bucket \
  --bucket my-cluster-velero-backups-dr \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2

# Enable S3 versioning (required for CRR)
aws s3api put-bucket-versioning \
  --bucket my-cluster-velero-backups-primary \
  --versioning-configuration Status=Enabled

# Configure CRR from primary → DR bucket
# (replicates all backup files to DR region automatically)

# Install Velero in primary cluster
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-cluster-velero-backups-primary \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-node-agent \                   # enables file-level PV backup
  --secret-file ./credentials-velero

# Configure backup schedules
# Full cluster backup every 6 hours
velero schedule create cluster-full \
  --schedule="0 */6 * * *" \
  --include-cluster-resources=true \
  --ttl 720h \                         # keep 30 days
  --storage-location default

# Frequent namespace backup for critical namespaces (every 15 min)
velero schedule create production-frequent \
  --schedule="*/15 * * * *" \
  --include-namespaces production \
  --ttl 48h \                          # keep 2 days
  --snapshot-volumes=true              # include PV snapshots

# Check backup status
velero backup get
velero backup describe cluster-full-20240115100000 --details
velero backup logs cluster-full-20240115100000

# DISASTER RECOVERY PROCEDURE:
# 1. Provision new cluster in DR region (Terraform)
# 2. Install Velero in DR cluster pointing to DR S3 bucket
# 3. Create BackupStorageLocation pointing to DR bucket
# 4. Restore

# Restore from backup in DR cluster
velero restore create \
  --from-backup cluster-full-20240115100000 \
  --restore-volumes=true \
  --namespace-mappings production:production  # map namespace if needed

velero restore get
velero restore describe my-restore-20240115120000 --details
```

---

### Runbook — Accidental Namespace Deletion

```bash
# INCIDENT: kubectl delete namespace production (accidentally)
# RESPONSE RUNBOOK:

# STEP 1: Immediate assessment
date                                          # note incident time
kubectl get namespace production              # confirm it's gone
kubectl get events -A | grep production       # confirm deletion event

# STEP 2: Check Velero for most recent backup
velero backup get | grep production
# NAME                          STATUS     CREATED
# production-frequent-...       Completed  2024-01-15 09:45:00   ← 15 min ago

# STEP 3: Restore (Velero preserves all K8s objects)
velero restore create recovery-production-20240115 \
  --from-backup production-frequent-20240115094500 \
  --include-namespaces production \
  --restore-volumes=true

# STEP 4: Monitor restore
watch velero restore describe recovery-production-20240115 --details

# STEP 5: Verify (once restore shows Completed)
kubectl get pods -n production
kubectl get pvc -n production
kubectl get services -n production

# STEP 6: Verify application health
kubectl rollout status deployment -n production
curl https://api.production.com/health

# STEP 7: If Velero not available — restore from GitOps
# Argo CD auto-sync will recreate all manifests
kubectl apply -k overlays/production/
# (restores all K8s objects but NOT PV data)
# Restore PV data from EBS snapshots separately

# PREVENTION: Admission webhook to block namespace deletion
# (OPA/Gatekeeper or Kyverno policy to require confirmation label)
```

---

### Active-Passive Multi-Region DR

```
MULTI-REGION ACTIVE-PASSIVE ARCHITECTURE
═══════════════════════════════════════════════════════════════

PRIMARY REGION (us-east-1):
  EKS cluster + running workloads
  Velero → S3 bucket (primary)
  S3 CRR → S3 bucket (DR)
  Argo CD → deploys from Git

DR REGION (us-west-2):
  EKS cluster (warm standby — running but serving no traffic)
    OR: cluster defined in Terraform but not provisioned (cold standby)
  Velero → S3 bucket (DR copy)
  Argo CD → same Git repo, same manifests

TRAFFIC ROUTING:
  Route53: primary DNS weighted routing → 100% to us-east-1
  Health check: Route53 monitors api.production.com/health

FAILOVER PROCEDURE:
  1. Detect: Route53 health check fails primary region
  2. Promote: Route53 weight changes → 100% to us-west-2
     OR: manual DNS update (if health check not configured)
  3. Scale up: Cluster Autoscaler adds nodes in DR cluster
              (warm standby already has some nodes)
  4. Restore data: Velero restore from DR S3 bucket
                   (if needed — stateless apps skip this)
  5. Verify: run integration tests against DR cluster

WARM vs COLD STANDBY:
  Warm: DR cluster running with same deployments, 0 traffic
        RTO: 5-15 minutes (Route53 failover + scale up)
        Cost: ~100% of primary cluster cost (full duplicate)

  Cold: DR cluster defined in IaC but not provisioned
        RTO: 30-60 minutes (provision + deploy + restore)
        Cost: ~10-20% of primary (just S3 storage + Velero)

  Hot: DR cluster running and serving real traffic (active-active)
       RTO: 0 (already serving)
       Cost: 100%+ of primary
       Complexity: highest (data synchronization required)
```

---

### DR Testing

```bash
# DR DRILL SCHEDULE — test quarterly minimum

# TEST 1: Velero restore test (monthly, non-production)
# Create test cluster in dev account
eksctl create cluster --name dr-test --region us-west-2 \
  --without-nodegroup
# Install Velero pointing to production backup S3 bucket (read-only)
# Restore a single namespace
velero restore create dr-test \
  --from-backup production-frequent-latest \
  --include-namespaces staging
# Verify: all pods running, data accessible
# Measure: time to complete restore

# TEST 2: Accidental deletion simulation (quarterly)
# Create a copy of production namespace in test cluster
# Delete it: kubectl delete namespace production-copy
# Restore from Velero
# Measure: time to detect, time to restore, data integrity

# TEST 3: Region failover drill (bi-annually)
# Simulate us-east-1 outage:
# 1. Update Route53 to point to us-west-2
# 2. Run integration test suite against us-west-2
# 3. Measure RTO
# 4. Fail back to us-east-1
# 5. Document findings and improve runbook

# METRICS TO TRACK:
echo "DR METRICS:"
echo "  RPO achieved: time since last successful backup"
echo "  RTO achieved: minutes from failure detection to full recovery"
echo "  Backup success rate: Velero backup success % last 30 days"
echo "  Restore test pass rate: quarterly drill results"

# Check Velero backup success rate
velero backup get | grep -c "Completed"
velero backup get | grep -c "Failed"
```

---

### 🎤 Short Crisp Interview Answer

> *"Kubernetes DR is layered: infrastructure as code (Terraform) to reprovision clusters, GitOps (Argo CD) to redeploy all application workloads, Velero for Kubernetes object and PV snapshots, and cloud-native backups for persistent data. The most common DR scenario is accidental deletion — Velero restores from the most recent scheduled backup, with RPO determined by backup frequency (we run every 15 minutes for critical namespaces). For region-level DR, the active-passive pattern has a warm standby cluster in a secondary region synced via Velero S3 cross-region replication — failover is a Route53 weight update with 5-15 minute RTO. The critical non-negotiables: test your DR quarterly (untested backups are not backups), back up etcd off-cluster (S3), and ensure your GitOps repo can recreate all manifests independently."*

---

### ⚠️ Gotchas

1. **GitOps alone is not DR** — Argo CD can redeploy K8s objects but cannot restore PV data. You need Velero or cloud-native snapshots for stateful data.
2. **Velero backup of secrets stores base64, not plaintext** — if you use External Secrets Operator, secrets aren't in etcd at all. Velero will backup empty placeholder secrets. The real values are in Secrets Manager — ensure that's also replicated cross-region.
3. **etcd backup alone is not sufficient for EKS** — AWS doesn't expose etcd on managed EKS. Your DR strategy for EKS must be GitOps + Velero, not etcd snapshots.
4. **DR drill is mandatory** — backups that have never been tested will fail when you actually need them. A restore test is the only way to validate that backup files are complete and procedures work.

---

---

# 🏁 Category 9 — Complete Operations Map

```
CLUSTER OPERATIONS DECISION MAP
═══════════════════════════════════════════════════════════════════

DAILY OPERATIONS:
  Need to connect to cluster?      → kubeconfig / kubectx        9.1
  Pod is misbehaving?              → kubectl describe/logs/exec  9.2, 8.7
  Check namespace resources?       → kubectl get all -n <ns>     9.3

NODE OPERATIONS:
  Maintenance on a node?           → cordon → drain → work → uncordon  9.4
  Node stuck in drain?             → check PDB, scale up first   9.4
  Nodes filling up?                → Cluster Autoscaler          9.9
  Need GPU/spot nodes?             → Node pools + taints         9.10

CLUSTER LIFECYCLE:
  Time to upgrade K8s version?     → upgrade control plane first 9.5
  What APIs will break?            → kubent scan before upgrade  9.5
  Back up cluster state?           → etcdctl snapshot / Velero   9.6
  Something got deleted?           → velero restore              9.12

APPLICATION DEPLOYMENT:
  Deploy third-party software?     → Helm install                9.7
  Manage own app config?           → Kustomize overlays          9.8
  Deploy same app to 3 envs?       → Kustomize base + overlays   9.8

MULTI-TENANCY:
  Share cluster with teams?        → Namespace isolation         9.11
  Give teams full K8s API?         → vcluster                   9.11
  Strict compliance isolation?     → Separate clusters           9.11

DISASTER RECOVERY:
  Region outage?                   → Route53 failover to DR      9.12
  Namespace deleted?               → velero restore from backup  9.12
  How to test DR?                  → Quarterly restore drills    9.12
```

---

# Quick Reference — Category 9 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **kubeconfig** | clusters + users + contexts. KUBECONFIG env var overrides ~/.kube/config |
| **context** | cluster + user + default namespace. kubectl config use-context to switch |
| **EKS kubeconfig** | aws eks update-kubeconfig — uses exec plugin with 15-min tokens |
| **kubectl -o jsonpath** | Extract specific fields. `'{.status.podIP}'` for pod IP |
| **kubectl --dry-run=server** | Server validates + admission webhooks run. Better than client |
| **cordon** | Stop new pods scheduling. Existing pods unaffected |
| **drain** | Evict all pods gracefully. Respects PDB. DaemonSets skipped by default |
| **drain --force** | Evicts pods with no controller — they won't reschedule! |
| **PDB blocking drain** | Most common drain issue. Scale up first to create slack |
| **uncordon** | Re-enable scheduling. Doesn't rebalance existing pods |
| **Version skew** | kubelet up to 2 minor behind API Server. Upgrade API Server first |
| **EKS upgrade order** | Control plane → Add-ons → Node groups |
| **kubent** | Detect deprecated API usage before upgrade |
| **etcd backup** | etcdctl snapshot save. Verify with snapshot status. Off-cluster to S3 |
| **etcd restore** | Stop API Server → snapshot restore → update etcd manifest → restart |
| **EKS etcd** | Managed by AWS — use Velero instead |
| **Helm release** | Deployed instance of a chart. Stored as Secrets in namespace |
| **helm --atomic** | Auto-rollback on upgrade failure |
| **Values precedence** | --set overrides -f values.yaml overrides chart defaults |
| **Kustomize base** | Clean valid YAML. No templates |
| **Kustomize overlays** | Patches applied over base. Per-environment customization |
| **CA scale-up** | Pending pod → simulate → add node → 2-4 min on AWS |
| **CA scale-down** | 10 min underutilized + all safety checks → drain + terminate |
| **CA blocked scale-down** | safe-to-evict: false annotation, PDB, bare pods |
| **Karpenter vs CA** | Karpenter: 30-60s, any instance type, consolidation. CA: 2-4min, fixed types |
| **Namespace isolation** | Soft. RBAC + Quota + NetworkPolicy + PSS |
| **vcluster** | Virtual K8s API Server as pod. Shared nodes. Good for SaaS tenants |
| **Separate clusters** | Hard isolation. Required for PCI/HIPAA compliance |
| **DR RPO** | How much data loss acceptable. Determined by backup frequency |
| **DR RTO** | How long recovery can take. Warm standby: 5-15 min |
| **Velero** | K8s object + PV backup to S3. Works on EKS |
| **DR testing** | Mandatory quarterly. Untested backups are not backups |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| CA scale-down check interval | 10 seconds |
| CA scale-down-unneeded-time | 10 minutes (default) |
| CA scale-down-delay-after-add | 10 minutes (default) |
| CA utilization threshold | 50% (CPU + memory requested) |
| EC2 node launch time (CA) | 2-4 minutes |
| Karpenter node launch time | 30-60 seconds |
| K8s minor releases per year | 3 (approx every 4 months) |
| K8s version support window | ~14 months |
| kubelet max version skew | 2 minor versions behind API Server |
| etcd snapshot frequency (min) | Every 6 hours |
| vcluster overhead | ~3 pods (API Server, etcd, syncer) |
| Velero backup frequency (critical NS) | Every 15 minutes |
| Route53 health check failover | 1-2 minutes |
| Warm standby DR RTO | 5-15 minutes |
| Cold standby DR RTO | 30-60 minutes |
| EKS control plane upgrade time | 15-30 minutes |

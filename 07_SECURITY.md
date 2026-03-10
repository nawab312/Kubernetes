# Kubernetes Interview Mastery
# CATEGORY 7: SECURITY

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Deep Answer → Gotchas → Interview Q&A → Connections.
> ⚠️ = High priority, frequently asked, commonly misunderstood.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 7.1 | RBAC — Roles, ClusterRoles, Bindings ⚠️ | 🟢 Beginner |
| 7.2 | Service Accounts | 🟢 Beginner |
| 7.3 | Pod Security Standards / Pod Security Admission | 🟢 Beginner |
| 7.4 | Admission Controllers ⚠️ | 🟡 Intermediate |
| 7.5 | Webhook Admission — Mutating vs Validating ⚠️ | 🟡 Intermediate |
| 7.6 | Network Policies for security (zero-trust) | 🟡 Intermediate |
| 7.7 | OIDC integration — external identity providers | 🟡 Intermediate |
| 7.8 | Audit logging | 🟡 Intermediate |
| 7.9 | Seccomp, AppArmor, SELinux profiles | 🟡 Intermediate |
| 7.10 | OPA/Gatekeeper — policy as code ⚠️ | 🔴 Advanced |
| 7.11 | Kyverno — policy engine alternative | 🔴 Advanced |
| 7.12 | Workload identity (IRSA on EKS) | 🔴 Advanced |
| 7.13 | Image signing & verification (Cosign, Connaisseur) | 🔴 Advanced |
| 7.14 | Runtime security (Falco) | 🔴 Advanced |
| 7.15 | CIS Benchmark hardening | 🔴 Advanced |

---

# ⚠️ 7.1 RBAC — Roles, ClusterRoles, Bindings

## 🟢 Beginner — HIGH PRIORITY

### What it is in simple terms

RBAC (Role-Based Access Control) is the **authorization system** that controls who can do what in Kubernetes. Every API request carries an identity (user, ServiceAccount, or group), and RBAC determines whether that identity is allowed to perform the requested action on the requested resource.

---

### The Four RBAC Objects

```
RBAC OBJECT MODEL
═══════════════════════════════════════════════════════════════

ROLE (namespaced):
  Defines WHAT can be done in ONE namespace.
  Contains: rules (apiGroups + resources + verbs)
  Scope: namespace-scoped — only valid within its namespace

CLUSTERROLE (cluster-scoped):
  Defines WHAT can be done across the WHOLE cluster.
  Same rule format as Role.
  Used for: cluster-scoped resources (Nodes, PVs, Namespaces)
            OR namespaced resources where you want reuse across namespaces

ROLEBINDING (namespaced):
  Binds a Role OR ClusterRole to subjects in ONE namespace.
  If binding a ClusterRole → ClusterRole rules apply in this namespace only
  Scope: namespace-scoped

CLUSTERROLEBINDING (cluster-scoped):
  Binds a ClusterRole to subjects across the WHOLE cluster.
  Always requires ClusterRole (not Role).
  Scope: cluster-scoped — applies everywhere

SUBJECTS (who is being granted access):
  User:           External identity (human, CI system) — string name
  Group:          Collection of users — string name
  ServiceAccount: Pod identity — namespace + name
```

---

### RBAC Verbs and Resources

```
KUBERNETES API VERBS:
═══════════════════════════════════════════════════════════════

HTTP verb  → K8s RBAC verb(s)
GET        → get (single resource), list (collection), watch
POST       → create
PUT        → update
PATCH      → patch
DELETE     → delete (single), deletecollection (bulk)

SPECIAL VERBS:
  use       → PodSecurityPolicy (deprecated), ResourceClaim
  bind      → Roles/ClusterRoles (can grant to others)
  escalate  → Roles/ClusterRoles (grant permissions you don't have)
  impersonate → Users/Groups/ServiceAccounts
  approve   → CertificateSigningRequest
  sign      → CertificateSigningRequest

RESOURCE NAMES (optional, per-object restriction):
  rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["my-specific-secret"]  # only THIS secret, not all
    verbs: ["get"]
  → Subject can only get the secret named "my-specific-secret"
  ← Can't list secrets (list doesn't work with resourceNames)

COMMON APIGROUPS:
  ""            → core group: pods, services, configmaps, secrets, nodes, endpoints
  "apps"        → deployments, replicasets, statefulsets, daemonsets
  "batch"       → jobs, cronjobs
  "autoscaling" → horizontalpodautoscalers
  "networking.k8s.io" → ingresses, networkpolicies
  "rbac.authorization.k8s.io" → roles, clusterroles, bindings
  "storage.k8s.io" → storageclasses, volumeattachments
  "*"           → ALL apiGroups (admin-level)
```

---

### RBAC YAML — Full Examples

```yaml
# Role — namespaced, grants access within "production" namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]                    # core API group
  resources: ["pods", "pods/log"]    # pods AND pod logs
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]

---
# ClusterRole — cluster-wide, also reusable across namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]               # nodes are cluster-scoped
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]          # namespaces are cluster-scoped
  verbs: ["get", "list"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "persistentvolumes"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding — bind Role to subjects in "production" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-pod-reader
  namespace: production
subjects:
# Human user
- kind: User
  name: "jane@company.com"           # exact string from ID token sub/email
  apiGroup: rbac.authorization.k8s.io
# ServiceAccount (pod identity)
- kind: ServiceAccount
  name: monitoring-agent
  namespace: monitoring              # SA can be from different namespace
# Group
- kind: Group
  name: "platform-team"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role                         # Role or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# Binding a ClusterRole at namespace scope (common pattern for reuse)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-clusterrole-in-staging
  namespace: staging
subjects:
- kind: User
  name: "developer@company.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole                  # ClusterRole used in namespace RoleBinding
  name: view                         # built-in view ClusterRole
  apiGroup: rbac.authorization.k8s.io
# Result: developer can view all resources in staging namespace only
# NOT cluster-wide — ClusterRole narrowed to one namespace by RoleBinding

---
# ClusterRoleBinding — cluster-wide access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
- kind: User
  name: "platform-admin@company.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin               # built-in: full access to everything
  apiGroup: rbac.authorization.k8s.io
```

---

### Built-in ClusterRoles

```bash
# Kubernetes ships with these ClusterRoles:
kubectl get clusterrole | grep -v "system:"

# cluster-admin:  Full access to everything — ALL verbs, ALL resources
# admin:          Full namespace access (no namespace CRUD, no quota changes)
# edit:           Read/write most namespace resources (no RBAC changes)
# view:           Read-only access to most namespace resources

# Grant view to a user in a specific namespace:
kubectl create rolebinding readonly-user \
  --clusterrole=view \
  --user=developer@company.com \
  --namespace=staging

# Grant cluster-admin (careful!):
kubectl create clusterrolebinding admin-user \
  --clusterrole=cluster-admin \
  --user=platform-admin@company.com
```

---

### kubectl RBAC Commands

```bash
# Check if you can perform an action (as yourself)
kubectl auth can-i get pods -n production
# yes

kubectl auth can-i delete deployments -n production
# no

# Check what another user can do (impersonate — requires admin)
kubectl auth can-i get secrets \
  --as=jane@company.com \
  -n production
# no

# Check what a ServiceAccount can do
kubectl auth can-i list pods \
  --as=system:serviceaccount:monitoring:prometheus-sa \
  -n production
# yes

# Get all permissions for a role
kubectl describe role pod-reader -n production
# Rules:
#   Resources          Non-Resource URLs  Resource Names  Verbs
#   pods               []                 []              [get list watch]
#   pods/log           []                 []              [get list watch]

# Find all bindings for a subject
kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq '
  .items[] |
  select(.subjects[]?.name == "jane@company.com") |
  .metadata.name + " → " + .roleRef.name'

# Audit: list all ClusterRoleBindings to cluster-admin
kubectl get clusterrolebindings -o json | jq '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  {name: .metadata.name, subjects: .subjects}'

# Create Role from command line
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods,pods/log \
  -n production

# Create ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create RoleBinding
kubectl create rolebinding jane-pod-reader \
  --role=pod-reader \
  --user=jane@company.com \
  -n production
```

---

### RBAC Principle of Least Privilege

```
SECURITY BEST PRACTICES
═══════════════════════════════════════════════════════════════

1. NEVER use cluster-admin for applications
   Apps need: get/list specific resources, not cluster-admin
   cluster-admin compromise = full cluster compromise

2. NEVER use wildcard (*) verbs or resources in production
   rules:
   - apiGroups: ["*"]      ← dangerous
     resources: ["*"]      ← dangerous
     verbs: ["*"]          ← dangerous

3. Secrets access is especially dangerous
   GET secrets = read all secret values
   LIST secrets = read ALL secrets in namespace at once
   Restrict: only services that truly need secrets
   Prefer: volume mount access over RBAC access to secrets

4. "list" vs "get" distinction
   get:  one specific resource (knows the name)
   list: all resources of this type in namespace
   watch: stream all changes
   list + watch = can see everything — more powerful than get

5. Avoid RBAC escalation paths
   bind verb:     can grant ANY role to anyone (escalation)
   escalate verb: can grant permissions beyond what you have
   impersonate:   can act as any user — effectively cluster-admin
   These three verbs should be extremely restricted.

6. Use namespace-scoped roles, not ClusterRoles, for app access
   Apps don't need cluster-wide access in most cases.
   ClusterRoleBinding = cluster-wide access = large blast radius.
```

---

### 🎤 Short Crisp Interview Answer

> *"RBAC has four objects: Role (namespaced rules), ClusterRole (cluster-wide or reusable rules), RoleBinding (grants a Role or ClusterRole within a namespace), and ClusterRoleBinding (grants a ClusterRole cluster-wide). The key pattern is: you can bind a ClusterRole with a RoleBinding to scope it to one namespace — this lets you reuse ClusterRole definitions without granting cluster-wide access. Subjects are Users, Groups, or ServiceAccounts. Verbs map to HTTP operations: get/list/watch for reads, create/update/patch/delete for writes. LIST on secrets is more dangerous than GET — it returns all secrets at once. Always follow least privilege — applications rarely need more than get/list/watch on specific resources in their namespace."*

---

### ⚠️ Gotchas

1. **list permission on Secrets returns all values** — granting `list` on secrets in a namespace effectively exposes all secret data. Grant only `get` with specific resourceNames where possible.
2. **ClusterRole + RoleBinding ≠ cluster-wide access** — this is a common confusion. Binding a ClusterRole with a RoleBinding scopes it to the namespace of the RoleBinding. Only ClusterRoleBinding grants cluster-wide access.
3. **RBAC is additive, never restrictive** — you cannot "deny" a specific action. Rules only add permissions. If any rule allows an action, it's allowed. Implement deny via admission webhooks (OPA/Gatekeeper), not RBAC.
4. **User identity is just a string** — Kubernetes has no user store. The user name is whatever string appears in the authentication token or certificate CN field. There's no way to "create a user" in K8s natively.

---

---

# 7.2 Service Accounts

## 🟢 Beginner

### What it is in simple terms

A Service Account is the **identity of a process running inside a pod** when it talks to the Kubernetes API. Every pod runs as a ServiceAccount — if you don't specify one, it runs as the `default` ServiceAccount of its namespace.

---

### ServiceAccount Mechanics

```
HOW SERVICE ACCOUNT AUTHENTICATION WORKS
═══════════════════════════════════════════════════════════════

OLD WAY (K8s < 1.24 — long-lived tokens):
  Creating a ServiceAccount auto-created a Secret:
    Secret type: kubernetes.io/service-account-token
    Secret data:  token: <long-lived JWT, never expires>
                  ca.crt: <cluster CA certificate>
                  namespace: <namespace bytes>

  Pod mount: /var/run/secrets/kubernetes.io/serviceaccount/
    token     ← JWT for API calls
    ca.crt    ← CA to verify API server cert
    namespace ← pod's namespace

  Problem: Token never expires. Service account token compromised
           = permanent cluster access until manually rotated.

NEW WAY (K8s 1.22+, default K8s 1.24+ — bound tokens):
  Projected ServiceAccount token (topic 5.5).
  Auto-injected via projected volume:
    expirationSeconds: 3607 (default ~1 hour)
    audience: "https://kubernetes.default.svc"
    pod-bound: invalidated when pod terminates

  Kubelet auto-rotates the token at 80% of its lifetime.
  No Secret object created for the token.

  Pod still gets same mount path:
    /var/run/secrets/kubernetes.io/serviceaccount/token

WHY IT MATTERS:
  Old: token stolen from pod → attacker has permanent K8s access
  New: token stolen → expires in <1 hour, can't be renewed
```

---

### ServiceAccount YAML

```yaml
# Create a dedicated ServiceAccount for an app (best practice)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    # IRSA annotation (AWS EKS — covered in 7.12)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/MyAppRole
automountServiceAccountToken: true  # default: true
# Set false to opt-out of token injection (for pods that don't need API access)

---
# Pod using a specific ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa        # use our SA, not "default"
  automountServiceAccountToken: false # override: don't mount token (most apps don't need API access)
  containers:
  - name: app
    image: myapp:v1

---
# Grant ServiceAccount permissions via RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-sa-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ServiceAccount with image pull secret (for private registries)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
imagePullSecrets:
- name: ecr-pull-secret
# All pods using this SA automatically get this imagePullSecret
# No need to specify imagePullSecrets in every pod spec
```

---

### kubectl ServiceAccount Commands

```bash
# Create ServiceAccount
kubectl create serviceaccount myapp-sa -n production

# List ServiceAccounts
kubectl get serviceaccount -n production
# NAME       SECRETS  AGE
# default    0        30d    ← auto-created, bound token (no Secret in 1.24+)
# myapp-sa   0        5m

# Describe SA (in K8s 1.24+, no auto-created token Secret)
kubectl describe serviceaccount myapp-sa -n production

# Get pods and their service accounts
kubectl get pods -n production \
  -o custom-columns="NAME:.metadata.name,SA:.spec.serviceAccountName"

# Verify token mounted in pod
kubectl exec -it myapp -n production -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# {
#   "aud": ["https://kubernetes.default.svc"],
#   "exp": 1720000000,          ← expiry timestamp
#   "iat": 1719996400,
#   "iss": "https://kubernetes.default.svc",
#   "sub": "system:serviceaccount:production:myapp-sa"
#   "kubernetes.io": {
#     "pod": {"name": "myapp-xyz", "uid": "..."},  ← pod-bound
#   }
# }

# Create long-lived token for CI/CD (explicit, not auto-generated)
kubectl create token myapp-sa \
  --duration=8760h \           # 1 year
  -n production
# ⚠️ Use sparingly — prefer short-lived tokens or IRSA

# Create a Secret-based token (legacy, K8s allows explicit creation):
apiVersion: v1
kind: Secret
metadata:
  name: myapp-sa-token
  namespace: production
  annotations:
    kubernetes.io/service-account.name: myapp-sa
type: kubernetes.io/service-account-token
# K8s will populate .data.token with a non-expiring JWT
```

---

### Security Best Practices for ServiceAccounts

```
SERVICE ACCOUNT SECURITY GUIDELINES
═══════════════════════════════════════════════════════════════

1. NEVER use the default ServiceAccount for app workloads
   default SA in a namespace often accumulates permissions.
   Create dedicated SAs: one per application component.

2. Set automountServiceAccountToken: false for pods that
   don't need Kubernetes API access
   Most application pods don't call the K8s API.
   Disabling token mounting removes the attack surface.
   Can set at SA level or pod spec level.

3. One SA per application — least privilege
   Don't share SAs between different applications.
   Compromise of one app = only that app's permissions.

4. Namespace isolation
   An SA in namespace A cannot be used in namespace B
   (unless explicitly referenced in RoleBinding in B).

5. Audit SA permissions regularly
   kubectl get rolebindings,clusterrolebindings -A -o json | \
   jq '.items[] | select(.subjects[]?.kind == "ServiceAccount")'
```

---

### 🎤 Short Crisp Interview Answer

> *"Service Accounts are the pod identity for Kubernetes API calls. Every pod runs as a ServiceAccount — if unspecified, the namespace's default SA. In Kubernetes 1.24+, tokens are projected directly into pods via bound tokens that expire in about an hour and are pod-bound — when the pod dies, the token becomes invalid. This replaced the old long-lived Secret-based tokens that never expired. Best practices: create a dedicated SA per application, set automountServiceAccountToken: false for apps that don't call the K8s API, and grant only the specific RBAC permissions the app actually needs. On EKS, ServiceAccounts are also used for IRSA — the pod's SA token is exchanged for AWS IAM credentials."*

---

### ⚠️ Gotchas

1. **default ServiceAccount accumulates permissions over time** — teams add RoleBindings to the default SA for convenience. Over time it has more access than intended. Always use dedicated SAs.
2. **automountServiceAccountToken: true by default** — every pod gets a token even if the app never uses it. A container exploit can use this token to attack the API server. Disable when not needed.
3. **Token is readable by any process in the pod** — all containers in a pod share the same SA and can read the same token. Design pod contents carefully.

---

---

# 7.3 Pod Security Standards (PSS) / Pod Security Admission (PSA)

## 🟢 Beginner

### What it is in simple terms

Pod Security Standards (PSS) define **three security profiles** for pods — Privileged, Baseline, and Restricted. Pod Security Admission (PSA) enforces them at the namespace level. When a pod is created that violates the namespace's security profile, PSA can warn, audit-log, or reject the pod.

---

### Three Security Profiles

```
PSS SECURITY PROFILES
═══════════════════════════════════════════════════════════════

PRIVILEGED (no restrictions):
  Allows: privileged containers, hostNetwork, hostPID,
          hostIPC, hostPath volumes, any capabilities,
          any securityContext setting
  Use for: system-level DaemonSets (CNI, CSI, monitoring agents)
           that legitimately need host-level access
  Example namespaces: kube-system, calico-system, monitoring

BASELINE (prevents known privilege escalations):
  Blocks: privileged containers, hostNetwork/hostPID/hostIPC,
          hostPath volumes, dangerous Linux capabilities
          (NET_ADMIN, SYS_ADMIN, etc.), host port binding
  Allows: most app workloads without modification
  Neutral: doesn't require securityContext changes to pass
  Use for: general application namespaces (default secure)
  Example: most production app namespaces

RESTRICTED (hardened — current security best practices):
  Requires:
    runAsNonRoot: true
    seccompProfile: RuntimeDefault or Localhost
    allowPrivilegeEscalation: false
    drop all capabilities (capabilities.drop: ["ALL"])
  Blocks: everything Baseline blocks PLUS all capability additions
  Use for: high-security namespaces, multi-tenant clusters
  Example: PCI, HIPAA workloads

PROFILE HIERARCHY:
  Privileged ⊃ Baseline ⊃ Restricted
  (Restricted is strictest, Privileged is most permissive)
```

---

### PSA Namespace Labels and Modes

```
PSA MODES — THREE ENFORCEMENT BEHAVIORS
═══════════════════════════════════════════════════════════════

enforce:  Pod REJECTED if it violates the profile
          API Server returns error, pod not created
          Use: production enforcement

warn:     Pod CREATED but client gets a warning message
          kubectl returns: Warning: would violate PodSecurity
          Useful: migration period, testing

audit:    Pod CREATED, violation recorded in audit log
          No user-visible warning
          Useful: discovery — what would break if enforced?

NAMESPACE LABELS:
  pod-security.kubernetes.io/<mode>: <profile>
  pod-security.kubernetes.io/<mode>-version: <version>

  Modes: enforce, warn, audit
  Profiles: privileged, baseline, restricted
  Version: v1.27, v1.28, latest (pinned version of profile definition)
```

---

### PSA YAML — Namespace Labels

```yaml
# Namespace with PSA enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted — pods rejected if they violate
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28

    # Warn on baseline violations (less strict than enforce)
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28

    # Audit log any baseline violations
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28

---
# Namespace for system components (privileged — no restrictions)
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: privileged

---
# Migration pattern: audit first, then warn, then enforce
# Phase 1: audit only (see what would break)
    pod-security.kubernetes.io/audit: restricted
# Phase 2: add warn (developers see warnings)
    pod-security.kubernetes.io/warn: restricted
# Phase 3: enforce
    pod-security.kubernetes.io/enforce: restricted
```

---

### Pod securityContext for Restricted Profile

```yaml
# Pod that passes the "restricted" PSS profile
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production            # enforce: restricted
spec:
  securityContext:
    runAsNonRoot: true             # required by restricted
    runAsUser: 1000                # non-root UID
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault         # required by restricted (syscall filtering)

  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false  # required by restricted
      readOnlyRootFilesystem: true     # best practice (not required)
      capabilities:
        drop: ["ALL"]              # required by restricted: drop all caps
        add: []                    # restricted: cannot add ANY capabilities
    volumeMounts:
    - name: tmp
      mountPath: /tmp              # needed if readOnlyRootFilesystem: true

  volumes:
  - name: tmp
    emptyDir: {}
```

---

### kubectl PSA Commands

```bash
# Check what profile a namespace uses
kubectl get namespace production \
  -o jsonpath='{.metadata.labels}' | python3 -m json.tool
# {
#   "pod-security.kubernetes.io/enforce": "restricted",
#   "pod-security.kubernetes.io/enforce-version": "v1.28"
# }

# Apply PSA label to existing namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.28

# Test: create a privileged pod in restricted namespace
kubectl run privileged-test \
  --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"test","image":"nginx","securityContext":{"privileged":true}}]}}' \
  -n production
# Error: pods "privileged-test" is forbidden:
#        violates PodSecurity "restricted:v1.28":
#        privileged (container "test")

# Dry-run to check if a pod would pass
kubectl apply --dry-run=server -f mypod.yaml -n production
# Warning: would violate PodSecurity "restricted:v1.28": ...
# OR: pod/mypod created (dry run) — means it passes

# Audit all existing violations in a namespace
kubectl label namespace production \
  pod-security.kubernetes.io/audit=restricted --overwrite
# Then check audit logs for violations from running pods
kubectl get events -n production | grep PodSecurity
```

---

### PSS vs PodSecurityPolicy (PSP)

```
PSP DEPRECATED AND REMOVED (K8s 1.25)
PSS/PSA REPLACED IT
═══════════════════════════════════════════════════════════════

PodSecurityPolicy (removed K8s 1.25):
  CRD-like object, referenced by pods via RBAC
  Complex: need PSP + RBAC binding to SA to work
  Easy to misconfigure silently
  Namespace-wide policy not straightforward
  Deprecated K8s 1.21, Removed K8s 1.25

Pod Security Standards (PSS) + PSA:
  Built into API Server as admission controller
  Namespace-level labels — simple to apply
  Three well-defined profiles: privileged/baseline/restricted
  No RBAC required — just namespace labels
  Three modes: enforce/warn/audit — gradual rollout
  Cannot express custom policies (use OPA/Gatekeeper for that)

If you need custom policies beyond three profiles:
  → OPA/Gatekeeper (7.10) or Kyverno (7.11)
```

---

### 🎤 Short Crisp Interview Answer

> *"Pod Security Standards define three profiles: Privileged (no restrictions, for system daemonsets), Baseline (blocks known exploits like privileged containers and hostPath), and Restricted (requires runAsNonRoot, seccomp profile, drop all capabilities, no privilege escalation). Pod Security Admission enforces these via namespace labels with three modes: enforce rejects violating pods, warn lets them through with a client warning, audit records violations in audit logs without user impact. The typical migration path is: audit first to discover violations, add warn so developers see them, then enforce. PSA replaced PodSecurityPolicy which was removed in Kubernetes 1.25."*

---

### ⚠️ Gotchas

1. **Switching to enforce on a namespace with running pods doesn't evict them** — PSA checks at admission time only. Existing violating pods keep running. You must restart them to trigger enforcement.
2. **kube-system must be Privileged** — system components like CNI, CSI, and kube-proxy need privileged access. Never apply Restricted or Baseline to kube-system.
3. **Restricted requires container image to support non-root** — many official Docker Hub images run as root by default. They need explicit USER directives in their Dockerfile or runAsUser in securityContext.

---

---

# ⚠️ 7.4 Admission Controllers — What They Are, Key Ones

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Admission controllers are **plugins that intercept API Server requests after authentication and authorization but before the object is persisted to etcd**. They can mutate (modify) objects or validate (approve/reject) them. They are the enforcement layer between "you have permission to do this" and "this request meets cluster policy."

---

### The Request Pipeline

```
API SERVER REQUEST FLOW
═══════════════════════════════════════════════════════════════

kubectl apply → API Server:

  1. AUTHENTICATION:  Who are you?
     (certificates, OIDC tokens, bootstrap tokens)

  2. AUTHORIZATION:  Are you allowed to do this? (RBAC)
     (can jane@company.com create pods in production?)

  3. ADMISSION:  Does this request meet cluster policy?
     ├── MUTATING admission controllers (modify the object)
     │     All mutating controllers run in series
     │     Can ADD fields, change values, inject containers
     │
     ├── OBJECT VALIDATION (JSON schema check)
     │     API Server validates the object structure
     │
     └── VALIDATING admission controllers (approve/reject)
           All validating controllers run in parallel
           Any single rejection = request rejected

  4. WRITE TO ETCD (if all admission passed)

  5. RESPONSE to client

KEY INSIGHT:
  RBAC = "are you ALLOWED to do this operation?"
  Admission = "is this specific REQUEST OK to proceed?"
  Both must pass. RBAC passes first, then admission.
```

---

### Key Built-in Admission Controllers

```
ADMISSION CONTROLLERS ENABLED BY DEFAULT (selection):
═══════════════════════════════════════════════════════════════

NamespaceLifecycle:
  Prevents: creating resources in a terminating namespace
            creating resources in non-existent namespace
  Always enabled — critical.

LimitRanger:
  Evaluates LimitRange objects — injects default requests/limits
  into containers that don't specify them.
  Without this: LimitRange objects exist but do nothing.

ResourceQuota:
  Evaluates ResourceQuota objects — rejects requests that would
  exceed namespace quota.
  Without this: ResourceQuota objects exist but do nothing.

ServiceAccount:
  Auto-injects default ServiceAccount into pods that don't specify one.
  Auto-injects automountServiceAccountToken projected volume.

PersistentVolumeClaimResize:
  Validates that PVC resize requests are valid and allowed.

DefaultStorageClass:
  Injects the default StorageClass into PVCs with no storageClassName.
  Without this: PVCs without explicit SC get no storage class.

PodSecurity (built-in):
  Enforces Pod Security Standards (PSS) via namespace labels.
  Replaced PodSecurityPolicy.

MutatingAdmissionWebhook:
  Calls external mutating webhooks (user-defined).
  Enables: Istio sidecar injection, ESO mutations, custom policies.

ValidatingAdmissionWebhook:
  Calls external validating webhooks (user-defined).
  Enables: OPA/Gatekeeper, Kyverno, custom validation.

NodeRestriction:
  Limits what a kubelet can modify via the API.
  Kubelet can only modify its own Node and pods on its node.
  Prevents compromised kubelet from modifying other nodes.

Priority:
  Sets pod.spec.priority from priorityClassName.
  Without this: PriorityClass objects don't affect pods.

StorageObjectInUseProtection:
  Adds finalizer to PVCs and PVs in use by pods.
  Prevents PVC/PV deletion while pod is using them.

PodNodeSelector:
  Restricts pod.spec.nodeSelector to specific labels per namespace.
  Useful for enforcing pod → node pool assignment.
```

---

### The Critical Mutating → Validating Order

```
WHY ORDER MATTERS
═══════════════════════════════════════════════════════════════

Mutating controllers run FIRST (in series, alphabetically by name
for webhooks at the same reinvocationPolicy level).

This means:
  1. Mutating controller A adds: securityContext.runAsUser: 1000
  2. Mutating controller B adds: a sidecar container
  3. Object now looks different from what user submitted
  4. Validating controllers evaluate the FINAL MUTATED object

If validating ran before mutating:
  OPA/Gatekeeper might reject a pod for missing securityContext
  even though Istio's mutating webhook would have added it.

REINVOCATION POLICY (K8s 1.15+):
  If webhook A's mutation causes another webhook B to run again
  (because B's objectSelector now matches), B must re-run.
  reinvocationPolicy: IfNeeded
    → webhook called again if another webhook mutated the object
  reinvocationPolicy: Never (default)
    → webhook called exactly once

PRACTICAL IMPLICATION:
  Ensure mutating webhooks (sidecar injectors, defaults) run BEFORE
  validating webhooks (OPA/Gatekeeper) see the pod.
  Configure webhook reinvocation if order matters.
```

---

### kubectl Admission Controller Commands

```bash
# View enabled admission plugins (self-managed clusters)
kube-apiserver --help | grep enable-admission-plugins

# For kubeadm clusters — check API server manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission

# For EKS — admission controllers are managed, cannot directly view
# EKS enables: NamespaceLifecycle, LimitRanger, ServiceAccount,
#              DefaultStorageClass, DefaultTolerationSeconds,
#              MutatingAdmissionWebhook, ValidatingAdmissionWebhook,
#              ResourceQuota, PodSecurity, NodeRestriction, etc.

# View registered admission webhooks
kubectl get mutatingwebhookconfigurations
# NAME                               WEBHOOKS   AGE
# istio-sidecar-injector             1          30d
# cert-manager-webhook               1          30d
# aws-load-balancer-webhook          1          30d

kubectl get validatingwebhookconfigurations
# NAME                              WEBHOOKS   AGE
# gatekeeper-validating-webhook     2          20d
# kyverno-policy-validating-webhook 2          15d

# Describe a webhook to see its rules
kubectl describe mutatingwebhookconfiguration istio-sidecar-injector
```

---

### 🎤 Short Crisp Interview Answer

> *"Admission controllers intercept API Server requests after authentication and authorization but before writing to etcd. They run in two phases: mutating first (can modify objects — used for sidecar injection, default injection, label addition), then validating (can only approve or reject — used for policy enforcement). The order is critical: mutating runs first so validating sees the final, mutated object. Key built-in controllers include LimitRanger which applies LimitRange defaults, ResourceQuota which enforces quota, NodeRestriction which limits what a kubelet can do, and PodSecurity which enforces Pod Security Standards. MutatingAdmissionWebhook and ValidatingAdmissionWebhook are the extension points that enable tools like Istio, OPA/Gatekeeper, and Kyverno."*

---

### ⚠️ Gotchas

1. **A broken webhook can block all pod creation** — if a webhook's failurePolicy is Fail and the webhook service is down, ALL pod creations in matching namespaces fail. Use failurePolicy: Ignore for non-critical webhooks or ensure high availability.
2. **Webhooks create circular dependency risk** — a webhook pod in namespace X that applies to namespace X will try to call itself on its own creation, potentially deadlocking. Use namespaceSelector to exclude the webhook's own namespace.
3. **Admission controllers don't run on etcd writes** — if someone writes directly to etcd (bypassing the API server), admission controllers are skipped. Protect etcd access.

---

---

# ⚠️ 7.5 Webhook Admission — MutatingWebhook vs ValidatingWebhook

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Webhook admission extends the Kubernetes API Server with **external HTTP services** that can intercept object creation/modification requests. MutatingWebhookConfiguration registers webhooks that can modify objects. ValidatingWebhookConfiguration registers webhooks that can approve or reject them.

---

### Webhook Architecture

```
WEBHOOK CALL FLOW
═══════════════════════════════════════════════════════════════

User: kubectl apply -f pod.yaml

API Server:
  1. Checks MutatingWebhookConfigurations — which webhooks match?
     Match criteria: namespace labels, object labels, resource type
  2. Calls each matching mutating webhook (HTTP POST):
     Request body: AdmissionReview{request: {object: <pod JSON>}}
     Webhook responds: AdmissionReview{response: {
       allowed: true,
       patch: <base64 JSON patch>  ← mutation here
     }}
  3. Applies patch to the object
  4. Checks ValidatingWebhookConfigurations — which match?
  5. Calls all matching validating webhooks IN PARALLEL:
     Response: AdmissionReview{response: {
       allowed: false,
       status: {message: "policy violated: missing label"}
     }}
  6. If ANY validating webhook returns allowed: false → request rejected

WEBHOOK COMPONENTS:
  WebhookConfiguration: K8s object declaring what to intercept
  Webhook service:       Your HTTP server (deployed in cluster or external)
  CA bundle:             Webhook's TLS cert CA (API Server validates)
  AdmissionReview:       JSON payload (request in, response out)
```

---

### MutatingWebhookConfiguration YAML

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-defaults-injector
webhooks:
- name: pod-defaults.company.com           # unique name
  clientConfig:
    service:
      name: pod-defaults-webhook           # K8s Service for webhook
      namespace: kube-system
      path: "/mutate"
      port: 443
    caBundle: <base64-encoded-CA-cert>     # CA that signed webhook's TLS cert

  rules:                                   # WHAT to intercept
  - apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
    operations: ["CREATE"]                 # CREATE, UPDATE, DELETE, CONNECT, *

  namespaceSelector:                       # WHICH namespaces to intercept
    matchLabels:
      inject-defaults: "true"              # only namespaces with this label
    # OR exclude kube-system:
    # matchExpressions:
    # - key: kubernetes.io/metadata.name
    #   operator: NotIn
    #   values: ["kube-system", "kube-public"]

  objectSelector:                          # WHICH objects (by their labels)
    matchLabels:
      inject-sidecar: "true"              # only pods with this label

  failurePolicy: Fail                      # Fail or Ignore
  # Fail:   webhook unavailable → request rejected
  # Ignore: webhook unavailable → request proceeds (less safe)

  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None                        # None, NoneOnDryRun, Some, Unknown
  # None: webhook has no side effects (safe for dry-run calls)
  # NoneOnDryRun: has side effects but handles dry-run flag
  # Some: has side effects (not called for dry-run)

  timeoutSeconds: 10                       # fail after 10s (max 30s)

  reinvocationPolicy: IfNeeded             # re-call if another webhook mutated
```

---

### ValidatingWebhookConfiguration YAML

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: policy-validator
webhooks:
- name: validate-pods.company.com
  clientConfig:
    service:
      name: policy-webhook
      namespace: kube-system
      path: "/validate"
      port: 443
    caBundle: <base64-encoded-CA-cert>

  rules:
  - apiGroups: ["", "apps"]
    apiVersions: ["v1"]
    resources: ["pods", "deployments", "statefulsets"]
    operations: ["CREATE", "UPDATE"]

  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "kube-public", "kube-node-lease"]

  failurePolicy: Fail
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

---

### Writing a Simple Webhook Server

```go
// Minimal Go webhook that injects a label on every pod
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    admissionv1 "k8s.io/api/admission/v1"
)

type patchOperation struct {
    Op    string      `json:"op"`
    Path  string      `json:"path"`
    Value interface{} `json:"value,omitempty"`
}

func mutateHandler(w http.ResponseWriter, r *http.Request) {
    var admissionReview admissionv1.AdmissionReview
    json.NewDecoder(r.Body).Decode(&admissionReview)

    // Create JSON patch to add label
    patch := []patchOperation{
        {
            Op:    "add",
            Path:  "/metadata/labels/injected-by",
            Value: "webhook",
        },
    }
    patchBytes, _ := json.Marshal(patch)
    patchType := admissionv1.PatchTypeJSONPatch

    admissionReview.Response = &admissionv1.AdmissionResponse{
        UID:       admissionReview.Request.UID,
        Allowed:   true,
        Patch:     patchBytes,
        PatchType: &patchType,
    }

    json.NewEncoder(w).Encode(admissionReview)
}
```

---

### Debugging Webhooks

```bash
# Check webhook configurations
kubectl get mutatingwebhookconfigurations -o yaml
kubectl get validatingwebhookconfigurations -o yaml

# Test why a pod is being rejected
kubectl apply -f pod.yaml -v=8 2>&1 | grep -E "webhook|admission"
# Shows: which webhook was called, what it returned

# Temporarily bypass webhook for emergency (requires cluster-admin)
# Option 1: Add namespace to exclusion selector
kubectl label namespace production webhook-skip=true

# Option 2: Temporarily scale down webhook deployment
kubectl scale deployment policy-webhook --replicas=0 -n kube-system
# ⚠️ Only for emergencies — all policy enforcement disabled

# Check webhook service endpoints
kubectl get endpoints policy-webhook -n kube-system
# NAME              ENDPOINTS                          AGE
# policy-webhook    10.244.1.15:443,10.244.2.8:443    5d
# (must have endpoints — if empty, webhook service has no pods)

# Check webhook TLS certificate expiry
kubectl get mutatingwebhookconfiguration pod-defaults-injector \
  -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | \
  base64 -d | openssl x509 -noout -dates
# notAfter=Dec  1 00:00:00 2025 GMT  ← check this doesn't expire

# View admission events in API server logs
# On self-managed clusters:
journalctl -u kube-apiserver | grep -i "admission\|webhook" | tail -50
```

---

### 🎤 Short Crisp Interview Answer

> *"Webhook admission extends the API Server with external HTTP services. Mutating webhooks receive an AdmissionReview JSON payload and can return a JSON patch to modify the object — used for sidecar injection, adding default labels, injecting security contexts. Validating webhooks receive the same payload but can only return allowed: true or false — used for policy enforcement like OPA Gatekeeper. Mutating webhooks run first, all in series; validating webhooks run in parallel after, and any single rejection blocks the request. The most critical operational concern is failurePolicy: if set to Fail and the webhook service is down, ALL matching requests fail. Webhooks must exclude kube-system from their namespaceSelector to avoid circular dependency on their own pod creation."*

---

### ⚠️ Gotchas

1. **Webhook failing = cluster operations failing** — a webhook deployment crash during a deployment rollout means you can't create new pods until the webhook is restored. Run webhooks with multiple replicas and PodDisruptionBudgets.
2. **TLS certificate expiry breaks all webhooks** — the caBundle in the webhook config must match the webhook service's current TLS certificate. Certificates expire. Use cert-manager to auto-rotate.
3. **sideEffects must be None for dry-run** — if a webhook has sideEffects: Some, it's not called during `kubectl apply --dry-run=server`. This means dry-run validation doesn't test the webhook. Set sideEffects: None whenever possible.
4. **timeoutSeconds max is 30** — API Server enforces a hard 30-second maximum timeout. Webhooks that take longer are treated as failure (per failurePolicy).

---

---

# 7.6 Network Policies for Security (Zero-Trust)

## 🟡 Intermediate

### What it is in simple terms

By default, all pods in a Kubernetes cluster can communicate with all other pods with no restrictions — essentially flat networking. **NetworkPolicy** objects define firewall rules that restrict which pods can talk to which other pods, and which pods can access external services. Implementing NetworkPolicy correctly achieves zero-trust networking where nothing is allowed unless explicitly permitted.

---

### Zero-Trust Networking Pattern

```
DEFAULT-DENY + EXPLICIT-ALLOW PATTERN
═══════════════════════════════════════════════════════════════

STEP 1: Default deny all ingress and egress in namespace
STEP 2: Explicitly allow only required communication

This is the zero-trust pattern. Without step 1, forgetting
to add an allow rule is safe. With step 1, forgetting = denied.

COMMON MISTAKE: Apply default-deny without DNS egress exception
→ All DNS lookups fail → all apps break immediately
Always add UDP/TCP 53 egress to kube-dns FIRST.
```

---

### Network Policy YAML — Zero-Trust

```yaml
# Step 1: Default deny ALL ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}              # matches ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
  # No ingress/egress rules = deny all
  # (ingress: [] and egress: [] would be identical)

---
# Step 2: Allow DNS egress (MUST ADD BEFORE DEFAULT DENY TAKES EFFECT)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}              # all pods need DNS
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
    # No to: selector = allows DNS to any destination
    # (CoreDNS in kube-system is the target)
    # Alternatively, restrict to kube-dns pods:
    # to:
    # - namespaceSelector:
    #     matchLabels:
    #       kubernetes.io/metadata.name: kube-system
    #   podSelector:
    #     matchLabels:
    #       k8s-app: kube-dns

---
# Step 3: Allow app to receive traffic from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server          # this policy applies to api-server pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    # AND condition: namespace AND pod selector in SAME item
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: production
      podSelector:
        matchLabels:
          app: frontend        # only frontend pods in production namespace
    ports:
    - port: 8080
      protocol: TCP

---
# Step 4: Allow api-server to reach database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server          # api-server pods can send to...
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres        # ...postgres pods
    ports:
    - port: 5432
      protocol: TCP

---
# Restrict database to only receive from api-server
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-server
    ports:
    - port: 5432
      protocol: TCP
  # Also allow replication between postgres replicas
  - from:
    - podSelector:
        matchLabels:
          app: postgres        # postgres → postgres (replication)
    ports:
    - port: 5432
      protocol: TCP

---
# Allow monitoring scrape (Prometheus → all pods on /metrics)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}              # all pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - port: 9090              # metrics port
      protocol: TCP

---
# Allow egress to external APIs (by CIDR range)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0       # allow all external
        except:
        - 10.0.0.0/8          # except internal IPs
        - 172.16.0.0/12       # except pod CIDR
        - 192.168.0.0/16      # except service CIDR
    ports:
    - port: 443
      protocol: TCP
```

---

### The AND vs OR Gotcha (Critical)

```
AND vs OR IN from/to SELECTORS — MOST COMMON MISTAKE
═══════════════════════════════════════════════════════════════

CASE 1: Two conditions in SAME list item = AND
  from:
  - namespaceSelector:        ← SAME item (dash before namespaceSelector)
      matchLabels:
        env: production
    podSelector:              ← NO dash = same item as namespaceSelector
      matchLabels:
        app: frontend
  RESULT: namespace=production AND pod=frontend
          Only frontend pods IN production namespace

CASE 2: Two conditions in SEPARATE list items = OR
  from:
  - namespaceSelector:        ← first item
      matchLabels:
        env: production
  - podSelector:              ← second item (separate dash)
      matchLabels:
        app: frontend
  RESULT: namespace=production OR pod=frontend
          ALL pods in production namespace
          PLUS frontend pods in ANY namespace (dangerous!)

THIS IS THE #1 NetworkPolicy MISTAKE IN INTERVIEWS AND PRODUCTION.
```

---

### kubectl NetworkPolicy Commands

```bash
# View NetworkPolicies
kubectl get networkpolicy -n production
kubectl describe networkpolicy allow-frontend-to-api -n production

# Test connectivity (requires a debug pod)
# Install netshoot for testing:
kubectl run netshoot --image=nicolaka/netshoot \
  -n production --rm -it -- /bin/bash

# Inside netshoot pod:
# Test DNS
nslookup kubernetes.default.svc.cluster.local
# Test TCP connection to api-server pod
nc -zv api-server-svc 8080
# Test blocked connection
nc -zv postgres-svc 5432
# Connection timed out ← correctly blocked by NetworkPolicy

# Visualize NetworkPolicies (use np-viewer or similar tools)
kubectl get networkpolicies -n production -o yaml | \
  python3 -c "import sys, yaml; [print(p['metadata']['name']) for p in yaml.safe_load_all(sys.stdin)]"

# Check if CNI supports NetworkPolicy
# Flannel: does NOT support NetworkPolicy
# Calico: YES
# Cilium: YES (+ L7 policies)
# EKS VPC CNI alone: does NOT — need Calico or Network Policy Controller
```

---

### 🎤 Short Crisp Interview Answer

> *"NetworkPolicy implements zero-trust by default-denying all traffic and then explicitly allowing only required paths. The pattern is: create a default-deny-all policy first, then allow-dns-egress (critical — otherwise all DNS fails), then add specific allow rules per service. The most important gotcha is the AND vs OR semantics: namespaceSelector and podSelector in the SAME list item are AND (namespace X AND pod Y), but as SEPARATE list items they're OR (namespace X OR pod Y). NetworkPolicy requires a CNI that supports it — Flannel does not, Calico and Cilium do. On EKS, the VPC CNI alone doesn't support NetworkPolicy; you need the Network Policy Controller add-on or Calico."*

---

### ⚠️ Gotchas

1. **NetworkPolicy requires a supporting CNI** — on EKS with default VPC CNI, NetworkPolicy objects exist but do nothing. Install the Network Policy Controller add-on or use Calico/Cilium.
2. **DNS egress must be allowed before default deny** — applying default-deny without first allowing UDP/TCP port 53 egress breaks all pod DNS resolution immediately.
3. **podSelector: {} in a policy matches ALL pods** — an empty podSelector is NOT "match nothing" — it matches ALL pods in the namespace. Be careful when writing default-deny rules.
4. **NetworkPolicy doesn't affect host network pods** — pods with hostNetwork: true bypass NetworkPolicy enforcement in some CNI implementations.

---

---

# 7.7 OIDC Integration — External Identity Providers

## 🟡 Intermediate

### What it is in simple terms

Kubernetes has no built-in user store — it delegates human authentication to external identity providers via **OIDC (OpenID Connect)**. When configured, users log in through their corporate IdP (Azure AD, Okta, Google, Keycloak), receive a JWT token, and present it to the Kubernetes API Server, which validates the token and extracts the username and group memberships.

---

### OIDC Authentication Flow

```
OIDC AUTHENTICATION FLOW
═══════════════════════════════════════════════════════════════

1. Developer runs: kubectl get pods
   kubeconfig has: user.auth-provider.oidc or exec credential plugin

2. kubectl needs a token → calls auth helper (kubelogin)
   kubelogin redirects developer to IdP login page (browser)
   Developer authenticates with SSO/MFA

3. IdP issues ID Token (JWT) containing:
   sub:    "user123" (user ID)
   email:  "jane@company.com"
   groups: ["platform-team", "developers"]
   aud:    "kubernetes"
   exp:    (expiry — short-lived, typically 1 hour)

4. kubectl sends API request with:
   Authorization: Bearer <id_token>

5. API Server validates:
   - Token signature using OIDC provider's public key (JWKS)
   - aud claim matches configured --oidc-client-id
   - Token is not expired
   - iss claim matches --oidc-issuer-url

6. API Server extracts identity:
   Username = claims[--oidc-username-claim]  (default: sub or email)
   Groups   = claims[--oidc-groups-claim]    (default: groups)

7. RBAC checks: is jane@company.com / platform-team allowed to get pods?

8. Response returned
```

---

### API Server OIDC Configuration

```yaml
# kube-apiserver flags (kubeadm: /etc/kubernetes/manifests/kube-apiserver.yaml)
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    # OIDC configuration flags:
    - --oidc-issuer-url=https://login.microsoftonline.com/<tenant-id>/v2.0
      # For Azure AD — or Okta, Google, Keycloak URL
    - --oidc-client-id=kubernetes
      # Client ID registered in the IdP for Kubernetes
    - --oidc-username-claim=email
      # Which JWT claim to use as K8s username (sub, email, preferred_username)
    - --oidc-username-prefix=oidc:
      # Prefix prevents collision: "oidc:jane@company.com" vs service account names
    - --oidc-groups-claim=groups
      # Which JWT claim contains group memberships
    - --oidc-groups-prefix=oidc:
      # Prefix groups: "oidc:platform-team"
    - --oidc-ca-file=/etc/kubernetes/pki/oidc-ca.crt
      # CA cert for IdP TLS (if internal/self-signed)
    - --oidc-required-claim=aud=kubernetes
      # Optional: require specific claim value in token

# EKS: Configured via OIDC identity provider association
# aws eks associate-identity-provider-config \
#   --cluster-name my-cluster \
#   --oidc {...}
# OR use Kubernetes OIDC add-on
```

---

### kubeconfig for OIDC Users

```yaml
# ~/.kube/config — user section for OIDC
users:
- name: jane-oidc
  user:
    # Option A: exec plugin (recommended — kubelogin)
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://login.microsoftonline.com/<tenant>/v2.0
      - --oidc-client-id=kubernetes
      - --oidc-client-secret=<client-secret>
      # kubelogin handles: token caching, refresh, browser redirect
      # Install: brew install int128/kubelogin/kubelogin

    # Option B: manual token (for testing only, expires quickly)
    # token: <id_token_jwt>

contexts:
- name: production
  context:
    cluster: production-cluster
    user: jane-oidc
    namespace: production

---
# RBAC binding for OIDC user and groups
# The username is "oidc:jane@company.com" (with prefix)
# The group is "oidc:platform-team" (with prefix)

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-team-admin
subjects:
- kind: Group
  name: "oidc:platform-team"    # matches --oidc-groups-prefix + group name
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-production-access
  namespace: production
subjects:
- kind: User
  name: "oidc:jane@company.com"  # matches --oidc-username-prefix + claim value
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

---

### EKS OIDC Configuration

```bash
# EKS has built-in OIDC provider per cluster for IRSA
# For HUMAN authentication to EKS, options:
# 1. aws-auth ConfigMap (older approach — IAM users/roles mapped to K8s users)
# 2. EKS Access Entries (newer, recommended)
# 3. OIDC + API Server flags (for non-AWS IdPs)

# EKS Access Entries (recommended for human auth):
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/DeveloperRole \
  --type STANDARD \
  --kubernetes-groups platform-team

aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/DeveloperRole \
  --access-scope type=namespace,namespaces=production \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy

# View aws-auth ConfigMap (legacy)
kubectl get configmap aws-auth -n kube-system -o yaml
```

---

### 🎤 Short Crisp Interview Answer

> *"Kubernetes has no user store — authentication is delegated to external OIDC identity providers like Azure AD, Okta, or Keycloak. The API Server is configured with the IdP's issuer URL and validates tokens by fetching the JWKS public keys. The token's claims are mapped to K8s username and group using --oidc-username-claim and --oidc-groups-claim flags. A prefix like 'oidc:' prevents name collisions between OIDC users and ServiceAccounts. Once the API Server extracts the identity, normal RBAC applies. On EKS, human authentication to the cluster uses IAM identities mapped via aws-auth ConfigMap or the newer Access Entries, while IRSA uses the cluster's OIDC provider for pod workload identity."*

---

### ⚠️ Gotchas

1. **API Server can only have ONE OIDC provider** — if you need multiple IdPs, use a proxy like Dex or Pinniped that federates multiple providers and presents a single OIDC endpoint to the API Server.
2. **Token expiry requires re-authentication** — OIDC tokens are short-lived (1 hour typically). kubelogin handles refresh automatically, but static tokens in kubeconfig silently fail after expiry.
3. **EKS doesn't support --oidc-* flags directly** — EKS manages its API Server and doesn't expose arbitrary kube-apiserver flags. Use EKS OIDC provider integration or third-party solutions like Teleport.

---

---

# 7.8 Audit Logging

## 🟡 Intermediate

### What it is in simple terms

Kubernetes audit logging records every API Server request — who did what, when, on which object, and what the outcome was. It's the cluster's security audit trail, required for compliance (SOC 2, PCI-DSS, HIPAA) and essential for incident investigation.

---

### Audit Policy Levels

```
AUDIT LOG LEVELS (from least to most verbose):
═══════════════════════════════════════════════════════════════

None:
  Do not log this request at all.
  Use for: health checks, metrics scrapes, high-volume reads

Metadata:
  Log request metadata only:
    who (user, groups), what (verb, resource, namespace),
    when (timestamp), outcome (status code)
  Does NOT log request body or response body.
  Use for: most read operations (get, list, watch)

Request:
  Log metadata + request body.
  Does NOT log response body.
  Use for: write operations (create, update, patch)

RequestResponse:
  Log metadata + request body + response body.
  Highest verbosity — generates large log volume.
  Use for: sensitive operations (secret access, RBAC changes)
```

---

### Audit Policy YAML

```yaml
# /etc/kubernetes/audit-policy.yaml
# (referenced by kube-apiserver --audit-policy-file flag)
apiVersion: audit.k8s.io/v1
kind: Policy

# OmitStages: skip logging during these request phases
# (default: log all stages including RequestReceived)
omitStages:
- "RequestReceived"    # skip — creates duplicate entries with ResponseComplete

rules:
# Rule: evaluated TOP TO BOTTOM — first matching rule wins

# 1. Never log these (high-volume, low-value)
- level: None
  users: ["system:kube-proxy"]          # kube-proxy watch is very noisy
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services", "services/status"]

- level: None
  userGroups: ["system:authenticated"]
  nonResourceURLs:
  - "/api*"          # API discovery — very high volume, low value
  - "/version"

- level: None
  users: ["system:kube-scheduler", "system:kube-controller-manager"]
  verbs: ["get", "list", "watch"]        # internal components, very noisy

# 2. Log secret access at RequestResponse level (high sensitivity)
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
  # Log request + response for secrets — who read what value

# 3. Log RBAC changes at RequestResponse
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]
  verbs: ["create", "update", "patch", "delete"]

# 4. Log pod exec/port-forward (potential breakout vector)
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods/exec", "pods/portforward", "pods/attach"]
  verbs: ["create"]

# 5. Log writes at Request level
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  omitStages:
  - "RequestReceived"

# 6. Default: Metadata for everything else
- level: Metadata
  # Catches all remaining requests with just metadata
```

---

### Audit Backend Configuration

```yaml
# kube-apiserver flags for audit logging
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    # Policy file
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml

    # BACKEND 1: Log file (simple, good for CloudWatch/Splunk ingestion)
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30         # retain 30 days
    - --audit-log-maxbackup=10      # keep 10 rotated files
    - --audit-log-maxsize=100       # rotate at 100 MB
    - --audit-log-format=json       # json (default) or legacy

    # BACKEND 2: Webhook (real-time streaming to SIEM)
    - --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
    - --audit-webhook-batch-max-size=400
    - --audit-webhook-batch-max-wait=30s

# EKS: Control plane audit logs sent to CloudWatch Logs
# Enable in aws console or:
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["audit"],"enabled":true}]}'
# Logs appear in: /aws/eks/<cluster-name>/cluster log group
# Log streams: kube-apiserver-audit-*
```

---

### Reading Audit Logs

```bash
# Sample audit log entry (JSON format):
# {
#   "kind": "Event",
#   "apiVersion": "audit.k8s.io/v1",
#   "level": "RequestResponse",
#   "auditID": "abc-def-123",
#   "stage": "ResponseComplete",
#   "requestURI": "/api/v1/namespaces/production/secrets/db-credentials",
#   "verb": "get",
#   "user": {
#     "username": "oidc:jane@company.com",
#     "groups": ["oidc:platform-team", "system:authenticated"]
#   },
#   "sourceIPs": ["10.0.1.5"],
#   "objectRef": {
#     "resource": "secrets",
#     "namespace": "production",
#     "name": "db-credentials"
#   },
#   "responseStatus": {"code": 200},
#   "requestReceivedTimestamp": "2024-01-15T10:30:00Z",
#   "stageTimestamp": "2024-01-15T10:30:00.050Z"
# }

# Find all secret access events
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets") |
      {user: .user.username, secret: .objectRef.name, 
       verb: .verb, time: .requestReceivedTimestamp}'

# Find who deleted what
cat /var/log/kubernetes/audit.log | \
  jq 'select(.verb == "delete") |
      {user: .user.username, resource: .objectRef.resource,
       name: .objectRef.name, ns: .objectRef.namespace}'

# Find kubectl exec events (potential intrusion)
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.subresource == "exec") |
      {user: .user.username, pod: .objectRef.name, 
       ns: .objectRef.namespace, time: .requestReceivedTimestamp}'

# CloudWatch Logs Insights query for EKS:
# fields @timestamp, user.username, verb, objectRef.resource, objectRef.name
# | filter objectRef.resource = "secrets"
# | filter verb in ["get", "list"]
# | sort @timestamp desc
# | limit 100
```

---

### 🎤 Short Crisp Interview Answer

> *"Audit logging records every API Server request — who, what, when, on which object, and the outcome. There are four log levels: None (skip), Metadata (just headers and user), Request (plus request body), and RequestResponse (plus response body). The audit policy file defines rules evaluated top-to-bottom, first match wins. Production practice is to log secrets at RequestResponse level to capture who read what, log pod exec at RequestResponse since it's a breakout vector, skip system component watch traffic that's extremely noisy, and log all writes at Request level. On EKS, control plane audit logs flow to CloudWatch Logs when enabled."*

---

---

# 7.9 Seccomp, AppArmor, SELinux Profiles

## 🟡 Intermediate

### What it is in simple terms

These are **Linux kernel security mechanisms** that restrict what system calls and kernel operations containerized processes can perform — an additional defense layer beyond container isolation.

---

### Seccomp — System Call Filtering

```
SECCOMP (Secure Computing Mode):
Restricts which system calls a process can make.
A container normally has access to ~300+ Linux syscalls.
Many syscalls are powerful/dangerous:
  ptrace:   debug/control other processes
  setuid:   change user identity
  mount:    mount filesystems (container escape vector)
  kexec:    replace kernel (ultimate escalation)
  etc.

Seccomp profile = allowlist or denylist of syscalls.
If process makes a denied syscall:
  SCMP_ACT_ERRNO: returns EPERM (operation not permitted)
  SCMP_ACT_KILL:  process killed immediately (SIGKILL)
  SCMP_ACT_TRAP:  send SIGSYS to process
```

---

### Seccomp YAML

```yaml
# Apply RuntimeDefault seccomp profile (recommended baseline)
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
      # RuntimeDefault: containerd/CRI-O's default profile
      # Blocks ~30-40 most dangerous syscalls (ptrace, mount, etc.)
      # Allows all syscalls typical apps need
      # Required by PSS Restricted profile

# Apply custom seccomp profile from file
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-app.json
      # File must exist at:
      # /var/lib/kubelet/seccomp/profiles/my-app.json
      # on the node where pod runs

# Generate a custom profile with audit mode first:
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
      # audit.json: all syscalls LOG but not blocked
      # Collect for a week → use syscalls seen → create allowlist

---
# Seccomp profile JSON example (minimal — audit log mode)
# /var/lib/kubelet/seccomp/profiles/audit.json
{
  "defaultAction": "SCMP_ACT_LOG",   # log all syscalls (audit mode)
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": []                      # empty = use defaultAction
}

# Production allowlist profile:
{
  "defaultAction": "SCMP_ACT_ERRNO",  # block by default
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "fstat",
                "lstat", "poll", "lseek", "mmap", "mprotect", "munmap",
                "brk", "rt_sigaction", "rt_sigprocmask", "ioctl",
                "pread64", "pwrite64", "readv", "writev", "access",
                "pipe", "select", "sched_yield", "mremap", "msync",
                "socket", "connect", "accept", "sendto", "recvfrom",
                "sendmsg", "recvmsg", "bind", "listen", "getsockname",
                "getpeername", "setsockopt", "getsockopt", "fork",
                "execve", "exit", "wait4", "getpid", "getuid", "geteuid"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

---

### AppArmor

```
APPARMOR — Mandatory Access Control (MAC):
Kernel module (Linux Security Module — LSM).
Restricts what FILES, CAPABILITIES, and NETWORK operations
a process can perform, based on named profiles.
Available on: Ubuntu/Debian nodes (default), some distros.
NOT available on: RHEL/CentOS/Amazon Linux (use SELinux instead).

EKS AMI:
  Amazon Linux 2:  uses SELinux (AppArmor not available)
  Bottlerocket OS: has custom security model
  Ubuntu EKS AMI:  AppArmor available

PROFILE MODES:
  enforce: violations denied + logged
  complain: violations logged only (discovery mode)
  disabled: profile inactive
```

```yaml
# Apply AppArmor profile via annotation (pre-K8s 1.30)
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
    # OR:    localhost/my-custom-profile   (profile on node)
    # OR:    unconfined                    (no AppArmor — less secure)
spec:
  containers:
  - name: app
    image: myapp:v1

# K8s 1.30+: securityContext field (stable)
spec:
  containers:
  - name: app
    securityContext:
      appArmorProfile:
        type: RuntimeDefault   # RuntimeDefault, Localhost, Unconfined
        # localhostProfile: my-profile  (if type: Localhost)
```

---

### SELinux

```
SELINUX — Security-Enhanced Linux:
Kernel security module for Mandatory Access Control.
Used on: RHEL, CentOS, Amazon Linux, Fedora.
Every process, file, port has a LABEL (context).
Policy rules define: which process labels can access which file labels.

SELINUX CONTEXTS:
  system_u:system_r:container_t:s0:c123,c456
  user:role:type:level:categories

CONTAINER SELINUX:
  Container runtime assigns a unique MCS category pair (c123,c456)
  to each container's process.
  Ensures container A cannot access container B's files even on same node
  (different categories → kernel blocks access).

In pods:
  spec.containers[].securityContext.seLinuxOptions:
    user: "_"
    role: "_"
    type: "container_t"
    level: "s0:c123,c456"
  Usually left to container runtime — it assigns correct labels.
  Override only if specific file access requirements exist.
```

---

### Security Layers Summary

```
DEFENSE IN DEPTH — LINUX SECURITY LAYERS
═══════════════════════════════════════════════════════════════

Layer 1: RBAC + Pod Security Admission
  Prevents: privileged pod creation, hostNetwork, hostPath

Layer 2: Seccomp
  Restricts: system calls available to process
  Defense: if container exploited, many dangerous syscalls unavailable

Layer 3: AppArmor / SELinux
  Restricts: file/network/capability access by MAC policy
  Defense: process cannot access files outside its policy even with exploit

Layer 4: Capabilities (securityContext.capabilities.drop)
  Restricts: elevated Linux capabilities (CAP_NET_ADMIN, CAP_SYS_ADMIN)
  Default caps: 14 capabilities → drop all → add only what's needed

Layer 5: Read-only root filesystem
  Restricts: malware cannot write to container filesystem
  Use emptyDir for /tmp if needed

Layer 6: runAsNonRoot + runAsUser
  Restricts: even if container breaks out, not running as root on host

ALL LAYERS APPLIED = no single exploit grants full access
```

---

### 🎤 Short Crisp Interview Answer

> *"These are Linux kernel security mechanisms layered on top of container isolation. Seccomp filters system calls — the RuntimeDefault profile blocks the ~30-40 most dangerous syscalls like ptrace and mount. It's required by the PSS Restricted profile. AppArmor is a MAC module that restricts file, network, and capability access by named profiles — available on Ubuntu/Debian nodes but not Amazon Linux. SELinux is the RHEL/Amazon Linux equivalent using security labels and categories — the container runtime automatically assigns unique MCS categories to each container so they can't access each other's files. Together with capabilities.drop: ALL, readOnlyRootFilesystem, and runAsNonRoot, they form defense-in-depth where no single exploit grants full access."*

---

---

# ⚠️ 7.10 OPA/Gatekeeper — Policy as Code

## 🔴 Advanced — HIGH PRIORITY

### What it is in simple terms

**OPA (Open Policy Agent)** is a general-purpose policy engine. **Gatekeeper** is its Kubernetes integration — a validating (and mutating) webhook that evaluates custom policies written in the **Rego** language against API Server requests. It replaces ad-hoc admission webhooks with a declarative, GitOps-friendly policy framework.

---

### Why OPA/Gatekeeper Exists

```
THE POLICY GAP IN KUBERNETES
═══════════════════════════════════════════════════════════════

RBAC answers: can you do this operation?
PSS answers:  is the pod's security context safe?
OPA/Gatekeeper answers: does this object meet OUR business rules?

Examples of policies OPA/Gatekeeper handles that PSS/RBAC cannot:
  ✓ All Deployments must have resource requests set
  ✓ All containers must use images from our private registry only
  ✓ No two Ingresses can use the same hostname
  ✓ All pods must have required labels (team, cost-center, environment)
  ✓ Namespaces must follow naming convention
  ✓ PVCs in production namespace must use SSD storage class
  ✓ Deployments must not use :latest image tag
  ✓ HPA must be defined for any Deployment over 1 replica
  ✓ ConfigMaps cannot contain strings that look like passwords

These are ORGANIZATIONAL POLICIES, not security primitives.
PSS Restricted is too rigid (specific security model).
RBAC can't express "only if field X has value Y".
OPA/Gatekeeper fills this gap.
```

---

### Gatekeeper Architecture

```
GATEKEEPER COMPONENTS
═══════════════════════════════════════════════════════════════

ConstraintTemplate (CRD definition):
  Defines a new CRD type and the Rego policy logic.
  Written by policy authors.
  Example: "RequiredLabels" — a template for label policies

Constraint (instance of ConstraintTemplate):
  Configures the template for a specific use case.
  Specifies: what resources to check, what parameters.
  Example: "all-pods-must-have-team-label"

Gatekeeper Controller:
  Validates incoming requests against all Constraints.
  Also enforces audit mode on existing objects.

Audit Controller:
  Periodically scans EXISTING cluster resources.
  Reports violations even for objects created before policy.
  Violations stored in Constraint.status.violations[]

OPA Cache:
  Keeps a copy of cluster resources for policy evaluation.
  Rego policies can reference other cluster objects
  (e.g., "does this namespace have the required quota?")
```

---

### OPA/Gatekeeper YAML — Full Example

```yaml
# Step 1: ConstraintTemplate — defines the policy logic in Rego
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels           # new CRD type this creates
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:                        # parameter: list of required labels
              type: array
              items:
                type: string

  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      # violation block: if ANY iteration produces a violation{}, request fails
      violation[{"msg": msg, "details": {"missing_labels": missing}}] {
        # 'input' is the AdmissionReview request object
        provided := {label | input.review.object.metadata.labels[label]}
        # input.parameters comes from the Constraint (not ConstraintTemplate)
        required := {label | label := input.parameters.labels[_]}
        # Set difference: required labels not in provided labels
        missing := required - provided
        # Condition: if missing is NOT empty, this rule fires
        count(missing) > 0
        # Format the violation message
        msg := sprintf("Missing required labels: %v", [missing])
      }

---
# Step 2: Constraint — configure the template for specific resources
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels              # Kind from ConstraintTemplate.spec.crd.names.kind
metadata:
  name: pods-must-have-team-label
spec:
  enforcementAction: deny            # deny (reject), warn, or dryrun
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    # Scope: which namespaces to apply
    namespaces: []                   # empty = all namespaces
    excludedNamespaces:
    - kube-system
    - monitoring
    # Object selector: only apply to objects with these labels
    # labelSelector: {}

  parameters:                        # passed to input.parameters in Rego
    labels: ["team", "cost-center", "environment"]

---
# More examples: no-latest-tag policy

apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snolatesttag
spec:
  crd:
    spec:
      names:
        kind: K8sNoLatestTag
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8snolatesttag
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        # Check if image ends with :latest or has no tag
        endswith(container.image, ":latest")
        msg := sprintf("Container '%v' uses :latest tag. Pin to specific version.", [container.name])
      }
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        # Image with no tag at all = :latest implicitly
        not contains(container.image, ":")
        msg := sprintf("Container '%v' has no image tag. Pin to specific version.", [container.name])
      }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoLatestTag
metadata:
  name: no-latest-tag
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: ["", "apps"]
      kinds: ["Pod", "Deployment", "StatefulSet", "DaemonSet", "Job", "CronJob"]
    excludedNamespaces: ["kube-system"]

---
# Require resource requests policy
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequireresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequireResources
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequireresources
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.requests.cpu
        msg := sprintf("Container '%v' missing CPU request", [container.name])
      }
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.requests.memory
        msg := sprintf("Container '%v' missing memory request", [container.name])
      }
```

---

### kubectl Gatekeeper Commands

```bash
# Install Gatekeeper
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system --create-namespace

# View ConstraintTemplates
kubectl get constrainttemplate
# NAME                   AGE
# k8srequiredlabels      5d
# k8snolatesttag         5d
# k8srequireresources    5d

# View Constraints and their violation counts
kubectl get k8srequiredlabels
# NAME                         ENFORCEMENT-ACTION  TOTAL-VIOLATIONS
# pods-must-have-team-label    deny                12

# View violations on a specific constraint
kubectl describe k8srequiredlabels pods-must-have-team-label
# Status:
#   Audit Timestamp: 2024-01-15T10:00:00Z
#   Total Violations: 12
#   Violations:
#   - Enforcement Action: deny
#     Kind: Deployment
#     Message: Missing required labels: {"cost-center"}
#     Name: api-deployment
#     Namespace: production

# Test a policy before deploying
kubectl apply --dry-run=server -f my-pod.yaml
# Error: admission webhook denied the request:
#        Missing required labels: {"team", "cost-center"}

# View all violations across all constraints
kubectl get constraint -A \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.totalViolations}{"\n"}{end}'

# Rego unit testing
opa test my_policy.rego my_policy_test.rego -v
```

---

### 🎤 Short Crisp Interview Answer

> *"OPA Gatekeeper is a validating (and mutating) webhook that evaluates custom policies written in Rego against API Server requests. Policies are split into ConstraintTemplates which define the Rego logic and create a new CRD, and Constraints which are instances that configure the template for specific resources. This separation lets platform teams write templates once and teams instantiate them with different parameters. The three enforcement actions are deny (reject the request), warn (proceed with a warning), and dryrun (just record violations for audit). The audit controller also scans existing objects and reports violations on Constraint status — so you can see what's currently violating policy without waiting for the next create/update. Common use cases: enforce required labels, block :latest image tags, require resource requests, restrict to approved registries."*

---

### ⚠️ Gotchas

1. **Gatekeeper webhook outage = all matching requests blocked** — with failurePolicy: Fail, if Gatekeeper pods are down, no pods can be created. Run 3+ Gatekeeper replicas in production.
2. **Rego learning curve** — Rego's logic is declarative and different from imperative languages. Violations fire when ANY iteration of the violation rule produces a result. Test with `opa eval` before deploying.
3. **Audit violations are snapshot, not real-time** — audit runs every minute by default. A resource can exist in violation for up to a minute before it appears in constraint status.

---

---

# 7.11 Kyverno — Policy Engine Alternative

## 🔴 Advanced

### What it is in simple terms

Kyverno is a **Kubernetes-native policy engine** that uses YAML (not Rego) to define policies. It can validate, mutate, generate, and clean up Kubernetes resources. Compared to OPA/Gatekeeper, Kyverno is more accessible (no new language), natively understands Kubernetes structure, and has more built-in actions.

---

### Kyverno vs OPA/Gatekeeper

```
KYVERNO vs OPA/GATEKEEPER COMPARISON
═══════════════════════════════════════════════════════════════

Policy language:
  Kyverno:         YAML + JMESPath expressions
  OPA/Gatekeeper:  Rego (purpose-built policy language)

Learning curve:
  Kyverno:    Low — familiar YAML syntax, no new language
  OPA:        Higher — Rego is non-trivial to learn

Policy types:
  Kyverno:    validate, mutate, generate, cleanup
  Gatekeeper: validate (+ mutate via mutation policies, alpha)

Generate resources:
  Kyverno:    YES — can create resources (e.g., auto-create NetworkPolicy)
  Gatekeeper: NO

Cleanup:
  Kyverno:    YES — can delete resources based on conditions
  Gatekeeper: NO

Ecosystem:
  Kyverno:         250+ policies at kyverno.io/policies
  Gatekeeper:      Gatekeeper Library, OPA policy library

Complexity:
  Kyverno:    Simpler for common cases
  OPA:        More powerful for complex logic (Rego is Turing-complete)

Maturity:
  Both are CNCF projects, widely used in production.
  OPA/Gatekeeper: longer history, broader policy ecosystem
  Kyverno:        faster-growing, simpler for K8s-native policies
```

---

### Kyverno Policy YAML — Examples

```yaml
# Kyverno ClusterPolicy — validate: require labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-label
spec:
  validationFailureAction: Enforce  # Enforce (deny) or Audit (just warn/log)
  background: true                  # apply to existing resources in audit mode

  rules:
  - name: check-team-label
    match:
      any:
      - resources:
          kinds: ["Deployment", "StatefulSet", "DaemonSet"]
          namespaces:
          - "production"
          - "staging"
    validate:
      message: "The 'team' label is required on all Deployments."
      pattern:
        metadata:
          labels:
            team: "?*"             # must exist and be non-empty

---
# Kyverno — mutate: add default label if missing
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
  - name: add-environment-label
    match:
      any:
      - resources:
          kinds: ["Pod"]
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(environment): "unknown"   # + prefix = add only if missing
                                        # (don't overwrite existing value)

---
# Kyverno — mutate: set security defaults
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: set-security-context
spec:
  rules:
  - name: set-runasnonroot
    match:
      any:
      - resources:
          kinds: ["Pod"]
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"              # apply to all containers (glob match)
            securityContext:
              +(allowPrivilegeEscalation): false
              +(readOnlyRootFilesystem): true

---
# Kyverno — generate: auto-create NetworkPolicy on namespace creation
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-networkpolicy
spec:
  rules:
  - name: default-deny-ingress
    match:
      any:
      - resources:
          kinds: ["Namespace"]
          selector:
            matchLabels:
              enforce-network-policy: "true"
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: default-deny-ingress
      namespace: "{{request.object.metadata.name}}"  # target namespace
      synchronize: true       # keep generated resource in sync with this policy
      data:
        spec:
          podSelector: {}
          policyTypes: ["Ingress"]

---
# Kyverno — block :latest image tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Using :latest or no tag is not allowed. Pin to a specific version."
      foreach:
      - list: "request.object.spec.containers"
        deny:
          conditions:
            any:
            - key: "{{ element.image }}"
              operator: Equals
              value: "*:latest"
            - key: "{{ element.image }}"
              operator: NotContains
              value: ":"               # no tag at all

---
# Kyverno — cleanup: delete completed Jobs older than 1 hour
apiVersion: kyverno.io/v2alpha1
kind: ClusterCleanupPolicy
metadata:
  name: cleanup-completed-jobs
spec:
  match:
    any:
    - resources:
        kinds: ["Job"]
  conditions:
    any:
    - key: "{{ target.status.completionTime }}"
      operator: LessThan
      value: "{{ time_now_utc() | time_diff_from_now_utc(@) | [0] }}"
  schedule: "*/30 * * * *"      # run cleanup every 30 minutes
```

---

### kubectl Kyverno Commands

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno \
  --namespace kyverno --create-namespace

# View policies
kubectl get clusterpolicy
# NAME                     BACKGROUND  VALIDATE ACTION  READY   AGE
# require-team-label       true        Enforce          true    5d
# disallow-latest-tag      true        Enforce          true    5d
# add-networkpolicy        true        -                true    5d

# View policy reports (audit results)
kubectl get policyreport -A
kubectl get clusterpolicyreport

# Describe policy report for a namespace
kubectl describe policyreport -n production
# Results:
#   Message: validation rule 'check-team-label' failed...
#   Policy:  require-team-label
#   Result:  fail
#   Resource: Deployment/api-server

# Test policy against a resource without applying
kyverno apply policy.yaml --resource pod.yaml
# PASS: 1, FAIL: 0, WARN: 0, ERROR: 0, SKIP: 0

# Generate policy from existing resources (policy discovery)
kyverno generate policy --from-resource my-deployment.yaml
```

---

### 🎤 Short Crisp Interview Answer

> *"Kyverno is a Kubernetes-native policy engine that uses YAML instead of a custom policy language. It supports four operations: validate (reject or audit policy violations), mutate (patch objects before they're stored), generate (create additional resources automatically), and cleanup (delete resources based on conditions). The main advantages over Gatekeeper are the lower learning curve (pure YAML, no Rego), the generate capability which can auto-create NetworkPolicies or ConfigMaps when namespaces are created, and cleaner Kubernetes-native syntax. For complex multi-object logic or non-Kubernetes use cases, OPA/Gatekeeper's Rego is more powerful. Both are production-grade — choice depends on team familiarity and specific policy requirements."*

---

---

# 7.12 Workload Identity (IRSA on EKS)

## 🔴 Advanced

### What it is in simple terms

**IRSA (IAM Roles for Service Accounts)** lets pods running on EKS **assume AWS IAM roles without static credentials**. A pod's Kubernetes ServiceAccount token is exchanged for temporary AWS STS credentials via OpenID Connect federation. No AWS access keys stored in Secrets — pods get fresh, short-lived credentials automatically.

---

### IRSA Deep Dive

```
IRSA MECHANISM — STEP BY STEP
═══════════════════════════════════════════════════════════════

SETUP:
  1. EKS cluster has an OIDC provider URL:
     https://oidc.eks.us-east-1.amazonaws.com/id/<cluster-id>
     This URL is the EKS cluster's built-in OIDC issuer.
     AWS IAM trusts tokens issued by this URL.

  2. Create IAM role with trust policy allowing this OIDC provider:
     {
       "Effect": "Allow",
       "Principal": {
         "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<id>"
       },
       "Action": "sts:AssumeRoleWithWebIdentity",
       "Condition": {
         "StringEquals": {
           "oidc.eks.us-east-1.amazonaws.com/id/<id>:sub":
             "system:serviceaccount:production:myapp-sa",
           "oidc.eks.us-east-1.amazonaws.com/id/<id>:aud":
             "sts.amazonaws.com"
         }
       }
     }
     Note: Condition restricts which SA can assume this role.

  3. Annotate K8s ServiceAccount with IAM role ARN:
     eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/MyAppRole

  4. Pod Admission Webhook (Amazon EKS Pod Identity Webhook):
     When pod with annotated SA is created:
       Mutating webhook injects environment variables:
         AWS_ROLE_ARN=arn:aws:iam::123456789:role/MyAppRole
         AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
       Injects projected volume with SA token:
         audience: sts.amazonaws.com
         expirationSeconds: 86400 (24h, auto-rotated)

RUNTIME:
  5. AWS SDK inside pod calls STS AssumeRoleWithWebIdentity:
     - Reads token from: AWS_WEB_IDENTITY_TOKEN_FILE
     - Sends to STS: AssumeRoleWithWebIdentity(token, roleARN)
  6. STS validates token against EKS OIDC endpoint (JWKS)
  7. STS verifies sub claim matches allowed SA in trust policy
  8. STS returns temporary credentials:
     AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN
     (valid for 1 hour, auto-renewed by SDK)
  9. Pod uses temporary credentials for AWS API calls
     No static keys anywhere.
```

---

### IRSA YAML Setup

```yaml
# Step 1: Create ServiceAccount with IRSA annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyAppIAMRole
    eks.amazonaws.com/token-expiration: "3600"    # optional: 1h token lifetime
    eks.amazonaws.com/sts-regional-endpoints: "true"  # use regional STS endpoint

---
# Step 2: Pod using the annotated ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa    # pod must use the annotated SA
  containers:
  - name: app
    image: myapp:v1
    # Webhook injects these automatically:
    # env:
    # - name: AWS_ROLE_ARN
    #   value: arn:aws:iam::123456789012:role/MyAppIAMRole
    # - name: AWS_WEB_IDENTITY_TOKEN_FILE
    #   value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

```bash
# Terraform: create IAM role with trust policy for IRSA
resource "aws_iam_role" "myapp" {
  name = "myapp-irsa-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${module.eks.oidc_provider}"
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${module.eks.oidc_provider}:sub" = "system:serviceaccount:production:myapp-sa"
          "${module.eks.oidc_provider}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

# eksctl shortcut
eksctl create iamserviceaccount \
  --name myapp-sa \
  --namespace production \
  --cluster my-cluster \
  --role-name "myapp-irsa-role" \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Verify IRSA working in pod
kubectl exec -it myapp -n production -- \
  aws sts get-caller-identity
# {
#   "UserId": "AROAI3EXAMPLE:botocore-session-...",
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/myapp-irsa-role/botocore-session-..."
# }

# EKS Pod Identity (newer, simpler alternative to IRSA):
# aws eks create-pod-identity-association \
#   --cluster-name my-cluster \
#   --namespace production \
#   --service-account myapp-sa \
#   --role-arn arn:aws:iam::123456789012:role/MyAppIAMRole
```

---

### 🎤 Short Crisp Interview Answer

> *"IRSA eliminates static AWS credentials from pods by using OIDC federation. The EKS cluster has a built-in OIDC issuer URL. An IAM role's trust policy specifies which Kubernetes ServiceAccount (by namespace and name) can assume it via AssumeRoleWithWebIdentity. The pod's ServiceAccount is annotated with the IAM role ARN. When the pod starts, a mutating admission webhook injects the role ARN and a projected SA token file path as environment variables. The AWS SDK reads the token, calls STS to exchange it for temporary credentials, and uses those for API calls. Tokens expire after 24 hours by default and are auto-rotated. No IAM access keys anywhere in the cluster."*

---

---

# 7.13 Image Signing & Verification (Cosign, Connaisseur)

## 🔴 Advanced

### What it is

**Image signing** cryptographically attests that a container image was built by a trusted pipeline and hasn't been tampered with. **Cosign** (Sigstore project) signs images and attestations. **Connaisseur** (or Kyverno/OPA policies) verifies signatures at admission time, blocking unsigned or incorrectly-signed images from running.

---

### Why Image Signing Matters

```
SUPPLY CHAIN ATTACK PREVENTION
═══════════════════════════════════════════════════════════════

THREAT: Malicious image injected into registry
  Attacker compromises:
    - Container registry (malicious layer injected)
    - CI/CD pipeline (malicious image built and pushed)
    - Image tag (re-tagged to point to malicious image)

  Without signing: cluster pulls and runs malicious image
  With signing:    cluster verifies signature → rejects unsigned images

COSIGN SIGNS:
  The image manifest digest (SHA256 hash of the image content)
  Signature stored in same registry as OCI artifact
  Cannot tamper with image without invalidating signature

ALSO SIGNS:
  SBOM (Software Bill of Materials) — what's in the image
  Vulnerability scan attestations — "this image was scanned, here are results"
  Build provenance — which CI pipeline built this, from which commit
```

---

### Cosign Signing and Verification

```bash
# Generate signing key pair
cosign generate-key-pair
# cosign.key   ← private key (store securely — Vault, AWS KMS)
# cosign.pub   ← public key (share with everyone, put in policy)

# Sign an image (after building and pushing)
cosign sign \
  --key cosign.key \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3
# Creates a signature OCI artifact in the registry:
# 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:sha256-abc123...sig

# Sign with keyless signing (Sigstore / Fulcio — uses OIDC identity)
cosign sign \
  --identity-token=$(gcloud auth print-identity-token) \
  myapp:v1.2.3
# Uses your OIDC identity (GitHub Actions, Google, etc.)
# Signature stored in Rekor transparency log
# No key management required

# Verify an image signature
cosign verify \
  --key cosign.pub \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3
# Verification for 123456789.../myapp:v1.2.3
# The following checks were performed:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key
# [{"critical":{"type":"cosign container image signature"}, ...}]

# Attach SBOM
cosign attach sbom \
  --sbom sbom.spdx.json \
  myapp:v1.2.3

# Attach vulnerability attestation
cosign attest \
  --predicate scan-results.json \
  --type vuln \
  --key cosign.key \
  myapp:v1.2.3

# Verify attestation
cosign verify-attestation \
  --type vuln \
  --key cosign.pub \
  myapp:v1.2.3
```

---

### Admission Enforcement with Kyverno

```yaml
# Kyverno policy: only allow signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-image-signature
    match:
      any:
      - resources:
          kinds: ["Pod"]
    verifyImages:
    - imageReferences:
      - "123456789.dkr.ecr.us-east-1.amazonaws.com/*"  # our registry
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
            signatureAlgorithm: sha256
      # Optional: also verify attestations (e.g., no critical CVEs)
      attestations:
      - predicateType: https://cosign.sigstore.dev/attestation/vuln/v1
        attestors:
        - entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                ...
        conditions:
        - all:
          - key: "{{ scanner.result.summary.CRITICAL }}"
            operator: Equals
            value: "0"             # reject if any CRITICAL CVEs
```

---

### 🎤 Short Crisp Interview Answer

> *"Image signing prevents supply chain attacks where malicious images are injected into registries or CI pipelines. Cosign (from the Sigstore project) signs the image manifest digest and stores the signature as an OCI artifact in the same registry. The signature can be key-based — using a generated key pair — or keyless using an OIDC identity from GitHub Actions or similar, recorded in the Rekor transparency log. Admission enforcement is done via Kyverno's verifyImages rule or OPA policy that calls cosign verify at admission time, rejecting pods with unsigned or incorrectly-signed images. In a full supply chain security setup, you also sign and verify SBOMs and vulnerability scan attestations to ensure only images that passed security scanning can run in production."*

---

---

# 7.14 Runtime Security (Falco)

## 🔴 Advanced

### What it is in simple terms

**Falco** is a runtime security tool that **detects anomalous behavior in running containers** by monitoring Linux kernel system calls in real time. While admission controllers prevent bad configurations from being deployed, Falco catches active attacks after deployment — like a process spawning a shell inside a container, reading /etc/passwd, or making unexpected network connections.

---

### How Falco Works

```
FALCO ARCHITECTURE
═══════════════════════════════════════════════════════════════

DATA SOURCE — kernel system calls:
  eBPF probe (recommended): loads eBPF program into kernel
    Captures: every syscall made by every process on the node
    Safe: eBPF is sandboxed, verified by kernel verifier
    Modern approach (no kernel module required)
  
  Kernel module (legacy): loadable kernel module (kmod)
    Captures same syscalls
    Less safe: kernel module crashes = node crashes
    Requires kernel headers

  All containers on the node are monitored.
  No agent inside containers needed.
  Minimal overhead (~2-3% CPU per node).

RULE ENGINE:
  Falco Rules: YAML files defining what "anomalous" means
  Rule evaluation: for each syscall event, evaluate all rules
  Alert: if rule matches, send to output channel(s)

OUTPUT CHANNELS:
  stdout (default)
  File
  Syslog
  HTTP webhook → Slack, PagerDuty, Splunk, DataDog
  gRPC API → custom consumers
  Falcosidekick (fan-out to multiple outputs)
```

---

### Falco Rules — Built-in and Custom

```yaml
# Falco rule syntax (rules.yaml)
- rule: Terminal shell in container
  desc: Detect interactive terminal shell opened in a container
  condition: >
    spawned_process and
    container and
    shell_procs and
    proc.tty != 0
    # shell_procs: matches bash, sh, zsh, etc.
    # proc.tty != 0: means an interactive terminal
  output: >
    Shell opened in container
    (user=%user.name user_loginuid=%user.loginuid
    container_id=%container.id
    image=%container.image.repository
    shell=%proc.name parent=%proc.pname
    cmdline=%proc.cmdline terminal=%proc.tty)
  priority: WARNING
  tags: [container, shell, mitre_execution]

---
# Rule: unexpected file read
- rule: Read sensitive file in container
  desc: Reading /etc/passwd, /etc/shadow in a container
  condition: >
    open_read and
    container and
    (fd.name startswith /etc/shadow or
     fd.name startswith /etc/passwd or
     fd.name startswith /etc/sudoers)
  output: >
    Sensitive file read
    (user=%user.name container_id=%container.id
    image=%container.image.repository
    file=%fd.name)
  priority: WARNING

---
# Rule: unexpected outbound connection
- rule: Unexpected outbound connection from prod container
  desc: Container making unexpected outbound connections
  condition: >
    outbound and
    container and
    not expected_outbound_destinations
    # Define macro: expected_outbound_destinations
    # List allowed destinations (internal services, AWS APIs)
  output: >
    Unexpected outbound connection
    (user=%user.name container=%container.name
    image=%container.image.repository
    connection=%fd.name)
  priority: WARNING

---
# Rule: privilege escalation
- rule: Privilege escalation in container
  desc: Process attempting privilege escalation
  condition: >
    evt.type in (setuid, setgid) and
    container and
    not user.uid=0 and
    evt.args contains "uid=0"
  output: >
    Privilege escalation attempt
    (user=%user.name container=%container.id
    image=%container.image.repository
    syscall=%evt.type)
  priority: CRITICAL

---
# Falco deployment as DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: falco
spec:
  selector:
    matchLabels:
      app: falco
  template:
    spec:
      tolerations:
      - operator: Exists           # run on all nodes including tainted ones
      hostPID: true                # required: see host processes
      hostNetwork: true            # required: see host network activity
      containers:
      - name: falco
        image: falcosecurity/falco-no-driver:0.36.0
        securityContext:
          privileged: true         # required: eBPF needs this
        env:
        - name: FALCO_BPF_PROBE
          value: ""                # use eBPF
        volumeMounts:
        - name: rules
          mountPath: /etc/falco/rules.d
        - name: falco-config
          mountPath: /etc/falco/falco.yaml
          subPath: falco.yaml
      volumes:
      - name: rules
        configMap:
          name: falco-rules
      - name: falco-config
        configMap:
          name: falco-config
```

---

### Falco Response Actions

```bash
# View Falco alerts in real-time
kubectl logs -f daemonset/falco -n falco | grep -v DEBUG

# Sample Falco alert:
# 10:30:15.123456789: Warning Shell opened in container
#   (user=root user_loginuid=1000
#    container_id=abc123def456
#    image=myapp:v1
#    shell=bash parent=runc
#    cmdline=bash terminal=34816)

# Falcosidekick — fan-out to multiple backends
# Send to Slack:
#   - alerts → Slack channel immediately
# Send to Prometheus:
#   - falco_events_total counter → alert on spike
# Send to Elasticsearch:
#   - full event data for SIEM correlation
# Send to AWS SNS:
#   - trigger Lambda for automated response

# Automated response example:
# Alert: shell opened in container →
# Lambda: kubectl delete pod <container_id's pod> immediately
```

---

### 🎤 Short Crisp Interview Answer

> *"Falco monitors Linux kernel system calls via an eBPF probe on every node, evaluating all container activity against rules in real time. Unlike admission controllers that prevent bad configurations, Falco detects active attacks: a shell spawned inside a container, sensitive file reads, unexpected outbound connections, or privilege escalation attempts. Rules are written in YAML with conditions using Falco's field syntax. Falco runs as a DaemonSet needing privileged access to load the eBPF probe. Alerts go to Slack, PagerDuty, Splunk, or custom webhooks via Falcosidekick. The typical incident response integration is: Falco alert → SNS → Lambda → automated pod termination."*

---

---

# 7.15 CIS Benchmark Hardening

## 🔴 Advanced

### What it is in simple terms

The **CIS (Center for Internet Security) Kubernetes Benchmark** is a prescriptive set of security configuration recommendations for Kubernetes clusters. Running automated tools like **kube-bench** against your cluster identifies which CIS controls are passing and failing, guiding hardening efforts.

---

### CIS Benchmark Categories

```
CIS KUBERNETES BENCHMARK SECTIONS
═══════════════════════════════════════════════════════════════

1. CONTROL PLANE COMPONENTS
   1.1 API Server flags:
       --anonymous-auth=false           (no unauthenticated access)
       --audit-log-path=<path>          (audit logging enabled)
       --audit-policy-file=<path>
       --encryption-provider-config=<> (secrets encrypted at rest)
       --tls-min-version=VersionTLS12  (no TLS 1.0/1.1)
       --disable-admission-plugins=AlwaysAdmit (no open door)
       --enable-admission-plugins includes NodeRestriction
       --profiling=false               (no profiling endpoint)
       --request-timeout=300s

   1.2 etcd:
       --client-cert-auth=true         (mTLS for etcd)
       --auto-tls=false                (no self-signed)
       --peer-client-cert-auth=true    (peer auth)

   1.3 Controller Manager:
       --use-service-account-credentials=true
       --service-account-private-key-file=<path>
       --root-ca-file=<path>
       --profiling=false

   1.4 Scheduler:
       --profiling=false
       --bind-address=127.0.0.1        (not 0.0.0.0)

2. ETCD
   2.1 etcd data dir permissions: 700 (only etcd user)
   2.2 etcd data dir owner: etcd:etcd

3. CONTROL PLANE CONFIGURATION
   3.1 Ensure authentication is enabled
   3.2 Logging and audit trail

4. WORKER NODE SECURITY
   4.1 kubelet flags:
       --anonymous-auth=false          (no unauthenticated kubelet API)
       --authorization-mode=Webhook    (RBAC for kubelet API)
       --client-ca-file=<path>
       --rotate-certificates=true      (auto-rotate kubelet certs)
       --protect-kernel-defaults=true  (prevent kernel param changes)
       --event-qps=0                   (no event rate limiting issues)
       --read-only-port=0              (disable read-only kubelet port)

   4.2 kubelet configuration file:
       Prefer config file over command-line flags
       File permissions: 644 or stricter
       Owner: root:root

5. KUBERNETES POLICIES
   5.1 RBAC:
       No wildcard permissions in production
       Minimize cluster-admin bindings
       ServiceAccounts: automountServiceAccountToken: false default
       Minimize secret access

   5.2 Pod Security:
       Pod Security Standards applied
       No privileged containers
       No hostNetwork/hostPID/hostIPC
       No hostPath volumes (use PVCs)
       capabilities.drop: ALL

   5.3 Network:
       NetworkPolicies applied (default-deny)
       No exposed services of type NodePort unless required

   5.4 Secrets:
       Encryption at rest enabled (EncryptionConfiguration)
       External secret manager used (Vault, AWS SM)
```

---

### kube-bench Automated Scanning

```bash
# Run kube-bench on a node (control plane)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml
kubectl logs -f job/kube-bench-master

# Output sample:
# [INFO] 1 Master Node Security Configuration
# [PASS] 1.1.1 Ensure that the API server pod specification file permissions
#         are set to 644 or more restrictive (Automated)
# [FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd
#         (Automated)
#         ** Result  : etcd data directory ownership is not etcd:etcd
# [WARN] 1.2.6 Ensure that the --kubelet-certificate-authority argument is set
#         as appropriate (Automated)

# Run on worker node
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-node.yaml
kubectl logs -f job/kube-bench-node

# For EKS (managed control plane — subset of checks applicable)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml
# Note: EKS control plane is managed by AWS — some CIS controls
# are handled by AWS, some require your configuration

# Run kube-bench directly on a node (for detailed output)
# On the node:
kube-bench run --targets master \
  --config-dir /etc/kube-bench/cfg \
  --version 1.27

# Summary output:
# 58 checks PASS
# 12 checks FAIL
# 8 checks WARN
# 0 checks INFO
```

---

### EKS-Specific Hardening

```bash
# EKS CIS hardening (controls you own, not AWS-managed):

# 1. Enable audit logging
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["audit","api","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# 2. Enable secrets encryption (at cluster creation)
aws eks create-cluster \
  --name my-cluster \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:..."}}]'

# 3. Restrict cluster endpoint (no public access)
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config \
    endpointPublicAccess=false,endpointPrivateAccess=true,publicAccessCidrs=[]

# 4. Enable node security groups tightly
# Worker node SG: allow only from control plane SG, no 0.0.0.0/0

# 5. Use Bottlerocket AMI (immutable, security-hardened)
# eksctl nodegroup with --node-ami-family=Bottlerocket

# 6. Use node groups with least-privilege IAM roles
# Worker node IAM: only EC2ContainerRegistryReadOnly + CNI permissions

# 7. Enable EKS add-on for VPC CNI with security groups for pods
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy":"true"}'

# Trivy for vulnerability and misconfiguration scanning:
trivy k8s --report summary cluster
# Checks: misconfigurations, exposed secrets, vulnerable images
```

---

### 🎤 Short Crisp Interview Answer

> *"The CIS Kubernetes Benchmark is a prescriptive checklist of security configurations across the API Server, etcd, controller manager, kubelet, and cluster policies. kube-bench automates evaluation against the benchmark and reports PASS/FAIL/WARN per control. Key controls that are commonly missed: disabling anonymous authentication on the kubelet API (--anonymous-auth=false), enabling audit logging with a retention policy, encrypting secrets at rest, applying Pod Security Standards, setting default-deny NetworkPolicies, and minimizing RBAC permissions (no wildcard roles). On EKS, AWS handles control plane hardening but you own worker node configuration, RBAC, NetworkPolicy, and secrets encryption. I run kube-bench in CI and as a periodic CronJob to catch drift."*

---

---

# 🏁 Category 7 — Complete Security Map

```
KUBERNETES SECURITY LAYERS
═══════════════════════════════════════════════════════════════════

IDENTITY & ACCESS
  Who are you?        Authentication (OIDC, certificates)     → 7.7
  What can you do?    RBAC (Roles, Bindings)                   → 7.1
  Pod identity:       ServiceAccounts + bound tokens           → 7.2
  AWS identity:       IRSA (SA token → IAM role)               → 7.12

ADMISSION (what can be deployed)
  Policy types?       Admission Controllers                    → 7.4
  Custom policy?      Mutating/Validating Webhooks             → 7.5
  Pod security?       PSS/PSA (Privileged/Baseline/Restricted) → 7.3
  Org policies?       OPA/Gatekeeper (Rego)                    → 7.10
  K8s-native policy?  Kyverno (YAML)                           → 7.11

RUNTIME SECURITY (what runs and how it behaves)
  Network isolation?  NetworkPolicy (zero-trust)               → 7.6
  Syscall filtering?  Seccomp, AppArmor, SELinux               → 7.9
  Anomaly detection?  Falco (eBPF runtime monitoring)          → 7.14
  Image integrity?    Cosign signing + Kyverno verification    → 7.13

AUDIT & COMPLIANCE
  Who did what?       Audit logging (API Server events)        → 7.8
  Benchmark?          CIS Benchmark + kube-bench               → 7.15
  Secret exposure?    EncryptionConfiguration (etcd)           → Cat 5 (5.3)

DEFENSE-IN-DEPTH ORDER:
  1. Build: sign images (Cosign)
  2. Deploy: verify signatures (Kyverno/Gatekeeper)
  3. Schedule: check PSS, OPA/Kyverno policies
  4. Runtime: isolate with NetworkPolicy, drop capabilities
  5. Monitor: Falco for anomalies, audit logs for audit trail
  6. Audit: kube-bench for CIS compliance, regular review
```

---

# Quick Reference — Category 7 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **RBAC** | Role (ns), ClusterRole (cluster). RoleBinding (ns). ClusterRoleBinding (cluster). Additive only — no deny |
| **ClusterRole + RoleBinding** | = namespace-scoped access only, NOT cluster-wide |
| **list secrets** | More dangerous than get — returns ALL secret values at once |
| **Service Accounts** | Bound tokens since 1.24+. 1hr expiry. Pod-bound. automountServiceAccountToken: false if not needed |
| **PSS Profiles** | Privileged (unrestricted) → Baseline (no privilege escalation) → Restricted (runAsNonRoot, seccomp, drop ALL) |
| **PSA modes** | enforce (reject), warn (client warning), audit (log only) |
| **Admission flow** | Authn → Authz (RBAC) → Mutating → Schema Validate → Validating → etcd |
| **Webhook failurePolicy** | Fail = webhook outage = request blocked. Ignore = webhook outage = request allowed |
| **NetworkPolicy default** | All allowed. Add default-deny first, then allow DNS, then specific rules |
| **AND vs OR** | Same item = AND. Separate items = OR. Most common NetworkPolicy mistake |
| **OIDC** | One provider per API Server. Identity is just a string. kubelogin handles token refresh |
| **Audit levels** | None → Metadata → Request → RequestResponse. First match wins |
| **Seccomp** | RuntimeDefault blocks ~30-40 dangerous syscalls. Required by Restricted PSS |
| **AppArmor** | Ubuntu/Debian nodes. MAC for files/network. Not available on Amazon Linux |
| **SELinux** | RHEL/Amazon Linux. MCS categories per container. Container isolation |
| **OPA/Gatekeeper** | ConstraintTemplate (Rego logic) + Constraint (config). Audit scans existing objects |
| **Kyverno** | YAML policies. validate/mutate/generate/cleanup. No Rego needed |
| **IRSA** | SA annotation → IAM trust policy → STS exchange → temp creds. No static keys |
| **Cosign** | Signs image digest. Signature stored as OCI artifact in same registry |
| **Falco** | eBPF syscall monitoring. DaemonSet. Detects runtime anomalies post-deployment |
| **kube-bench** | Automates CIS Benchmark evaluation. Run as Job on each node |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| Bound SA token default expiry | 3607 seconds (~1 hour) |
| SA token auto-rotation trigger | 80% of expiry |
| EKS IRSA token expiry (default) | 86400 seconds (24 hours) |
| Max webhook timeout | 30 seconds |
| PSS profile count | 3 (Privileged, Baseline, Restricted) |
| PSA enforcement modes | 3 (enforce, warn, audit) |
| CIS Benchmark kube-bench typical PASS rate | 80-90% out of box, target 100% |
| Falco overhead per node | ~2-3% CPU |
| Audit log levels | 4 (None, Metadata, Request, RequestResponse) |
| RBAC subjects kinds | 3 (User, Group, ServiceAccount) |

# Kubernetes Interview Mastery
# CATEGORY 14: TROUBLESHOOTING

---

> **How to use this document:**
> Every topic follows: Mental Model → Symptoms → Decision Tree → Commands → Root Causes → Fix → Gotchas.
> This is a scenario-first category — start from the symptom, work to the cause.
> ⚠️ = High priority, frequently asked in interviews and production incidents.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 14.1 | Systematic debugging methodology | 🟢 Beginner |
| 14.2 | Pod not starting — Pending, ImagePullBackOff, CrashLoopBackOff ⚠️ | 🟢 Beginner |
| 14.3 | Reading kubectl describe output | 🟢 Beginner |
| 14.4 | Networking failures — Service unreachable, DNS not resolving ⚠️ | 🟡 Intermediate |
| 14.5 | NetworkPolicy lockout | 🟡 Intermediate |
| 14.6 | Storage failures — PVC stuck Pending, volume mount errors ⚠️ | 🟡 Intermediate |
| 14.7 | OOMKilled & CPU throttling ⚠️ | 🟡 Intermediate |
| 14.8 | Node issues — NotReady, pressure conditions | 🟡 Intermediate |
| 14.9 | Deployment stuck — rollout not progressing | 🟡 Intermediate |
| 14.10 | RBAC failures — Forbidden errors ⚠️ | 🟡 Intermediate |
| 14.11 | crictl — debugging below the kubelet | 🟡 Intermediate |
| 14.12 | Control plane debugging | 🔴 Advanced |
| 14.13 | Webhook outages ⚠️ | 🔴 Advanced |
| 14.14 | Cluster Autoscaler not scaling | 🔴 Advanced |
| 14.15 | Network deep-dive tools | 🔴 Advanced |
| 14.16 | Performance profiling | 🔴 Advanced |
| 14.17 | Incident runbooks — correlated multi-signal diagnosis | 🔴 Advanced |

---

# 14.1 Systematic Debugging Methodology

## 🟢 Beginner

### The Mental Model

Every Kubernetes problem follows a **layered investigation order**. Going out of order wastes time. The sequence is deterministic: broad → narrow, cluster-level → pod-level → process-level.

---

### The Five-Layer Debugging Stack

```
KUBERNETES DEBUGGING — UNIVERSAL SEQUENCE
═══════════════════════════════════════════════════════════════

LAYER 1: WHAT IS THE SCOPE?
  One pod broken?         → workload problem
  All pods in namespace?  → namespace/quota/NetworkPolicy problem
  All pods on one node?   → node problem
  All pods everywhere?    → control plane / cluster-wide problem
  Service unreachable?    → networking problem
  Data loss / mount fail? → storage problem

LAYER 2: CLUSTER-LEVEL EVENTS (fastest first signal)
  kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -20
  kubectl get events -A --field-selector type=Warning | tail -30
  → Events expire in ~1 hour. Check them FIRST before they're gone.

LAYER 3: OBJECT STATUS
  kubectl get pod <name> -n <ns>            → STATUS column
  kubectl get pod <name> -n <ns> -o wide    → also NODE, IP
  kubectl describe pod <name> -n <ns>       → Conditions + Events

LAYER 4: LOGS
  kubectl logs <pod> -n <ns>                → current container stdout
  kubectl logs <pod> -n <ns> --previous     → last crashed container
  kubectl logs <pod> -n <ns> -c <container> → specific container
  kubectl logs <pod> -n <ns> --timestamps   → with timestamps

LAYER 5: INTERACTIVE INVESTIGATION
  kubectl exec -it <pod> -n <ns> -- /bin/sh → shell inside container
  kubectl debug -it <pod> --image=nicolaka/netshoot → debug sidecar
  crictl logs <container-id>                → below kubelet level

THE GOLDEN RULE:
  Events   → tells you WHAT happened (OOMKill, FailedScheduling, etc.)
  Logs     → tells you WHY it happened (app error, config missing)
  exec     → lets you reproduce and confirm (test connectivity, read files)
  Never skip layers. Never start with exec if events answer the question.
```

---

### The Debugging Command Sequence

```bash
# ─────────────────────────────────────────────────────
# STEP 1: Orient — what's broken?
# ─────────────────────────────────────────────────────
kubectl get pods -n production
# NAME                     READY   STATUS             RESTARTS   AGE
# api-7d9f8c-abc           0/1     CrashLoopBackOff   5          10m  ← broken
# api-7d9f8c-def           1/1     Running            0          10m
# worker-6b5c9d-xyz        0/1     Pending            0          5m   ← broken

# ─────────────────────────────────────────────────────
# STEP 2: Events first (fastest, most informative)
# ─────────────────────────────────────────────────────
kubectl get events -n production \
  --sort-by='.lastTimestamp' | tail -20

# ─────────────────────────────────────────────────────
# STEP 3: Describe the broken pod
# ─────────────────────────────────────────────────────
kubectl describe pod api-7d9f8c-abc -n production
# → Read: Conditions, Events sections

# ─────────────────────────────────────────────────────
# STEP 4: Logs (including previous container)
# ─────────────────────────────────────────────────────
kubectl logs api-7d9f8c-abc -n production
kubectl logs api-7d9f8c-abc -n production --previous

# ─────────────────────────────────────────────────────
# STEP 5: Exec for interactive investigation
# ─────────────────────────────────────────────────────
kubectl exec -it api-7d9f8c-def -n production -- /bin/sh
# (use a WORKING pod to investigate shared resources)

# ─────────────────────────────────────────────────────
# QUICK DIAGNOSTIC COMMANDS (run in parallel)
# ─────────────────────────────────────────────────────
# Pods not Running in ALL namespaces
kubectl get pods -A | grep -v -E "Running|Completed|Succeeded"

# Recent Warning events across entire cluster
kubectl get events -A --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -30

# Node health
kubectl get nodes
kubectl describe nodes | grep -A 5 "Conditions:"

# Resource usage
kubectl top pods -n production
kubectl top nodes

# HPA status
kubectl get hpa -n production

# PVC status
kubectl get pvc -n production
```

---

### Debugging Tools Reference

```bash
# QUICK DEBUGGING POD (netshoot — has curl, dig, nc, tcpdump, etc.)
kubectl run debug-pod \
  --image=nicolaka/netshoot \
  --rm -it --restart=Never \
  -n production \
  -- /bin/bash

# EPHEMERAL CONTAINER (attach to running pod — K8s 1.23+)
kubectl debug -it my-pod \
  --image=busybox \
  --target=app \              # share PID namespace with 'app' container
  -n production

# COPY POD FOR DEBUGGING (override entrypoint to prevent crash)
kubectl debug my-crashing-pod \
  --copy-to=debug-copy \
  --image=my-api:v1 \
  -- sleep infinity
kubectl exec -it debug-copy -n production -- /bin/sh

# NODE DEBUGGING (SSH-like access to node)
kubectl debug node/worker-2 \
  -it \
  --image=ubuntu
# Mounts host filesystem at /host inside debug container

# PORT-FORWARD for local testing
kubectl port-forward svc/my-api 8080:80 -n production &
curl http://localhost:8080/health

# CHECK API SERVER RESPONSIVENESS
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /livez
```

---

### 🎤 Short Crisp Interview Answer

> *"My debugging sequence is always: scope first (is it one pod, one node, or cluster-wide?), then events (kubectl get events -- they expire in an hour and often contain the answer immediately), then object status with kubectl describe, then logs including --previous for crashed containers, then exec for interactive investigation. I never start with exec if events already explain the problem. The most common mistake I see is jumping straight to logs or exec without reading events, which wastes time when the scheduler already left a clear 'Insufficient CPU' message."*

---

---

# ⚠️ 14.2 Pod Not Starting — Pending, ImagePullBackOff, CrashLoopBackOff

## 🟢 Beginner — HIGH PRIORITY

### Complete Decision Tree

```
POD STATUS → ROOT CAUSE DECISION TREE
═══════════════════════════════════════════════════════════════

STATUS: Pending
  └── kubectl describe pod → Events:
      ├── "FailedScheduling: 0/N nodes available: N Insufficient cpu"
      │     → Nodes don't have enough CPU to satisfy requests
      │     → Fix: add nodes, reduce requests, or check ResourceQuota
      │
      ├── "FailedScheduling: 0/N nodes available: N Insufficient memory"
      │     → Same as above for memory
      │
      ├── "FailedScheduling: 0/N nodes: N didn't match node affinity"
      │     → nodeAffinity or nodeSelector has no matching nodes
      │     → kubectl get nodes --show-labels | grep <required-label>
      │
      ├── "FailedScheduling: 0/N nodes: N node(s) had taint X, not tolerated"
      │     → Node taint not tolerated by pod
      │     → kubectl describe node | grep Taints
      │
      ├── "FailedScheduling: 0/N nodes: N didn't match topology spread"
      │     → TopologySpreadConstraints too strict
      │     → Check maxSkew and available zones
      │
      ├── "FailedScheduling: 0/N nodes: N had volume node affinity conflict"
      │     → PVC is bound to a volume in AZ-A but pod would go to AZ-B
      │     → EBS volumes are AZ-locked — pod must go to same AZ
      │
      ├── No events at all
      │     → Check ResourceQuota: kubectl get resourcequota -n <ns>
      │     → Check if scheduler is running: kubectl get pods -n kube-system
      │
      └── "unbound immediate PersistentVolumeClaims"
            → PVC not bound → pod stuck waiting for storage
            → kubectl get pvc -n <ns>

STATUS: ImagePullBackOff / ErrImagePull
  └── kubectl describe pod → Events:
      ├── "Failed to pull image: not found"
      │     → Image tag doesn't exist in registry
      │     → Typo in image name or tag
      │     → Fix: verify exact image:tag in registry
      │
      ├── "Failed to pull image: unauthorized"
      │     → Registry auth failure
      │     → Private registry without imagePullSecret
      │     → Fix: create imagePullSecret, attach to ServiceAccount
      │
      ├── "Failed to pull image: network timeout"
      │     → Node can't reach registry (firewall, VPC config)
      │     → NAT gateway issue, security group blocking 443
      │
      └── "Back-off pulling image"
            → Docker Hub rate limit hit (100/6h for unauthenticated)
            → Fix: authenticate to Docker Hub or use ECR mirror

STATUS: CrashLoopBackOff
  └── Container starts, crashes, K8s backs off (10s → 20s → 40s → 5m)
      Step 1: kubectl logs <pod> --previous     ← MOST IMPORTANT
      Step 2: Check exit code in describe output
              Exit 1  : application error (read logs)
              Exit 137: OOMKilled (increase memory limit)
              Exit 139: Segfault in application
              Exit 143: SIGTERM (graceful stop — why is it stopping?)
              Exit 255: container runtime error
      Step 3: Common causes:
      ├── App can't connect to dependency (DB, Redis)
      │     → Readiness probe as liveness = cascade restart
      │     → Fix: make liveness probe check only process health
      │
      ├── Missing environment variable / config
      │     → App exits 1 on startup with "required env var missing"
      │     → kubectl exec (working pod) -- env | grep <var>
      │
      ├── Missing secret / configmap
      │     → Events: "unable to mount secret: not found"
      │     → kubectl get secret/configmap <name> -n <ns>
      │
      ├── Liveness probe too aggressive during startup
      │     → No startupProbe + slow app = liveness kills it during boot
      │     → Fix: add startupProbe with generous failureThreshold
      │
      └── Wrong CMD / ENTRYPOINT in image
            → Container immediately exits
            → kubectl debug --copy-to=debug -- sleep infinity → inspect

STATUS: OOMKilled (shows in RESTARTS column as increasing)
  └── Container hit memory limit → kernel kills it (exit 137)
      kubectl describe pod → "Last State: OOMKilled"
      kubectl top pod <name>  ← see current memory usage
      Fix: increase memory limit OR fix memory leak

STATUS: Init:Error / Init:CrashLoopBackOff
  └── Init container failing
      kubectl logs <pod> -c <init-container-name>
      kubectl describe pod → Init Containers section

STATUS: Terminating (stuck)
  └── Pod not terminating after grace period
      kubectl describe pod → look for finalizers
      kubectl get pod <name> -o jsonpath='{.metadata.finalizers}'
      Common causes:
        - Finalizer not being removed (controller crashed)
        - PVC in use protection
        - Custom finalizer stuck
      Fix (last resort):
      kubectl patch pod <name> \
        -p '{"metadata":{"finalizers":[]}}' --type=merge
```

---

### Commands for Each Status

```bash
# ── PENDING ──────────────────────────────────────────
# Why is it pending?
kubectl describe pod <name> -n <ns> | grep -A 20 "Events:"
# Check scheduler is running
kubectl get pods -n kube-system | grep scheduler
# Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes
# Check ResourceQuota
kubectl get resourcequota -n <ns>
kubectl describe resourcequota -n <ns>

# ── IMAGEPULLBACKOFF ─────────────────────────────────
# What exact error?
kubectl describe pod <name> | grep -A 5 "Failed to pull"
# Check imagePullSecret exists
kubectl get secret -n <ns> | grep regcred
# Test pull manually on node
docker pull <image>   # or: crictl pull <image>
# Check ECR login
aws ecr get-login-password | docker login --username AWS \
  --password-stdin <ecr-endpoint>

# ── CRASHLOOPBACKOFF ─────────────────────────────────
# Get last crash output (MOST USEFUL)
kubectl logs <pod> -n <ns> --previous
kubectl logs <pod> -n <ns> --previous --timestamps
# Check exit code
kubectl describe pod <pod> -n <ns> | grep "Exit Code"
# If crash before any logs → override entrypoint
kubectl debug <pod> --copy-to=debug-pod -- sleep 3600 -n <ns>
kubectl exec -it debug-pod -n <ns> -- /bin/sh
# then manually run the entrypoint to see the error

# ── STUCK TERMINATING ────────────────────────────────
kubectl get pod <name> -n <ns> \
  -o jsonpath='{.metadata.finalizers}'
# Force delete (use only as last resort)
kubectl delete pod <name> -n <ns> --grace-period=0 --force
```

---

### 🎤 Short Crisp Interview Answer

> *"For pod startup failures I work through the status. Pending means the scheduler can't place it — events will say exactly why: insufficient CPU/memory, no nodes matching affinity/taint, or PVC in wrong AZ. ImagePullBackOff means the image can't be fetched — check image name/tag, check imagePullSecret exists and is attached to the ServiceAccount, check registry reachability. CrashLoopBackOff means the container starts but immediately dies — kubectl logs --previous is the first command, which shows what the container printed before crashing. The exit code in kubectl describe tells me whether it's an OOMKill (137), SIGTERM (143), or app error (1). The most dangerous CrashLoopBackOff cause in production is a liveness probe checking a database dependency — if the DB goes down, every pod restarts in a loop simultaneously."*

---

---

# 14.3 Reading kubectl describe Output

## 🟢 Beginner

### Full Annotated Output

```bash
kubectl describe pod api-7d9f8c-abc -n production
```

```
Name:             api-7d9f8c-abc
Namespace:        production
Priority:         0                         ← PriorityClass value
Node:             worker-2/10.0.1.15        ← which node + node IP
                                              "none" = not yet scheduled
Start Time:       Mon, 15 Jan 2024 10:00:00 +0000
Labels:           app=api
                  pod-template-hash=7d9f8c  ← auto-added by ReplicaSet
Annotations:      ...

Status:           Running                   ← pod phase (not container)
IP:               10.244.2.5               ← pod IP in cluster network
IPs:
  IP:  10.244.2.5

Init Containers:                           ← check here if Init:x/N status
  db-migrate:
    State:       Terminated
    Reason:      Completed                 ← init container succeeded
    Exit Code:   0
    Started:     Mon, 15 Jan 2024 10:00:05
    Finished:    Mon, 15 Jan 2024 10:00:25

Containers:
  api:
    Container ID: containerd://abc123...   ← runtime container ID
    Image:        myapp:v1.2.3
    Image ID:     docker.io/myapp@sha256:def456...  ← actual digest pulled
    Port:         8080/TCP
    Host Port:    0/TCP                    ← 0 = not using host port
    State:        Running                  ← container state
      Started:    Mon, 15 Jan 2024 10:00:30
    Ready:        False                    ← ⚠️ Running but NOT Ready
    Restart Count: 3                       ← ⚠️ 3 restarts = probe issue or crash
    Limits:
      cpu:        1
      memory:     1Gi
    Requests:
      cpu:        250m
      memory:     256Mi
    Liveness:     http-get http://:8080/health/live
                  delay=0s timeout=5s period=15s #success=1 #failure=3
    Readiness:    http-get http://:8080/health/ready
                  delay=0s timeout=3s period=5s  #success=1 #failure=3
    Environment:
      DB_HOST:    postgres.production.svc.cluster.local
      LOG_LEVEL:  INFO
      DB_PASS:    <set to the key 'password' in secret 'db-credentials'>
    Mounts:
      /etc/config from app-config (rw)    ← configmap mount
      /etc/secrets from db-creds (ro)     ← secret mount
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access (ro)

Conditions:                               ← ⚠️ READ THESE CAREFULLY
  Type              Status  Reason
  Initialized       True               ← init containers done
  Ready             False  ContainersNotReady   ← pod NOT in LB endpoints
  ContainersReady   False  ContainersNotReady   ← readiness probe failing
  PodScheduled      True               ← assigned to node

                  ↑
  If PodScheduled = False: scheduling failure (check events)
  If Initialized  = False: init container running or failed
  If ContainersReady = False: probe failing or container crashed
  If Ready        = False: pod is Running but not serving traffic

Volumes:
  app-config:
    Type:       ConfigMap
    Name:       api-config              ← configmap must exist in namespace
  db-creds:
    Type:       Secret
    SecretName: db-credentials          ← secret must exist in namespace
  kube-api-access:
    Type:       Projected               ← auto-injected SA token

QoS Class:      Burstable               ← requests < limits = Burstable
Node-Selectors: <none>
Tolerations:    node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s

Events:                                 ← ⚠️ MOST DIAGNOSTIC SECTION
  Type    Reason     Age    From       Message
  Normal  Scheduled  10m    scheduler  Assigned to worker-2
  Normal  Pulled     10m    kubelet    Image pulled in 5s
  Normal  Started    10m    kubelet    Container started
  Warning Unhealthy  8m     kubelet    Readiness probe failed:
                                       HTTP probe failed: 503
  Warning Unhealthy  5m     kubelet    Readiness probe failed:
                                       HTTP probe failed: 503
  Warning BackOff    2m     kubelet    Back-off restarting failed container
```

---

### What Each Section Tells You

```
SECTION             WHAT TO LOOK FOR
═══════════════════════════════════════════════════════════════

Node:               "none" = pod never scheduled (Pending)
                    a node name = scheduler placed it

State + Ready:      State=Running, Ready=False = readiness failing
                    State=Waiting, Reason=CrashLoopBackOff = crash
                    State=Terminated, Exit Code=137 = OOMKill

Restart Count:      0 = healthy
                    > 0 = container has crashed at least N times
                    High = CrashLoopBackOff in progress

Limits/Requests:    No requests = BestEffort QoS, evicted first
                    Limits = Requests = Guaranteed QoS

Liveness/Readiness: Shows probe config — check timeout values
                    "delay=0s" with slow app = restart loop

Environment:        "<set to the key X in secret Y>" = secret injection
                    If secret Y doesn't exist → pod won't start

Mounts:             If referenced ConfigMap/Secret doesn't exist
                    → pod stuck in ContainerCreating (FailedMount event)

CONDITIONS:         The ordered lifecycle checklist
                    First False condition = where pod is stuck

Events:             The STORY of what happened to this pod
                    Unhealthy = probe failing
                    BackOff = CrashLoopBackOff
                    FailedMount = missing ConfigMap/Secret
                    FailedScheduling = scheduler can't place pod
                    OOMKilling = exceeded memory limit
```

---

---

# ⚠️ 14.4 Networking Failures — Service Unreachable, DNS Not Resolving

## 🟡 Intermediate — HIGH PRIORITY

### Mental Model for K8s Networking

```
NETWORK PATH FOR A REQUEST
═══════════════════════════════════════════════════════════════

Pod A wants to reach Service "api" in namespace "production":

  1. DNS resolution:
     curl http://api.production.svc.cluster.local
     → Pod A sends DNS query to 10.96.0.10 (CoreDNS ClusterIP)
     → CoreDNS resolves "api.production.svc.cluster.local" to Service ClusterIP
     → Returns: 10.100.50.25 (the Service ClusterIP)

  2. Packet hits Service ClusterIP (virtual IP):
     → kube-proxy iptables/IPVS rule intercepts
     → DNAT: 10.100.50.25:80 → 10.244.1.5:8080 (a Pod IP)

  3. Packet routed to Pod:
     → CNI routes packet to correct node
     → Delivered to target pod

FAILURE POINTS:
  DNS fails         → step 1 breaks (CoreDNS issue, ndots misconfiguration)
  No endpoints      → step 2 breaks (no healthy pods, selector mismatch)
  NetworkPolicy     → step 3 breaks (firewall rule drops packet)
  Port mismatch     → all steps pass but connection refused
  Wrong namespace   → DNS resolves to wrong or nothing

DEBUGGING ORDER:
  1. Can I reach the Service ClusterIP directly?     → tests steps 2+3
  2. Does DNS resolve the service name?              → tests step 1
  3. Are there endpoints behind the Service?         → tests step 2
  4. Is there a NetworkPolicy blocking?              → tests step 3
```

---

### Service Unreachable — Full Debug Sequence

```bash
# SCENARIO: curl http://api-svc:8080 times out from another pod

# ─── STEP 1: Verify the Service exists ───────────────────────────
kubectl get svc api-svc -n production
# NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# api-svc   ClusterIP   10.100.50.25   <none>        80/TCP    5d
# If not found: wrong namespace, wrong name, service not deployed

# ─── STEP 2: Check endpoints (are pods behind the service?) ──────
kubectl get endpoints api-svc -n production
# NAME      ENDPOINTS                         AGE
# api-svc   10.244.1.5:8080,10.244.2.3:8080   5d  ← healthy, has endpoints
# OR:
# api-svc   <none>                             5d  ← NO ENDPOINTS = problem!

# If no endpoints → selector mismatch or all pods NotReady
kubectl describe svc api-svc -n production | grep Selector
# Selector: app=my-api
kubectl get pods -n production -l app=my-api   # must match!
kubectl get pods -n production -l app=my-api \
  -o wide --show-labels                        # confirm pod labels

# Common selector mismatch:
# Service selector:  app=my-api
# Pod label:         app=myapi   ← missing hyphen → no match!

# ─── STEP 3: Check if pods are Ready ─────────────────────────────
kubectl get pods -n production -l app=my-api
# If READY = 0/1: readiness probe failing → pod excluded from endpoints
kubectl describe pod <pod-name> -n production | grep -A 5 "Readiness"

# ─── STEP 4: Test from inside cluster (debug pod) ────────────────
kubectl run debug --image=nicolaka/netshoot \
  --rm -it --restart=Never -n production \
  -- /bin/bash

# Inside debug pod:
# Test direct pod IP (bypass Service)
curl http://10.244.1.5:8080/health
# If this works: Service routing problem
# If this fails: pod itself not responding

# Test via ClusterIP
curl http://10.100.50.25:80/health
# If this works: DNS problem
# If this fails: kube-proxy / iptables problem

# Test via service name (DNS)
curl http://api-svc.production.svc.cluster.local/health
# If this works: short name DNS problem (ndots)
# If this fails: CoreDNS problem

# ─── STEP 5: Check port mapping ──────────────────────────────────
kubectl get svc api-svc -n production -o yaml | grep -A 5 "ports:"
# ports:
# - port: 80           ← Service port (what clients use)
#   targetPort: 8080   ← Container port (what pod listens on)
#   protocol: TCP
# If targetPort doesn't match container's actual port → connection refused

# ─── STEP 6: Check NetworkPolicy ─────────────────────────────────
kubectl get networkpolicy -n production
kubectl describe networkpolicy -n production
# Look for policies that might block ingress/egress on relevant ports
```

---

### DNS Not Resolving — Full Debug Sequence

```bash
# SCENARIO: nslookup api-svc.production fails from a pod

# ─── STEP 1: CoreDNS is healthy? ─────────────────────────────────
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                READY   STATUS    RESTARTS   AGE
# coredns-abc123      1/1     Running   0          30d
# coredns-def456      1/1     Running   0          30d
# If not Running: CoreDNS crash → all DNS fails

kubectl logs -n kube-system deployment/coredns
# Look for: SERVFAIL, loop detected, forward errors

# ─── STEP 2: Can pods reach CoreDNS? ─────────────────────────────
kubectl run dns-debug --image=nicolaka/netshoot \
  --rm -it --restart=Never -n production -- /bin/bash

# Inside debug pod:
cat /etc/resolv.conf
# nameserver 10.96.0.10        ← CoreDNS ClusterIP
# search production.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# Test DNS resolution step by step:
nslookup kubernetes.default.svc.cluster.local
# If this fails: CoreDNS itself broken, not a search domain issue

nslookup api-svc.production.svc.cluster.local
# Full FQDN test - should always work if CoreDNS is up

nslookup api-svc.production
# Short form test

nslookup api-svc
# Shortest form - relies on search domains

# ─── STEP 3: The ndots:5 gotcha ──────────────────────────────────
# ndots:5 means: if name has fewer than 5 dots, try search domains first
# "api-svc" has 0 dots → tries:
#   api-svc.production.svc.cluster.local  (fails)
#   api-svc.svc.cluster.local             (fails)
#   api-svc.cluster.local                 (fails)
#   api-svc.                              (fails)
# = 4 failed lookups before resolving
# "external.domain.com" has 2 dots → tries all search domains first!
# = 3 extra failed lookups before resolving external name

# Fix for external names: use trailing dot or lower ndots
curl http://external.domain.com./api     # trailing dot = always FQDN
# OR in pod spec:
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"    # Only 2-dot names get search domain treatment

# ─── STEP 4: CoreDNS ConfigMap ───────────────────────────────────
kubectl get configmap coredns -n kube-system -o yaml
# Check: forward . /etc/resolv.conf   ← upstream DNS for external names
# Check: loop detection plugin enabled (prevents CoreDNS forwarding to itself)
# Check: health, ready endpoints enabled

# Restart CoreDNS if needed
kubectl rollout restart deployment/coredns -n kube-system
```

---

### 🎤 Short Crisp Interview Answer

> *"Kubernetes networking failures split into three categories. No endpoints means the Service selector doesn't match pod labels, or all pods are NotReady — kubectl get endpoints shows '<none>' immediately. DNS failures are either CoreDNS being down (check kube-system pods) or the ndots:5 gotcha where external hostnames get search domain prefixes appended causing 3-4 extra failed lookups before resolving. The third category is NetworkPolicy blocks — a packet reaches the right pod IP but gets silently dropped by a CNI policy. My debugging sequence: check endpoints exist, test direct pod IP to bypass Service, test ClusterIP to bypass DNS, test FQDN to bypass search domain resolution — each step isolates a different layer."*

---

---

# 14.5 NetworkPolicy Lockout

## 🟡 Intermediate

### What happens and why

```
NETWORKPOLICY LOCKOUT SCENARIO
═══════════════════════════════════════════════════════════════

BEFORE policy:   all traffic allowed (K8s default)
APPLY default-deny: all traffic BLOCKED
RESULT: if you forget to add back DNS egress → everything breaks

CLASSIC MISTAKE SEQUENCE:
  1. Apply default-deny NetworkPolicy to namespace
  2. Forget to allow UDP/TCP 53 egress to CoreDNS
  3. All pods immediately lose DNS resolution
  4. All pods can't connect to anything (service names don't resolve)
  5. Panic

SECOND CLASSIC MISTAKE:
  Apply NetworkPolicy that allows ingress from pods with label X
  But the source pods DON'T have label X
  → Traffic silently blocked, no error message
  → curl times out, looks like a network problem
```

---

```bash
# ─── DIAGNOSE NetworkPolicy lockout ──────────────────────────────
# Which NetworkPolicies exist?
kubectl get networkpolicy -n production
kubectl describe networkpolicy -n production

# Does the default-deny policy exist?
kubectl get networkpolicy default-deny-all -n production
# If yes: all traffic blocked unless explicitly allowed

# What does the blocking policy look like?
kubectl get networkpolicy -n production -o yaml

# ─── TEST from inside a pod ───────────────────────────────────────
kubectl exec -it my-pod -n production -- /bin/sh

# Test DNS (most common lockout symptom)
nslookup kubernetes.default.svc.cluster.local
# If SERVFAIL: DNS egress is blocked
# Fix: add NetworkPolicy allowing egress to port 53

# Test specific pod-to-pod
nc -zv other-pod-ip 8080
# If timeout: ingress to other-pod is blocked

# ─── IDENTIFY what's being blocked with Cilium/Hubble ─────────────
# (if using Cilium CNI)
hubble observe --namespace production --verdict DROPPED
# Shows EXACTLY which flows are being dropped and which policy blocks them

# ─── FIX: Add DNS egress (always first!) ──────────────────────────
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF

# ─── DEBUG: AND vs OR selector gotcha ─────────────────────────────
# This allows traffic from namespace=production AND pod=frontend:
  from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: production
    podSelector:                    # ← NO DASH = same list item = AND
      matchLabels:
        app: frontend

# This allows from namespace=production OR any pod=frontend:
  from:
  - namespaceSelector:              # ← first item
      matchLabels:
        env: production
  - podSelector:                    # ← SEPARATE DASH = second item = OR
      matchLabels:
        app: frontend
# OR = DANGEROUS! Allows frontend pods from ANY namespace

# ─── TEMPORARILY DISABLE policies for debugging ───────────────────
# Rename the policy (kubectl can't easily disable)
kubectl annotate networkpolicy default-deny-all \
  -n production \
  disabled=true
# Still enforced! Annotations don't disable policies.
# Only way to disable: kubectl delete or change podSelector to not match

# Delete temporarily (save YAML first!)
kubectl get networkpolicy default-deny-all -n production -o yaml > /tmp/policy-backup.yaml
kubectl delete networkpolicy default-deny-all -n production
# Test... then reapply
kubectl apply -f /tmp/policy-backup.yaml
```

---

---

# ⚠️ 14.6 Storage Failures — PVC Stuck Pending, Volume Mount Errors

## 🟡 Intermediate — HIGH PRIORITY

```
STORAGE FAILURE DECISION TREE
═══════════════════════════════════════════════════════════════

PVC STATUS: Pending
  └── kubectl describe pvc <name>
      ├── "no persistent volumes available for this claim"
      │     → Static provisioning: no matching PV exists
      │     → Check: kubectl get pv (look for Available PVs)
      │     → Check: access mode and storageClass match
      │
      ├── "waiting for first consumer to be created before binding"
      │     → StorageClass has volumeBindingMode: WaitForFirstConsumer
      │     → PVC binds ONLY after pod is scheduled
      │     → This is NORMAL behavior (not a bug!)
      │     → PVC will bind once pod gets scheduled to a node
      │
      ├── "failed to provision volume with StorageClass X"
      │     → Dynamic provisioner failed (CSI driver error)
      │     → Check CSI driver pods: kubectl get pods -n kube-system | grep csi
      │     → Check CSI driver logs
      │     → On EKS: check EBS CSI driver has IRSA permissions
      │
      └── StorageClass does not exist
            → kubectl get storageclass
            → PVC references non-existent SC → never provisions

POD STATUS: ContainerCreating (stuck, with volume)
  └── kubectl describe pod → Events:
      ├── "Unable to mount volumes: unable to find secret X"
      │     → Secret referenced in volume doesn't exist in namespace
      │     → kubectl get secret X -n <ns>
      │
      ├── "Unable to mount volumes: unable to find configmap X"
      │     → Same for ConfigMap
      │
      ├── "failed to attach volume: volume is already exclusively attached"
      │     → RWO EBS volume still attached to crashed/terminated pod's node
      │     → K8s waiting for node to release it
      │     → If node is gone: force detach in AWS console or wait ~6 min
      │     → Force detach: aws ec2 detach-volume --volume-id vol-xxx --force
      │
      ├── "failed to mount volume: node does not have capacity"
      │     → EBS volume in wrong AZ vs pod's node
      │     → StorageClass needs WaitForFirstConsumer
      │
      └── "permission denied" writing to volume
            → Security context mismatch
            → fsGroup in pod spec sets group ownership of mounted volume
            → Fix: spec.securityContext.fsGroup: <gid>
```

---

```bash
# ─── PVC DEBUGGING ────────────────────────────────────────────────
kubectl get pvc -n production
# NAME        STATUS    VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS
# data-pvc    Pending                                          gp3          ← stuck
# logs-pvc    Bound     pvc-abc123   10Gi       RWO            gp3

kubectl describe pvc data-pvc -n production
# Events section will have the reason

# List all StorageClasses
kubectl get storageclass
# NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# gp2 (default)   kubernetes.io/aws-ebs   Delete          Immediate
# gp3             ebs.csi.aws.com         Delete          WaitForFirstConsumer

# List available PVs (for static provisioning)
kubectl get pv
# NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# pv-data-01   10Gi       RWO            Retain           Available ← could bind
# pv-data-02   10Gi       RWO            Retain           Bound     ← already used

# Check CSI driver health (EKS EBS CSI)
kubectl get pods -n kube-system | grep ebs-csi
kubectl logs -n kube-system deployment/ebs-csi-controller \
  -c csi-provisioner | tail -20

# ─── STUCK VOLUME ATTACHMENT ──────────────────────────────────────
# Find which node had the volume
kubectl get volumeattachment
# NAME                                       ATTACHER         PV              NODE
# csi-abc123...                              ebs.csi.aws.com  pvc-def456...   worker-3

# If worker-3 is dead/unreachable → volume stuck
# Check node status
kubectl get node worker-3
# STATUS: NotReady

# Force detach in AWS (if node is truly gone)
aws ec2 detach-volume \
  --volume-id vol-xxxxx \        # find from PV spec
  --force

# ─── FSGROUP / PERMISSION ISSUES ─────────────────────────────────
# Container can't write to mounted volume
kubectl exec -it my-pod -- ls -la /data
# drwxr-xr-x 2 root root 4096 ← owned by root, container user can't write

# Fix: add fsGroup to pod security context
spec:
  securityContext:
    fsGroup: 1000    # Kubernetes chowns volume to this GID at mount time
  containers:
  - name: app
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
```

---

# ⚠️ 14.7 OOMKilled & CPU Throttling

## 🟡 Intermediate — HIGH PRIORITY

```
RESOURCE PROBLEM PATTERNS
═══════════════════════════════════════════════════════════════

OOMKILLED (memory limit exceeded):
  Symptom:   Pod keeps restarting, exit code 137
  Mechanism: Container exceeded memory limit →
             Linux OOM killer sends SIGKILL to process
  kubectl describe pod → "Last State: OOMKilled"

CPU THROTTLING (over CPU limit):
  Symptom:   High latency, slow responses, but pod NOT restarting
  Mechanism: CFS (Completely Fair Scheduler) enforces CPU quota
             Container gets its limit in CPU time per 100ms window
             Over limit → throttled (paused) for rest of window
  NOT visible in kubectl top (shows requests, not throttle %)
  Need metrics: container_cpu_cfs_throttled_periods_total

MEMORY LEAK vs WRONG LIMIT:
  OOMKilled once: limit likely too low → increase limit
  OOMKilled repeatedly with growing memory: likely leak
  Check: kubectl top pod over time → growing memory = leak
```

---

```bash
# ─── DIAGNOSE OOMKILL ─────────────────────────────────────────────
# Check pod restarts + OOMKill
kubectl get pod my-pod -n production
# RESTARTS: 5 ← suspicious

kubectl describe pod my-pod -n production | grep -A 5 "Last State"
# Last State:    Terminated
#   Reason:      OOMKilled          ← confirmed OOMKill
#   Exit Code:   137
#   Started:     Mon, 15 Jan 2024 10:00:00
#   Finished:    Mon, 15 Jan 2024 10:05:32

# Current memory usage
kubectl top pod my-pod -n production
# NAME      CPU(cores)   MEMORY(bytes)
# my-pod    250m         980Mi        ← near the 1Gi limit!

# Memory usage trend (via Prometheus if available)
# container_memory_working_set_bytes{pod="my-pod"}

# Check limits
kubectl get pod my-pod -n production \
  -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"memory":"1Gi"},"requests":{"memory":"256Mi"}}

# Fix: increase memory limit
kubectl patch deployment my-api -n production \
  --patch '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"2Gi"}}}]}}}}'

# ─── DIAGNOSE CPU THROTTLING ──────────────────────────────────────
# kubectl top shows usage — doesn't show throttling
kubectl top pod my-pod -n production --containers
# NAME    NAME   CPU(cores)   MEMORY(bytes)
# my-pod  app    800m         512Mi     ← using 800m of 1000m limit

# Throttling from Prometheus:
# rate(container_cpu_cfs_throttled_periods_total{pod="my-pod"}[5m])
#   / rate(container_cpu_cfs_periods_total{pod="my-pod"}[5m])
# > 0.25 = 25% of time throttled → significant latency impact

# High throttling with low usage? → Java JVM issue
# JVM doesn't honor container CPU limits by default
# Fix: -XX:+UseContainerSupport (enabled by default in JDK 10+)
# Or: -XX:ActiveProcessorCount=2

# Fix CPU throttling options:
# Option 1: Increase CPU limit
kubectl patch deployment my-api -n production \
  --patch '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"limits":{"cpu":"2000m"}}}]}}}}'

# Option 2: Remove CPU limit entirely (controversial but valid for latency-sensitive)
# No CPU limit = container can use any available CPU, no throttling
# Risk: can starve other pods on same node

# ─── FINDING RESOURCE WASTE / RIGHTSIZING ─────────────────────────
# See all pods with their requests/limits/actual usage
kubectl top pods -n production --containers | sort -k3 -r

# VPA recommendations (if VPA installed)
kubectl describe vpa my-api -n production
# Recommendation:
#   Container Recommendations:
#     Container Name: app
#     Lower Bound:    cpu: 150m  memory: 400Mi
#     Target:         cpu: 350m  memory: 650Mi   ← suggested
#     Upper Bound:    cpu: 700m  memory: 1Gi
```

---

# 14.8 Node Issues — NotReady, Pressure Conditions

## 🟡 Intermediate

```bash
# ─── NODE NOTREADY ────────────────────────────────────────────────
kubectl get nodes
# NAME       STATUS     ROLES    AGE   VERSION
# worker-2   NotReady   <none>   30d   v1.28.5  ← problem!

# What condition is False/Unknown?
kubectl describe node worker-2 | grep -A 15 "Conditions:"
# Type               Status  Reason
# MemoryPressure     False
# DiskPressure       False
# PIDPressure        False
# Ready              False   KubeletNotReady   ← kubelet stopped reporting

# Common causes of NotReady:
# 1. kubelet crashed on the node
# 2. Node lost network connectivity (partitioned)
# 3. Node ran out of disk/memory → pressure taint added

# ─── PRESSURE TAINT DEBUGGING ─────────────────────────────────────
kubectl describe node worker-2 | grep Taints
# Taints: node.kubernetes.io/disk-pressure:NoSchedule   ← pressure!
#         node.kubernetes.io/memory-pressure:NoSchedule

# What's consuming disk?
# SSH to node, or use kubectl debug node:
kubectl debug node/worker-2 -it --image=ubuntu -- bash
# Inside debug container (host fs at /host):
df -h /host                        # overall disk usage
du -sh /host/var/lib/docker/*      # Docker storage
du -sh /host/var/log/*             # log files
du -sh /host/var/lib/kubelet/*     # kubelet state

# Clean up disk (careful!)
# Remove stopped containers
crictl rmp $(crictl ps -a -q --state exited)
# Remove unused images
crictl rmi --prune
# Truncate large logs
find /host/var/log -name "*.log" -size +500M -exec truncate -s 100M {} \;

# ─── NODE NOT READY — KUBELET INVESTIGATION ───────────────────────
# SSH to the node
ssh worker-2
sudo systemctl status kubelet
# ● kubelet.service - kubelet
#    Active: failed (Result: exit-code) ← kubelet crashed!

sudo journalctl -u kubelet -n 100 --no-pager
# Look for: certificate expired, config errors, OOM on node

# Restart kubelet
sudo systemctl restart kubelet
sudo systemctl enable kubelet

# ─── EVICTED PODS CLEANUP ─────────────────────────────────────────
# Find all evicted pods
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted

# Delete all evicted pods
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.status.reason=="Evicted") |
         .metadata.namespace + "/" + .metadata.name' | \
  while read ns_name; do
    ns=$(echo $ns_name | cut -d/ -f1)
    name=$(echo $ns_name | cut -d/ -f2)
    kubectl delete pod $name -n $ns
  done
```

---

# 14.9 Deployment Stuck — Rollout Not Progressing

## 🟡 Intermediate

```bash
# ─── STUCK ROLLOUT ────────────────────────────────────────────────
kubectl rollout status deployment/my-api -n production
# Waiting for deployment "my-api" rollout to finish:
# 2 out of 5 new replicas have been updated...

# Why is it stuck?
kubectl describe deployment my-api -n production | \
  grep -A 10 "Conditions:"
# Available    True    MinimumReplicasAvailable
# Progressing  False   ProgressDeadlineExceeded  ← STUCK! deadline hit

# What are the new pods doing?
kubectl get pods -n production | grep my-api
# my-api-OLD-abc   1/1   Running            0    10m  ← old pods fine
# my-api-NEW-xyz   0/1   CrashLoopBackOff   5    8m   ← new pods crashing

# Why are new pods crashing?
kubectl logs my-api-NEW-xyz -n production --previous
# Error: DATABASE_URL environment variable required
# → New image version needs a new env var that wasn't set

# Fix: update deployment with missing env var + roll forward
kubectl set env deployment/my-api -n production \
  DATABASE_URL=postgres://...

# Or: rollback to previous version
kubectl rollout undo deployment/my-api -n production

# ─── PODS STUCK TERMINATING DURING ROLLOUT ───────────────────────
kubectl get pods -n production
# my-api-OLD-abc   1/1   Terminating   0   20m  ← stuck!

# Why won't it terminate?
kubectl describe pod my-api-OLD-abc -n production
# Check Events for: "error killing container"
# Check finalizers
kubectl get pod my-api-OLD-abc -n production \
  -o jsonpath='{.metadata.finalizers}'

# preStop hook hanging? Check grace period
kubectl get pod my-api-OLD-abc -n production \
  -o jsonpath='{.spec.terminationGracePeriodSeconds}'

# Force delete (data loss risk — understand before using)
kubectl delete pod my-api-OLD-abc -n production \
  --grace-period=0 --force

# ─── ROLLOUT PAUSED ──────────────────────────────────────────────
kubectl rollout status deployment/my-api -n production
# deployment "my-api" is paused   ← someone ran kubectl rollout pause

kubectl rollout resume deployment/my-api -n production

# ─── progressDeadlineSeconds EXCEEDED ───────────────────────────
# Default: 600 seconds (10 minutes)
# If rollout doesn't complete in 600s → Progressing=False
kubectl patch deployment my-api -n production \
  --patch '{"spec":{"progressDeadlineSeconds": 1200}}'
# OR fix the underlying reason new pods aren't becoming Ready
```

---

# ⚠️ 14.10 RBAC Failures — Forbidden Errors

## 🟡 Intermediate — HIGH PRIORITY

```bash
# ─── SYMPTOM: Forbidden error ─────────────────────────────────────
# Error from server (Forbidden):
# pods is forbidden: User "jane@company.com" cannot list resource "pods"
#                    in API group "" in the namespace "production"

# ─── STEP 1: Confirm who you are ─────────────────────────────────
kubectl auth whoami
# ATTRIBUTE   VALUE
# Username    jane@company.com
# Groups      [system:authenticated oidc:developers]

# ─── STEP 2: Check what they can do ──────────────────────────────
kubectl auth can-i list pods -n production \
  --as=jane@company.com
# no

kubectl auth can-i list pods -n production \
  --as=system:serviceaccount:production:my-app-sa
# no

# Check all permissions for a user
kubectl auth can-i --list \
  --as=jane@company.com \
  -n production
# Resources   Non-Resource URLs   Resource Names   Verbs
# pods        []                  []               [get]   ← can get but not list!

# ─── STEP 3: Find what RoleBindings exist ────────────────────────
# All bindings in namespace
kubectl get rolebindings -n production -o yaml | \
  grep -A 5 "subjects:"

# Find bindings for specific user
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.subjects[]?.name == "jane@company.com") |
      {name: .metadata.name, ns: .metadata.namespace, role: .roleRef.name}'

# Find bindings for a ServiceAccount
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(
    .subjects[]? |
    .kind == "ServiceAccount" and .name == "my-app-sa"
  ) | {name: .metadata.name, role: .roleRef.name}'

# ─── STEP 4: Check what role grants ──────────────────────────────
kubectl describe role pod-reader -n production
# Rules:
# Resources  Verbs
# pods       [get]    ← only get, not list!

kubectl describe clusterrole view | grep pods
# pods  [get list watch]  ← view includes list

# ─── STEP 5: Create or fix binding ───────────────────────────────
# Grant list permission
kubectl create rolebinding jane-pod-viewer \
  --clusterrole=view \
  --user=jane@company.com \
  -n production

# For ServiceAccount
kubectl create rolebinding myapp-sa-binding \
  --role=pod-reader \
  --serviceaccount=production:my-app-sa \
  -n production

# ─── COMMON RBAC GOTCHAS ─────────────────────────────────────────
# Gotcha 1: OIDC prefix mismatch
# API Server configured with --oidc-username-prefix=oidc:
# → Kubernetes sees: "oidc:jane@company.com"
# → RoleBinding has subject: "jane@company.com" (no prefix)
# → Binding never matches!
# Fix: add prefix to binding subject
kubectl get rolebinding -n production -o yaml | grep name | grep jane
# Check if it has: "oidc:jane@company.com" or "jane@company.com"

# Gotcha 2: Wrong subject namespace for ServiceAccount
# RoleBinding in namespace "production" with subject:
#   kind: ServiceAccount
#   name: my-app-sa
#   namespace: staging   ← wrong! SA is in production, not staging
# → ServiceAccount in "production" namespace gets no permissions

# Gotcha 3: ClusterRoleBinding needed for cluster-scoped resources
# Nodes, PersistentVolumes, StorageClasses are cluster-scoped
# RoleBinding cannot grant access to cluster-scoped resources
# Must use ClusterRoleBinding

# ─── IN-CLUSTER APP GETTING FORBIDDEN ────────────────────────────
# App pod logs: "403 Forbidden calling /api/v1/pods"
# The pod's ServiceAccount lacks RBAC permissions

# Find what SA the pod uses
kubectl get pod my-pod -n production \
  -o jsonpath='{.spec.serviceAccountName}'
# my-app-sa

# Check SA's permissions
kubectl auth can-i list pods \
  --as=system:serviceaccount:production:my-app-sa \
  -n production
# no → need to add RoleBinding
```

---

# 14.11 crictl — Debugging Below the Kubelet

## 🟡 Intermediate

### What it is and when to use it

```
CRICTL — CONTAINER RUNTIME INTERFACE TOOL
═══════════════════════════════════════════════════════════════

kubectl talks to: API Server → kubelet → containerd → runc
crictl talks to:             kubelet → containerd directly

USE crictl WHEN:
  ✓ Pod appears in kubectl but container won't start
  ✓ Node-level container issues (containerd errors)
  ✓ Node is semi-broken and kubectl is unreliable
  ✓ Debugging container runtime configuration
  ✓ Checking actual container state below Kubernetes level

crictl runs ON THE NODE (SSH or kubectl debug node)
Communicates with: /run/containerd/containerd.sock (containerd)
                   /var/run/crio/crio.sock (CRI-O)
```

---

```bash
# SSH to node first (or: kubectl debug node/worker-2 -it --image=ubuntu)

# ─── CRICTL COMMANDS ──────────────────────────────────────────────
# List running pods (from containerd's perspective)
crictl pods
# POD ID              CREATED             STATE     NAME               NAMESPACE
# abc123def456        5 minutes ago       Ready     my-api-xyz         production
# def456ghi789        2 minutes ago       NotReady  worker-abc         production

# List containers (not pods)
crictl ps                              # running containers
crictl ps -a                           # all containers including stopped
# CONTAINER ID   IMAGE                  COMMAND    CREATED        STATE
# abc123def456   myapp@sha256:...       "node s…"  5 minutes ago  Running
# def456ghi789   myapp@sha256:...       "node s…"  2 minutes ago  Exited

# Get container logs (when kubectl logs doesn't work)
crictl logs abc123def456
crictl logs --tail=100 abc123def456
crictl logs --since=1h abc123def456    # last 1 hour of logs

# Inspect container (full runtime config)
crictl inspect abc123def456
# Shows: pid, state, mounts, namespaces, cgroup path

# Execute command in container (when kubectl exec is broken)
crictl exec -it abc123def456 /bin/sh

# Inspect pod (pod sandbox)
crictl inspectp abc123def456pod        # pod-level info

# ─── CRICTL IMAGE MANAGEMENT ──────────────────────────────────────
crictl images                          # list pulled images
crictl pull myapp:v1.2.3               # pull image directly
crictl rmi myapp:old-version           # remove image
crictl imagefsinfo                     # image filesystem info

# ─── DIAGNOSING CONTAINERD ISSUES ────────────────────────────────
# Check containerd status
systemctl status containerd
journalctl -u containerd -n 50 --no-pager

# Check containerd config
cat /etc/containerd/config.toml

# Check containerd snapshotter (OverlayFS) health
# If OverlayFS corrupted: containers won't start, cryptic errors
ls /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/

# ─── READING CONTAINER STATE FROM FILE SYSTEM ────────────────────
# Container log files (what kubectl logs reads)
ls /var/log/pods/<namespace>_<pod-name>_<uid>/<container>/
cat /var/log/pods/production_my-api-xyz_abc123.../app/0.log

# Container runtime state
ls /var/lib/containerd/
# io.containerd.content.v1.content/  ← image layers stored here
# io.containerd.snapshotter.v1.overlayfs/  ← container filesystems

# ─── CRICTL vs DOCKER COMPARISON ─────────────────────────────────
# docker ps       → crictl ps
# docker images   → crictl images
# docker logs     → crictl logs
# docker exec     → crictl exec
# docker inspect  → crictl inspect
# docker pull     → crictl pull
```

---

---

# 14.12 Control Plane Debugging — API Server Slow, etcd Latency, Scheduler Backlog

## 🔴 Advanced

### Mental Model

```
CONTROL PLANE HEALTH = CLUSTER HEALTH
═══════════════════════════════════════════════════════════════

SYMPTOMS OF CONTROL PLANE DEGRADATION:
  kubectl commands timeout or are slow
  Pods stuck in Pending despite available nodes
  New Deployments take minutes to create pods
  Rollouts stall or never start
  Controllers stop reconciling (Deployments don't scale)
  Admission webhooks timing out
  WATCH connections drop and reconnect frequently

CONTROL PLANE COMPONENT CHAIN:
  kubectl → API Server → etcd (writes)
              ↓
          Scheduler ← watches pods (via Informer)
              ↓
          Controller Manager ← watches all objects
              ↓
          kubelet ← watches assigned pods
              ↓
          Pod starts

BOTTLENECK IDENTIFICATION:
  API Server slow?    → etcd latency, admission webhooks, high cardinality
  Scheduler slow?     → scheduler queue backlog, plugin overhead
  Controller slow?    → controller queue depth, rate limiting
  etcd slow?          → disk I/O, compaction, defrag needed, large objects
```

---

### API Server Debugging

```bash
# ─── API SERVER HEALTH ─────────────────────────────────────────────
# Check if API Server is responding
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /livez

# Verbose API call timing (see how long requests take)
kubectl get pods -n production -v=6
# I0115 10:00:00.000] GET https://api-server:6443/api/v1/namespaces/production/pods
# Response Status: 200 OK in 450 milliseconds    ← normal: <100ms, slow: >500ms

# API Server metrics (if accessible)
# Self-managed: curl https://api-server:6443/metrics (requires auth)
# EKS: CloudWatch → EKS control plane logs → API server

# KEY API SERVER METRICS:
# apiserver_request_duration_seconds_bucket  → latency per verb/resource
# apiserver_request_total{code="429"}        → rate limiting (too many requests)
# apiserver_current_inflight_requests        → concurrent requests in flight
# etcd_request_duration_seconds_bucket       → etcd call latency from API Server

# Check API Server pods (self-managed)
kubectl get pods -n kube-system | grep apiserver
kubectl logs -n kube-system kube-apiserver-master-1 \
  --tail=100 | grep -E "slow|timeout|error|etcd"

# Watch list requests consuming memory (list paging)
kubectl get --raw '/metrics' | grep apiserver_watch_events_total

# ─── COMMON API SERVER PROBLEMS ───────────────────────────────────
# Problem 1: Too many LIST requests (controllers hammering API)
kubectl get --raw '/metrics' | \
  grep 'apiserver_request_total{verb="LIST"'
# If LIST count is huge → controller is not using Informer/watch cache
# Fix: ensure controllers use shared informers, not direct LIST calls

# Problem 2: Large objects in etcd
# Objects > 1.5MB cause slow API Server responses
# ConfigMaps/Secrets with large data, or CRDs with large status fields
kubectl get configmap huge-config -n production \
  -o json | wc -c
# > 1000000 = 1MB+ object = problem

# Problem 3: Admission webhook timeout cascade
# Every API request hits webhooks → if webhook is slow → all requests slow
# Check webhook latency
kubectl get --raw '/metrics' | \
  grep admission_webhook_admission_duration

# ─── EKS CONTROL PLANE LOGS ────────────────────────────────────────
# Enable logging (if not already enabled)
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Query API Server logs in CloudWatch
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/cluster \
  --log-stream-name-prefix kube-apiserver \
  --filter-pattern '"slow"' \
  --start-time $(date -d '1 hour ago' +%s000)

# Query scheduler logs
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/cluster \
  --log-stream-name-prefix kube-scheduler \
  --filter-pattern '"pending"'
```

---

### etcd Debugging

```bash
# ─── ETCD HEALTH ──────────────────────────────────────────────────
# (self-managed clusters only — EKS etcd is managed by AWS)
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Health check
etcdctl endpoint health
# https://127.0.0.1:2379 is healthy: successfully committed proposal in 2ms

# Member list (HA clusters — all 3 members healthy?)
etcdctl member list -w table
# +------------------+---------+----------+----------------------------+
# |        ID        | STATUS  |   NAME   |        CLIENT URLS         |
# +------------------+---------+----------+----------------------------+
# | 8e9e05c52164694d | started | master-1 | https://10.0.1.10:2379     |
# | 91bc3c398fb3c146 | started | master-2 | https://10.0.1.11:2379     |
# | fd422379fda50e48 | started | master-3 | https://10.0.1.12:2379     |

# Endpoint status (shows who is leader)
etcdctl endpoint status -w table
# +------------------------+------------------+--------+---------+--------+
# |       ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER |
# +------------------------+------------------+--------+---------+--------+
# | https://10.0.1.10:2379 | 8e9e05c52164694d |  3.5.9 |  512 MB |    true   |

# ─── ETCD PERFORMANCE ISSUES ──────────────────────────────────────
# DB size growing → needs compaction
etcdctl endpoint status -w json | python3 -m json.tool | grep dbSize
# "dbSize": 536870912   ← 512MB — if > 2GB, performance degrades

# Compact etcd (removes old revisions, reduces logical DB size)
REVISION=$(etcdctl endpoint status --write-out="json" | \
  python3 -c "import json,sys; print(json.load(sys.stdin)[0]['Status']['header']['revision'])")
etcdctl compact $REVISION

# Defrag (physically reclaims disk space after compaction)
# WARNING: etcd briefly unavailable during defrag
etcdctl defrag                          # defrag leader
etcdctl defrag --endpoints=https://10.0.1.11:2379  # defrag follower first

# Check etcd latency (fsync commit time)
etcdctl check perf
# This tests if disk can sustain etcd write latency requirements
# PASS: Throughput is 62 writes/s
# FAIL: disk too slow → use SSDs (gp3, io1) for etcd, not gp2 HDD

# Find large keys causing slow reads
etcdctl get /registry --prefix --keys-only | \
  awk -F/ '{print $3}' | sort | uniq -c | sort -rn | head -20
# 2847 secrets      ← most objects
# 1523 pods
# 891  configmaps
# 23   customresourcedefinition

# Find actual large objects
etcdctl get /registry/configmaps/production/huge-config \
  | wc -c    # check size of specific object

# ─── SCHEDULER BACKLOG ────────────────────────────────────────────
# Scheduler metrics (self-managed)
kubectl get --raw '/metrics' | grep scheduler_pending_pods
# scheduler_pending_pods{queue="activeQ"} 145      ← 145 pods waiting to schedule!
# scheduler_pending_pods{queue="backoffQ"} 12      ← 12 in backoff (failed recently)
# scheduler_pending_pods{queue="unschedulableQ"} 8 ← 8 unschedulable

# High activeQ: scheduler not fast enough, or cluster full
# High unschedulableQ: pods that CANNOT be placed (check their events)

# Scheduler logs
kubectl logs -n kube-system kube-scheduler-master-1 \
  --tail=50 | grep -E "Preempting|Unable to schedule|error"
```

---

### 🎤 Short Crisp Interview Answer

> *"Control plane debugging starts with measuring API Server response latency using kubectl -v=6 which shows request timing per call. Slow API Servers are usually caused by: etcd I/O latency (check etcdctl check perf — etcd needs NVMe/SSD, not spinning disk), admission webhook timeouts cascading across all requests, or controllers doing raw LIST calls instead of using the watch cache which hammers the API Server. On EKS, I enable control plane logging to CloudWatch and filter for 'slow' in API Server logs. For etcd specifically: if DB size exceeds 2GB, compact and defrag to restore performance. Scheduler backlogs show up in scheduler_pending_pods metrics — a high unschedulableQ count means pods can never be placed and need investigation of their scheduling constraints."*

---

---

# ⚠️ 14.13 Webhook Outages — Admission Webhook Blocking All Pod Creation

## 🔴 Advanced — HIGH PRIORITY

### Why this is critical

```
WEBHOOK OUTAGE = CLUSTER-WIDE IMPACT
═══════════════════════════════════════════════════════════════

When a MutatingWebhookConfiguration or ValidatingWebhookConfiguration
has failurePolicy: Fail (the default) and the webhook server is DOWN:

  Every matching API request → webhook call → connection refused/timeout
  → API Server returns 500 or timeout error
  → NO matching resources can be created or updated

WORST CASE:
  Webhook matches: all pods in all namespaces
  Webhook server crashes
  failurePolicy: Fail
  → ZERO new pods can be created cluster-wide
  → Existing pods still running (webhooks only on admission, not runtime)
  → Any rolling update, new deployment, or pod restart = BLOCKED
  → This is a production P1 incident

REAL WORLD EXAMPLE:
  OPA Gatekeeper webhook crashes during maintenance
  failurePolicy: Fail
  → All pod creates blocked
  → Kubernetes can't even reschedule system pods if they crash
  → Must emergency-delete the webhook configuration to recover
```

---

### Diagnosing Webhook Outage

```bash
# SYMPTOM: kubectl apply -f deployment.yaml fails with:
# Error from server (InternalError): Internal error occurred:
# failed calling webhook "validate.gatekeeper.sh":
# Post "https://gatekeeper-webhook-service.gatekeeper-system.svc:443/v1/admit":
# dial tcp 10.96.45.23:443: connect: connection refused

# ─── STEP 1: Identify which webhook is failing ───────────────────
# List all webhook configurations
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Which ones are set to Fail?
kubectl get mutatingwebhookconfigurations -o json | \
  jq '.items[] | {name: .metadata.name, failurePolicy: .webhooks[].failurePolicy}'
# {"name":"gatekeeper-mutating","failurePolicy":"Fail"}
# {"name":"istio-sidecar-injector","failurePolicy":"Ignore"}  ← safe if down

# ─── STEP 2: Check webhook server pods ───────────────────────────
# Find OPA/Gatekeeper webhook pods
kubectl get pods -n gatekeeper-system
# NAME                        READY   STATUS    RESTARTS
# gatekeeper-controller-0     0/1     Pending   0         ← webhook server down!
# gatekeeper-controller-1     0/1     Pending   0

# Why are they Pending? (usually circular: new pods need webhook to create)
kubectl describe pod gatekeeper-controller-0 -n gatekeeper-system
# FailedScheduling: nodes don't match... OR:
# Error from server: failed calling webhook  ← circular!

# ─── STEP 3: Emergency recovery — change failurePolicy ───────────
# Option A: Change failurePolicy to Ignore (webhook still called, failure ignored)
kubectl patch validatingwebhookconfiguration gatekeeper-validating-webhook-configuration \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

# Option B: Delete the webhook entirely (nuclear — removes all policy enforcement)
kubectl delete validatingwebhookconfiguration gatekeeper-validating-webhook-configuration
kubectl delete mutatingwebhookconfiguration gatekeeper-mutating-webhook-configuration

# Now gatekeeper pods can be created (webhook no longer blocking)
# Once gatekeeper is healthy, reapply webhook configuration

# ─── STEP 4: Fix root cause (why did webhook server go down?) ────
# Common causes:
# 1. Node failure evicted webhook pods + new pods can't create (circular)
# 2. Resource exhaustion (OOM on webhook pods)
# 3. TLS certificate expiry
# 4. Webhook service misconfigured (wrong port/selector)

# Check TLS certificate expiry
kubectl get secret -n gatekeeper-system gatekeeper-webhook-server-cert \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -dates
# notAfter=Jan 15 10:00:00 2024 GMT  ← expired!

# Renew cert (cert-manager usually handles this automatically)
kubectl delete secret gatekeeper-webhook-server-cert -n gatekeeper-system
# cert-manager will recreate it

# ─── WEBHOOK BEST PRACTICES (prevent outages) ─────────────────────
# 1. Always exclude kube-system from webhooks
webhooks:
- name: my-webhook.example.com
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "kube-public"]
  failurePolicy: Fail

# 2. Always exclude the webhook's own namespace
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["gatekeeper-system", "kube-system"]

# 3. Run at least 3 webhook replicas
spec:
  replicas: 3
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule

# 4. Set appropriate timeout (default 10s — API Server drops after 30s max)
webhooks:
- timeoutSeconds: 5      # fail fast rather than hanging

# 5. Use Ignore for non-critical webhooks
  failurePolicy: Ignore  # mutation webhook for non-critical labels

# 6. Test webhook is reachable before applying
kubectl get endpoints -n gatekeeper-system \
  gatekeeper-webhook-service
# Must have at least one endpoint
```

---

---

# 14.14 Cluster Autoscaler Not Scaling

## 🔴 Advanced

### Scale-Up Not Working

```bash
# SYMPTOM: Pods stuck Pending, CA should add nodes but doesn't

# ─── CHECK CA IS RUNNING ──────────────────────────────────────────
kubectl get pods -n kube-system | grep cluster-autoscaler
kubectl logs -n kube-system deployment/cluster-autoscaler \
  --tail=100 | grep -E "scale|Scale|ERROR|error"

# ─── WHY CA WON'T SCALE UP ────────────────────────────────────────
# Reason 1: Pod has scheduling constraints CA can't satisfy
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep "cannot be scheduled"
# "Pod worker/ml-job cannot be scheduled on any node group.
#  Node group gpu-nodes: max size already reached"
#                         ↑
#  Node group hit maxSize limit → CA won't exceed it

# Increase max size of node group
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name gpu-nodes \
  --scaling-config minSize=0,maxSize=20,desiredSize=5

# Reason 2: Pod has unsatisfiable node selector
# Pod requires: accelerator=nvidia-tesla-v100
# Node group only has: accelerator=nvidia-tesla-t4
# → CA simulates scheduling → no group can satisfy → won't scale
kubectl describe pod stuck-ml-job -n production | grep -A 5 "Node-Selectors"

# Reason 3: Pod too large for any node in any group
kubectl describe pod stuck-ml-job | grep -A 5 "Limits"
# Limits: memory: 200Gi
# Largest instance in node group: r5.4xlarge = 128GB
# → No node can ever fit this pod

# Reason 4: CA not discovering the node group (missing ASG tags)
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep "node group"
# If node group not listed → missing tags:
# k8s.io/cluster-autoscaler/enabled = true
# k8s.io/cluster-autoscaler/<cluster-name> = owned

# Verify ASG tags
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-nodegroup-asg \
  --query 'AutoScalingGroups[0].Tags'

# Reason 5: CA lacks IAM permissions
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep "AccessDenied\|UnauthorizedOperation"
# "AccessDenied: autoscaling:SetDesiredCapacity"
# Fix: attach correct IAM policy to CA's ServiceAccount (IRSA)

# ─── VERIFY SCALE-UP SIMULATION ───────────────────────────────────
# CA logs when it decides to scale up
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep -E "Expanding|Scale up|new nodes"
# "Scale up triggered, adding 2 nodes to node group general-workers"
# "Waiting for nodes to become ready"
```

---

### Scale-Down Not Working

```bash
# SYMPTOM: Nodes underutilized for >10 minutes but CA won't remove them

# ─── CHECK CA SCALE-DOWN LOGS ─────────────────────────────────────
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep -E "scale.down|Scale down|not removing|cannot remove|No candidates"

# ─── COMMON REASONS CA WON'T SCALE DOWN ──────────────────────────

# Reason 1: Pod with safe-to-evict: false annotation
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep "safe-to-evict"
# "Node worker-5 has pod kube-system/aws-node which cannot be evicted"
# OR: "Pod has annotation cluster-autoscaler.kubernetes.io/safe-to-evict: false"

# Find which pods block scale-down
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.metadata.annotations}{"\n"}{end}' | \
  grep "safe-to-evict.*false"

# Reason 2: Pod violates PDB during scale-down simulation
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep "PodDisruptionBudget"
# "Cannot evict pod production/api-xyz: would violate PDB"

# Find the blocking PDB
kubectl get pdb -n production
# NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# api-pdb   2               N/A               0   ← 0 disruptions allowed!
# If allowed disruptions = 0: CA cannot evict ANY pod from this deployment

# Fix: Scale up the deployment so PDB has slack
kubectl scale deployment my-api --replicas=3 -n production
# Now: 3 replicas, minAvailable=2 → 1 allowed disruption

# Reason 3: Pod with local storage (emptyDir)
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep "local storage"
# "Node worker-5 has pod with local storage: production/cache-pod"

# If safe to evict (data is ephemeral):
# Add annotation to the pod's deployment template:
spec:
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"

# OR: configure CA with --skip-nodes-with-local-storage=false

# Reason 4: Node has been underutilized but not long enough
# Default: scale-down-unneeded-time=10m
# Check when CA started considering this node:
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep "worker-5.*unneeded"
# "worker-5 was unneeded for 7m30s, will be scaled down in 2m30s"

# Reason 5: Node scale-down disabled via annotation
kubectl describe node worker-5 | grep -i scale-down
# Annotations: cluster-autoscaler.kubernetes.io/scale-down-disabled: true
# Remove annotation to re-enable
kubectl annotate node worker-5 \
  cluster-autoscaler.kubernetes.io/scale-down-disabled-

# ─── FORCE VIEWING CA DECISION ────────────────────────────────────
# CA expvar endpoint (if accessible)
kubectl port-forward -n kube-system deployment/cluster-autoscaler 8085:8085
curl http://localhost:8085/debug/vars | python3 -m json.tool | \
  grep -A 5 "scaleDown"
# Shows internal CA state: which nodes are candidates, timers, blockers
```

---

---

# 14.15 Network Deep-Dive Tools — tcpdump, conntrack, iptables

## 🔴 Advanced

### When to use network deep-dive tools

```
ESCALATION PATH FOR NETWORK DEBUGGING
═══════════════════════════════════════════════════════════════

Level 1: kubectl exec → curl / nc / nslookup      (most issues resolved here)
Level 2: kubectl debug → netshoot image            (DNS, routing, connectivity)
Level 3: tcpdump on pod/node                       (packet-level: is traffic arriving?)
Level 4: conntrack / iptables inspection           (NAT, Service routing)
Level 5: eBPF / Hubble / Cilium observe            (kernel-level, zero overhead)

USE LEVEL 3+ WHEN:
  curl succeeds but app sees no traffic → tcpdump verifies packets arrive
  Service ClusterIP unreachable → iptables rules might be missing
  Intermittent connection drops → conntrack table full or NAT collision
  NetworkPolicy seems right but traffic still dropped → packet trace
```

---

### tcpdump — Packet Capture

```bash
# ─── TCPDUMP INSIDE A POD ─────────────────────────────────────────
# Most app containers don't have tcpdump
# Use netshoot ephemeral container (shares pod network namespace)

kubectl debug -it my-pod \
  --image=nicolaka/netshoot \
  --target=app \              # share network namespace with 'app' container
  -n production \
  -- tcpdump -i eth0 -nn port 8080

# Common tcpdump filters for K8s:
# Capture all traffic on pod's interface
tcpdump -i eth0 -nn -s 0

# Capture HTTP traffic to/from specific port
tcpdump -i eth0 -nn port 80 or port 8080

# Capture DNS queries
tcpdump -i eth0 -nn port 53

# Capture traffic to specific pod IP
tcpdump -i eth0 -nn host 10.244.2.5

# Capture SYN packets (new connections)
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-syn != 0'

# Save to file and analyse later
tcpdump -i eth0 -nn -w /tmp/capture.pcap
# Copy out: kubectl cp debug-pod:/tmp/capture.pcap ./capture.pcap
# Analyse: wireshark capture.pcap

# ─── TCPDUMP ON THE NODE ──────────────────────────────────────────
# Useful for seeing ALL pod traffic on a node

# Get node debug container
kubectl debug node/worker-2 -it --image=nicolaka/netshoot

# Inside debug container (host network accessible):
# List interfaces
ip link show
# eth0: host interface
# vethXXXXXX: one per container on this node (veth pair)

# Capture on host interface
tcpdump -i eth0 -nn host 10.244.2.5    # traffic to/from pod IP

# Capture on specific veth pair (one container's traffic)
tcpdump -i veth3b2a1c9 -nn

# Which veth corresponds to which pod?
# On node: check /sys/class/net/<veth>/ifindex
# In pod: ip link → shows peer ifindex
kubectl exec my-pod -- ip link show eth0
# 5: eth0@if6: ...        ← ifindex 5, peer is index 6
# On node: ip link | grep "^6:"
# 6: veth3b2a1c9@if5: ... ← this is the veth for my-pod

# ─── CONNTRACK — NAT SESSION TRACKING ─────────────────────────────
# conntrack tracks NAT sessions (Service → Pod IP translation)
# On node (via kubectl debug node):

# Install conntrack if not present
apt-get install -y conntrack

# View all tracked connections
conntrack -L
# tcp 6 86392 ESTABLISHED src=10.244.1.5 dst=10.100.50.25 sport=45678 dport=80
#                          src=10.244.2.3 dst=10.244.1.5  sport=8080   dport=45678 ASSURED
#          ↑ client pod      ↑ Service ClusterIP             ↑ actual pod IP after DNAT

# Filter by destination (find connections to a Service)
conntrack -L | grep "dst=10.100.50.25"

# Count connections per source (detect connection exhaustion)
conntrack -L | awk '{print $5}' | sort | uniq -c | sort -rn | head -10

# Conntrack table size (if full → new connections dropped silently!)
cat /proc/sys/net/netfilter/nf_conntrack_max
# 131072  ← max connections
cat /proc/sys/net/netfilter/nf_conntrack_count
# 131068  ← current connections → almost full!

# Fix: increase conntrack table size
sysctl -w net.netfilter.nf_conntrack_max=262144

# ─── IPTABLES — SERVICE ROUTING RULES ─────────────────────────────
# kube-proxy writes iptables rules for every Service
# On node via kubectl debug node:

# Show all kube-proxy rules (verbose, thousands of rules possible)
iptables-save | grep <service-name>

# Find rules for a specific Service ClusterIP
SERVICE_IP="10.100.50.25"
iptables-save | grep $SERVICE_IP
# -A KUBE-SERVICES -d 10.100.50.25/32 -p tcp --dport 80
#   -j KUBE-SVC-ABCDEF123456      ← jumps to this chain

# Follow the chain
iptables-save | grep "KUBE-SVC-ABCDEF123456"
# -A KUBE-SVC-ABCDEF123456 -m statistic --mode random --probability 0.5
#   -j KUBE-SEP-POD1       ← 50% to pod 1
# -A KUBE-SVC-ABCDEF123456
#   -j KUBE-SEP-POD2       ← remaining 50% to pod 2

# KUBE-SEP (Service EndPoint)
iptables-save | grep "KUBE-SEP-POD1"
# -A KUBE-SEP-POD1 -p tcp -j DNAT --to-destination 10.244.1.5:8080
#                                                    ↑ actual pod IP:port

# If Service has NO endpoints → no KUBE-SEP rules → traffic blackholed
# Verify endpoints rule exists:
iptables-save | grep "KUBE-SVC-ABCDEF123456" | grep KUBE-SEP
# If empty: no endpoints → check kubectl get endpoints <svc>

# ─── IPVS MODE (alternative to iptables) ─────────────────────────
# If kube-proxy runs in IPVS mode:
ipvsadm -L -n
# IP Virtual Server version 1.2.1
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port Forward Weight ActiveConn InActConn
# TCP  10.100.50.25:80 rr          ← round-robin
#   -> 10.244.1.5:8080  Masq    1      5          0
#   -> 10.244.2.3:8080  Masq    1      3          0

# IPVS is more efficient than iptables for large clusters (O(1) vs O(n))
```

---

---

# 14.16 Performance Profiling — CPU Flame Graphs, Go pprof

## 🔴 Advanced

### When to Profile K8s Components

```
PROFILING SCENARIOS
═══════════════════════════════════════════════════════════════

WHEN TO PROFILE:
  API Server using 100% CPU → what code path?
  Controller Manager slow to reconcile → which controller?
  etcd high memory → goroutine leak?
  Custom operator consuming too much CPU → which reconcile call?

KUBERNETES COMPONENTS EXPOSE pprof:
  API Server:           :6443/debug/pprof/
  Controller Manager:   :10257/debug/pprof/
  Scheduler:            :10259/debug/pprof/
  kubelet:              :10250/debug/pprof/
  Custom operators:     expose via controller-runtime metrics endpoint

pprof PROFILES:
  /debug/pprof/profile    → CPU profile (30s sample by default)
  /debug/pprof/heap       → heap memory snapshot
  /debug/pprof/goroutine  → goroutine dump (detect leaks)
  /debug/pprof/trace      → execution trace
```

---

```bash
# ─── PPROF FROM API SERVER ────────────────────────────────────────
# Port-forward to API Server metrics port
kubectl port-forward -n kube-system \
  pod/kube-apiserver-master-1 6443:6443

# Capture 30-second CPU profile
curl -sk https://localhost:6443/debug/pprof/profile?seconds=30 \
  --cert /etc/kubernetes/pki/admin.crt \
  --key /etc/kubernetes/pki/admin.key \
  -o cpu-profile.prof

# View profile interactively
go tool pprof cpu-profile.prof
# (pprof) top10
# (pprof) web          ← opens flame graph in browser
# (pprof) list <func>  ← show source with sample counts

# Generate flame graph (requires Brendan Gregg's FlameGraph tool)
go tool pprof -http=:8080 cpu-profile.prof
# Opens browser at localhost:8080 with interactive flame graph

# ─── HEAP MEMORY PROFILE ──────────────────────────────────────────
curl -sk https://localhost:6443/debug/pprof/heap \
  --cert /etc/kubernetes/pki/admin.crt \
  --key /etc/kubernetes/pki/admin.key \
  -o heap-profile.prof

go tool pprof heap-profile.prof
# (pprof) top        ← top memory allocators
# (pprof) alloc_space ← by total allocations
# (pprof) inuse_space ← by currently held memory

# ─── GOROUTINE LEAK DETECTION ─────────────────────────────────────
# If memory grows indefinitely → goroutine leak common cause
curl -sk https://localhost:6443/debug/pprof/goroutine?debug=2 \
  --cert /etc/kubernetes/pki/admin.crt \
  --key /etc/kubernetes/pki/admin.key \
  -o goroutines.txt

# Count goroutines
grep "^goroutine " goroutines.txt | wc -l
# > 10000 goroutines = likely leak

# Find what's goroutines are doing
grep -A 3 "goroutine " goroutines.txt | \
  grep -v "^--$" | \
  awk '/goroutine/{cmd=""} /\.go/{cmd=$0} /goroutine/{if(cmd) print cmd}' | \
  sort | uniq -c | sort -rn | head -20

# ─── CUSTOM OPERATOR PROFILING ────────────────────────────────────
# controller-runtime automatically exposes pprof if enabled
# In main.go:
import _ "net/http/pprof"
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()

# Or enable via controller-runtime manager options:
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    PprofBindAddress: ":8082",   # exposes pprof on port 8082
})

# Port-forward to operator
kubectl port-forward -n my-operator deployment/my-operator 8082:8082

# Profile operator
curl http://localhost:8082/debug/pprof/profile?seconds=30 \
  -o operator-cpu.prof
go tool pprof -http=:9090 operator-cpu.prof

# ─── NODE-LEVEL CPU PROFILING ─────────────────────────────────────
# For node CPU saturation (not Go process)

# perf on node (via kubectl debug node)
kubectl debug node/worker-2 -it --image=ubuntu -- bash
# Inside: access host at /proc

# Install perf
apt-get install -y linux-tools-generic

# Record 30-second system-wide CPU profile
perf record -F 99 -a -g -- sleep 30
perf script > perf.out

# Generate flame graph (FlameGraph toolkit)
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl perf.out > perf.folded
./FlameGraph/flamegraph.pl perf.folded > flamegraph.svg
# Copy out and open in browser

# Quick top-N functions consuming CPU
perf report --stdio --no-children | head -30
```

---

---

# 14.17 Incident Runbooks — Correlated Multi-Signal Diagnosis

## 🔴 Advanced

### The Correlated Diagnosis Model

```
MULTI-SIGNAL CORRELATION
═══════════════════════════════════════════════════════════════

Single-signal debugging: "Pod X is CrashLoopBackOff"
Multi-signal debugging:  "API error rate spiked at 14:32.
                          Pod restarts began at 14:31.
                          Node memory pressure event at 14:30.
                          Deploy happened at 14:29."
                          ↑ Root cause: deploy caused memory pressure
                            which evicted pods which caused API errors

THE 5-MINUTE INCIDENT TRIAGE:
  T+0:00  What is the user-visible impact? (error rate, latency, downtime)
  T+0:30  When did it start? (Grafana: zoom to incident window)
  T+1:00  What changed? (deploys, config changes, cronjobs, traffic spike)
  T+2:00  What scope? (one pod, one node, one namespace, cluster-wide?)
  T+3:00  First signal in which system? (app logs, K8s events, node metrics)
  T+4:00  Correlate signals to build causal chain
  T+5:00  Apply fix or rollback, verify recovery
```

---

### Runbook: Application Error Rate Spike

```bash
# ══════════════════════════════════════════════════════
# RUNBOOK: APPLICATION ERROR RATE > 1% (P1 INCIDENT)
# ══════════════════════════════════════════════════════

# SIGNAL 1: Grafana alert fires
# HTTP 5xx rate > 1% for service "api" in production

# STEP 1: Confirm impact and scope
kubectl get pods -n production -l app=api
# NAME              READY   STATUS             RESTARTS   AGE
# api-abc           0/1     CrashLoopBackOff   15         45m  ← unhealthy
# api-def           1/1     Running            0          45m
# api-ghi           0/1     CrashLoopBackOff   12         45m

# STEP 2: Correlate with recent changes
# Check recent Kubernetes events
kubectl get events -n production \
  --sort-by='.lastTimestamp' | tail -30
# 45m: Normal   ScalingReplicaSet  Scaled up to 5
# 44m: Normal   Pulled             Pulled image myapp:v2.1.0  ← new deploy!
# 43m: Warning  BackOff            Back-off restarting failed container
# 43m: Warning  OOMKilling         killing process 1234 with memory 1Gi

# TIMELINE RECONSTRUCTED:
# 44 min ago: Deploy of v2.1.0 started
# 43 min ago: New pods crashing (OOMKilled)
# 43 min ago: Old pods still running but now overloaded (2 pods serving 5x traffic)
# Now: error rate high because pods are insufficient

# STEP 3: Confirm root cause
kubectl logs api-abc -n production --previous | tail -30
# java.lang.OutOfMemoryError: Java heap space    ← OOM in new version

kubectl describe pod api-abc -n production | grep -A 3 "Last State"
# Last State: Terminated  Reason: OOMKilled  Exit Code: 137

# STEP 4: Check if rollback is safe
kubectl rollout history deployment/api -n production
# REVISION  CHANGE-CAUSE
# 5         v2.0.5 (previous stable)
# 6         v2.1.0 (current — crashing)

# STEP 5: Immediate mitigation — rollback
kubectl rollout undo deployment/api -n production
# Rolled back to revision 5

# Monitor recovery
kubectl rollout status deployment/api -n production
kubectl get pods -n production -l app=api -w
# Watch READY change from 0/1 to 1/1

# Verify error rate dropping in Grafana

# STEP 6: Document and post-mortem
# Root cause: v2.1.0 introduced a memory leak in Java heap
# Detection: 3 minutes (alert fired 3 min after deploy)
# MTTR: 8 minutes (rollback + pod stabilization)
# Action items:
#   1. Add memory limit increase to v2.1.0 PR (double heap size)
#   2. Add load test with memory profiling to CI pipeline
#   3. Improve canary strategy (10% traffic before full rollout)
```

---

### Runbook: Cluster-Wide Pod Creation Failure

```bash
# ══════════════════════════════════════════════════════
# RUNBOOK: ALL NEW POD CREATES FAILING (P0 INCIDENT)
# ══════════════════════════════════════════════════════

# SIGNAL: Multiple teams report deployments stuck, rollouts not progressing
# SIGNAL: Kubernetes operators paged

# STEP 1: Confirm scope is cluster-wide
kubectl run test-pod --image=nginx --restart=Never
# Error from server (InternalError): Internal error occurred:
# failed calling webhook "validate.gatekeeper.sh": ...connection refused

# CONFIRMED: Webhook outage (see 14.13)

# Check all namespaces
kubectl get pods -A | grep -v Running | grep -v Completed | wc -l
# 47 pods not running

# STEP 2: Identify the failing webhook
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Find which one is set to Fail (dangerous)
for wh in $(kubectl get validatingwebhookconfigurations -o name); do
  policy=$(kubectl get $wh \
    -o jsonpath='{.webhooks[*].failurePolicy}')
  echo "$wh: $policy"
done
# validatingwebhookconfiguration.admissionregistration.k8s.io/gatekeeper: Fail Fail

# STEP 3: Check webhook server pods
kubectl get pods -n gatekeeper-system
# gatekeeper-controller-0   0/1   Pending   0   15m  ← all down!
# gatekeeper-controller-1   0/1   Pending   0   15m

# Why Pending? Events:
kubectl describe pod gatekeeper-controller-0 -n gatekeeper-system
# Events: FailedScheduling: 0/3 nodes available: 3 Insufficient memory
# → Nodes full! Gatekeeper needs memory to start but nodes are full

# STEP 4: Emergency — change failurePolicy to Ignore
kubectl patch validatingwebhookconfiguration \
  gatekeeper-validating-webhook-configuration \
  --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"},
    {"op":"replace","path":"/webhooks/1/failurePolicy","value":"Ignore"}
  ]'

# STEP 5: Verify pod creation works now
kubectl run test-pod --image=nginx --restart=Never
# pod/test-pod created  ← success!
kubectl delete pod test-pod

# STEP 6: Fix underlying cause (node memory pressure)
kubectl top nodes
# NAME       CPU   MEMORY
# worker-1   45%   94%   ← near full!
# worker-2   38%   91%
# worker-3   41%   89%

# Cordon all nodes and investigate what's consuming memory
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount' | \
  tail -20    # Find high-restart pods (memory leaks)

# Scale down non-critical workloads to free memory
kubectl scale deployment dev-tools -n development --replicas=0
kubectl scale deployment batch-workers -n production --replicas=2  # was 10

# Gatekeeper pods can now schedule
kubectl get pods -n gatekeeper-system -w
# gatekeeper-controller-0   1/1   Running   0   2m

# STEP 7: Restore failurePolicy to Fail
kubectl patch validatingwebhookconfiguration \
  gatekeeper-validating-webhook-configuration \
  --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/failurePolicy","value":"Fail"},
    {"op":"replace","path":"/webhooks/1/failurePolicy","value":"Fail"}
  ]'

# Verify webhook is working
kubectl run test-pod --image=nginx --restart=Never
kubectl delete pod test-pod
```

---

### Runbook: Node NotReady Cascading Failure

```bash
# ══════════════════════════════════════════════════════
# RUNBOOK: MULTIPLE NODES NotReady, PODS EVICTING
# ══════════════════════════════════════════════════════

# SIGNAL: Multiple PagerDuty alerts: nodes NotReady + pod error rates

# STEP 1: Assess scope
kubectl get nodes
# NAME       STATUS     ROLES
# worker-1   Ready      <none>    ← OK
# worker-2   NotReady   <none>    ← DOWN
# worker-3   NotReady   <none>    ← DOWN
# worker-4   Ready      <none>    ← OK

# 2 of 4 nodes down = 50% capacity loss

# STEP 2: What's happening to pods that were on those nodes?
kubectl get pods -A \
  --field-selector=spec.nodeName=worker-2 | grep -v Running
# production/api-abc: Terminating (5 min)   ← stuck terminating
# production/db-xyz: Terminating (5 min)    ← stuck (PV issue?)

# Pods stuck Terminating because node is gone (kubelet can't send completion)
# They'll auto-delete after node is NotReady for 300s (5 min)

# STEP 3: Are pods rescheduling?
kubectl get pods -A | grep Pending
# production/api-rescheduled   Pending   2m   ← waiting for a node

kubectl describe pod api-rescheduled -n production | grep -A 5 "Events"
# FailedScheduling: 0/2 nodes available: 2 Insufficient cpu
# → Remaining nodes (worker-1, worker-4) are full!

# STEP 4: Check if CA is adding nodes
kubectl logs -n kube-system deployment/cluster-autoscaler | tail -20
# "Scale up triggered, adding 2 nodes to node group general"
# "Waiting for nodes..." (nodes still launching — 2-4 min)

# STEP 5: Immediate relief — check if we can free capacity
# Any low-priority pods we can evict?
kubectl get pods -A -o json | \
  jq '.items[] | select(.spec.priorityClassName == "low-priority") |
      .metadata.namespace + "/" + .metadata.name'

# Scale down non-critical
kubectl scale deployment -n development --all --replicas=0

# STEP 6: Investigate why nodes went down
# Check AWS console: were EC2 instances terminated? Status checks failing?
aws ec2 describe-instances \
  --filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Status:StateReason.Message}'

# Check node logs before death (if node still partially accessible)
kubectl logs -n kube-system daemonset/aws-node | grep worker-2

# Look for spot interruption events (common cause)
aws ec2 describe-spot-instance-requests \
  --filters "Name=state,Values=cancelled" \
  --query 'SpotInstanceRequests[?CreateTime>=`2024-01-15T14:00:00`]'
# If spot interruptions: those nodes were reclaimed by AWS

# STEP 7: Once new nodes are ready, verify recovery
kubectl get nodes
# All 4+ Ready

kubectl get pods -A | grep -v -E "Running|Completed"
# Should be empty (or only expected pending pods)

# STEP 8: Post-incident — prevent recurrence
# If spot interruptions caused this: ensure on-demand baseline
# Karpenter: set minValues for on-demand in NodePool
# CA: use --balance-similar-node-groups with mixed instance policies
```

---

### The Incident Diagnosis Checklist

```
UNIVERSAL INCIDENT CHECKLIST
═══════════════════════════════════════════════════════════════

INITIAL TRIAGE (< 5 minutes):
  □ What is user-visible impact? (error rate, latency, total downtime)
  □ When did it start exactly? (check Grafana, correlate with deploys)
  □ What is the scope? (one pod / namespace / node / cluster-wide)
  □ What changed recently? (kubectl rollout history, Argo CD history, Terraform)
  □ Is this a known issue? (check runbook, past incidents, Slack history)

EVIDENCE COLLECTION (next 5 minutes):
  □ kubectl get events -A --sort-by=.lastTimestamp | tail -50
  □ kubectl get pods -A | grep -v Running
  □ kubectl get nodes
  □ kubectl top nodes && kubectl top pods -A --sort-by=memory
  □ Grafana: error rate, latency, pod restarts, node CPU/memory
  □ Relevant app logs: kubectl logs -n <ns> -l app=<x> --tail=100

MITIGATION OPTIONS (ordered by risk):
  □ Rollback deployment: kubectl rollout undo (low risk, fast)
  □ Scale up replicas: kubectl scale (low risk, temporary)
  □ Cordon broken node: kubectl cordon (low risk)
  □ Change webhook failurePolicy: Ignore (medium risk, removes enforcement)
  □ Delete stuck resources: kubectl delete --force (high risk, data loss)
  □ Restart control plane components (extreme, cluster instability risk)

COMMUNICATION:
  □ Acknowledge alert within SLA
  □ Status page update (if external-facing)
  □ Stakeholder update every 15 minutes
  □ Declare resolved only after metrics confirm recovery

POST-INCIDENT:
  □ 5-why root cause analysis
  □ Timeline: detection time, MTTR, impact duration
  □ Action items: prevent recurrence
  □ Runbook updated with new scenario
  □ Alert improved (faster detection next time)
```

---

# Quick Reference — Category 14 Cheat Sheet

| Symptom | First Command | Likely Cause |
|---------|---------------|--------------|
| Pod Pending | `kubectl describe pod → Events` | No resources, taint, affinity, AZ PVC mismatch |
| ImagePullBackOff | `kubectl describe pod → Events` | Wrong image tag, missing imagePullSecret, rate limit |
| CrashLoopBackOff | `kubectl logs --previous` | App error, OOM, missing config/secret |
| Exit 137 | `kubectl describe pod` | OOMKilled — increase memory limit |
| Exit 143 | `kubectl logs --previous` | SIGTERM received — why is it being stopped? |
| Running 0/1 | `kubectl describe pod → Conditions` | Readiness probe failing |
| Stuck Terminating | `kubectl get pod -o json | jq .metadata.finalizers` | Finalizer not removed |
| Service unreachable | `kubectl get endpoints <svc>` | No endpoints (selector mismatch / pods NotReady) |
| DNS SERVFAIL | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | CoreDNS down |
| PVC Pending | `kubectl describe pvc` | No matching PV, CSI error, wrong StorageClass |
| Node NotReady | `kubectl describe node → Conditions` | kubelet crash, network partition, pressure |
| Forbidden error | `kubectl auth can-i --as=<user>` | Missing RoleBinding or wrong subject |
| All pod creates fail | `kubectl run test --image=nginx` | Webhook outage (change failurePolicy to Ignore) |
| CA not scaling up | `kubectl logs cluster-autoscaler | grep "cannot"` | Max size, unsatisfiable constraints, IAM |
| CA not scaling down | `kubectl logs cluster-autoscaler | grep "not removing"` | PDB, safe-to-evict:false, local storage |
| High CPU throttle | Prometheus: `cfs_throttled_periods_total` | CPU limit too low |
| Memory growing | `kubectl top pod -w` | Memory leak — profile with pprof heap |

---

## Key Numbers for Troubleshooting

| Fact | Value |
|------|-------|
| Events expire after | ~1 hour |
| Default pod termination grace period | 30 seconds |
| Node NotReady → pods auto-deleted after | 300 seconds (5 min) |
| Webhook max timeout | 30 seconds |
| CA scale-down check interval | 10 seconds |
| CA scale-down-unneeded-time | 10 minutes |
| CrashLoopBackOff max backoff | 5 minutes |
| Exit code OOMKill | 137 (128+9) |
| Exit code SIGTERM | 143 (128+15) |
| Exit code segfault | 139 (128+11) |
| conntrack table full impact | New connections silently dropped |
| pprof CPU sample default | 30 seconds |
| API Server typical response time | < 100ms (warn if > 500ms) |
| etcd DB size before performance degrades | > 2 GB |

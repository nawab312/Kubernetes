# Kubernetes Interview Mastery
# CATEGORY 8: OBSERVABILITY

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Deep Answer → Gotchas → Interview Q&A → Connections.
> ⚠️ = High priority, frequently asked, commonly misunderstood.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 8.1 | Logs — kubectl logs, log aggregation basics | 🟢 Beginner |
| 8.2 | Liveness, Readiness, Startup Probes ⚠️ | 🟢 Beginner |
| 8.3 | Events — cluster events, how to read them | 🟢 Beginner |
| 8.4 | Metrics — kube-state-metrics, metrics-server | 🟡 Intermediate |
| 8.5 | Prometheus + Grafana stack on Kubernetes | 🟡 Intermediate |
| 8.6 | Distributed tracing (Jaeger, OpenTelemetry) | 🟡 Intermediate |
| 8.7 | Pod conditions & debugging lifecycle ⚠️ | 🟡 Intermediate |
| 8.8 | Custom metrics for HPA (Prometheus adapter) | 🔴 Advanced |
| 8.9 | Alerting — PrometheusRules, Alertmanager routing | 🔴 Advanced |
| 8.10 | eBPF-based observability (Hubble/Cilium, Pixie) | 🔴 Advanced |
| 8.11 | SLO-based alerting in Kubernetes | 🔴 Advanced |

---

# 8.1 Logs — kubectl logs, Log Aggregation Basics

## 🟢 Beginner

### What it is in simple terms

Container logs in Kubernetes are **stdout/stderr streams** written by processes inside containers. Kubernetes captures these via the container runtime and stores them on the node as files. `kubectl logs` reads those files through the API Server. In production, a log aggregation system (Fluentd, Fluent Bit, Loki) ships logs to a central backend before node storage fills up.

---

### How Container Logs Work

```
LOG FLOW IN KUBERNETES
═══════════════════════════════════════════════════════════════

Container process
  │  writes to stdout/stderr
  ▼
Container runtime (containerd)
  │  captures and writes to log file
  ▼
Node filesystem: /var/log/pods/<namespace>_<pod>_<uid>/<container>/0.log
                 /var/log/containers/<pod>_<namespace>_<container>-<id>.log
                   (symlink to above)
  │
  ├── kubectl logs → API Server → kubelet → reads log file → streams to client
  │
  └── Log agent (Fluent Bit DaemonSet)
        reads /var/log/pods/**/*.log
        parses + enriches with pod metadata
        ships to: Loki / Elasticsearch / CloudWatch

LOG ROTATION:
  kubelet rotates log files when they exceed containerLogMaxSize (default: 10Mi)
  Keeps last containerLogMaxFiles (default: 5) rotated files
  Total per container: ~50Mi on disk
  Older logs DISAPPEAR after rotation unless shipped to aggregator
  This is why log aggregation is essential for production.

LOG FORMAT:
  /var/log/pods/.../0.log content:
  2024-01-15T10:30:00.123456789Z stdout F {"level":"info","msg":"request received"}
                                   │     │ └── log line (F = full line, P = partial)
                                   │     └── stream (stdout or stderr)
                                   └── RFC3339Nano timestamp added by runtime
```

---

### kubectl logs Commands

```bash
# Basic log retrieval
kubectl logs pod-name -n production
kubectl logs pod-name -n production -c container-name   # specific container
kubectl logs pod-name -n production --all-containers     # all containers in pod

# Stream logs in real-time (like tail -f)
kubectl logs -f pod-name -n production
kubectl logs -f deployment/my-api -n production          # follows ALL pods in deployment

# Show last N lines
kubectl logs --tail=100 pod-name -n production
kubectl logs --tail=50 -f pod-name -n production         # last 50 then stream

# Logs since a time window
kubectl logs --since=1h pod-name -n production           # last 1 hour
kubectl logs --since=30m pod-name -n production          # last 30 minutes
kubectl logs --since-time=2024-01-15T10:00:00Z pod-name  # since specific time

# Logs from previous (crashed) container instance
kubectl logs pod-name -n production --previous
kubectl logs pod-name -n production -p                   # shorthand for --previous
# Essential for debugging CrashLoopBackOff — see what crashed

# Add timestamps to log output
kubectl logs pod-name -n production --timestamps

# Follow logs for all pods matching a label selector
kubectl logs -f -l app=my-api -n production --max-log-requests=10
# --max-log-requests: limits parallel log connections (default: 5)

# Practical debugging: get logs from all pods in a failed deployment
kubectl get pods -n production -l app=my-api \
  -o jsonpath='{.items[*].metadata.name}' | \
  xargs -I {} kubectl logs {} -n production --tail=50

# Using stern (better multi-pod log tailing)
# stern "my-api" -n production              # all pods matching pattern
# stern "my-api" -n production --since 1h  # with time window
# stern . -n production                    # ALL pods in namespace

# Using kubetail
# kubetail my-api -n production
```

---

### Log Aggregation Architecture

```
LOG AGGREGATION STACK OPTIONS
═══════════════════════════════════════════════════════════════

OPTION A: EFK (Elasticsearch + Fluentd/Fluent Bit + Kibana)
  Fluent Bit DaemonSet → Elasticsearch → Kibana
  Full-text search, complex queries
  High operational complexity and cost
  Good for: compliance, long retention, complex search

OPTION B: PLG (Promtail/Fluent Bit + Loki + Grafana)
  Fluent Bit DaemonSet → Loki → Grafana
  Log-like Prometheus (label-based, not full-text indexed)
  Much cheaper (only index labels, store logs compressed)
  Good for: K8s-native, cost-conscious, Prometheus users
  Query language: LogQL (similar to PromQL)
  RECOMMENDED for most K8s deployments

OPTION C: AWS CloudWatch Logs (EKS)
  Fluent Bit DaemonSet (aws-for-fluent-bit) → CloudWatch Logs
  Managed, no infrastructure to operate
  Log groups per namespace or per pod
  Good for: EKS, AWS-first orgs, low ops overhead

OPTION D: Datadog / Splunk / Dynatrace
  Vendor DaemonSet agent → cloud backend
  Full observability platform (logs + metrics + traces)
  High cost, low ops overhead
```

---

### Fluent Bit DaemonSet (Log Forwarder)

```yaml
# Fluent Bit reads node logs and ships to Loki
# Deployed as DaemonSet — one per node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - operator: Exists              # run on ALL nodes (even tainted)
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 256Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc/fluent-bit.conf
          subPath: fluent-bit.conf
      volumes:
      - name: varlog
        hostPath:
          path: /var/log              # read node log files
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluent-bit-config

---
# Fluent Bit ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush           5
        Daemon          Off
        Log_Level       info

    [INPUT]
        Name            tail
        Path            /var/log/pods/*/*/*.log
        Parser          docker
        Tag             kube.*
        Refresh_Interval 5
        Mem_Buf_Limit   50MB
        Skip_Long_Lines On

    [FILTER]
        Name            kubernetes
        Match           kube.*
        Kube_URL        https://kubernetes.default.svc:443
        Kube_CA_File    /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log       On          # merge JSON log lines into structured fields
        K8S-Logging.Parser  On      # use parser annotations on pods
        Labels          On          # include pod labels in log metadata
        Annotations     Off

    [OUTPUT]
        Name            loki
        Match           kube.*
        Host            loki.logging.svc.cluster.local
        Port            3100
        Labels          job=fluent-bit,namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name'],container=$kubernetes['container_name']
        Auto_kubernetes_labels On
```

---

### LogQL Queries (Loki)

```bash
# LogQL — query language for Loki (similar to PromQL)

# View all logs from a namespace
{namespace="production"}

# Filter by pod name pattern
{namespace="production", pod=~"my-api.*"}

# Filter logs containing "error"
{namespace="production"} |= "error"

# Parse JSON log lines and filter on field
{namespace="production"} | json | level="error"

# Count error rate over time
count_over_time({namespace="production"} |= "error" [5m])

# Top 10 endpoints by request count
{namespace="production", container="api"}
  | json
  | line_format "{{.path}}"
  | count_over_time([1m])
```

---

### 🎤 Short Crisp Interview Answer

> *"Container logs are stdout/stderr captured by the container runtime and stored as files on the node at /var/log/pods. kubectl logs reads these files through the API Server. Log rotation happens when files exceed 10Mi (default), keeping 5 files — so older logs disappear without aggregation. In production, Fluent Bit runs as a DaemonSet, reads node log files, enriches with pod metadata, and ships to a backend. The most common stacks are PLG (Fluent Bit + Loki + Grafana) for cost-efficient K8s-native logging, or EFK for full-text search, or CloudWatch on EKS. The critical kubectl logs flags are --previous for seeing crashed container logs, --since for time-windowed retrieval, and -f for streaming."*

---

### ⚠️ Gotchas

1. **Logs disappear after pod termination without aggregation** — once a pod is deleted (not just restarted), its log files are cleaned up. You need log aggregation to retain post-mortem logs.
2. **--previous only works if the container restarted, not if the pod was deleted** — `kubectl logs --previous` shows the last terminated container instance, but only if that same pod object still exists. If the pod was deleted and rescheduled, it's a new pod — `--previous` shows nothing.
3. **Multi-line logs break per-line shipping** — Java stack traces and Python tracebacks span multiple lines. Without multi-line parser configuration in Fluent Bit, each line becomes a separate log entry, making traces unreadable.
4. **kubectl logs -f on Deployments follows one pod** — `kubectl logs -f deployment/myapp` picks one pod. For multi-pod tailing use stern or `-l` label selector with `--max-log-requests`.

---

---

# ⚠️ 8.2 Liveness, Readiness, Startup Probes

## 🟢 Beginner — HIGH PRIORITY

### What it is in simple terms

Probes are **health checks that kubelet runs against containers** to determine if they're alive, ready to receive traffic, or still starting up. Getting probes wrong is one of the most common causes of production outages — either missed restarts of hung containers, or unnecessary removal of healthy pods from load balancers.

---

### Three Probe Types — What Each Does

```
THREE PROBES — DISTINCT PURPOSES, DISTINCT CONSEQUENCES
═══════════════════════════════════════════════════════════════

LIVENESS PROBE — "Is the container still alive?"
  Question: Is the process in a recoverable state?
  Failure action: RESTART the container (kill + recreate)
  Use when: App can get into deadlock/zombie state that can't
            self-recover — restart is the only fix
  Examples: HTTP server stops accepting connections,
            goroutine deadlock, memory leak causing hang
  NOT for: slow startup (use startupProbe instead)
  NOT for: dependency unavailability (restart won't help if DB is down)

READINESS PROBE — "Is the container ready to receive traffic?"
  Question: Is the app ready to serve requests right now?
  Failure action: REMOVE pod from Service endpoints
                  (pod stays running, just gets no traffic)
  Use when: App needs time to warm up (cache load, DB connect)
            App wants to shed load temporarily
            Rolling update: don't send traffic to unready new pods
  NOT for: permanent failures (liveness is better then)
  Critical for: zero-downtime deployments

STARTUP PROBE — "Has the container finished starting up?"
  Question: Has the app completed its initialization phase?
  Failure action: RESTART the container if startup takes too long
  Use when: App has slow or variable startup time (JVM, DB migrations)
  Effect:   DISABLES liveness probe until startupProbe passes first
  Without:  Liveness probe fires during slow startup → restart loop
  With:     Liveness probe only activates AFTER startup is confirmed

PROBE LIFECYCLE INTERACTION:
  Pod created → startupProbe runs (if configured)
    startupProbe PASSES → livenessProbe + readinessProbe activate
    startupProbe FAILS (threshold exceeded) → container RESTARTED

  startupProbe not configured → livenessProbe runs immediately
    livenessProbe FAILS → container RESTARTED
    readinessProbe FAILS → pod removed from endpoints (not restarted)
```

---

### Three Probe Mechanisms

```
PROBE IMPLEMENTATION OPTIONS:
═══════════════════════════════════════════════════════════════

1. httpGet — HTTP request to container
   Probe succeeds: HTTP status 200-399
   Probe fails:    HTTP status 400+, connection refused, timeout
   Most common for web services

2. exec — Run command inside container
   Probe succeeds: command exits with code 0
   Probe fails:    command exits with non-zero code
   Good for: custom health checks, checking file existence

3. tcpSocket — TCP connection to port
   Probe succeeds: TCP connection established (port open)
   Probe fails:    Connection refused or timeout
   Good for: non-HTTP services (databases, message queues)

4. grpc — gRPC Health Check protocol (K8s 1.24+)
   Probe succeeds: gRPC status SERVING
   Probe fails:    Any other gRPC status
   Good for: gRPC services implementing grpc_health_v1.Health
```

---

### Probe Configuration Fields

```
PROBE TIMING PARAMETERS:
═══════════════════════════════════════════════════════════════

initialDelaySeconds (default: 0):
  Time to wait after container starts before first probe.
  Replace with startupProbe — don't use this for slow starts.
  Use: small value (5-10s) as safety buffer only.

periodSeconds (default: 10):
  How often to run the probe.
  Lower = faster detection, more load
  Typical: 10s for liveness, 5-10s for readiness

timeoutSeconds (default: 1):
  How long to wait for probe response before marking as failure.
  Set this >= your app's P99 health check latency.
  Typical: 3-5s

failureThreshold (default: 3):
  How many consecutive failures before taking action.
  Liveness: 3 failures = restart container
  Readiness: 3 failures = remove from endpoints
  Typical: 3-5 for liveness, 3 for readiness

successThreshold (default: 1):
  How many consecutive successes to go back to healthy.
  Must be 1 for liveness and startup probes.
  Readiness: can be >1 (e.g., 2 = must succeed twice before re-adding to LB)

LIVENESS TOTAL TIME TO RESTART:
  initialDelaySeconds + (failureThreshold × periodSeconds)
  Example: delay=0, threshold=3, period=10 → restart after 30s
  Set appropriately for your SLA.
```

---

### Probe YAML — Production Pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-api
spec:
  containers:
  - name: api
    image: my-api:v1
    ports:
    - containerPort: 8080

    # STARTUP PROBE — handles slow JVM/cache-loading startup
    startupProbe:
      httpGet:
        path: /health/startup      # or /healthz — must be fast, no DB calls
        port: 8080
        scheme: HTTP
      failureThreshold: 30         # 30 failures × 10s = 5 minutes max startup
      periodSeconds: 10
      timeoutSeconds: 3
      # After this passes, liveness and readiness activate

    # LIVENESS PROBE — detect hung/deadlocked process
    livenessProbe:
      httpGet:
        path: /health/live         # MUST NOT check dependencies
        port: 8080                 # returns 200 if process is responsive
        httpHeaders:               # optional: add headers for auth bypass
        - name: X-Health-Check
          value: "liveness"
      initialDelaySeconds: 0       # startupProbe handles slow start
      periodSeconds: 15
      timeoutSeconds: 5
      failureThreshold: 3          # 3 × 15s = 45 seconds before restart
      successThreshold: 1          # must be 1 for liveness

    # READINESS PROBE — control traffic routing
    readinessProbe:
      httpGet:
        path: /health/ready        # CAN check dependencies (DB, cache)
        port: 8080                 # returns 503 if not ready to serve
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1

---
# exec probe example — check file existence
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - "test -f /tmp/healthy && test $(date +%s) -lt $(cat /tmp/healthy)"
    # App writes /tmp/healthy with expiry timestamp
    # Liveness checks file is fresh (updated within window)
  periodSeconds: 30
  failureThreshold: 3

---
# tcpSocket probe — for non-HTTP services
livenessProbe:
  tcpSocket:
    port: 5432                     # just check port is open
  initialDelaySeconds: 30
  periodSeconds: 10

---
# gRPC probe (K8s 1.24+)
livenessProbe:
  grpc:
    port: 9090
    service: "liveness"            # optional: service name in health check
  periodSeconds: 10

---
# Sidecar readiness — main container ready only when sidecar is ready
# Use init container pattern or shared readiness endpoint
```

---

### Health Endpoint Design

```
WHAT LIVENESS AND READINESS ENDPOINTS SHOULD CHECK
═══════════════════════════════════════════════════════════════

/health/startup (startupProbe):
  Check: can the HTTP server bind and respond?
  Do NOT check: database, external services
  Returns: 200 immediately after HTTP server starts
  Goal: signal that startup is complete, not that all deps are ready

/health/live (livenessProbe):
  Check: is the process responsive?
  Check: goroutine/thread pool not exhausted
  Check: critical internal queues not full
  Do NOT check: database connectivity
  Do NOT check: external service availability
  Rationale: restarting the pod won't fix a down database
             But it WILL fix a deadlocked goroutine pool

/health/ready (readinessProbe):
  Check: database connection pool has connections
  Check: cache warmed up (if required for correctness)
  Check: required external services reachable
  Check: leader election completed (StatefulSets)
  Returns: 200 if ready to serve, 503 if not
  Rationale: pod should not receive traffic until fully operational

BAD LIVENESS PROBE PATTERN (VERY COMMON MISTAKE):
  livenessProbe:
    httpGet:
      path: /health             # checks DB connectivity
  
  If DB goes down:
    → Liveness probe fails (DB unreachable)
    → Container restarted
    → Container starts up, DB still down
    → Liveness probe fails again
    → Restart loop
    → ALL pods are restarting simultaneously
    → Your entire service is down because of DB connectivity
    
  THIS IS A CASCADING FAILURE CAUSED BY INCORRECT PROBE DESIGN.
  Liveness must ONLY check what a restart can fix.
```

---

### kubectl Probe Debugging

```bash
# View probe configuration on a running pod
kubectl describe pod my-api -n production | grep -A 15 "Liveness\|Readiness\|Startup"
# Liveness:   http-get http://:8080/health/live delay=0s timeout=5s period=15s
#             #success=1 #failure=3
# Readiness:  http-get http://:8080/health/ready delay=0s timeout=3s period=5s
#             #success=1 #failure=3
# Startup:    http-get http://:8080/health/startup delay=0s timeout=3s period=10s
#             #success=1 #failure=30

# Check probe failure events
kubectl describe pod my-api -n production | grep -A 5 Events
# Events:
#   Warning  Unhealthy  30s  kubelet  Liveness probe failed:
#            HTTP probe failed with statuscode: 500
#   Warning  Unhealthy  25s  kubelet  Readiness probe failed:
#            dial tcp 10.244.1.5:8080: connect: connection refused

# Check restart count (indicates liveness probe failures)
kubectl get pod my-api -n production
# NAME      READY   STATUS    RESTARTS   AGE
# my-api    1/1     Running   5          10m    ← 5 restarts = probe issue

# Check if pod is in endpoints (readiness check)
kubectl get endpoints my-api-svc -n production
# NAME        ENDPOINTS                         AGE
# my-api-svc  10.244.1.5:8080,10.244.2.3:8080  5d
# If fewer endpoints than pod count → some pods not ready

# Manually test health endpoints from inside cluster
kubectl run probe-test --image=curlimages/curl --rm -it -- \
  curl -v http://my-api-svc.production.svc.cluster.local:8080/health/ready

# Watch probe events in real-time
kubectl get events -n production --field-selector reason=Unhealthy -w
```

---

### 🎤 Short Crisp Interview Answer

> *"Three probes serve distinct purposes. Liveness restarts the container on failure — use it only for conditions a restart can fix, like a deadlocked thread pool. Readiness removes the pod from Service endpoints on failure — use it for dependency availability like database connectivity. Startup disables liveness until startup completes — essential for JVMs and apps with DB migrations that take variable time to start. The most critical design rule is: liveness probes must NEVER check external dependencies like databases. If the database goes down and liveness checks it, every pod restarts in a loop simultaneously — a self-inflicted outage. The startup probe's failureThreshold × periodSeconds defines the maximum allowed startup time; exceed it and the container is killed."*

---

### ⚠️ Gotchas

1. **Liveness probe checking database = cascading failure** — the most dangerous misconfiguration. If liveness calls a DB endpoint and DB is down, all pods restart repeatedly, amplifying the outage.
2. **No startup probe + slow app + aggressive liveness = restart loop** — without a startup probe, liveness fires immediately. A JVM taking 60 seconds to start with a 30-second liveness timeout = container killed before it ever starts.
3. **Readiness failure doesn't remove pod from StatefulSet headless DNS** — StatefulSet headless service DNS (`pod-0.svc.ns.svc.cluster.local`) still resolves to unready pods. Only Service-backed endpoints (ClusterIP) respect readiness.
4. **timeoutSeconds: 1 (default) causes false failures** — under load, your health endpoint might take 2-3 seconds. Default 1-second timeout = false probe failures → unnecessary restarts. Set `timeoutSeconds: 3-5`.

---

---

# 8.3 Events — Cluster Events, How to Read Them

## 🟢 Beginner

### What it is in simple terms

Kubernetes Events are **cluster-level log entries** that record significant occurrences — pod scheduling, container restarts, probe failures, OOM kills, scale-up actions. They are the first place to look when debugging any cluster issue.

---

### Event Structure

```
EVENT ANATOMY
═══════════════════════════════════════════════════════════════

kubectl describe pod my-pod shows:
  Events:
    Type    Reason             Age    From               Message
    ──────────────────────────────────────────────────────────────
    Normal  Scheduled          5m     default-scheduler  Successfully assigned production/my-pod to worker-2
    Normal  Pulling            5m     kubelet            Pulling image "myapp:v1.2.3"
    Normal  Pulled             4m     kubelet            Successfully pulled image in 30s
    Normal  Created            4m     kubelet            Created container app
    Normal  Started            4m     kubelet            Started container app
    Warning Unhealthy          2m     kubelet            Readiness probe failed: HTTP probe failed with statuscode: 503
    Warning BackOff            1m     kubelet            Back-off restarting failed container app
    Warning OOMKilling         30s    kubelet            Memory limit reached. Killing container app

TYPE:
  Normal  = informational (expected behavior)
  Warning = something wrong or unusual

REASON (common ones to memorize):
  Scheduled:        Pod assigned to a node
  Pulling/Pulled:   Image pull in progress/complete
  Failed:           Image pull failed (bad tag, registry auth issue)
  Created/Started:  Container lifecycle
  Killing:          Container being terminated
  Unhealthy:        Probe failure
  BackOff:          CrashLoopBackOff — container restart delay increasing
  OOMKilling:       Container killed for exceeding memory limit
  FailedMount:      Volume mount failed (PVC not found, secret missing)
  FailedScheduling: Pod cannot be scheduled (no nodes fit)
  Evicted:          Pod evicted from node (resource pressure)
  NodeNotReady:     Node went not-ready while pod was on it
  Preempting:       Higher-priority pod preempting lower-priority pods
  SuccessfulCreate: ReplicaSet created a pod
  SuccessfulDelete: ReplicaSet deleted a pod
  ScalingReplicaSet:HPA changed replica count

FROM:
  default-scheduler: scheduling decisions
  kubelet:           node-level events (probe, container lifecycle)
  replicaset-controller: RS management
  horizontal-pod-autoscaler: HPA scaling events
  cluster-autoscaler: node scaling events
```

---

### kubectl Events Commands

```bash
# View events for a specific pod
kubectl describe pod my-pod -n production
# Events section at the bottom — most useful for debugging

# View ALL events in a namespace
kubectl get events -n production
kubectl get events -n production --sort-by='.lastTimestamp'   # chronological
kubectl get events -n production --sort-by='.metadata.creationTimestamp'

# Filter to Warning events only
kubectl get events -n production --field-selector type=Warning

# Filter by reason
kubectl get events -n production --field-selector reason=OOMKilling
kubectl get events -n production --field-selector reason=FailedScheduling
kubectl get events -n production --field-selector reason=BackOff

# Watch events in real-time
kubectl get events -n production -w
kubectl get events -A -w                    # all namespaces, real-time

# Events for a specific object (pod, deployment, node)
kubectl get events -n production \
  --field-selector involvedObject.name=my-pod

# Events on a node
kubectl get events -n default \
  --field-selector involvedObject.kind=Node,involvedObject.name=worker-2

# Events across ALL namespaces, sorted by time
kubectl get events -A --sort-by='.lastTimestamp' | tail -30

# IMPORTANT: Events expire after ~1 hour by default
# For longer retention: ship to log aggregation (Fluent Bit → Loki)
# kube-state-metrics also exposes event counts as metrics
```

---

### Reading Events for Common Failures

```bash
# SCENARIO: Pod stuck in Pending
kubectl describe pod my-pod -n production
# Events:
#   Warning  FailedScheduling  pod/my-pod:
#            0/3 nodes are available:
#            1 Insufficient cpu,
#            2 node(s) didn't match pod topology spread constraints.
# → Not enough CPU or topology constraint preventing scheduling

# SCENARIO: CrashLoopBackOff
kubectl describe pod my-pod -n production
# Events:
#   Warning  BackOff  kubelet  Back-off restarting failed container
# → Container keeps crashing. Check:
kubectl logs my-pod --previous    # what was the last output before crash?

# SCENARIO: Image pull failure
# Events:
#   Warning  Failed  kubelet  Failed to pull image "myapp:v99":
#            rpc error: code = NotFound
# → Image doesn't exist in registry, wrong tag, or registry auth issue

# SCENARIO: OOMKilled
# Events:
#   Warning  OOMKilling  kubelet  Memory limit reached. Killing container
# → Increase memory limit or fix memory leak

# SCENARIO: Volume mount failure
# Events:
#   Warning  FailedMount  kubelet  Unable to attach or mount volumes:
#            unable to mount configmap: configmap "my-config" not found
# → ConfigMap/Secret referenced in pod doesn't exist in namespace
```

---

### 🎤 Short Crisp Interview Answer

> *"Kubernetes Events are short-lived cluster log entries recording significant occurrences — scheduling decisions, image pulls, container restarts, probe failures, OOM kills. They expire after about 1 hour by default. Events have a Type (Normal or Warning), a Reason (Scheduled, BackOff, OOMKilling, FailedScheduling), and a From field indicating which component generated the event. The first debugging step for any pod issue is `kubectl describe pod` — the Events section at the bottom shows exactly what happened. Common patterns: FailedScheduling means the scheduler couldn't place the pod, BackOff means CrashLoopBackOff, OOMKilling means memory limit exceeded."*

---

### ⚠️ Gotchas

1. **Events expire after ~1 hour** — if a pod crashed overnight and you investigate in the morning, events are gone. Ship events to a log aggregation backend for retention.
2. **Events are deduplicated with a count** — if a probe fails 100 times, you see `Warning Unhealthy (×100)` not 100 separate events. The count helps diagnose frequency.
3. **Events are namespace-scoped** — events for a pod in namespace `production` are in namespace `production`. Events about nodes are in namespace `default`.

---

---

# 8.4 Metrics — kube-state-metrics, metrics-server

## 🟡 Intermediate

### What it is in simple terms

Two distinct metric sources expose Kubernetes state as Prometheus-format metrics. **metrics-server** provides real-time resource usage (CPU/memory) for `kubectl top` and HPA. **kube-state-metrics** exposes the state of Kubernetes objects (how many pods are running, deployment desired vs available replicas) as metrics — a translation of the Kubernetes API into Prometheus.

---

### metrics-server vs kube-state-metrics

```
TWO COMPLETELY DIFFERENT METRIC SOURCES
═══════════════════════════════════════════════════════════════

metrics-server:
  What: Real-time resource USAGE from nodes and pods
  Source: Pulls from kubelet's /metrics/resource endpoint
          Kubelet aggregates cgroups data
  Metrics:
    node_cpu_usage_seconds_total
    node_memory_working_set_bytes
    container_cpu_usage_seconds_total
    container_memory_working_set_bytes
  Stored: In-memory ONLY — no historical data
  Used by: kubectl top pods/nodes, HPA CPU/memory scaling
  API: metrics.k8s.io/v1beta1 (Kubernetes Aggregated API)
  One instance per cluster

kube-state-metrics:
  What: Kubernetes OBJECT STATE as metrics
  Source: Watches Kubernetes API Server (list+watch all objects)
  Metrics:
    kube_pod_status_phase{phase="Running"}
    kube_deployment_status_replicas_available
    kube_deployment_spec_replicas
    kube_node_status_condition
    kube_pod_container_status_restarts_total
    kube_job_status_succeeded
    kube_persistentvolumeclaim_status_phase
    kube_horizontalpodautoscaler_status_current_replicas
    ...hundreds more
  Stored: In Prometheus (with retention)
  Used by: Prometheus alerts, Grafana dashboards, HPA custom metrics
  Does NOT measure resource usage
  Deployed as Deployment (1 replica)

ANALOGY:
  metrics-server  = "how much CPU is this pod using RIGHT NOW?"
  kube-state-metrics = "is this deployment fully available? how many
                        times did this container restart?"
```

---

### metrics-server Installation and Usage

```bash
# Install metrics-server (EKS often has it as an add-on)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# EKS: Install as add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name metrics-server

# Verify metrics-server is working
kubectl top nodes
# NAME       CPU(cores)  CPU%   MEMORY(bytes)  MEMORY%
# worker-1   1250m       15%    4Gi            25%

kubectl top pods -n production
# NAME            CPU(cores)   MEMORY(bytes)
# api-abc123      245m         312Mi
# worker-def456   1200m        896Mi

kubectl top pods -n production --containers    # per-container breakdown
kubectl top pods -A --sort-by=memory           # sort all pods by memory

# metrics-server serves the metrics.k8s.io API
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | python3 -m json.tool
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | python3 -m json.tool

# If metrics-server is down, kubectl top fails:
# Error from server (ServiceUnavailable): the server is currently unable
# to handle the request (get nodes.metrics.k8s.io)
```

---

### kube-state-metrics Key Metrics

```bash
# Install kube-state-metrics (via kube-prometheus-stack Helm chart)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
# Includes: Prometheus, Grafana, kube-state-metrics, node-exporter

# Critical KSM metrics to know for interviews:

# Pod health
kube_pod_status_phase{namespace="production", phase="Running"}
kube_pod_status_phase{namespace="production", phase="Pending"}
kube_pod_status_phase{namespace="production", phase="Failed"}

# Container restarts (key alert metric)
kube_pod_container_status_restarts_total{namespace="production"}

# Deployment availability
kube_deployment_status_replicas_available{namespace="production"}
kube_deployment_spec_replicas{namespace="production"}
# Alert: available < desired for > 5 minutes

# Node readiness
kube_node_status_condition{condition="Ready", status="true"}
kube_node_status_condition{condition="MemoryPressure", status="true"}
kube_node_status_condition{condition="DiskPressure", status="true"}

# PVC status
kube_persistentvolumeclaim_status_phase{phase="Bound"}
kube_persistentvolumeclaim_status_phase{phase="Pending"}

# HPA status
kube_horizontalpodautoscaler_status_current_replicas
kube_horizontalpodautoscaler_spec_max_replicas
kube_horizontalpodautoscaler_spec_min_replicas

# ResourceQuota usage
kube_resourcequota{resource="requests.cpu", type="used"}
kube_resourcequota{resource="requests.cpu", type="hard"}

# Job completion
kube_job_status_succeeded
kube_job_status_failed
kube_job_complete
```

---

### node-exporter — Node-Level Metrics

```bash
# node-exporter runs as DaemonSet — exposes host OS metrics
# Part of kube-prometheus-stack

# Key node metrics:
node_cpu_seconds_total                    # CPU usage
node_memory_MemAvailable_bytes            # available memory
node_disk_io_time_seconds_total           # disk I/O
node_network_transmit_bytes_total         # network TX
node_filesystem_avail_bytes              # disk space
node_load1, node_load5, node_load15      # system load average

# Process count (PID pressure alert)
node_processes_max_processes             # max PIDs (kernel.pid_max)
node_processes_threads                   # current thread count
```

---

### 🎤 Short Crisp Interview Answer

> *"Two completely separate metric systems. metrics-server pulls real-time CPU and memory usage from kubelet's cgroups data and serves it via the metrics.k8s.io API — it's what kubectl top and HPA use for resource-based scaling. It's in-memory only with no history. kube-state-metrics watches the Kubernetes API and translates object state into Prometheus metrics — deployment replicas available vs desired, pod phases, container restart counts, PVC binding status, HPA replica counts. It has no resource usage data. Together with node-exporter for OS-level metrics, these three form the foundation of a Kubernetes observability stack before you add application-specific metrics."*

---

### ⚠️ Gotchas

1. **metrics-server doesn't provide historical data** — `kubectl top` shows current usage only. For trends and capacity planning, you need Prometheus scraping the kubelet metrics endpoint.
2. **kube-state-metrics can create cardinality explosion** — with thousands of pods and labels, the number of metric series multiplies rapidly. Tune `--metric-labels-allowlist` to only include labels that matter.
3. **metrics-server needs the right kubelet flags** — if kubelet's `--authentication-token-webhook` or `--authorization-mode=Webhook` aren't set correctly, metrics-server can't authenticate to kubelet.

---

---

# 8.5 Prometheus + Grafana Stack on Kubernetes

## 🟡 Intermediate

### What it is in simple terms

Prometheus is a **time-series database and monitoring system** that scrapes metrics from HTTP endpoints on a schedule. Grafana is the **visualization layer** that queries Prometheus and renders dashboards. Together they form the de-facto standard Kubernetes observability stack, commonly deployed via the `kube-prometheus-stack` Helm chart.

---

### Prometheus Architecture on Kubernetes

```
PROMETHEUS ON KUBERNETES — FULL STACK
═══════════════════════════════════════════════════════════════

DISCOVERY (how Prometheus finds targets):
  Kubernetes Service Discovery: Prometheus watches K8s API
    → Lists all Pods, Services, Nodes, Endpoints
    → Filters based on annotations/labels (ServiceMonitor/PodMonitor)
    → Gets target IP:port for scraping

SCRAPING (how metrics are collected):
  Prometheus HTTP GET /metrics every scrapeInterval (default: 30s)
  Target must expose Prometheus format:
    # HELP http_requests_total Total HTTP requests
    # TYPE http_requests_total counter
    http_requests_total{method="GET",status="200"} 1547
    http_requests_total{method="POST",status="500"} 3

STORAGE:
  Prometheus TSDB (time-series database) — local to the pod
  Default retention: 15 days, 50GB
  Compaction: data blocks compacted over time for efficiency
  For long-term storage: Thanos, Cortex, or VictoriaMetrics

QUERY (PromQL):
  Prometheus exposes HTTP API for PromQL queries
  Grafana, Alertmanager, and HPA adapter query this API

ALERTING:
  PrometheusRule CRDs define alert expressions
  Prometheus evaluates every evaluationInterval (default: 1m)
  Fired alerts sent to Alertmanager
  Alertmanager: deduplication, grouping, routing to PagerDuty/Slack

COMPONENTS OF kube-prometheus-stack:
  prometheus-operator:   CRD controller for Prometheus, Alertmanager,
                         ServiceMonitor, PodMonitor, PrometheusRule
  Prometheus:            Metrics scraper and TSDB
  Alertmanager:          Alert routing and deduplication
  Grafana:               Dashboards and visualization
  kube-state-metrics:    K8s object state → Prometheus metrics
  node-exporter:         Node OS metrics → Prometheus metrics
  blackbox-exporter:     External endpoint probing (HTTP, TCP, DNS)
```

---

### ServiceMonitor — How Prometheus Discovers Targets

```yaml
# ServiceMonitor tells Prometheus to scrape a Service's endpoints
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-api-monitor
  namespace: production
  labels:
    release: kube-prometheus-stack  # must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-api                   # Service must have this label
  namespaceSelector:
    matchNames:
    - production
    - staging
  endpoints:
  - port: http-metrics              # port name in Service spec
    path: /metrics                  # metrics endpoint path
    interval: 30s                   # scrape every 30 seconds
    scrapeTimeout: 10s
    scheme: http
    # For TLS:
    # scheme: https
    # tlsConfig:
    #   caFile: /etc/ssl/certs/ca.crt

---
# The Service being monitored must have the matching label
apiVersion: v1
kind: Service
metadata:
  name: my-api
  namespace: production
  labels:
    app: my-api                     # matches ServiceMonitor selector
spec:
  ports:
  - name: http-metrics              # must match ServiceMonitor endpoint port name
    port: 9090
    targetPort: 9090
  selector:
    app: my-api

---
# PodMonitor — scrape pods directly (no Service needed)
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-worker-monitor
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-worker
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 60s                   # less frequent for batch workers
```

---

### Prometheus Deployment YAML

```yaml
# Prometheus instance (via prometheus-operator)
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2                       # HA: 2 Prometheus instances
  version: v2.48.0
  serviceAccountName: prometheus

  # Which ServiceMonitors to pick up
  serviceMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack

  # Which PrometheusRules to pick up
  ruleSelector:
    matchLabels:
      release: kube-prometheus-stack

  # Storage
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: gp3
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi

  # Retention
  retention: 15d
  retentionSize: 45GB               # also bounded by size

  # Resources
  resources:
    requests:
      memory: 4Gi
      cpu: 500m
    limits:
      memory: 8Gi
      cpu: 2000m

  # Remote write (long-term storage)
  remoteWrite:
  - url: https://thanos-receive.monitoring.svc:19291/api/v1/receive
    # OR: Amazon Managed Prometheus (AMP)
    # url: https://aps-workspaces.us-east-1.amazonaws.com/workspaces/<id>/api/v1/remote_write
    sigv4:
      region: us-east-1
```

---

### Essential PromQL Queries

```promql
# CPU usage percentage per pod
100 * sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m]))
  by (pod) /
  sum(kube_pod_container_resource_requests{namespace="production",resource="cpu"})
  by (pod)

# Memory usage vs limit
container_memory_working_set_bytes{namespace="production"}
  /
container_spec_memory_limit_bytes{namespace="production"}
# → Alert if > 0.85 (85% of limit)

# Container restart rate
rate(kube_pod_container_status_restarts_total{namespace="production"}[15m]) * 60
# → restarts per minute. Alert if > 0.

# Deployment availability ratio
kube_deployment_status_replicas_available / kube_deployment_spec_replicas
# → Alert if < 1 for > 5 minutes

# HTTP error rate (from app custom metrics)
sum(rate(http_requests_total{namespace="production",status=~"5.."}[5m]))
  /
sum(rate(http_requests_total{namespace="production"}[5m]))
# → Alert if > 0.01 (1% error rate)

# P99 latency (from histogram metric)
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{namespace="production"}[5m]))
  by (le, service)
)

# Node memory pressure
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
# → Alert if > 0.85

# PVC usage
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes
# → Alert if > 0.85

# Pod CPU throttling
rate(container_cpu_cfs_throttled_seconds_total[5m])
  /
rate(container_cpu_cfs_periods_total[5m])
# → Alert if > 0.25 (25% throttled — latency impact)
```

---

### 🎤 Short Crisp Interview Answer

> *"Prometheus uses Kubernetes service discovery to find targets via ServiceMonitor and PodMonitor CRDs — the prometheus-operator watches these and configures Prometheus accordingly. Prometheus scrapes each target's /metrics endpoint on a schedule, stores time-series data locally with typically 15-day retention, and evaluates PrometheusRules for alerting. Grafana queries Prometheus via PromQL and renders dashboards. The kube-prometheus-stack Helm chart deploys the full stack: Prometheus, Alertmanager, Grafana, kube-state-metrics, and node-exporter. For long-term retention beyond 15 days, remote write to Thanos, Cortex, or Amazon Managed Prometheus. The most important metrics to alert on: container restart rate, deployment availability ratio, CPU throttling percentage, memory usage vs limit ratio, and HTTP error rate."*

---

### ⚠️ Gotchas

1. **ServiceMonitor labels must match Prometheus serviceMonitorSelector** — if the Prometheus CR's selector doesn't match the ServiceMonitor's labels, Prometheus never discovers those targets. The `release: kube-prometheus-stack` label is the convention.
2. **High cardinality labels in metrics** — adding labels with many unique values (user IDs, UUIDs, request IDs) to metrics creates millions of time series, crashing Prometheus. Only use low-cardinality labels (service, method, status_code).
3. **Prometheus is not distributed by default** — two Prometheus replicas scrape the same targets independently (no sharding). For very large clusters use Thanos or VictoriaMetrics for proper HA with deduplication.

---

---

# 8.6 Distributed Tracing (Jaeger, OpenTelemetry)

## 🟡 Intermediate

### What it is in simple terms

Distributed tracing tracks **a single request as it flows through multiple services**, capturing timing, dependencies, and errors at each hop. When a user request is slow or failing, tracing shows exactly which service or database call is the bottleneck — something impossible to determine from logs or metrics alone.

---

### Tracing Concepts

```
DISTRIBUTED TRACING CONCEPTS
═══════════════════════════════════════════════════════════════

TRACE:
  A complete end-to-end record of one request.
  Has a unique trace ID (e.g., UUID).
  Contains multiple spans.

SPAN:
  A single unit of work within a trace.
  Has: span ID, parent span ID, operation name,
       start timestamp, duration, status, tags, logs.
  Types: root span (no parent), child spans.

CONTEXT PROPAGATION:
  Trace ID + span ID passed between services in HTTP headers:
    W3C Trace Context (modern standard):
      traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
    B3 headers (older, Zipkin-compatible):
      X-B3-TraceId, X-B3-SpanId, X-B3-ParentSpanId
  Without propagation: each service creates a new trace → disconnected.

TRACE EXAMPLE:
  User request to frontend → 150ms total

  [frontend.handleRequest]          0-150ms    (150ms)
    [auth.validateToken]            5-25ms     (20ms)
    [catalog.getProducts]           30-100ms   (70ms)  ← BOTTLENECK
      [catalog.dbQuery]             35-95ms    (60ms)  ← slow DB query
      [cache.get]                   30-32ms    (2ms)
    [recommendation.getPersonalized]105-140ms  (35ms)

  Without tracing: "the API is slow" — no idea why
  With tracing: "catalog.dbQuery takes 60ms — needs an index"
```

---

### OpenTelemetry — The Standard

```
OPENTELEMETRY (OTel) — VENDOR-NEUTRAL STANDARD
═══════════════════════════════════════════════════════════════

OpenTelemetry is a CNCF project providing:
  - APIs: instrument your code (create spans, add attributes)
  - SDKs: language implementations (Go, Java, Python, Node.js, etc.)
  - Collector: receive, process, and export telemetry
  - Protocol: OTLP (OpenTelemetry Protocol)

BEFORE OTel: Vendor lock-in
  Using Jaeger SDK → switching to Zipkin = rewrite all instrumentation
  Using Datadog SDK → switching to Honeycomb = rewrite all instrumentation

WITH OTel:
  Instrument once with OTel SDK → send to OTel Collector
  OTel Collector routes to: Jaeger, Zipkin, Tempo, Datadog, Honeycomb
  Switch backends without changing application code

COMPONENTS:
  Application (OTel SDK):    creates spans, propagates context
  OTel Collector (sidecar or DaemonSet):
                             receives OTLP, processes, exports
  Backend (Jaeger/Tempo):    stores and queries traces
  UI (Jaeger UI/Grafana):    visualizes traces
```

---

### OTel Collector Deployment

```yaml
# OTel Collector as sidecar (per pod, low latency)
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: OTEL_EXPORTER_OTLP_ENDPOINT
      value: "http://localhost:4317"      # sidecar collector gRPC port
    - name: OTEL_SERVICE_NAME
      value: "my-api"
    - name: OTEL_TRACES_SAMPLER
      value: "parentbased_traceidratio"
    - name: OTEL_TRACES_SAMPLER_ARG
      value: "0.1"                       # sample 10% of traces

  - name: otel-collector
    image: otel/opentelemetry-collector-contrib:0.88.0
    args: ["--config=/conf/collector.yaml"]
    ports:
    - containerPort: 4317               # OTLP gRPC
    - containerPort: 4318               # OTLP HTTP
    volumeMounts:
    - name: collector-config
      mountPath: /conf

---
# OTel Collector ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  collector.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:                           # batch before exporting (efficiency)
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        limit_mib: 400
      resource:                        # add K8s metadata to all spans
        attributes:
        - key: k8s.cluster.name
          value: "production-cluster"
          action: upsert
        - key: k8s.node.name
          from_attribute: MY_NODE_NAME
          action: upsert

    exporters:
      jaeger:
        endpoint: jaeger-collector.monitoring.svc:14250
        tls:
          insecure: true
      # OR: Grafana Tempo
      otlp:
        endpoint: tempo.monitoring.svc:4317
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [jaeger]

---
# OTel Operator — auto-instrumentation (no code changes needed!)
# Install: kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

# Define auto-instrumentation for Java apps
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: java-instrumentation
  namespace: production
spec:
  exporter:
    endpoint: http://otel-collector.monitoring.svc:4317
  sampler:
    type: TraceIdRatioBased
    argument: "0.25"             # sample 25%
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.31.0

# Annotate pod to get auto-instrumented (no code changes!)
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-java: "true"
    # OTel operator injects JAVA_TOOL_OPTIONS with javaagent
```

---

### 🎤 Short Crisp Interview Answer

> *"Distributed tracing tracks a request across multiple services by propagating a trace ID in HTTP headers. Each service creates spans that are children of the parent span, recording timing and status. OpenTelemetry is the vendor-neutral standard — you instrument once with the OTel SDK, export to the OTel Collector, and route to any backend: Jaeger, Grafana Tempo, or commercial tools. On Kubernetes, the OTel Operator enables auto-instrumentation via annotations — no code changes needed for Java, Python, and Node.js services. Sampling is critical — sampling 100% of traces at high traffic would overwhelm storage; 1-10% tail-based sampling is typical. The three pillars of observability are logs, metrics, and traces — they're complementary: metrics tell you something is wrong, logs tell you what happened in one service, traces tell you where across the system."*

---

### ⚠️ Gotchas

1. **Context propagation must be enabled end-to-end** — if one service in the chain doesn't propagate trace context (missing the `traceparent` header in outbound calls), the trace breaks and appears as disconnected fragments.
2. **100% sampling at high traffic = trace backend overload** — start with 1-10% head-based sampling, then consider tail-based sampling (sample 100% of error traces, 1% of success traces) for better coverage of failures.
3. **Clock skew breaks trace visualization** — spans from different pods/nodes use their local clocks. If nodes have significant clock drift (> 1ms), spans appear in wrong order or with negative durations. Ensure NTP sync.

---

---

# ⚠️ 8.7 Pod Conditions & Debugging Lifecycle

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Every pod has a set of **Conditions** — boolean flags that track its progression through the lifecycle. Understanding conditions and their failure messages is the foundation of Kubernetes debugging. This topic covers the complete mental model for diagnosing any pod issue.

---

### Pod Phase vs Pod Conditions

```
PHASE AND CONDITIONS — TWO DIFFERENT VIEWS
═══════════════════════════════════════════════════════════════

PHASE (coarse-grained, single value):
  Pending:    Pod accepted, but not all containers running yet
  Running:    Pod bound to node, at least one container running
  Succeeded:  All containers completed successfully (exit 0)
  Failed:     All containers terminated, at least one non-zero exit
  Unknown:    Node unreachable, pod state cannot be determined

⚠️ IMPORTANT: Running phase ≠ pod is ready to serve traffic
  A pod in Running phase can have READY = 0/1 (readiness probe failing)

CONDITIONS (fine-grained, multiple boolean flags):
  PodScheduled:      Has the pod been assigned to a node?
  Initialized:       Have all init containers completed?
  ContainersReady:   Are all containers ready (probes passing)?
  Ready:             Is the pod ready to serve traffic?
                     (Ready = ContainersReady + custom readinessGates)

CONDITION ORDER (sequential requirements):
  PodScheduled → Initialized → ContainersReady → Ready

  If PodScheduled = False: pod is Pending — scheduler can't place it
  If Initialized = False:  init container running or failed
  If ContainersReady = False: container starting, probe failing, or crashed
  If Ready = False:  readiness probe failing (may be running but not in LB)
```

---

### Complete Pod Debugging Decision Tree

```
POD DEBUGGING DECISION TREE
═══════════════════════════════════════════════════════════════

kubectl get pod my-pod
                │
                ├── STATUS: Pending
                │     kubectl describe pod my-pod → Events:
                │     ├── "FailedScheduling: Insufficient cpu"
                │     │     → Nodes don't have enough CPU
                │     │       kubectl top nodes
                │     │       kubectl describe nodes | grep -A 5 "Allocated"
                │     │       Solution: add nodes or reduce requests
                │     │
                │     ├── "FailedScheduling: didn't match node affinity"
                │     │     → nodeAffinity/nodeSelector has no matching nodes
                │     │       kubectl get nodes --show-labels
                │     │       Check affinity rules in pod spec
                │     │
                │     ├── "FailedScheduling: 0/3 nodes have 0 available..."
                │     │     → Taint not tolerated by pod
                │     │       kubectl describe nodes | grep Taints
                │     │
                │     └── No events? → Check RBAC, ResourceQuota
                │           kubectl get resourcequota -n <ns>
                │
                ├── STATUS: Pending > 30s with PodScheduled=True
                │     Pod is scheduled but Init/ContainersReady False
                │     ├── Init container issue
                │     │     kubectl logs my-pod -c init-container-name
                │     │     kubectl describe pod → "Init:Error" or "Init:0/1"
                │     └── Image pull issue
                │           kubectl describe pod → "ImagePullBackOff"
                │           Events: "Failed to pull image"
                │           Check: image name, tag, registry auth
                │
                ├── STATUS: CrashLoopBackOff
                │     Container starts, crashes, K8s backs off restart
                │     kubectl logs my-pod --previous   ← last run's output
                │     kubectl describe pod → "Exit Code: N"
                │     Exit Code 1:   application error (check logs)
                │     Exit Code 137: OOMKilled (increase memory limit)
                │     Exit Code 143: SIGTERM (graceful termination — check why)
                │     Exit Code 1 + empty logs: crash before logging started
                │       → Run container with /bin/sh manually to debug
                │
                ├── STATUS: Running, READY: 0/1
                │     Container running but readiness probe failing
                │     kubectl describe pod → Events: "Readiness probe failed"
                │     kubectl exec -it my-pod -- curl localhost:8080/health/ready
                │     Check: app started? deps available? config correct?
                │
                ├── STATUS: Running, READY: 1/1
                │     Pod appears healthy. Traffic issues?
                │     kubectl get endpoints my-svc   ← is pod in endpoints?
                │     kubectl exec -it debug-pod -- curl my-svc:port
                │     Check: NetworkPolicy, Service selector, port names
                │
                ├── STATUS: Error / OOMKilled
                │     kubectl describe pod → "OOMKilled" or "Exit Code: 137"
                │     kubectl top pod my-pod    ← current memory usage
                │     Solution: increase memory limit or fix leak
                │
                ├── STATUS: Terminating (stuck)
                │     Pod not fully terminating
                │     kubectl describe pod → look for finalizers
                │     kubectl get pod my-pod -o jsonpath='{.metadata.finalizers}'
                │     kubectl patch pod my-pod -p '{"metadata":{"finalizers":[]}}' --type=merge
                │     (removes finalizers — use as last resort)
                │
                └── STATUS: Evicted
                      Node was under memory/disk pressure
                      kubectl get events -n <ns> | grep Evicted
                      kubectl get pods -A | grep Evicted   ← find all
                      kubectl delete pods --field-selector=status.phase=Failed -A
```

---

### kubectl describe pod — Full Breakdown

```bash
# kubectl describe pod output anatomy
kubectl describe pod my-api-abc123 -n production

# Name:             my-api-abc123
# Namespace:        production
# Priority:         0
# Service Account:  my-api-sa
# Node:             worker-2/10.0.1.15   ← which node, node IP
# Start Time:       Mon, 15 Jan 2024 10:00:00 +0000
# Labels:           app=my-api
#                   pod-template-hash=xyz789
# Annotations:      kubectl.kubernetes.io/last-applied-configuration: ...

# Status:           Running   ← pod phase
# IP:               10.244.2.5    ← pod IP (cluster-internal)
# IPs:
#   IP:  10.244.2.5

# Init Containers:
#   db-migrate:
#     State:          Terminated
#     Reason:         Completed    ← init container completed successfully
#     Exit Code:      0

# Containers:
#   api:
#     Image:          my-api:v1.2.3
#     Port:           8080/TCP
#     State:          Running
#     Started:        Mon, 15 Jan 2024 10:00:30 +0000
#     Ready:          True    ← readiness probe passing
#     Restart Count:  0       ← no restarts = healthy
#     Limits:
#       cpu:          1000m
#       memory:       1Gi
#     Requests:
#       cpu:          250m
#       memory:       256Mi
#     Liveness:   http-get http://:8080/health/live delay=0s timeout=5s period=15s #success=1 #failure=3
#     Readiness:  http-get http://:8080/health/ready delay=0s timeout=3s period=5s #success=1 #failure=3
#     Environment:
#       DB_HOST:    postgres.production.svc.cluster.local
#     Mounts:
#       /etc/secrets from db-credentials (ro)

# Conditions:
#   Type              Status
#   Initialized       True      ← init containers done
#   Ready             True      ← pod in Service endpoints
#   ContainersReady   True      ← all probes passing
#   PodScheduled      True      ← assigned to node

# Volumes:
#   db-credentials:
#     Type:    Secret (a volume populated by a Secret)
#     SecretName:  database-credentials

# QoS Class:       Burstable   ← req < lim → Burstable

# Node-Selectors:  <none>
# Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s

# Events:                     ← MOST IMPORTANT FOR DEBUGGING
#   Normal   Scheduled    5m  default-scheduler  Assigned to worker-2
#   Normal   Pulled       5m  kubelet            Image pulled
#   Normal   Started      5m  kubelet            Container started
```

---

### Exit Code Reference

```
CONTAINER EXIT CODES — DIAGNOSTIC GUIDE
═══════════════════════════════════════════════════════════════

Exit Code 0:    Successful termination (Succeeded phase)
Exit Code 1:    General application error (check logs)
Exit Code 2:    Misuse of shell command or scripting error
Exit Code 137:  SIGKILL (9) received = 128 + 9
                Caused by: OOMKill, manual kill, liveness failure
                kubectl describe pod → Reason: OOMKilled
Exit Code 139:  SIGSEGV (11) received = 128 + 11
                Segmentation fault in application
Exit Code 143:  SIGTERM (15) received = 128 + 15
                Graceful termination requested
                Normal for kubectl delete / rolling update
Exit Code 255:  Container image or container definition error
                (exit -1 sometimes shows as 255)

HOW TO CHECK:
kubectl describe pod my-pod | grep "Exit Code"
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

---

### 🎤 Short Crisp Interview Answer

> *"Pod debugging follows a decision tree based on status. Pending means either not scheduled (check events for FailedScheduling — insufficient resources, affinity mismatch, taint not tolerated) or scheduled but init containers running. CrashLoopBackOff means the container keeps crashing — always check `kubectl logs --previous` for the last output before crash, and look at the exit code: 137 is OOMKilled, 143 is SIGTERM. Running but 0/1 Ready means readiness probe is failing — exec into the pod and test the health endpoint manually. The most useful commands in order: kubectl get pod (get status), kubectl describe pod (get conditions and events), kubectl logs --previous (crash output), kubectl exec (interactive debugging). Pod conditions — PodScheduled, Initialized, ContainersReady, Ready — show exactly where in the lifecycle the pod is stuck."*

---

### ⚠️ Gotchas

1. **Running ≠ Ready** — a pod in Running phase with 0/1 Ready is not serving traffic. Services only route to pods with Ready=True. This is the most common confusion.
2. **CrashLoopBackOff hides logs** — containers that crash immediately (in < 1 second) often produce no logs. The initial delay before the first crash might be the only window. Use `--previous` flag and check exit code.
3. **Terminating pods block namespace deletion** — pods stuck in Terminating (usually due to finalizers or hung processes) prevent namespace deletion. Must remove finalizers or force-delete.

---

---

# 8.8 Custom Metrics for HPA (Prometheus Adapter)

## 🔴 Advanced

### What it is in simple terms

The default HPA scales on CPU and memory. **Custom metrics HPA** lets you scale on any Prometheus metric — queue depth, active HTTP connections, request latency, or any business metric. The **Prometheus Adapter** bridges Prometheus and the Kubernetes custom metrics API that HPA queries.

---

### Custom Metrics API Architecture

```
CUSTOM METRICS FLOW
═══════════════════════════════════════════════════════════════

Application → emits: http_active_requests_total metric
Prometheus  → scrapes and stores the metric
                    │
                    ▼
Prometheus Adapter → queries Prometheus every 30s
                   → translates to Kubernetes Custom Metrics API
                   → serves: custom.metrics.k8s.io/v1beta1
                    │
                    ▼
HPA                → queries custom.metrics.k8s.io every 15s
                   → gets: "api-server has 450 active requests"
                   → computes desired replicas
                   → scales Deployment

THREE METRIC APIS AVAILABLE TO HPA:
  metrics.k8s.io         → CPU, memory (served by metrics-server)
  custom.metrics.k8s.io  → object metrics (served by Prometheus adapter)
  external.metrics.k8s.io → external metrics (SQS depth, etc. via KEDA)
```

---

### Prometheus Adapter Configuration

```yaml
# Prometheus Adapter ConfigMap — defines metric mappings
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    # Rule: expose http_requests_per_second per pod
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"          # custom metric name
      metricsQuery: |
        sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m]))
        by (<<.GroupBy>>)
      # Prometheus query executed when HPA polls the metric
      # <<.Series>>, <<.LabelMatchers>>, <<.GroupBy>> are template vars

    # Rule: expose queue depth (Gauge, not counter)
    - seriesQuery: 'app_queue_depth{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "app_queue_depth"
        as: "queue_depth"
      metricsQuery: 'avg(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)'

---
# HPA using custom metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-api
  minReplicas: 3
  maxReplicas: 50

  metrics:
  # Scale on CPU (standard)
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # Scale on custom metric: active HTTP requests per pod
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second    # must match adapter config name
      target:
        type: AverageValue
        averageValue: "100"               # scale when > 100 req/s per pod

  # Scale on external metric (SQS queue via KEDA)
  - type: External
    external:
      metric:
        name: sqs_queue_depth
        selector:
          matchLabels:
            queue: orders-queue
      target:
        type: AverageValue
        averageValue: "50"               # scale when > 50 messages per pod

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # 5 min cooldown before scale-down
      policies:
      - type: Percent
        value: 10                        # scale down max 10% per minute
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0     # scale up immediately
      policies:
      - type: Percent
        value: 100                      # scale up up to 100% per minute
        periodSeconds: 60
```

---

### Debugging Custom Metrics HPA

```bash
# Install Prometheus Adapter
helm install prometheus-adapter \
  prometheus-community/prometheus-adapter \
  -n monitoring \
  --set prometheus.url=http://prometheus-kube-prometheus-prometheus.monitoring.svc

# Verify custom metrics API is serving
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | python3 -m json.tool

# List available custom metrics
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | \
  python3 -m json.tool | grep '"name"'

# Check the metric value for a specific pod
kubectl get --raw \
  "/apis/custom.metrics.k8s.io/v1beta1/namespaces/production/pods/*/http_requests_per_second" | \
  python3 -m json.tool
# {
#   "kind": "MetricValueList",
#   "items": [
#     {
#       "describedObject": {"name": "my-api-abc123"},
#       "metric": {"name": "http_requests_per_second"},
#       "value": "143500m"   ← 143.5 requests/second (milliunits)
#     }
#   ]
# }

# Check HPA status
kubectl describe hpa my-api-hpa -n production
# Metrics:
#   resource cpu on pods (as a percentage of request):  65% (target 70%)
#   pods:    http_requests_per_second:                  143500m (target 100)
# Current replicas:  5
# Desired replicas:  8   ← HPA wants to scale up

# View HPA events
kubectl get events -n production | grep HorizontalPodAutoscaler
```

---

### 🎤 Short Crisp Interview Answer

> *"Custom metrics HPA uses the Prometheus Adapter to bridge Prometheus metrics to the Kubernetes custom.metrics.k8s.io API that HPA queries. You configure the adapter with rules that translate Prometheus queries into named metrics, then reference those metric names in the HPA spec under type: Pods. The HPA polls the custom metrics API every 15 seconds, computes desired replicas as ceil(current × currentValue / targetValue), and scales accordingly. Common use cases: scale on active HTTP connections, request queue depth, or business metrics like orders per minute. For external systems like SQS, KEDA provides the external.metrics.k8s.io API without needing Prometheus."*

---

---

# 8.9 Alerting — PrometheusRules, Alertmanager Routing

## 🔴 Advanced

### What it is in simple terms

**PrometheusRules** define alert expressions evaluated by Prometheus. When an expression fires, Prometheus sends the alert to **Alertmanager**, which handles deduplication, grouping, silencing, and routing to notification channels — Slack, PagerDuty, email, webhooks.

---

### PrometheusRule YAML

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: production-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack    # must match Prometheus ruleSelector
spec:
  groups:
  - name: pod-health
    interval: 30s                     # evaluate these rules every 30s
    rules:

    # Alert: high container restart rate
    - alert: ContainerHighRestartRate
      expr: |
        rate(kube_pod_container_status_restarts_total{
          namespace="production"
        }[15m]) * 60 > 0.5
      for: 5m                         # must fire for 5 minutes to alert
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Container {{ $labels.container }} in pod {{ $labels.pod }} restarting frequently"
        description: "Container restart rate is {{ $value | humanize }}/min for 5+ minutes"
        runbook_url: "https://wiki.internal/runbooks/container-restart"

    # Alert: deployment not fully available
    - alert: DeploymentNotAvailable
      expr: |
        (kube_deployment_status_replicas_available /
         kube_deployment_spec_replicas) < 0.8
      for: 10m
      labels:
        severity: critical
        pagerduty: "true"             # custom label for routing
      annotations:
        summary: "Deployment {{ $labels.deployment }} below 80% availability"
        description: "{{ $value | humanizePercentage }} of replicas available"

    # Alert: OOMKilled container
    - alert: ContainerOOMKilled
      expr: |
        kube_pod_container_status_last_terminated_reason{
          reason="OOMKilled",
          namespace="production"
        } == 1
      for: 0m                        # alert immediately (no for: clause)
      labels:
        severity: warning
      annotations:
        summary: "Container {{ $labels.container }} OOMKilled"
        description: "Container {{ $labels.container }} in pod {{ $labels.pod }} was OOMKilled. Increase memory limit."

  - name: latency
    rules:
    # Alert: P99 latency too high
    - alert: HighP99Latency
      expr: |
        histogram_quantile(0.99,
          sum(rate(http_request_duration_seconds_bucket{
            namespace="production",
            job="my-api"
          }[5m])) by (le, service)
        ) > 2.0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "P99 latency {{ $value | humanizeDuration }} for {{ $labels.service }}"

    # Alert: error rate SLO breach
    - alert: HighErrorRate
      expr: |
        (
          sum(rate(http_requests_total{
            namespace="production",
            status=~"5.."
          }[5m])) by (service)
          /
          sum(rate(http_requests_total{
            namespace="production"
          }[5m])) by (service)
        ) > 0.01
      for: 5m
      labels:
        severity: critical
        pagerduty: "true"
      annotations:
        summary: "Error rate {{ $value | humanizePercentage }} for {{ $labels.service }}"

  - name: node-health
    rules:
    # Alert: node memory pressure
    - alert: NodeMemoryPressure
      expr: kube_node_status_condition{condition="MemoryPressure", status="true"} == 1
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.node }} under memory pressure"

    # Alert: disk filling up
    - alert: NodeDiskFilling
      expr: |
        (1 - (node_filesystem_avail_bytes{mountpoint="/"} /
              node_filesystem_size_bytes{mountpoint="/"})) > 0.85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Node {{ $labels.instance }} disk {{ $value | humanizePercentage }} used"

  - name: recording-rules         # RECORDING RULES — pre-compute expensive queries
    rules:
    - record: job:http_requests_total:rate5m
      expr: sum(rate(http_requests_total[5m])) by (job, service)
      # Pre-computed — dashboards use this instead of re-computing
```

---

### Alertmanager Configuration

```yaml
# Alertmanager ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: "https://hooks.slack.com/services/..."
      pagerduty_url: "https://events.pagerduty.com/v2/enqueue"

    # INHIBIT: suppress lower-severity alerts when higher is firing
    inhibit_rules:
    - source_match:
        severity: critical
      target_match:
        severity: warning
      equal: [service, namespace]  # suppress warnings for same service when critical fires

    # ROUTE: how to route alerts to receivers
    route:
      group_by: [alertname, service, namespace]
      group_wait: 30s               # wait 30s to group related alerts
      group_interval: 5m            # wait 5m before sending next group update
      repeat_interval: 4h           # re-notify every 4h if still firing
      receiver: slack-platform      # default receiver

      routes:
      # Critical alerts with pagerduty label → PagerDuty
      - match:
          severity: critical
          pagerduty: "true"
        receiver: pagerduty-oncall
        continue: false

      # All critical → also Slack
      - match:
          severity: critical
        receiver: slack-critical-alerts
        continue: true              # continue = also match other routes

      # Warnings → team Slack only
      - match:
          severity: warning
          team: platform
        receiver: slack-platform

      # Silence during maintenance window
      # (use kubectl create silence via amtool CLI)

    receivers:
    - name: slack-platform
      slack_configs:
      - channel: "#platform-alerts"
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Namespace:* {{ .Labels.namespace }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}

    - name: pagerduty-oncall
      pagerduty_configs:
      - routing_key: "<pagerduty-routing-key>"
        send_resolved: true
        description: '{{ template "pagerduty.default.description" . }}'
        details:
          namespace: '{{ range .Alerts }}{{ .Labels.namespace }}{{ end }}'
          pod: '{{ range .Alerts }}{{ .Labels.pod }}{{ end }}'

    - name: slack-critical-alerts
      slack_configs:
      - channel: "#oncall-critical"
        send_resolved: true

    # SILENCES — temporary mute (use amtool or Alertmanager UI)
    # amtool silence add alertname=ContainerHighRestartRate --duration=2h
    # amtool silence expire <silence-id>
```

---

### Alert Lifecycle

```
ALERT STATE MACHINE
═══════════════════════════════════════════════════════════════

Prometheus evaluates expression every evaluationInterval:

  INACTIVE → expression first fires
  PENDING  → expression firing, but for: duration not elapsed yet
             (prevents flapping on transient spikes)
  FIRING   → for: duration elapsed, alert sent to Alertmanager
  RESOLVED → expression no longer fires

  FIRING alert sent to Alertmanager:
  Alertmanager:
    1. Fingerprint: hash of alert labels (for deduplication)
    2. Group: batch related alerts (group_by labels)
    3. Wait: group_wait for more alerts to join group
    4. Route: match against route tree → receiver
    5. Notify: send to Slack/PagerDuty
    6. Repeat: re-notify if still firing after repeat_interval

DEDUPLICATION:
  Same alert firing on 10 pods → ONE notification (grouped)
  Not 10 separate PagerDuty pages
  group_by: [alertname, service] groups all pod restarts for same service
```

---

### 🎤 Short Crisp Interview Answer

> *"PrometheusRules define alert expressions in PromQL with a 'for' duration that filters transient spikes — an alert must fire continuously for the full duration before Alertmanager receives it. Alertmanager then handles grouping (batching related alerts together), inhibition (suppressing warnings when a critical for the same service is already firing), silencing (temporary muting during maintenance), and routing via a route tree that matches on label values to direct alerts to the appropriate receiver — Slack, PagerDuty, or webhook. The key operational parameters are group_wait (how long to batch related alerts), repeat_interval (how often to re-notify if still firing), and inhibit_rules to prevent notification storms when one critical alert spawns dozens of warnings."*

---

---

# 8.10 eBPF-Based Observability (Hubble/Cilium, Pixie)

## 🔴 Advanced

### What it is in simple terms

**eBPF (extended Berkeley Packet Filter)** allows programs to run safely in the Linux kernel without kernel modules. For observability, eBPF can **capture every network packet, system call, and kernel event** with near-zero overhead — giving deep visibility into container behavior without any application changes or sidecars.

---

### eBPF for Kubernetes Observability

```
TRADITIONAL OBSERVABILITY vs eBPF OBSERVABILITY
═══════════════════════════════════════════════════════════════

TRADITIONAL:
  Application → instruments code → emits metrics/traces/logs
  Requires: SDK in every app, code changes, redeployment
  Blind spots: traffic between pods (captured at app layer only)
               kernel-level events
               third-party/legacy apps you can't instrument

eBPF:
  Kernel → eBPF program captures events → user-space processing
  Requires: eBPF program loaded once → all processes observed
  Visibility: EVERY network connection (pod-to-pod, pod-to-external)
              EVERY system call (file access, process exec)
              EVERY DNS query (resolved at kernel socket level)
              HTTP/gRPC traffic (parsed without TLS — if terminating in-pod)
  No application changes needed.
  No sidecars.
  ~2-3% overhead vs 5-15% for agent-based approaches.
```

---

### Hubble (Cilium) — Network Observability

```bash
# Hubble: eBPF-based network observability built into Cilium
# Requires: Cilium CNI

# Enable Hubble
helm upgrade cilium cilium/cilium \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Install Hubble CLI
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz
tar xzvf hubble-linux-amd64.tar.gz

# Port-forward to Hubble relay
kubectl port-forward -n kube-system svc/hubble-relay 4245:80

# Observe all traffic in production namespace in real-time
hubble observe --namespace production
# TIMESTAMP             SOURCE                       DESTINATION                   TYPE           VERDICT  SUMMARY
# Jan 15 10:30:01.123   production/api-abc123:47234   production/postgres:5432     to-endpoint    FORWARDED  TCP Established
# Jan 15 10:30:01.130   production/api-abc123         kube-system/coredns:53       DNS Query      FORWARDED  A production.svc
# Jan 15 10:30:01.200   external/1.2.3.4             production/api-abc123:8080   to-endpoint    FORWARDED  HTTP GET /api/v1/users
# Jan 15 10:30:02.500   production/worker-xyz        external/sqs.amazonaws.com   to-world       FORWARDED  TCP SYN

# Filter by verdict (dropped connections)
hubble observe --namespace production --verdict DROPPED
# Shows: NetworkPolicy blocks, policy violations

# Filter HTTP traffic
hubble observe --namespace production --protocol http
# Shows: every HTTP request with method, URL, status code

# Get metrics from Hubble
hubble observe --namespace production --type l7
# L7 (HTTP/gRPC/DNS) visibility — request-level detail

# Service map (visual)
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
# Open http://localhost:12000 → visual service dependency map
# Shows: which services talk to which, traffic volume, error rates

# Hubble metrics exported to Prometheus:
# hubble_flows_processed_total{source, destination, verdict, protocol}
# hubble_http_requests_total{source, destination, method, status_code}
# hubble_dns_queries_total{source, qtypes, rcode}
```

---

### Pixie — Full-Stack eBPF Observability

```bash
# Pixie: eBPF-based observability platform from New Relic (open-source core)
# Captures: HTTP, MySQL, PostgreSQL, gRPC, DNS, Redis — all without instrumentation

# Install Pixie
px deploy

# PxL (Pixie Query Language) — like Python for eBPF data

# Get all HTTP requests in the last 5 minutes
px run px/http_data -- -start_time '-5m' -namespace production

# Get all MySQL queries (captures at socket level, before encryption)
px run px/mysql_data -- -start_time '-5m' -namespace production

# Get all failing requests (status >= 400)
px run px/http_data_filtered -- \
  -start_time '-5m' \
  -namespace production \
  -req_method 'GET' \
  -resp_status '4[0-9][0-9]'

# Service topology map with latency
px run px/service_stats -- -start_time '-5m' -namespace production
# Shows: p50/p90/p99 latency, request rate, error rate per service pair

# DNS queries from a specific pod
px run px/dns_data -- \
  -start_time '-5m' \
  -pod my-api-abc123

# CPU flame graphs (no app changes needed)
px run px/perf_flamegraph -- -start_time '-5m' -namespace production
```

---

### eBPF vs Traditional Observability Comparison

```
WHEN TO USE EBPF OBSERVABILITY
═══════════════════════════════════════════════════════════════

USE eBPF (Hubble/Pixie) FOR:
  ✓ Zero-instrumentation visibility (no code changes)
  ✓ Legacy apps you can't modify
  ✓ Security investigation (what is this pod actually doing?)
  ✓ Network policy troubleshooting (is traffic being dropped?)
  ✓ Service dependency discovery (map unknown microservices)
  ✓ Database query performance (capture without DB agent)
  ✓ Compliance audit (prove what data left the cluster)

USE TRADITIONAL (Prometheus/OTel) FOR:
  ✓ Business metrics (user count, order value, feature flags)
  ✓ Application-specific metrics (queue depth, cache hit ratio)
  ✓ Custom SLI/SLO definitions
  ✓ Long-term trend storage and capacity planning
  ✓ Teams that want to own their observability domain

BEST PRACTICE: USE BOTH
  eBPF for infrastructure/network visibility (automatic)
  Application metrics/tracing for business metrics (instrumented)
  They complement, not replace, each other
```

---

### 🎤 Short Crisp Interview Answer

> *"eBPF observability captures kernel-level events — every network packet, system call, and DNS query — without any application changes or sidecars. Hubble is built into Cilium and provides real-time network flow visibility: every connection attempt, NetworkPolicy verdict (forwarded or dropped), HTTP request with status code, and DNS query, all visible through the hubble CLI or Hubble UI service map. Pixie extends this to L7 protocols — it captures MySQL queries, PostgreSQL queries, gRPC calls, and Redis commands by reading socket buffers at the kernel level before encryption. The key advantage over traditional observability is zero instrumentation overhead — particularly useful for legacy apps and security investigation. The limitation is that eBPF can't capture encrypted traffic in flight, and business-specific metrics still require application instrumentation."*

---

### ⚠️ Gotchas

1. **eBPF doesn't see inside TLS** — eBPF captures at the socket level. For TLS-encrypted traffic, it sees the connection but not the payload. Pixie reads from the plaintext socket buffer before the TLS layer, which works for in-pod TLS termination but not end-to-end encryption.
2. **Hubble requires Cilium CNI** — if you're using AWS VPC CNI, Flannel, or Calico, Hubble is not available. Pixie works with any CNI.
3. **eBPF kernel version requirements** — most features require Linux kernel 5.8+. Check node kernel versions before relying on eBPF features. EKS with Amazon Linux 2023 or Bottlerocket has sufficient kernel versions.

---

---

# 8.11 SLO-Based Alerting in Kubernetes

## 🔴 Advanced

### What it is in simple terms

**SLOs (Service Level Objectives)** define what "good" means for a service: "99.9% of requests succeed" or "P99 latency < 500ms." **Error budget** is how much you can fail within the SLO period. **Multi-window, multi-burn-rate alerting** catches both fast burns (immediate outage) and slow burns (gradual degradation) while minimizing false positives — a more sophisticated approach than simple threshold alerts.

---

### SLO Concepts

```
SLO TERMINOLOGY
═══════════════════════════════════════════════════════════════

SLA (Service Level Agreement):
  External contract. Breach = financial penalty.
  E.g., "99.9% uptime per month"

SLO (Service Level Objective):
  Internal target. Slightly tighter than SLA.
  E.g., "99.95% of requests succeed" over 30 days

SLI (Service Level Indicator):
  The metric that measures the SLO.
  E.g., rate(http_requests_total{status!~"5.."}[30d])
        / rate(http_requests_total[30d])

ERROR BUDGET:
  How much you can fail and still meet the SLO.
  99.9% SLO over 30 days = 0.1% error budget
  0.1% × 30 days × 24h × 60min = 43.2 minutes of downtime allowed

BURN RATE:
  How fast you're consuming the error budget.
  Burn rate 1 = consuming budget exactly at the rate it replenishes
  Burn rate 14 = consuming 14× faster = exhausting budget in 2 days
  (30 days × 1/14 = ~2.1 days)

MULTI-WINDOW MULTI-BURN-RATE ALERTING:
  Alert when burn rate is HIGH (fast burn = immediate action needed)
  Alert when burn rate is SUSTAINED (slow burn = gradual degradation)
  Uses two time windows per alert to filter transient spikes
```

---

### SLO Alerting PrometheusRules

```yaml
# SLO: 99.9% availability, 30-day window
# Error budget: 0.1% = 43.2 minutes/month

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-alerts-my-api
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  # Recording rules — pre-compute error rates for efficiency
  - name: slo-my-api-recording
    rules:
    # 5-minute error ratio
    - record: job:http_error_ratio:5m
      expr: |
        sum(rate(http_requests_total{
          job="my-api",
          status=~"5.."
        }[5m])) by (job)
        /
        sum(rate(http_requests_total{
          job="my-api"
        }[5m])) by (job)

    # 1-hour error ratio
    - record: job:http_error_ratio:1h
      expr: |
        sum(rate(http_requests_total{
          job="my-api",
          status=~"5.."
        }[1h])) by (job)
        /
        sum(rate(http_requests_total{
          job="my-api"
        }[1h])) by (job)

    # 6-hour error ratio
    - record: job:http_error_ratio:6h
      expr: |
        sum(rate(http_requests_total{job="my-api",status=~"5.."}[6h])) by (job)
        / sum(rate(http_requests_total{job="my-api"}[6h])) by (job)

    # 30-day error ratio (current budget burn)
    - record: job:http_error_ratio:30d
      expr: |
        sum(rate(http_requests_total{job="my-api",status=~"5.."}[30d])) by (job)
        / sum(rate(http_requests_total{job="my-api"}[30d])) by (job)

  - name: slo-my-api-alerts
    rules:
    # ALERT TIER 1: CRITICAL — Fast burn (page immediately)
    # Burn rate 14x = uses 14% of monthly budget in 1 hour
    # Error rate > 1.4% for 5m AND > 1.4% for 1h
    - alert: SLOBurnRateCritical
      expr: |
        job:http_error_ratio:5m{job="my-api"} > (14 * 0.001)
        and
        job:http_error_ratio:1h{job="my-api"} > (14 * 0.001)
      for: 2m
      labels:
        severity: critical
        pagerduty: "true"
        slo: "availability"
      annotations:
        summary: "CRITICAL: {{ $labels.job }} burning error budget at 14x rate"
        description: |
          Error rate: {{ $value | humanizePercentage }}
          At this rate, monthly error budget exhausted in ~2 days.
          Immediate action required.
        runbook_url: "https://wiki/runbooks/slo-critical"

    # ALERT TIER 2: HIGH — Medium burn (page during business hours)
    # Burn rate 6x = uses 6% of monthly budget in 1 hour
    - alert: SLOBurnRateHigh
      expr: |
        job:http_error_ratio:30m{job="my-api"} > (6 * 0.001)
        and
        job:http_error_ratio:6h{job="my-api"} > (6 * 0.001)
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "HIGH: {{ $labels.job }} burning error budget at 6x rate"
        description: |
          Error rate: {{ $value | humanizePercentage }}
          At this rate, monthly error budget exhausted in ~5 days.
          Investigation required.

    # ALERT TIER 3: WARNING — Slow burn (ticket, not page)
    # Burn rate 3x = uses 3% of monthly budget per hour
    - alert: SLOBurnRateSlow
      expr: |
        job:http_error_ratio:2h{job="my-api"} > (3 * 0.001)
        and
        job:http_error_ratio:24h{job="my-api"} > (3 * 0.001)
      for: 1h
      labels:
        severity: info
      annotations:
        summary: "SLOW BURN: {{ $labels.job }} consuming error budget"
        description: |
          Error rate: {{ $value | humanizePercentage }}
          Error budget will be exhausted in ~10 days at this rate.

    # Error budget remaining
    - alert: ErrorBudgetLow
      expr: |
        1 - job:http_error_ratio:30d{job="my-api"} / 0.001 < 0.1
      for: 0m
      labels:
        severity: warning
      annotations:
        summary: "Less than 10% error budget remaining for {{ $labels.job }}"
```

---

### Sloth — SLO Configuration Simplified

```yaml
# Sloth generates PrometheusRules from SLO definitions
# https://github.com/slok/sloth

apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: my-api-slos
  namespace: monitoring
spec:
  service: "my-api"
  labels:
    team: platform
    env: production

  slos:
  - name: requests-availability
    objective: 99.9                  # 99.9% target
    description: "99.9% of API requests must succeed"
    sli:
      events:
        errorQuery: |
          sum(rate(http_requests_total{job="my-api",status=~"(5..|429)"}[{{.window}}]))
        totalQuery: |
          sum(rate(http_requests_total{job="my-api"}[{{.window}}]))
    alerting:
      name: MyAPIAvailability
      labels:
        category: availability
      annotations:
        runbook: "https://wiki/runbooks/api-availability"
      pageAlert:
        labels:
          severity: critical
          pagerduty: "true"
      ticketAlert:
        labels:
          severity: warning

  - name: requests-latency
    objective: 99.0                  # 99% of requests < 500ms
    description: "99% of API requests complete within 500ms"
    sli:
      events:
        errorQuery: |
          sum(rate(http_request_duration_seconds_bucket{
            job="my-api",
            le="0.5"
          }[{{.window}}]))
        totalQuery: |
          sum(rate(http_request_duration_seconds_count{job="my-api"}[{{.window}}]))
    alerting:
      name: MyAPILatency
      labels:
        category: latency
```

---

### 🎤 Short Crisp Interview Answer

> *"SLO-based alerting is more nuanced than threshold alerting because it aligns alerts with actual user impact. The key concept is burn rate — how fast you're consuming your error budget. A single threshold like 'alert if error rate > 1%' creates both false positives (brief spikes that don't exhaust the budget) and false negatives (sustained 0.5% errors that exhaust the budget slowly). Multi-window multi-burn-rate alerting uses two windows per tier: a short window for responsiveness and a longer window for confidence, filtered by burn rate thresholds derived from the SLO. Burn rate 14 is critical (monthly budget gone in 2 days), burn rate 6 is high, burn rate 3 is warning. Tools like Sloth generate the PrometheusRules from a simple SLO YAML, so you don't have to hand-calculate all the thresholds."*

---

---

# 🏁 Category 8 — Complete Observability Map

```
OBSERVABILITY DECISION MAP
═══════════════════════════════════════════════════════════════════

SOMETHING IS WRONG → WHERE TO START?

  "Pod is not running" → Events → 8.3, 8.7
    kubectl describe pod → Events section
    Pending? → scheduling failure (resources, affinity, taints)
    CrashLoopBackOff? → kubectl logs --previous → exit code
    OOMKilled? → increase memory limit → 6.1

  "Service is slow" → Traces → 8.6
    Distributed trace shows which hop is slow
    If no traces: metrics → latency histogram → which service?
    Check CPU throttling: container_cpu_cfs_throttled → 6.1

  "Error rate spiking" → Metrics + Logs → 8.5, 8.1
    HTTP error rate from Prometheus → which service?
    Logs for that service → what error message?
    Recent deployment? → kubectl rollout history + undo?

  "Cluster is slow" → Node metrics → 8.4
    kubectl top nodes → CPU/memory saturation
    Node pressure conditions → 6.14
    Descheduler needed? → 6.13

  "What is this pod actually doing?" → eBPF → 8.10
    Hubble → what connections is it making?
    Pixie → what queries is it running?

OBSERVABILITY STACK LAYERS:
  Tier 1: Events, Probes, kubectl      → Basic (8.2, 8.3, 8.7)
  Tier 2: Metrics (Prometheus/Grafana) → Trending (8.4, 8.5)
  Tier 3: Logs (Fluentbit/Loki)        → Detailed (8.1)
  Tier 4: Traces (OTel/Jaeger)         → Request-level (8.6)
  Tier 5: eBPF (Hubble/Pixie)          → Zero-instrumentation (8.10)
  Tier 6: Alerts (PrometheusRules)     → Automated notification (8.9)
  Tier 7: SLOs                         → User-impact aligned (8.11)
```

---

# Quick Reference — Category 8 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **kubectl logs --previous** | Last terminated container's logs — essential for CrashLoopBackOff |
| **Log rotation** | 10Mi default, 5 files kept — without aggregation, old logs lost |
| **Liveness probe** | Restarts container on failure. NEVER check external deps |
| **Readiness probe** | Removes pod from LB on failure. CAN check deps |
| **Startup probe** | Disables liveness until startup complete. Saves from restart loops |
| **Probe timeoutSeconds** | Default is 1s — too low under load. Set 3-5s |
| **Events TTL** | ~1 hour default. Ship to log aggregation for retention |
| **metrics-server** | Real-time usage (CPU/memory). In-memory. Powers kubectl top + HPA |
| **kube-state-metrics** | K8s object state as metrics. Powers Prometheus alerts |
| **ServiceMonitor** | Tells Prometheus what to scrape. Labels must match Prometheus selector |
| **High cardinality** | Adding UUID/user ID labels to metrics = Prometheus crash |
| **Container exit 137** | OOMKilled. Exit 143 = SIGTERM. Exit 1 = app error |
| **Running ≠ Ready** | Pod Running phase with 0/1 Ready is not in LB endpoints |
| **Prometheus adapter** | Bridges Prometheus → custom.metrics.k8s.io for HPA |
| **Alert for: clause** | Transient spike filter. Alert must fire continuously for this duration |
| **Alertmanager grouping** | Batches related alerts. group_wait / group_interval / repeat_interval |
| **inhibit_rules** | Suppress warnings when a critical for same service is firing |
| **Hubble** | Cilium eBPF network observability. Every flow, DNS, HTTP |
| **Pixie** | eBPF L7 observability. MySQL/PostgreSQL/gRPC without instrumentation |
| **Burn rate** | How fast error budget is consumed. Rate 14 = critical (page now) |
| **SLO multi-window** | Short window (responsiveness) + long window (confidence) per tier |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| Default probe periodSeconds | 10 seconds |
| Default probe timeoutSeconds | 1 second (too low — use 3-5s) |
| Default probe failureThreshold | 3 |
| Container log file size before rotation | 10 MiB (default) |
| Container log files retained | 5 (default) |
| Event default TTL | ~1 hour |
| Prometheus default scrape interval | 30 seconds |
| Prometheus default retention | 15 days |
| HPA custom metrics poll interval | 15 seconds |
| Prometheus evaluation interval | 1 minute |
| Alert group_wait default | 30 seconds |
| SLO burn rate for critical (page) | 14× (uses monthly budget in 2 days) |
| SLO burn rate for high | 6× (uses monthly budget in 5 days) |
| SLO burn rate for warning | 3× (uses monthly budget in 10 days) |
| Falco eBPF overhead | ~2-3% CPU per node |
| eBPF minimum kernel version | 5.8+ for full features |

# Kubernetes Interview Mastery
# CATEGORY 11: ISTIO & SERVICE MESH

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Gotchas.
> ⚠️ = High priority, frequently asked in interviews.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 11.1 | What is a service mesh & why it exists ⚠️ | 🟢 Beginner |
| 11.2 | Istio architecture — control plane vs data plane | 🟢 Beginner |
| 11.3 | Envoy sidecar proxy — how it intercepts traffic | 🟢 Beginner |
| 11.4 | Istio injection — how sidecars get injected ⚠️ | 🟢 Beginner |
| 11.5 | VirtualService — traffic routing rules ⚠️ | 🟡 Intermediate |
| 11.6 | DestinationRule — load balancing, TLS, circuit breaking ⚠️ | 🟡 Intermediate |
| 11.7 | Gateway — ingress via Istio | 🟡 Intermediate |
| 11.8 | mTLS — PeerAuthentication, automatic vs strict mode ⚠️ | 🟡 Intermediate |
| 11.9 | AuthorizationPolicy — L7 RBAC ⚠️ | 🟡 Intermediate |
| 11.10 | Istio observability — Kiali, tracing, metrics | 🟡 Intermediate |
| 11.11 | Traffic management — canary, blue/green, mirroring ⚠️ | 🟡 Intermediate |
| 11.12 | ServiceEntry — bringing external services into the mesh | 🟡 Intermediate |
| 11.13 | Istio control plane internals — Istiod, Pilot, Citadel, Galley | 🔴 Advanced |
| 11.14 | xDS API — how Envoy gets its config ⚠️ | 🔴 Advanced |
| 11.15 | Istio ambient mesh — sidecarless architecture | 🔴 Advanced |
| 11.16 | Multi-cluster Istio | 🔴 Advanced |
| 11.17 | Circuit breaking & outlier detection in production | 🔴 Advanced |
| 11.18 | Istio performance tuning & resource overhead | 🔴 Advanced |

---

# ⚠️ 11.1 What Is a Service Mesh & Why It Exists

## 🟢 Beginner — HIGH PRIORITY

### The Problem It Solves

```
THE DISTRIBUTED SYSTEMS CROSS-CUTTING CONCERNS PROBLEM
═══════════════════════════════════════════════════════════════

EVERY microservice needs these capabilities:
  - Mutual TLS (encrypt + authenticate service-to-service traffic)
  - Retries (retry on transient failures)
  - Timeouts (don't wait forever for a slow service)
  - Circuit breaking (stop sending to a failing service)
  - Load balancing (L7: weighted, least-connection, locality-aware)
  - Distributed tracing (trace a request across 10 services)
  - Metrics (request rate, error rate, latency per service pair)
  - Authorization (service A is allowed to call service B's /api endpoint)
  - Rate limiting (service A can only call B 1000 req/sec)

WITHOUT A SERVICE MESH:
  Each team implements this IN THEIR APPLICATION CODE:
  ├── Python service: uses requests + tenacity for retries
  ├── Go service: uses net/http + custom circuit breaker
  ├── Java service: uses Resilience4j
  ├── Node service: uses axios with custom retry logic
  Problems:
    - Inconsistent: each language implements it differently
    - Hard to audit: "is mTLS actually enforced between all services?"
    - Can't be changed centrally: retry logic change = redeploy 50 services
    - Observability gaps: tracing only works where dev remembered to instrument
    - Maintenance burden: every team owns this complexity

WITH A SERVICE MESH:
  ALL cross-cutting concerns implemented ONCE at the network layer
  by a proxy (Envoy sidecar) that sits next to every application container.
  Application speaks plain HTTP/gRPC → proxy handles mTLS, retries, tracing.
  
  Benefits:
  ✓ Language-agnostic: works for Go, Python, Java, Node equally
  ✓ Zero app changes: inject sidecar, get all features
  ✓ Centrally configured: change retry policy via YAML, no redeployment
  ✓ Consistent observability: every request traced and measured
  ✓ Auditable security: mTLS enforcement verifiable centrally
  ✓ Gradual rollout: canary 10%/90% via config, no code change
```

---

### Service Mesh vs Alternatives

```
SERVICE MESH vs OTHER APPROACHES
═══════════════════════════════════════════════════════════════

Application libraries (Netflix OSS / Hystrix / Resilience4j):
  Pro:  no infra overhead, language-native
  Con:  per-language, per-team, inconsistent, can't centrally enforce
  Best when: monolith or few services, homogeneous language stack

Service Mesh (Istio, Linkerd, Cilium):
  Pro:  uniform, language-agnostic, centrally managed
  Con:  infrastructure complexity, latency overhead (~1-3ms), memory (~50MB/sidecar)
  Best when: many microservices, multi-language, security + observability required

API Gateway (Kong, AWS API Gateway):
  Pro:  handles north-south traffic (client → cluster) well
  Con:  not designed for east-west (service → service) traffic
  Often combined with service mesh: Gateway for ingress, mesh for internal

eBPF-based observability (Cilium Hubble):
  Pro:  zero overhead on application, no sidecar
  Con:  L4 visibility (no L7 HTTP method/path), less policy expressiveness
  Cilium also has L7 policy but not as rich as Istio

Istio Ambient Mesh (sidecarless, new):
  Pro:  no per-pod sidecar overhead, simpler injection
  Con:  less mature, some features still in progress
  Best when: want mesh benefits without sidecar overhead
```

---

### 🎤 Short Crisp Interview Answer

> *"A service mesh moves network cross-cutting concerns — mTLS, retries, timeouts, circuit breaking, tracing, and metrics — out of application code into a dedicated infrastructure layer. Without a mesh, every team reimplements these in their language of choice, inconsistently. Istio uses Envoy sidecars co-located with every pod: the app speaks plain HTTP, the sidecar handles encryption, observability, and policy. This means you can add mTLS between all services without touching a single line of application code, and change retry policies centrally without redeploying 50 services. The trade-off is ~1-3ms added latency per hop and ~50-100MB memory per sidecar — real costs that matter in very high-throughput or memory-constrained environments."*

---

---

# 11.2 Istio Architecture — Control Plane vs Data Plane

## 🟢 Beginner

### Two-Plane Architecture

```
ISTIO ARCHITECTURE
═══════════════════════════════════════════════════════════════

CONTROL PLANE (istiod — single binary since Istio 1.5):
  Runs as: Deployment in istio-system namespace
  Replicas: typically 2-3 for HA
  Responsibilities:
    - Pilot:   service discovery → Envoy routing config (xDS API)
    - Citadel: certificate authority → issue mTLS certs to sidecars
    - Galley:  config validation + processing (now integrated in istiod)
  Communicates with: API Server (watches Services, Endpoints, Istio CRDs)
                     Envoy proxies (push xDS config via gRPC)
  Does NOT handle: any data-plane traffic (no traffic flows through istiod)

DATA PLANE (Envoy sidecars):
  Runs as: istio-proxy container injected into every pod
  Responsibilities:
    - Intercept ALL inbound and outbound pod traffic (via iptables)
    - Enforce routing rules (VirtualService)
    - Enforce security policy (mTLS, AuthorizationPolicy)
    - Collect and report metrics + traces
  Scale: one Envoy per pod = scales with your workload
  Communicates with: istiod (receives config) + app container + other Envoys

TRAFFIC FLOW:
  Pod A (app + envoy-proxy) → Pod B (app + envoy-proxy)

  app-A (localhost:8080)
    ↓ plain HTTP (app doesn't know about mesh)
  envoy-proxy-A (intercepts outbound)
    ↓ mTLS over TCP (encrypted, authenticated)
  envoy-proxy-B (intercepts inbound)
    ↓ plain HTTP (decrypted, verified)
  app-B (localhost:8080)

  App A and App B both speak plain HTTP.
  Envoys handle mTLS transparently between them.
  App sees no difference with or without mesh.
```

---

### Istio Components in the Cluster

```bash
# Verify Istio installation
kubectl get pods -n istio-system
# NAME                                    READY   STATUS    RESTARTS   AGE
# istiod-7d8b9c-abc12                     1/1     Running   0          30d
# istiod-7d8b9c-def34                     1/1     Running   0          30d
# istio-ingressgateway-5b6c9f-xyz         1/1     Running   0          30d

# Istiod exposes:
kubectl get svc -n istio-system
# NAME                    TYPE           CLUSTER-IP      PORT(S)
# istiod                  ClusterIP      10.96.32.10     15010/TCP  ← xDS gRPC (unsecured)
#                                                        15012/TCP  ← xDS gRPC (secured)
#                                                        443/TCP    ← webhook (cert injection)
#                                                        15014/TCP  ← debug/metrics
# istio-ingressgateway    LoadBalancer   10.96.45.20     80/HTTP, 443/HTTPS

# istiod health
kubectl get deploy istiod -n istio-system
# Verify istiod is healthy before troubleshooting any mesh issues

# Check Istio version
istioctl version
# client version: 1.20.0
# control plane version: 1.20.0
# data plane version: 1.20.0 (11 proxies)
```

---

### 🎤 Short Crisp Interview Answer

> *"Istio has two planes. The control plane is istiod — a single Go binary running in istio-system that combines what were once three separate components: Pilot (distributes routing config to Envoys), Citadel (certificate authority issuing mTLS certs), and Galley (config validation). Istiod watches Kubernetes objects and Istio CRDs, computes the desired Envoy config, and pushes it to all sidecars via the xDS gRPC API. The data plane is the Envoy proxy container injected into every pod — it intercepts all traffic via iptables, enforces policy, handles mTLS, and reports telemetry. Istiod never touches data-plane traffic — if istiod goes down, existing traffic keeps flowing because sidecars have cached their last config."*

---

---

# 11.3 Envoy Sidecar Proxy — How It Intercepts Traffic

## 🟢 Beginner

### Envoy Architecture

```
HOW ENVOY INTERCEPTS TRAFFIC
═══════════════════════════════════════════════════════════════

IPTABLES REDIRECT (init container sets this up):
  The istio-init container (or CNI plugin) runs before the app container.
  It writes iptables rules:
    - All OUTBOUND traffic (except port 15001): redirect to port 15001 (Envoy)
    - All INBOUND traffic (except port 15006): redirect to port 15006 (Envoy)
    - Exception: traffic from UID 1337 (Envoy's own UID) passes through
                 (prevents Envoy from intercepting its own forwarded traffic)

  iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 15006  # inbound
  iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-ports 15001       # outbound
  iptables -t nat -A OUTPUT -m owner --uid-owner 1337 -j RETURN       # Envoy bypass

  App thinks it's connecting to service B:8080
  iptables silently redirects to Envoy:15001
  Envoy reads the ORIGINAL destination from SO_ORIGINAL_DST socket option
  Envoy applies routing rules, makes the real connection to service B

ENVOY INTERNAL ARCHITECTURE:
  Listeners:   ports Envoy listens on (15001 outbound, 15006 inbound)
  Filters:     chain of processing plugins per listener
               HTTP filters: JWT auth, RBAC, fault injection, stats
               Network filters: TCP proxy, MongoDB filter, Redis filter
  Routes:      match request → select cluster
  Clusters:    upstream service definitions (with endpoints)
  Endpoints:   actual pod IP:ports for each cluster

CONFIG DELIVERY:
  All of this config (listeners, routes, clusters, endpoints)
  comes from istiod via the xDS gRPC API.
  Envoy receives push updates when the config changes.
  Envoy NEVER reads Kubernetes objects directly — only istiod does.
```

---

### Envoy Port Reference

```
ENVOY SIDECAR PORTS
═══════════════════════════════════════════════════════════════

Port  Purpose
──────────────────────────────────────────────────────────────
15000 Envoy admin interface (debug, config dump, stats)
15001 Outbound traffic capture (iptables redirects outbound here)
15004 Debug port (mirror of 15000, accessible from istiod)
15006 Inbound traffic capture (iptables redirects inbound here)
15008 HBONE tunnel port (for ambient mesh)
15009 HBONE secure tunnel
15020 Istio agent health + metrics aggregation
15021 Health check endpoint (/healthz/ready)
15053 DNS proxy (captures DNS queries, resolves via istiod)
15090 Envoy Prometheus metrics endpoint (/stats/prometheus)
```

---

```bash
# Access Envoy admin interface for debugging
kubectl exec -it my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/help
# Lists all admin endpoints

# Dump full Envoy config (listeners, routes, clusters, endpoints)
kubectl exec -it my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/config_dump | python3 -m json.tool > /tmp/config.json

# Check which clusters (upstreams) Envoy knows about
kubectl exec -it my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/clusters | head -50

# Check active listeners
kubectl exec -it my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/listeners

# Check Envoy stats (request counts, errors)
kubectl exec -it my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/stats | grep "outbound|9080|http.requests"

# Check certificates
kubectl exec -it my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/certs

# istioctl proxy-config — friendlier wrapper
istioctl proxy-config listeners my-pod.production
istioctl proxy-config routes my-pod.production
istioctl proxy-config clusters my-pod.production
istioctl proxy-config endpoints my-pod.production
istioctl proxy-config secret my-pod.production   # check mTLS certs

# Check what traffic Envoy is intercepting
istioctl proxy-config listener my-pod.production \
  --port 8080 -o json
```

---

---

# ⚠️ 11.4 Istio Injection — How Sidecars Get Injected

## 🟢 Beginner — HIGH PRIORITY

### Injection Mechanism

```
TWO INJECTION METHODS
═══════════════════════════════════════════════════════════════

METHOD 1: AUTOMATIC INJECTION (recommended, production)
  Label a namespace: istio-injection=enabled
  All pods created in that namespace automatically get:
    - istio-proxy container injected (Envoy sidecar)
    - istio-init init container (sets up iptables rules)
    OR with CNI: no init container (CNI plugin handles iptables)

  How it works:
  kubectl apply -f pod.yaml
  → API Server → MutatingWebhookConfiguration (istio-sidecar-injector)
  → istiod webhook called with AdmissionReview
  → istiod injects istio-proxy + istio-init into pod spec
  → Modified pod spec returned to API Server
  → Modified pod stored in etcd and scheduled

  The webhook config:
  kubectl get mutatingwebhookconfiguration istio-sidecar-injector
  Matches: namespaces with label istio-injection=enabled
  AND: pods without annotation sidecar.istio.io/inject: "false"

METHOD 2: MANUAL INJECTION (for testing/debugging)
  istioctl kube-inject -f my-pod.yaml | kubectl apply -f -
  OR: generate injected manifest to inspect/commit to git
  istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml
```

---

### Injection Configuration

```bash
# ── NAMESPACE-LEVEL INJECTION ─────────────────────────────────────
# Enable injection in a namespace
kubectl label namespace production istio-injection=enabled

# Disable injection in a namespace (even if mesh-wide enabled)
kubectl label namespace production istio-injection=disabled

# Check current label
kubectl get namespace production --show-labels | grep istio

# ── POD-LEVEL OVERRIDES ───────────────────────────────────────────
# In pod/deployment annotations:
metadata:
  annotations:
    sidecar.istio.io/inject: "false"    # exclude THIS pod from injection
    sidecar.istio.io/inject: "true"     # force injection even if ns disabled

# ── VERIFY INJECTION ──────────────────────────────────────────────
kubectl get pod my-pod -n production \
  -o jsonpath='{.spec.containers[*].name}'
# my-app istio-proxy    ← both containers present = injected

kubectl get pods -n production
# NAME          READY   STATUS    RESTARTS   AGE
# my-api-abc    2/2     Running   0          5m   ← 2/2 = sidecar injected
# my-api-def    1/1     Running   0          5m   ← 1/1 = no sidecar!

# ── INJECTION RESOURCE CUSTOMIZATION ──────────────────────────────
metadata:
  annotations:
    # Override sidecar resources (important for resource-constrained environments)
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "500m"
    sidecar.istio.io/proxyMemoryLimit: "512Mi"
    # Log level for debugging
    sidecar.istio.io/logLevel: "debug"
    # Ports to exclude from interception (e.g., health check ports)
    traffic.sidecar.istio.io/excludeInboundPorts: "15020"
    traffic.sidecar.istio.io/excludeOutboundPorts: "9901"
    # Include specific ports only (selective interception)
    traffic.sidecar.istio.io/includeInboundPorts: "8080,8443"
```

---

### istio-init vs CNI Injection

```
INIT CONTAINER (default) vs CNI PLUGIN
═══════════════════════════════════════════════════════════════

istio-init (default):
  + Simpler to install (no cluster-level CNI change needed)
  - Requires NET_ADMIN capability (privileged init container)
  - Adds ~1s to pod startup (iptables setup)
  - Requires: PSA permitting NET_ADMIN (not "Restricted" PSS profile)

istio-cni (alternative):
  + No privileged init container needed
  + Compatible with "Restricted" PSS profile
  + Faster pod startup (iptables already set up when pod starts)
  - Cluster-level CNI plugin change needed
  - More complex to install and upgrade

USE CNI WHEN:
  PodSecurityAdmission "Restricted" profile enforced cluster-wide
  Security posture prohibits privileged containers
  Pod startup time is critical

INSTALL CNI MODE:
  helm install istio-cni istio/cni \
    --namespace kube-system \
    --set cni.cniBinDir=/opt/cni/bin \
    --set cni.cniConfDir=/etc/cni/net.d

  helm install istiod istio/istiod \
    --namespace istio-system \
    --set pilot.cni.enabled=true
```

---

### ⚠️ Injection Gotchas

```
COMMON INJECTION MISTAKES
═══════════════════════════════════════════════════════════════

GOTCHA 1: Label the namespace BEFORE deploying pods
  Wrong: deploy pods, then label namespace
  → Existing pods NOT re-injected (must restart/rollout restart)
  Right: label namespace first, then deploy pods

GOTCHA 2: CrashLoopBackOff after injection
  App starts before init container finishes iptables setup
  OR: app hardcodes 127.0.0.1 instead of container localhost
  Fix: use health checks to ensure readiness
       Or: add holdApplicationUntilProxyStarts: true in MeshConfig

GOTCHA 3: 2/2 READY but traffic not working
  Both containers running but Envoy hasn't received full xDS config
  Envoy starts before receiving config from istiod
  Fix: check istiod logs, check Envoy xDS sync status

GOTCHA 4: Namespace kube-system should NOT be injected
  System pods getting Envoy sidecars causes bootstrap loops
  Verify: kubectl get ns kube-system --show-labels | grep istio
  Should show: NO istio-injection label

GOTCHA 5: Pod restarts after sidecar injection
  istio-proxy memory OOM → pod restart
  Default istio-proxy memory: ~50-100MB request, 2GB limit
  In busy clusters: can use 200-500MB
  Fix: tune per-pod with annotations or globally in MeshConfig

GOTCHA 6: Deadlock on DNS startup
  App pod starts, tries to resolve DNS before iptables are ready
  istio-proxy intercepts DNS (port 15053) — if not ready, DNS fails
  Fix: holdApplicationUntilProxyStarts: true
```

---

### 🎤 Short Crisp Interview Answer

> *"Istio injection works via a MutatingAdmissionWebhook. Label a namespace with istio-injection=enabled, and every pod created there goes through the istiod webhook, which injects two things: an istio-init init container that writes iptables rules to intercept all pod traffic, and the istio-proxy (Envoy) container that handles it. The iptables rules redirect outbound traffic to Envoy port 15001 and inbound to 15006. Envoy uses SO_ORIGINAL_DST to learn where the packet was originally headed. The critical operational gotcha: labeling a namespace after pods are already running doesn't inject existing pods — you must rollout restart them. Also, kube-system should never be labeled for injection — it causes bootstrap loops."*

---

---

# ⚠️ 11.5 VirtualService — Traffic Routing Rules

## 🟡 Intermediate — HIGH PRIORITY

### What VirtualService Does

```
VIRTUALSERVICE — L7 TRAFFIC ROUTING
═══════════════════════════════════════════════════════════════

KUBERNETES SERVICE:   fixed selector → all matching pods
VIRTUALSERVICE:       rich matching rules → specific subsets of pods

VirtualService defines HOW traffic is routed to a destination.
DestinationRule defines WHAT the destinations are (subsets).
Together: complete traffic management.

VirtualService CAN:
  Weight traffic (90% to v1, 10% to v2)
  Route by HTTP header ("X-User-Type: premium" → v2)
  Route by URL path (/api/v2 → v2 subset)
  Route by source (traffic from mobile-app → v2)
  Add retries (retry 3 times on 503)
  Add timeouts (fail after 5 seconds)
  Inject faults (delay 50% of requests by 7s for chaos testing)
  Mirror traffic (copy 100% of traffic to a shadow service)
  Rewrite URLs (/old-path → /new-path)
  Add/remove headers

VirtualService CANNOT:
  Replace the Kubernetes Service (VS needs a real K8s Service to back it)
  Route TCP by L7 fields (TCP routing = by port only)
  Replace NetworkPolicy (VS is not a firewall, use AuthorizationPolicy)
```

---

### VirtualService YAML — Complete Reference

```yaml
# ── BASIC ROUTING — version-based canary ─────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-api-vs
  namespace: production
spec:
  # Hostnames this VS applies to (K8s Service names, external DNS, or *)
  hosts:
  - my-api                     # short name = my-api.production.svc.cluster.local
  - my-api.company.com         # also handle traffic for this external hostname

  # For ingress: which Gateways this VS is bound to
  gateways:
  - my-gateway                 # bind to Istio Gateway (for external traffic)
  - mesh                       # also for internal mesh traffic (default if omitted)

  http:                        # HTTP/HTTPS/gRPC routing rules (in order, first match wins)
  # ── Rule 1: header-based routing ──────────────────────────────
  - match:
    - headers:
        x-user-type:
          exact: premium       # exact match on header value
    - headers:
        x-beta-user:
          prefix: "true"       # prefix match
    route:
    - destination:
        host: my-api
        subset: v2             # defined in DestinationRule
      weight: 100

  # ── Rule 2: URI path routing ───────────────────────────────────
  - match:
    - uri:
        prefix: /api/v2        # prefix match
    - uri:
        regex: /api/v[0-9]+/.*  # regex match
    route:
    - destination:
        host: my-api
        subset: v2
      weight: 100

  # ── Rule 3: canary split (weighted) ──────────────────────────
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: my-api
        subset: v1
      weight: 90               # 90% to stable v1
    - destination:
        host: my-api
        subset: v2
      weight: 10               # 10% to canary v2

    # Retries on this route
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: "5xx,reset,connect-failure,retriable-4xx"

    # Timeout for this route
    timeout: 30s

    # Fault injection (for chaos engineering / testing)
    fault:
      delay:
        percentage:
          value: 10.0          # delay 10% of requests
        fixedDelay: 7s
      abort:
        percentage:
          value: 5.0           # abort 5% of requests
        httpStatus: 503

  # ── Default rule (catch-all) ───────────────────────────────────
  - route:
    - destination:
        host: my-api
        subset: v1
      weight: 100

---
# ── TCP ROUTING (non-HTTP) ────────────────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: postgres-vs
spec:
  hosts:
  - postgres
  tcp:
  - match:
    - port: 5432
    route:
    - destination:
        host: postgres
        port:
          number: 5432
        subset: primary
      weight: 100

---
# ── URL REWRITE ───────────────────────────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: rewrite-vs
spec:
  hosts:
  - my-api
  http:
  - match:
    - uri:
        prefix: /old/api
    rewrite:
      uri: /new/api           # rewrite before forwarding to upstream
    route:
    - destination:
        host: my-api
        port:
          number: 8080

---
# ── MIRROR (traffic shadowing) ────────────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: shadow-vs
spec:
  hosts:
  - my-api
  http:
  - route:
    - destination:
        host: my-api
        subset: v1
      weight: 100
    mirror:                   # copy 100% of traffic to v2 (async, no client impact)
      host: my-api
      subset: v2
    mirrorPercentage:
      value: 100.0
```

---

### Match Conditions Reference

```
MATCH CONDITIONS FOR HTTP ROUTING
═══════════════════════════════════════════════════════════════

uri:
  exact: "/api/v1/users"       exact string match
  prefix: "/api/v1"            prefix match
  regex: "/api/v[0-9]+/.*"     RE2 regex match

headers:
  "x-custom-header":
    exact: "value"
  "authorization":
    prefix: "Bearer "
  "x-region":
    regex: "us-.*"

method:
  exact: "GET"                 HTTP method match

queryParams:
  "version":
    exact: "v2"

authority:                     Host header
  exact: "api.company.com"

sourceLabels:                  Match on SOURCE pod labels
  app: mobile-frontend         traffic FROM pods with this label

sourceNamespace:               Traffic from specific namespace
  exact: "mobile-ns"

port: 8080                     Port the request is for

gateways:                      Only for traffic from this gateway
- my-gateway                   (needed when VS serves both mesh and ingress)

MULTIPLE MATCH CONDITIONS = AND logic
  match:
  - headers:           ← condition 1
      x-type: premium
    uri:               ← AND condition 2 (same list item)
      prefix: /api

MULTIPLE MATCH RULES = OR logic
  match:
  - headers:           ← match rule 1
      x-type: premium
  - uri:               ← OR match rule 2 (separate list item)
      prefix: /beta
```

---

### 🎤 Short Crisp Interview Answer

> *"VirtualService is Istio's L7 routing rule object. Where a Kubernetes Service is just 'send traffic to any pod matching this selector,' VirtualService lets you route based on HTTP headers, URI paths, source labels, or just weight percentages. Rules are evaluated in order, first match wins. The most common patterns: weighted canary split (90/10 between v1 and v2 subsets), header-based routing for A/B testing ('X-Beta-User: true' → v2), and adding retries and timeouts per route. VirtualService works with DestinationRule — VS says 'send 10% to subset v2,' DestinationRule says 'v2 means pods with label version=v2.' Without a matching DestinationRule defining the subset, traffic to that subset is dropped."*

---

---

# ⚠️ 11.6 DestinationRule — Load Balancing, TLS, Circuit Breaking

## 🟡 Intermediate — HIGH PRIORITY

### What DestinationRule Does

```
DESTINATIONRULE — UPSTREAM POLICY
═══════════════════════════════════════════════════════════════

DestinationRule defines policy for HOW to connect to a service.
Applied AFTER routing decision is made (after VirtualService picks destination).

DestinationRule controls:
  Subsets:         label-based pod groups ("v1" = pods with version=v1)
  Load balancing:  algorithm (ROUND_ROBIN, LEAST_CONN, RANDOM, PASSTHROUGH)
  Connection pool: max connections, max pending requests, max retries
  Outlier detection: automatic removal of unhealthy pods from load balancing
  TLS settings:    client TLS mode when connecting to upstream
  mTLS:            ISTIO_MUTUAL (use Istio certs) vs DISABLE vs SIMPLE

VIRTUALSERVICE + DESTINATIONRULE TOGETHER:
  VS: "send 10% of /api traffic to subset 'v2'"
  DR: "subset 'v2' = pods with label version=v2"
      "use LEAST_CONN load balancing for subset 'v2'"
  Together: 10% of /api traffic → least-connection across version=v2 pods
```

---

### DestinationRule YAML — Complete Reference

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-api-dr
  namespace: production
spec:
  host: my-api                 # K8s Service name this rule applies to

  # ── TRAFFIC POLICY (applies to ALL subsets unless overridden) ──
  trafficPolicy:

    # Load balancing algorithm
    loadBalancer:
      simple: LEAST_CONN       # ROUND_ROBIN, LEAST_CONN, RANDOM, PASSTHROUGH
      # OR: consistent hash (session affinity):
      # consistentHash:
      #   httpHeaderName: "x-user-id"     # hash on header value
      #   # OR: httpCookie, useSourceIp

    # Connection pool limits (prevents cascade failures)
    connectionPool:
      http:
        http1MaxPendingRequests: 100    # max queued requests (HTTP/1.1)
        http2MaxRequests: 1000         # max concurrent requests (HTTP/2)
        maxRequestsPerConnection: 10   # max requests per connection before close
        maxRetries: 3                  # max retries in flight across all hosts
        idleTimeout: 90s               # close idle connections after this
        h2UpgradePolicy: UPGRADE       # upgrade to HTTP/2 when possible
      tcp:
        maxConnections: 100            # max active TCP connections to service
        connectTimeout: 10s           # TCP connect timeout
        tcpKeepalive:
          time: 7200s                 # TCP keepalive interval
          interval: 75s

    # Outlier detection (circuit breaking on individual pods)
    outlierDetection:
      consecutiveGatewayErrors: 5     # eject pod after 5 consecutive 502/503/504
      consecutive5xxErrors: 5         # eject after 5 consecutive 5xx errors
      interval: 30s                   # how often to analyze for ejection
      baseEjectionTime: 30s           # minimum ejection duration
      maxEjectionPercent: 50          # never eject more than 50% of pods
      minHealthPercent: 30            # don't eject if < 30% would remain healthy

    # mTLS policy for connecting TO this service
    tls:
      mode: ISTIO_MUTUAL              # use Istio-managed mTLS certs
      # DISABLE:    no TLS (plain text)
      # SIMPLE:     one-way TLS (Envoy verifies server cert)
      # MUTUAL:     client cert from files (not Istio-managed)
      # ISTIO_MUTUAL: Istio-managed mTLS (most common for in-mesh)

  # ── SUBSETS (named pod groups) ────────────────────────────────
  subsets:
  - name: v1
    labels:
      version: v1              # selects pods with this label
    # trafficPolicy here overrides the global trafficPolicy for v1 only:
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN    # v1 uses round-robin (override)

  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 50  # tighter limits on canary
      outlierDetection:
        consecutiveGatewayErrors: 3   # eject v2 pods faster (stricter for canary)

  - name: stable
    labels:
      track: stable

---
# ── EXTERNAL SERVICE TLS ──────────────────────────────────────────
# When connecting to external services with TLS
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: external-api-dr
  namespace: production
spec:
  host: api.stripe.com         # external hostname (from ServiceEntry)
  trafficPolicy:
    tls:
      mode: SIMPLE             # one-way TLS (verify stripe.com cert)
      # caCertificates: /etc/certs/ca.pem  # custom CA if needed
```

---

### Circuit Breaking — How It Works

```
CIRCUIT BREAKING IN ISTIO
═══════════════════════════════════════════════════════════════

PROBLEM:
  Pod B is failing (returning 503s).
  Pod A keeps sending requests to Pod B.
  Pod B queue fills up → cascading failure.
  All of Pod A's threads blocked waiting for Pod B.
  Pod A becomes unavailable too → failure spreads.

ISTIO SOLUTION — TWO MECHANISMS:

1. CONNECTION POOL LIMITS (proactive, prevents overload):
   connectionPool.http.http1MaxPendingRequests: 100
   If > 100 requests queued to Pod B:
   → New requests immediately rejected with 503
   → Pod A gets fast failure instead of waiting
   → Pod A can fail fast or route to fallback

2. OUTLIER DETECTION (reactive, ejects failing pods):
   outlierDetection.consecutiveGatewayErrors: 5
   If Pod B returns 5 consecutive errors:
   → Pod B ejected from load balancing pool
   → Envoy stops sending to Pod B for baseEjectionTime (30s)
   → After 30s: Pod B gets 1 probe request
   → If succeeds: re-added to pool
   → If fails: ejected again for 60s (2x doubling)

COMBINED:
  Connection pool: reject requests when service is overloaded
  Outlier detection: eject specific failing pods from rotation

STATUS: WHERE ARE EJECTED HOSTS?
kubectl exec my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/clusters | grep "::cx_active"
# Shows active connections per upstream endpoint
# Ejected pods: "ejection_time" in stats

istioctl proxy-config endpoint my-pod.production \
  --cluster outbound|8080||my-api.production.svc.cluster.local
# ENDPOINT         STATUS      OUTLIER CHECK   CLUSTER
# 10.244.1.5:8080  HEALTHY     OK              outbound|8080||...
# 10.244.2.3:8080  UNHEALTHY   FAILED(5 GW)    outbound|8080||...  ← ejected!
```

---

### 🎤 Short Crisp Interview Answer

> *"DestinationRule has three main jobs. First, it defines named subsets — mapping a name like 'v2' to pods with label version=v2, which VirtualService references in routing rules. Second, it sets load balancing policy — ROUND_ROBIN, LEAST_CONN, or consistent hash for session affinity. Third, it configures circuit breaking via two mechanisms: connectionPool limits (reject requests fast when a service is overloaded — prevents thread exhaustion) and outlierDetection (automatically eject pods that return consecutive 5xx errors from the load balancing pool, restoring them after a timeout). The outlierDetection baseEjectionTime doubles on each consecutive ejection, so chronically failing pods get removed for increasingly longer periods."*

---

---

# 11.7 Gateway — Ingress via Istio

## 🟡 Intermediate

### Istio Gateway vs Kubernetes Ingress

```
ISTIO GATEWAY vs KUBERNETES INGRESS
═══════════════════════════════════════════════════════════════

Kubernetes Ingress:
  Single object: defines both LB/TLS (infra concern) and routing (app concern)
  HTTP only by default
  Controller-specific annotations for advanced features

Istio Gateway:
  Separation: Gateway = L4/TLS config, VirtualService = L7 routing
  Gateway handles: which ports, which protocols, which TLS certs
  VirtualService handles: which hostname routes to which service
  Supports: HTTP, HTTPS, TCP, TLS passthrough, gRPC
  Benefits: any Istio traffic management features work on ingress traffic
            (same VirtualService for internal AND external traffic)

Istio Gateway + VirtualService:
  Gateway says: "listen on port 443 for api.company.com, TLS with this cert"
  VirtualService says: "for api.company.com/v1 send to api-service v1 subset"
  Two separate YAML files, clear responsibility separation
```

---

### Gateway YAML

```yaml
# ── ISTIO GATEWAY (infrastructure concern) ───────────────────────
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: production-gateway
  namespace: production         # can be istio-system or any ns
spec:
  # Which Envoy deployment handles this gateway's traffic
  selector:
    istio: ingressgateway       # label on the ingress gateway pod
    # app: custom-gateway       # for custom gateway deployments

  servers:
  # ── HTTPS listener ──
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "api.company.com"
    - "admin.company.com"
    tls:
      mode: SIMPLE              # TLS termination at gateway
      credentialName: company-tls-secret  # K8s TLS Secret name
      # mode options:
      # PASSTHROUGH: don't terminate TLS, pass to backend (for backend TLS)
      # MUTUAL:      require client cert (mTLS with external clients)
      # SIMPLE:      terminate, no client cert required
      # AUTO_PASSTHROUGH: SNI routing, forward to pod with TLS intact
      minProtocolVersion: TLSV1_2

  # ── HTTP listener (redirect to HTTPS) ──
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "api.company.com"
    tls:
      httpsRedirect: true       # 301 redirect all HTTP → HTTPS

  # ── TCP listener (for non-HTTP services) ──
  - port:
      number: 5432
      name: postgres
      protocol: TCP
    hosts:
    - "*"

---
# ── VIRTUALSERVICE bound to Gateway ──────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-vs
  namespace: production
spec:
  hosts:
  - api.company.com             # must match Gateway host
  gateways:
  - production-gateway          # bind to our gateway
  - mesh                        # also route internal mesh traffic

  http:
  - match:
    - uri:
        prefix: /v1
    route:
    - destination:
        host: api-service
        subset: v1
        port:
          number: 8080
  - route:
    - destination:
        host: api-service
        subset: stable
        port:
          number: 8080
```

---

```bash
# Check gateway pods
kubectl get pods -n istio-system -l istio=ingressgateway
kubectl get svc -n istio-system istio-ingressgateway
# EXTERNAL-IP: a1b2c3d4.elb.amazonaws.com  ← point DNS here

# Check gateway status
kubectl get gateway -A
istioctl analyze -n production    # validate Gateway + VS config for issues

# TLS Certificate setup with cert-manager
kubectl create secret tls company-tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem \
  -n istio-system
# OR: use cert-manager Certificate CR targeting istio-system

# Debug: what Gateway config does Envoy have?
istioctl proxy-config listener \
  $(kubectl get pod -n istio-system -l istio=ingressgateway -o name | head -1) \
  -n istio-system

# Test gateway
curl -H "Host: api.company.com" \
  http://$(kubectl get svc -n istio-system istio-ingressgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/health
```

---

# ⚠️ 11.8 mTLS — PeerAuthentication, Automatic vs Strict Mode

## 🟡 Intermediate — HIGH PRIORITY

### What mTLS Provides

```
MTLS IN ISTIO — WHAT IT DOES
═══════════════════════════════════════════════════════════════

WITHOUT MTLS:
  Service A calls Service B
  B can't verify: who is calling me? (could be any pod, any attacker)
  A can't verify: is this really Service B? (could be a MITM)
  Traffic: plaintext on the wire

WITH ISTIO MTLS:
  Both sides authenticate using X.509 certificates issued by Istiod (Citadel)
  Certificate Subject: spiffe://cluster.local/ns/production/sa/my-service-account
                                              ↑ SPIFFE ID = cryptographic identity

  Authentication:
    A proves to B: "I am the pod running ServiceAccount my-api-sa in production"
    B proves to A: "I am the pod running ServiceAccount my-db-sa in production"
    Both verified by istiod's root CA

  Encryption:
    All traffic between A and B encrypted with ephemeral TLS keys
    Key rotation: Istiod-issued certs rotate every 24 hours by default

WORKLOAD IDENTITY:
  Identity tied to ServiceAccount, not IP address
  IP can change on pod restart → identity stays constant
  AuthorizationPolicy uses SPIFFE IDs → IP-independent authorization
  "Allow traffic from ServiceAccount api-sa in namespace production"
  ← this works even when pods restart and get new IPs

AUTOMATIC MTLS:
  Istio 1.5+: mTLS enabled by default in PERMISSIVE mode
  PERMISSIVE: accept both plaintext AND mTLS (migration-friendly)
  STRICT:     only mTLS accepted (fully zero-trust)
```

---

### PeerAuthentication YAML

```yaml
# ── CLUSTER-WIDE STRICT mTLS (ideal end state) ───────────────────
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system          # istio-system namespace = mesh-wide policy
spec:
  mtls:
    mode: STRICT                   # ALL traffic must be mTLS
    # PERMISSIVE: accept mTLS or plaintext (migration mode)
    # STRICT:     mTLS only (fully enforced)
    # DISABLE:    plaintext only (for debugging only)

---
# ── NAMESPACE-LEVEL (overrides cluster default for this namespace) ─
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: legacy-services       # only affects this namespace
spec:
  mtls:
    mode: PERMISSIVE               # legacy services still send plaintext

---
# ── WORKLOAD-SPECIFIC (overrides namespace default for these pods) ─
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: my-api-pa
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-api                  # only applies to these pods
  mtls:
    mode: STRICT
  # Port-level mtls override:
  portLevelMtls:
    8080:
      mode: STRICT
    9090:                          # metrics port: allow plaintext scraping
      mode: PERMISSIVE
```

---

### mTLS Migration Strategy

```
MIGRATION FROM PLAINTEXT TO STRICT MTLS
═══════════════════════════════════════════════════════════════

PHASE 1: Enable injection everywhere
  kubectl label namespace production istio-injection=enabled
  kubectl rollout restart deployment -n production
  All pods now have sidecars.
  Traffic: still plaintext (PERMISSIVE mode default)

PHASE 2: Verify all pods have sidecars
  kubectl get pods -n production
  # All pods show 2/2 READY ← sidecar injected
  # Any 1/1 pod = not injected = will fail when we go STRICT

PHASE 3: Enable STRICT mode (start with non-critical namespaces)
  kubectl apply -f - <<EOF
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: production
  spec:
    mtls:
      mode: STRICT
  EOF

PHASE 4: Verify mTLS enforcement
  # Check mTLS in Kiali: all edges should show "mTLS enabled" lock icon
  # OR: check from Envoy stats

  kubectl exec my-pod -n production -c istio-proxy -- \
    curl http://localhost:15000/stats | grep "ssl.handshake"
  # ssl.handshake: high number = mTLS working

  # Try connecting from a non-meshed pod (should fail)
  kubectl run plaintext-test --image=curlimages/curl --rm -it \
    --restart=Never -n default -- \
    curl http://my-api.production.svc.cluster.local:8080/health
  # Should return: "upstream connect error or disconnect/reset before headers"
  # This means STRICT is working — rejecting plaintext

PHASE 5: Cluster-wide STRICT (after all namespaces verified)
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: istio-system        # mesh-wide
  spec:
    mtls:
      mode: STRICT
```

---

### 🎤 Short Crisp Interview Answer

> *"Istio mTLS means every service-to-service call is mutually authenticated and encrypted. Istiod acts as the certificate authority, issuing X.509 certificates with SPIFFE IDs encoding the pod's ServiceAccount identity. Both sides verify each other's cert before any data flows. PeerAuthentication controls enforcement: PERMISSIVE (accept plaintext and mTLS, good for migration), STRICT (mTLS only, production target). The scope hierarchy: istio-system namespace = cluster-wide, namespace = that namespace, specific selector = those pods. For production, I target strict mode at the cluster level in istio-system, with namespace-level PERMISSIVE overrides for legacy services during migration. The certs rotate every 24 hours by default, and Envoy handles renewal transparently — the app sees none of it."*

---

---

# ⚠️ 11.9 AuthorizationPolicy — L7 RBAC

## 🟡 Intermediate — HIGH PRIORITY

### What AuthorizationPolicy Does

```
AUTHORIZATIONPOLICY — SERVICE MESH FIREWALL AT L7
═══════════════════════════════════════════════════════════════

NETWORKPOLICY (Kubernetes):   L3/L4 — allow/deny by IP and port
AUTHORIZATIONPOLICY (Istio):  L7 — allow/deny by service identity,
                              HTTP method, URL path, JWT claims

What NetworkPolicy cannot do:
  ✗ "Allow only GET /api/* but deny DELETE /admin/*"
  ✗ "Allow traffic from ServiceAccount api-sa (not just any pod on port 80)"
  ✗ "Allow JWT bearer tokens with claim role=admin to /admin"
  ✗ "Deny traffic from namespace staging to production"

What AuthorizationPolicy does:
  ✓ All of the above
  ✓ Evaluated per-request by Envoy (not per-connection like NetworkPolicy)
  ✓ Uses cryptographic identity (SPIFFE/mTLS cert), not IP addresses

ENFORCEMENT MODEL (same as NetworkPolicy):
  No policy on a workload: ALLOW all traffic (open by default)
  ANY DENY policy on a workload: explicitly denied traffic blocked
  ANY ALLOW policy on a workload: ONLY matching traffic allowed,
                                  everything else DENIED
  DENY evaluated before ALLOW (deny always wins)

POLICY SCOPE HIERARCHY:
  Mesh-wide:   namespace: istio-system
  Namespace:   selector: {} (all pods in namespace)
   or:         namespace: production (no selector = namespace-wide)
  Workload:    specific selector.matchLabels
```

---

### AuthorizationPolicy YAML — Complete Reference

```yaml
# ── STEP 1: Deny all (zero-trust baseline) ───────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}  # empty spec = deny ALL traffic to ALL workloads in namespace
  # action: DENY is default when no rules match

---
# ── STEP 2: Allow specific traffic ───────────────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  # WHO this policy protects (the server)
  selector:
    matchLabels:
      app: api               # policy applies TO api pods

  action: ALLOW              # ALLOW, DENY, or AUDIT

  rules:
  - from:
    # Source identity checks (mTLS required for these to work)
    - source:
        principals:          # SPIFFE IDs (ServiceAccount-based identity)
        - "cluster.local/ns/production/sa/frontend-sa"
        - "cluster.local/ns/mobile/sa/mobile-sa"
        # OR: namespaces (less specific)
        # namespaces: ["production", "staging"]
        # OR: ipBlocks (like NetworkPolicy, less preferred)
        # ipBlocks: ["10.0.0.0/8"]

    # Request-level checks
    to:
    - operation:
        methods: ["GET", "POST"]
        paths:               # URL path matching
        - "/api/*"
        - "/health"
        notPaths:            # NOT these paths (exclude /admin)
        - "/api/admin/*"
        ports: ["8080"]      # which port

    # Conditions (JWT claims, headers)
    when:
    - key: request.headers[x-internal-call]
      values: ["true"]       # only allow internal callers (header set by Envoy)

---
# ── JWT + RBAC (authenticated end-user requests) ─────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt-admin
  namespace: production
spec:
  selector:
    matchLabels:
      app: admin-api
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals:   # JWT subject (from verified JWT)
        - "*"                # any authenticated user
    to:
    - operation:
        methods: ["GET", "POST", "DELETE"]
        paths: ["/admin/*"]
    when:
    # JWT claim check
    - key: request.auth.claims[role]
      values: ["admin"]      # must have role=admin claim
    - key: request.auth.claims[org]
      values: ["company.com"]

---
# ── EXPLICIT DENY (evaluated before ALLOW) ────────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-staging-to-prod
  namespace: production
spec:
  action: DENY
  rules:
  - from:
    - source:
        namespaces: ["staging"]  # deny all traffic from staging namespace
  # No selector = applies to ALL workloads in production namespace

---
# ── ALLOW PROMETHEUS SCRAPING (common pattern) ────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-api
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["monitoring"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/metrics"]
        ports: ["9090"]

---
# ── MESH-WIDE DEFAULT DENY (strongest posture) ───────────────────
# In istio-system: applies across entire mesh
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: mesh-default-deny
  namespace: istio-system  # ← mesh-wide scope
spec:
  {}

# WARNING: this blocks ALL traffic including:
#   - health probes from kubelet (use portLevelMtls or exceptions)
#   - CoreDNS (add explicit allow for DNS)
#   - Prometheus scraping (add explicit allow)
# Only apply after adding all necessary ALLOW policies!
```

---

### RequestAuthentication — JWT Validation

```yaml
# RequestAuthentication configures JWT validation
# (separate from AuthorizationPolicy but works together)
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  jwtRules:
  - issuer: "https://auth.company.com"
    jwksUri: "https://auth.company.com/.well-known/jwks.json"
    # Envoy fetches JWKS (public keys) from this URL
    # Validates JWT signature, expiry, issuer
    audiences:
    - "api.company.com"
    forwardOriginalToken: true  # pass JWT to backend app
    outputClaimToHeaders:       # inject JWT claims as headers
    - header: x-user-id
      claim: sub
    - header: x-user-role
      claim: role

# RequestAuthentication: validates JWT if present, rejects invalid JWT
# BUT: does not require JWT (request without JWT still passes)
# To REQUIRE JWT: add AuthorizationPolicy with requestPrincipals: ["*"]
#   (requestPrincipal is only set when a valid JWT is present)
```

---

```bash
# Debug AuthorizationPolicy

# Check which policies apply
kubectl get authorizationpolicy -n production
kubectl describe authorizationpolicy allow-frontend-to-api -n production

# Test with istioctl
istioctl x authz check \
  $(kubectl get pod -l app=api -n production -o name | head -1) \
  -n production

# Check Envoy RBAC filter config
istioctl proxy-config listener \
  $(kubectl get pod -l app=api -n production -o name | head -1 | sed 's|pod/||') \
  -n production \
  --port 8080 -o json | grep -A 20 "rbac"

# Live test: send request and check if allowed
kubectl exec frontend-pod -n production -c frontend -- \
  curl -v http://api-service:8080/api/users
# 200 OK = allowed
# 403 Forbidden = AuthorizationPolicy blocked it

# Enable access log to see RBAC decisions
# In Envoy: access log shows "response_flags: UAEX" for RBAC deny
# UAEX = Unauthorized External Service (RBAC denied)
kubectl logs my-pod -n production -c istio-proxy | grep "UAEX"
```

---

### 🎤 Short Crisp Interview Answer

> *"AuthorizationPolicy is Istio's L7 RBAC — it controls which services can call which endpoints using cryptographic identity from mTLS certificates, not IP addresses. A policy with action: ALLOW and a selector means only matching traffic is permitted; everything else is denied. action: DENY is evaluated first — always wins. The most powerful feature is source.principals: the SPIFFE ID format encodes ServiceAccount identity, so 'allow traffic from ServiceAccount frontend-sa in namespace production' is a cryptographically verified check that survives pod restarts and IP changes. For authenticated users, RequestAuthentication validates JWTs, and then AuthorizationPolicy can check JWT claims like role=admin. The practical zero-trust pattern: namespace-wide deny-all, then explicit allow policies per communication path."*

---

---

# 11.10 Istio Observability — Kiali, Tracing, Metrics

## 🟡 Intermediate

### The Three Pillars

```
ISTIO OBSERVABILITY — THREE PILLARS
═══════════════════════════════════════════════════════════════

METRICS (via Envoy stats → Prometheus):
  Every Envoy sidecar exposes detailed stats at :15090/stats/prometheus
  Metrics collected per:
    - Request rate (requests per second)
    - Error rate (% 5xx responses)
    - Latency (p50, p95, p99 per service pair)
    - Connection pool stats (pending requests, active connections)
    - Circuit breaker stats (ejections)
  Standard labels: source_workload, destination_workload, response_code

DISTRIBUTED TRACING (via B3/W3C headers → Jaeger/Zipkin/Tempo):
  Envoy injects trace headers (x-b3-traceid, x-b3-spanid) on first hop
  Envoy creates a span for each proxy hop
  App MUST propagate headers (pass them through to outbound calls)
  End result: full trace across 10 services in a single view
  Tracing rate: typically 1-10% sampling (not 100% — too much data)

TOPOLOGY (via Kiali):
  Kiali reads metrics and traces from Prometheus + Jaeger
  Builds real-time service dependency graph
  Shows: traffic flow, error rates, latency, mTLS status, misconfigurations
  Detect: orphaned VirtualServices, missing DestinationRules, config errors
```

---

### Key Istio Metrics

```
STANDARD ISTIO METRICS (from Envoy sidecars)
═══════════════════════════════════════════════════════════════

REQUESTS:
  istio_requests_total
    Labels: source_workload, destination_service, response_code,
            response_flags, reporter (source or destination)
    Use: request rate, error rate by service pair

  istio_request_duration_milliseconds
    Labels: same as above + le (histogram bucket)
    Use: latency p50/p95/p99 per service

  istio_request_bytes_sum / _count   ← request payload sizes
  istio_response_bytes_sum / _count  ← response payload sizes

TCP (for non-HTTP services):
  istio_tcp_connections_opened_total
  istio_tcp_connections_closed_total
  istio_tcp_sent_bytes_total
  istio_tcp_received_bytes_total

CONTROL PLANE HEALTH:
  pilot_k8s_cfg_events         ← Istio config change events
  pilot_xds_pushes             ← xDS push rate to Envoys
  pilot_xds_push_time          ← how long xDS push takes
  pilot_conflict_outbound_listener ← config conflicts (missing DR)
  citadel_server_csr_count     ← certificate requests processed

RESPONSE FLAGS (important for debugging):
  UH: No healthy upstream (all pods down, circuit broken)
  UF: Upstream connection failure (TCP error)
  UO: Upstream overflow (connection pool exhausted → circuit breaker)
  NR: No route found (missing VirtualService or route rule)
  UAEX: Unauthorized (AuthorizationPolicy rejected)
  DC: Downstream connection terminated
  RL: Rate limited
```

---

### Prometheus Queries for Istio

```bash
# ── KEY PROMETHEUS QUERIES ────────────────────────────────────────

# Request rate per service (requests/sec)
rate(istio_requests_total{reporter="source"}[5m])

# Error rate per service (% 5xx)
sum(rate(istio_requests_total{
  reporter="source",
  destination_service="api-service.production.svc.cluster.local",
  response_code=~"5.*"
}[5m])) /
sum(rate(istio_requests_total{
  reporter="source",
  destination_service="api-service.production.svc.cluster.local"
}[5m]))

# P99 latency for a service
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{
    reporter="source",
    destination_service="api-service.production.svc.cluster.local"
  }[5m])) by (le)
)

# Circuit breaker ejections
rate(envoy_cluster_outlier_detection_ejections_total[5m])

# Pending requests (connection pool pressure)
envoy_cluster_upstream_rq_pending_total{
  cluster_name=~"outbound|8080||api.*"
}

# xDS config push latency (control plane health)
histogram_quantile(0.99,
  rate(pilot_xds_push_time_bucket[5m])
)

# mTLS status (% of requests using mTLS)
sum(rate(istio_requests_total{
  connection_security_policy="mutual_tls"
}[5m])) /
sum(rate(istio_requests_total[5m]))
```

---

### Distributed Tracing

```
TRACING — HOW IT WORKS WITH ISTIO
═══════════════════════════════════════════════════════════════

WHAT ENVOY DOES AUTOMATICALLY:
  Request enters mesh: Envoy generates trace ID + span
  Request exits a sidecar: Envoy creates a new span
  Span data sent to Jaeger/Zipkin/Tempo backend

WHAT YOUR APP MUST DO:
  When App A calls App B, it MUST forward these headers:
  x-request-id
  x-b3-traceid
  x-b3-spanid
  x-b3-parentspanid
  x-b3-sampled
  x-b3-flags
  b3 (single header format)
  traceparent (W3C format, newer)

  If your app doesn't forward these headers:
  → Each service shows as a separate disconnected trace
  → No end-to-end trace visibility
  → Very common gotcha: app handles tracing incorrectly

PROPAGATION EXAMPLE (Go):
  func handler(w http.ResponseWriter, r *http.Request) {
      // Extract incoming trace headers
      headers := map[string]string{
          "x-request-id":      r.Header.Get("x-request-id"),
          "x-b3-traceid":      r.Header.Get("x-b3-traceid"),
          "x-b3-spanid":       r.Header.Get("x-b3-spanid"),
          "x-b3-parentspanid": r.Header.Get("x-b3-parentspanid"),
          "x-b3-sampled":      r.Header.Get("x-b3-sampled"),
      }
      // Forward when calling downstream
      req, _ := http.NewRequest("GET", "http://downstream-service/api", nil)
      for k, v := range headers {
          req.Header.Set(k, v)
      }
      http.DefaultClient.Do(req)
  }

SAMPLING RATES:
  100% sampling: excellent visibility, massive data volume, high cost
  1% sampling:   minimal overhead, miss rare errors
  Recommended: 1-10% with tail-based sampling (always sample errors)
  Jaeger supports: probabilistic, rate-limiting, remote-controlled sampling
```

---

### Kiali

```bash
# Access Kiali dashboard
kubectl port-forward -n istio-system svc/kiali 20001:20001
# Open: http://localhost:20001

# Kiali features:
# 1. Service graph with real-time traffic
#    - Green edges: healthy traffic
#    - Red edges: error rate above threshold
#    - Lock icon: mTLS enabled
#    - Dashed edge: missing mTLS
# 2. Namespace overview: health at a glance
# 3. Config validation: detects misconfigurations
#    - "VirtualService has no matching DestinationRule"
#    - "DestinationRule references subset not in VirtualService"
#    - "Gateway references port not on ingress"
# 4. Traffic replay: replay traces in Jaeger
# 5. Workload health: pod status + proxy status

# Validate Istio config from CLI (same checks as Kiali)
istioctl analyze --all-namespaces
# Info [IST0102] (VirtualService my-api.production) VirtualService host
#   'nonexistent' not found
# Warning [IST0108] (DestinationRule my-api-dr.production) DestinationRule
#   subset 'v3' not found in service my-api

# Check Envoy stats for a service
kubectl exec my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/stats | \
  grep "outbound|8080||api-service"
```

---

---

# ⚠️ 11.11 Traffic Management — Canary, Blue/Green, Mirroring

## 🟡 Intermediate — HIGH PRIORITY

### Canary Deployment Pattern

```
CANARY DEPLOYMENT WITH ISTIO
═══════════════════════════════════════════════════════════════

GOAL: gradually shift traffic from v1 to v2, monitor, roll back if needed.

ADVANTAGE OVER KUBERNETES NATIVE CANARY:
  Kubernetes native: canary = percentage of pods
    5 pods v1 + 1 pod v2 = ~17% to v2 (determined by pod count)
    To change ratio: scale pod counts (wasteful)

  Istio canary: decouple traffic percentage from pod count
    5 pods v1 + 5 pods v2, but 90%/10% traffic split via VirtualService
    Change ratio: update VirtualService weight (no pod scaling needed)
    Can be: 99/1, 95/5, 90/10, 50/50, 0/100

CANARY PROGRESSION:
  Week 1: 5% to v2   → monitor error rate, latency
  Week 2: 10% to v2  → continue monitoring
  Week 3: 25% to v2  → expanding canary
  Week 4: 50% to v2  → half traffic
  Week 5: 100% to v2 → full rollout
  Any week: if error rate spikes → change weight to 0% immediately

AUTOMATED WITH ARGO ROLLOUTS:
  Argo Rollouts integrates with Istio VirtualService
  Automatically adjusts weights based on analysis metrics
  Automatic rollback if Prometheus metrics breach thresholds
```

---

### Canary YAML

```yaml
# SETUP: Both versions deployed as separate Deployments
# with different labels (version: v1 / version: v2)

# ── DestinationRule: define v1 and v2 subsets ────────────────────
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-api-dr
  namespace: production
spec:
  host: my-api
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

---
# ── VirtualService: 90/10 canary split ───────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-api-vs
  namespace: production
spec:
  hosts:
  - my-api
  http:
  - route:
    - destination:
        host: my-api
        subset: v1
      weight: 90
    - destination:
        host: my-api
        subset: v2
      weight: 10
    # Add retries to ensure user doesn't see v2 errors:
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: "5xx,reset,connect-failure"

---
# PHASE: Move to 100% v2 (after validation)
# Just change weights:
# weight: 0  (v1)
# weight: 100 (v2)

# ROLLBACK: change weights back to 100/0

# CLEANUP: remove v2 subset from DR, delete v2 Deployment
```

---

### Blue/Green Deployment Pattern

```yaml
# BLUE/GREEN: instant switch between two complete deployments
# Blue = current production (v1)
# Green = new version (v2) — deployed alongside but getting NO traffic

# ── DestinationRule ───────────────────────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-api-dr
  namespace: production
spec:
  host: my-api
  subsets:
  - name: blue
    labels:
      slot: blue      # pods labeled slot=blue
  - name: green
    labels:
      slot: green     # pods labeled slot=green

---
# ── VirtualService: currently all traffic to blue ─────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-api-vs
  namespace: production
spec:
  hosts:
  - my-api
  http:
  - route:
    - destination:
        host: my-api
        subset: blue
      weight: 100    # ALL to blue
    - destination:
        host: my-api
        subset: green
      weight: 0      # green deployed but receiving no traffic

# SWITCH TO GREEN (zero-downtime):
# kubectl patch virtualservice my-api-vs -n production \
#   --type=json \
#   -p '[{"op":"replace","path":"/spec/http/0/route/0/weight","value":0},
#        {"op":"replace","path":"/spec/http/0/route/1/weight","value":100}]'

# ROLLBACK TO BLUE (immediate):
# Same patch but reverse weights — takes effect in milliseconds

# ADVANTAGE OF BLUE/GREEN:
#   Instant switch (not gradual like canary)
#   Full green environment tested before switch
#   Instant rollback
# DISADVANTAGE:
#   2x resource cost (both versions fully deployed)
#   Users in flight get 50x responses if they hit wrong version mid-request
```

---

### Traffic Mirroring (Shadowing)

```yaml
# TRAFFIC MIRRORING: send real traffic to v2 without impacting users
# v1 still serves user-facing requests (authoritative response)
# v2 receives a copy of every request asynchronously
# v2 responses are DISCARDED (user never sees them)
# Purpose: test v2 with production traffic without risk

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-api-vs
  namespace: production
spec:
  hosts:
  - my-api
  http:
  - route:
    - destination:
        host: my-api
        subset: v1
      weight: 100           # users always get v1 response

    mirror:                 # copy traffic to v2 asynchronously
      host: my-api
      subset: v2
    mirrorPercentage:
      value: 100.0          # mirror 100% (or less for sampling)

# WHAT HAPPENS:
#   Request to my-api:
#   1. Envoy forwards to v1 (authoritative)
#   2. Envoy simultaneously copies request to v2 (fire-and-forget)
#   3. v2 receives: X-Forwarded-Host: my-api-shadow (marked as mirror)
#   4. v2 response: discarded
#   5. User gets v1 response

# MONITORING v2 SHADOW:
#   Check v2 logs: see production-realistic requests
#   Check v2 error rates in Prometheus (tagged with destination=v2)
#   Compare v1 vs v2 latency and error rates side-by-side

# WHEN TO USE:
#   Validate new version with production traffic patterns
#   Catch bugs that only appear with real data
#   Performance benchmarking under realistic load
#   Database schema changes (verify v2 can handle prod queries)
```

---

### 🎤 Short Crisp Interview Answer

> *"Istio's traffic management decouples traffic percentage from pod count — the biggest improvement over Kubernetes-native rollouts. For canary, I define v1 and v2 subsets in a DestinationRule, then use VirtualService weights to shift traffic gradually (5% → 10% → 25% → 100%) while monitoring error rates in Prometheus. Changing the split is a VirtualService patch — no pod scaling needed. Blue/green is an instant switch: both environments are always deployed, weight is flipped 0→100 atomically, rollback is reversing the weights in milliseconds. Traffic mirroring is the safest approach: v1 serves all user traffic, Envoy asynchronously copies every request to v2, v2 responses are discarded. You get production-realistic v2 metrics without any user impact — great for validating behavior with real data before any exposure."*

---

---

# 11.12 ServiceEntry — Bringing External Services into the Mesh

## 🟡 Intermediate

### Why ServiceEntry Exists

```
SERVICEENTRY — EXTENDING THE MESH BOUNDARY
═══════════════════════════════════════════════════════════════

BY DEFAULT (REGISTRY_ONLY mode):
  Istio can block all external traffic if configured strictly.
  Or with default mode: external traffic passes through but
  Envoy sees it as "unknown" and applies no policy.

WITH SERVICEENTRY:
  Register an external service (Stripe, Twilio, RDS, S3) as a
  first-class citizen in the mesh:
  - Envoy knows the hostname → applies routing rules
  - Can configure TLS origination (Envoy does TLS, app sends plain HTTP)
  - Can apply retries, timeouts, circuit breaking
  - Metrics tracked: see external call rates and latencies
  - Traffic auditing: log all calls to external payment APIs

USE CASES:
  1. Reach external APIs (api.stripe.com, hooks.slack.com)
  2. Reach external databases (RDS, managed Redis)
  3. Reach services in other clusters without Cluster Mesh
  4. Apply mTLS exit (Envoy adds client cert when calling external endpoint)
  5. Monitor external service dependency health
```

---

### ServiceEntry YAML

```yaml
# ── EXTERNAL HTTPS API ────────────────────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: stripe-api
  namespace: production
spec:
  hosts:
  - api.stripe.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL          # external to mesh
  resolution: DNS                  # resolve via DNS (not static IPs)

---
# DestinationRule for the external service (TLS settings)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: stripe-api-dr
  namespace: production
spec:
  host: api.stripe.com
  trafficPolicy:
    tls:
      mode: SIMPLE                 # Envoy verifies stripe.com's TLS cert
    connectionPool:
      http:
        http1MaxPendingRequests: 100
    outlierDetection:
      consecutiveGatewayErrors: 3
      interval: 30s
      baseEjectionTime: 60s       # eject Stripe for 60s after 3 failures

---
# ── EXTERNAL DATABASE (static IPs) ────────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: aws-rds
  namespace: production
spec:
  hosts:
  - mydb.cluster-abc123.us-east-1.rds.amazonaws.com
  ports:
  - number: 5432
    name: postgres
    protocol: TCP
  location: MESH_EXTERNAL
  resolution: DNS

---
# ── TLS ORIGINATION (app sends plaintext, Envoy adds TLS) ─────────
# Pattern: app calls http://external-service, Envoy upgrades to HTTPS
# Useful when: app can't do TLS itself, or you want centralized cert mgmt
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api-entry
  namespace: production
spec:
  hosts:
  - external-api.company.com
  ports:
  - number: 80                    # app calls port 80 (plain HTTP)
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL

---
# DestinationRule: upgrade the connection to HTTPS at Envoy level
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: external-api-origination
  namespace: production
spec:
  host: external-api.company.com
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 80
      tls:
        mode: SIMPLE             # Envoy does TLS on the port 80 outbound connection
        sni: external-api.company.com

---
# VirtualService: redirect port 80 to port 443
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: external-api-vs
  namespace: production
spec:
  hosts:
  - external-api.company.com
  http:
  - match:
    - port: 80
    route:
    - destination:
        host: external-api.company.com
        port:
          number: 443

---
# ── BLOCK ALL EXTERNAL TRAFFIC (egress control) ───────────────────
# Configure Istio to block unknown external traffic (REGISTRY_ONLY)
# Any external host NOT in a ServiceEntry → blocked
# Add this to IstioOperator or helm values:
#
# meshConfig:
#   outboundTrafficPolicy:
#     mode: REGISTRY_ONLY       # only allow defined ServiceEntries
#     # ALLOW_ANY (default): allow all external traffic
```

---

```bash
# Verify ServiceEntry is working
kubectl get serviceentry -n production
kubectl describe serviceentry stripe-api -n production

# Check Envoy clusters — external service should appear
istioctl proxy-config cluster my-pod.production | grep stripe

# Test egress to external service
kubectl exec my-pod -n production -c app -- \
  curl -v https://api.stripe.com/v1/charges \
  -H "Authorization: Bearer sk_test_..."

# Check metrics for external service calls
# In Prometheus:
# rate(istio_requests_total{
#   destination_service="api.stripe.com"
# }[5m])

# With REGISTRY_ONLY: test that unknown external traffic is blocked
kubectl exec my-pod -n production -c app -- \
  curl https://unknown-external.com
# Should get: upstream connect error (blocked by Envoy)
```

---

---

# 11.13 Istio Control Plane Internals — Istiod, Pilot, Citadel, Galley

## 🔴 Advanced

### Istiod — Consolidated Control Plane

```
ISTIO HISTORY AND ISTIOD CONSOLIDATION
═══════════════════════════════════════════════════════════════

PRE-1.5 ISTIO (three separate components):
  Pilot:    service discovery + xDS config generation
  Citadel:  certificate authority (CA) + secret management
  Galley:   config validation + distribution
  Mixer:    telemetry + policy (deprecated in 1.8, removed in 1.9)
  Problems: operational complexity, inter-component latency,
            multiple deployments to manage and scale

ISTIOD (Istio 1.5+): single binary containing all three
  Simpler: one Deployment, one Service, one certificate chain
  Faster:  no inter-component gRPC calls (all in-process)
  Lighter: ~50% reduction in control plane resources
  HA: run 2-3 replicas with leader election for critical functions

WHAT ISTIOD DOES:
  1. API Server Watch: watches Services, Endpoints, Pods, ConfigMaps,
     and all Istio CRDs (VirtualService, DestinationRule, etc.)
  2. xDS Config Generation: translates K8s + Istio objects into
     Envoy configuration (listeners, routes, clusters, endpoints)
  3. xDS Push: streams config to Envoy sidecars via gRPC
  4. Certificate Authority: signs CSRs from Envoy sidecars
     Issues SPIFFE X.509 certificates (24h TTL by default)
  5. Config Validation: MutatingWebhook + ValidatingWebhook for
     Istio resources (VirtualService, DestinationRule, etc.)
  6. Service Discovery: maintains registry of all mesh endpoints
```

---

### Pilot — Service Discovery and xDS

```
PILOT INTERNALS (inside istiod)
═══════════════════════════════════════════════════════════════

PILOT'S JOB:
  1. Maintain service registry (all K8s Services + Endpoints + Pods)
  2. Process Istio configs (VirtualService, DestinationRule, etc.)
  3. Generate Envoy config (xDS) from #1 + #2
  4. Push xDS to all connected Envoy proxies

SERVICE REGISTRY SOURCES:
  Primary:    Kubernetes API (Services, EndpointSlices, Pods, Nodes)
  Additional: Consul, Cloud Foundry (via MCP adapters — rare)

PILOT PUSH TRIGGERS:
  A Service changes → push new EDS (endpoints) to relevant Envoys
  A VirtualService changes → push new RDS (routes) to Envoys in that namespace
  A DestinationRule changes → push new CDS (clusters) to relevant Envoys
  A new pod with sidecar starts → Envoy connects, receives full config

INCREMENTAL xDS (delta xDS):
  Before: full config pushed on any change (expensive at scale)
  After (delta xDS): only changed resources pushed
  1,000 Services, 1 changes → push only the changed cluster/route
  Massive reduction in push size at scale (10,000+ services)

PILOT METRICS TO WATCH:
  pilot_xds_push_time:          how long pushes take (p99 should be <1s)
  pilot_xds_pushes:             push rate (spikes = config churn)
  pilot_conflict_outbound_listener: config conflicts (usually DR issues)
  pilot_total_xds_connections:  how many Envoys are connected
  pilot_xds_eds_instances:      total endpoint instances tracked
```

---

### Citadel — Certificate Authority

```
CITADEL (CERTIFICATE AUTHORITY) INSIDE ISTIOD
═══════════════════════════════════════════════════════════════

CERTIFICATE ISSUANCE FLOW:
  1. Pod starts, Envoy sidecar boots
  2. Envoy generates private key (never leaves pod)
  3. Envoy sends CSR (Certificate Signing Request) to istiod port 15012
     via mTLS (uses bootstrap cert from mounted secret)
  4. Istiod (Citadel) verifies: pod's ServiceAccount identity via K8s
  5. Istiod signs the CSR → issues X.509 cert
  6. Cert subject: spiffe://cluster.local/ns/{namespace}/sa/{service-account}
  7. Envoy uses cert for mTLS with other Envoys

CERT LIFECYCLE:
  Default cert TTL: 24 hours
  Rotation: Envoy requests new cert 12 hours before expiry
  Automatic: app/Envoy restart not needed for cert rotation

PLUGGABLE CA:
  Default: istiod as self-signed CA (fine for most clusters)
  Production option: plug in external CA (Vault, AWS ACM PCA, GCP CAS)
  External CA: istiod acts as intermediate CA,
               root CA is your PKI infrastructure
  Benefit: cert chain traces back to enterprise trust anchor

CHECKING CERTS:
  istioctl proxy-config secret my-pod.production
  # RESOURCE NAME        TYPE           STATUS     VALID CERT  SERIAL NUMBER
  # default              Cert Chain     ACTIVE     true        abc123def456
  # ROOTCA               CA             ACTIVE     true        rootcertserial

  kubectl exec my-pod -n production -c istio-proxy -- \
    curl http://localhost:15000/certs
  # Shows all certs held by this Envoy: workload cert + root CA
```

---

---

# ⚠️ 11.14 xDS API — How Envoy Gets Its Config

## 🔴 Advanced — HIGH PRIORITY

### xDS Fundamentals

```
XDS API — ENVOY'S CONFIGURATION PROTOCOL
═══════════════════════════════════════════════════════════════

xDS = "x Discovery Service" — a family of gRPC APIs that
      management servers (like istiod) use to configure Envoy.

THE FIVE CORE xDS APIS:
  LDS  Listener Discovery Service
       → Listeners: which ports Envoy listens on, which filter chains
       → Example: "Listen on port 15001, apply HTTP filters"

  RDS  Route Discovery Service
       → Routes: which HTTP request goes to which cluster
       → Example: "GET /api/* → api-v1-cluster, 10% to api-v2-cluster"

  CDS  Cluster Discovery Service
       → Clusters: upstream service definitions (connection config)
       → Example: "api-service: circuit breaker + LEAST_CONN LB"

  EDS  Endpoint Discovery Service
       → Endpoints: actual pod IP:port for each cluster
       → Example: "api-service endpoints: [10.244.1.5:8080, 10.244.2.3:8080]"

  SDS  Secret Discovery Service
       → Secrets: TLS certificates and keys
       → Example: "workload cert for this pod, root CA bundle"

FLOW (how it all connects):
  Listener (LDS) → uses FilterChain → references Route (RDS)
  Route (RDS) → references Cluster (CDS)
  Cluster (CDS) → references Endpoints (EDS)
  FilterChain → references TLS certs (SDS)

ENVOY BOOTSTRAP PROCESS:
  1. Envoy starts with minimal bootstrap config (istiod address, token)
  2. Envoy connects to istiod:15012 via gRPC (mTLS with bootstrap cert)
  3. Envoy sends DiscoveryRequest: "give me all LDS resources"
  4. istiod responds: DiscoveryResponse with all listener configs
  5. Envoy ACKs or NACKs (if config is invalid, sends NACK with error)
  6. Envoy sends DiscoveryRequest: "give me all RDS, CDS, EDS, SDS"
  7. istiod sends DiscoveryResponse for each
  8. This initial sync completes: Envoy is fully configured
  9. Istiod streams updates: any change → push to relevant Envoys
```

---

### xDS Request/Response Protocol

```
XDS PROTOCOL DETAIL
═══════════════════════════════════════════════════════════════

DISCOVERY REQUEST:
  {
    "version_info": "abc123",          ← version Envoy currently has
    "node": {
      "id": "sidecar~10.244.1.5~my-pod.production~production.svc.cluster.local",
      "cluster": "production",
      "metadata": {
        "ISTIO_VERSION": "1.20.0",
        "NAMESPACE": "production",
        "SERVICE_ACCOUNT": "my-api-sa"
      }
    },
    "resource_names": [],              ← empty = subscribe to all
    "type_url": "type.googleapis.com/envoy.config.listener.v3.Listener"
  }

DISCOVERY RESPONSE:
  {
    "version_info": "xyz789",         ← new version
    "resources": [
      { Listener config 1 },
      { Listener config 2 },
      ...
    ],
    "type_url": "type.googleapis.com/envoy.config.listener.v3.Listener",
    "nonce": "nonce-abc"              ← must ACK with this nonce
  }

ACK (Envoy applied the config):
  { "version_info": "xyz789", "response_nonce": "nonce-abc" }

NACK (Envoy rejected — config error):
  {
    "version_info": "abc123",         ← still on old version
    "response_nonce": "nonce-abc",
    "error_detail": {
      "message": "filter chain conflict on port 8080"
    }
  }
  Istiod sees NACK → logs error → does NOT roll back (Envoy keeps old config)
  Critical: NACK means your Istio config has an error
```

---

### Debugging xDS

```bash
# ── CHECK XDS SYNC STATUS ─────────────────────────────────────────
# Is this Envoy in sync with istiod?
istioctl proxy-status
# NAME                          CLUSTER   CDS   LDS   EDS   RDS   ECDS  ISTIOD
# my-pod-abc.production         prod      SYNCED SYNCED SYNCED SYNCED NOT SENT  istiod-xyz
# other-pod.production          prod      STALE  SYNCED SYNCED SYNCED NOT SENT  istiod-xyz
#                                          ↑ STALE = Envoy has old CDS config

# NOT SENT = istiod hasn't pushed this type (normal for some resource types)
# STALE    = push in progress or Envoy hasn't ACKed yet
# ERROR    = Envoy NACKed the config (config error!)

# Check specific Envoy's xDS connection details
istioctl proxy-config cluster my-pod.production
# Shows all CDS clusters Envoy currently has configured

# Full raw config dump from Envoy
kubectl exec my-pod -n production -c istio-proxy -- \
  curl http://localhost:15000/config_dump | \
  python3 -c "
import json, sys
cfg = json.load(sys.stdin)
for section in cfg['configs']:
    print(section['@type'])
"
# type.googleapis.com/envoy.admin.v3.BootstrapConfigDump
# type.googleapis.com/envoy.admin.v3.ListenersConfigDump
# type.googleapis.com/envoy.admin.v3.ClustersConfigDump
# type.googleapis.com/envoy.admin.v3.RoutesConfigDump
# type.googleapis.com/envoy.admin.v3.SecretsConfigDump

# Check istiod xDS push metrics
kubectl exec -n istio-system \
  $(kubectl get pod -n istio-system -l app=istiod -o name | head -1) -- \
  curl http://localhost:15014/metrics | \
  grep "pilot_xds"
# pilot_xds_pushes{type="cds"} 1234
# pilot_xds_push_time_bucket{type="lds", le="0.01"} 987  ← <10ms
# pilot_xds_errors_total 0  ← should be 0!

# View istiod internal state
kubectl port-forward -n istio-system svc/istiod 15014:15014
curl http://localhost:15014/debug/registryz    # service registry
curl http://localhost:15014/debug/configz      # processed Istio configs
curl http://localhost:15014/debug/pushstatus   # recent xDS push history
curl http://localhost:15014/debug/endpointz    # all endpoints istiod knows
```

---

### 🎤 Short Crisp Interview Answer

> *"xDS is the gRPC-based API protocol between istiod and Envoy. There are five discovery services: LDS (listeners — which ports to listen on), RDS (routes — which request goes where), CDS (clusters — upstream service config), EDS (endpoints — actual pod IPs), and SDS (secrets — TLS certs). On startup, Envoy connects to istiod and sends DiscoveryRequests for each type. Istiod responds with DiscoveryResponses containing the config, and Envoy ACKs if it successfully applied them or NACKs with an error message if not. NACKs are how you know your Istio config has an error — check istioctl proxy-status for STALE or ERROR states. After initial sync, istiod streams incremental updates — only changed resources are pushed, avoiding full-config pushes on every small change."*

---

---

# 11.15 Istio Ambient Mesh — Sidecarless Architecture

## 🔴 Advanced

### Why Ambient Mesh

```
SIDECAR PROBLEMS AMBIENT MESH SOLVES
═══════════════════════════════════════════════════════════════

SIDECAR (traditional) problems:
  Memory overhead: ~50-100MB per pod (in a 1,000-pod cluster: 50-100GB!)
  CPU overhead: ~0.5-1 vCPU per pod
  Startup latency: pods don't get traffic until Envoy is ready
  Restart coupling: Envoy and app restart together (version coupling)
  Injection complexity: MutatingWebhook required, init container, iptables
  Blast radius: Envoy bug affects every pod in the cluster

AMBIENT MESH APPROACH:
  No sidecar injected into pods.
  Two layers handle the mesh functionality instead:

  LAYER 1 — ztunnel (zero-trust tunnel):
    A DaemonSet — one ztunnel pod per node
    Handles: mTLS encryption, L4 authorization (ALLOW/DENY by identity)
    No L7: doesn't parse HTTP, no retries, no header manipulation
    Very lightweight: ~50MB per node (vs per pod!)

  LAYER 2 — waypoint proxy (per namespace, optional):
    A deployment of Envoy proxies, NOT per pod
    Handles: L7 routing, retries, circuit breaking, AuthorizationPolicy
    Created per namespace (or per service) only when L7 is needed
    Shared: all pods in namespace share waypoint proxies

AMBIENT vs SIDECAR COMPARISON:
  Sidecar:  every pod + Envoy (~100MB × 1,000 pods = 100GB)
  Ambient:  N nodes × ztunnel (~50MB × 100 nodes = 5GB)
            + waypoint proxy per namespace needing L7 (shared)
  Memory savings: typically 60-80% reduction
```

---

### Ambient Architecture

```
AMBIENT MESH TRAFFIC FLOW
═══════════════════════════════════════════════════════════════

L4 PATH (mTLS only, no waypoint needed):
  Pod A → ztunnel on Node A → HBONE tunnel (mTLS) → ztunnel on Node B → Pod B
  HBONE = HTTP-Based Overlay Network Encapsulation (HTTP/2 tunnel with mTLS)
  ztunnel: intercepts pod traffic via eBPF or iptables redirection
  No HTTP parsing, just L4 proxy with mTLS termination

L7 PATH (with waypoint proxy, when L7 features needed):
  Pod A → ztunnel → HBONE → waypoint proxy → ztunnel → Pod B
  Waypoint proxy: full Envoy, handles all L7 features
  Same VirtualService, DestinationRule, AuthorizationPolicy objects!

ENABLING AMBIENT:
  # Enable ambient for a namespace (no injection label needed!)
  kubectl label namespace production istio.io/dataplane-mode=ambient

  # For L7 features: create a waypoint proxy
  istioctl x waypoint apply --namespace production
  # Creates: deployment/waypoint in production namespace
  # All L7 traffic in namespace routes through waypoint

  # Check ambient enrollment
  istioctl x ztunnel-config workload

MIGRATING FROM SIDECAR TO AMBIENT:
  # Remove injection label
  kubectl label namespace production istio-injection-
  kubectl label namespace production istio.io/dataplane-mode=ambient
  # No pod restart needed! ztunnel intercepts existing pod traffic
  # Sidecar injected pods still work (ambient + sidecar coexist)
```

---

### Ambient Status (2024)

```
AMBIENT MESH MATURITY STATUS
═══════════════════════════════════════════════════════════════

Introduced: Istio 1.15 (September 2022, alpha)
Beta:        Istio 1.22 (May 2024)
Production ready: expected Istio 1.24+ / late 2024

SUPPORTED FEATURES (Beta):
  ✓ mTLS (via ztunnel)
  ✓ L4 AuthorizationPolicy
  ✓ L7 AuthorizationPolicy (via waypoint)
  ✓ HTTP VirtualService routing (via waypoint)
  ✓ DestinationRule (via waypoint)
  ✓ Telemetry (metrics + traces via waypoint)

STILL IN PROGRESS / LIMITATIONS:
  ⚠ Some advanced VirtualService features not supported
  ⚠ Multi-cluster ambient: in progress
  ⚠ IPv6: limited support
  ⚠ Windows nodes: not supported

WHEN TO CHOOSE AMBIENT:
  New cluster, resource cost is a concern, on Kubernetes 1.26+
  Large number of pods where per-sidecar overhead is significant
  Teams that find sidecar injection operational complexity challenging

WHEN TO STICK WITH SIDECAR:
  Production cluster that already uses sidecars (migration risk)
  Need features not yet supported by ambient
  Strict compliance requiring per-pod isolation at proxy level
```

---

---

# 11.16 Multi-Cluster Istio

## 🔴 Advanced

### Multi-Cluster Models

```
MULTI-CLUSTER ISTIO ARCHITECTURES
═══════════════════════════════════════════════════════════════

PRIMARY-REMOTE (most common):
  Primary cluster: runs istiod + data plane
  Remote cluster:  runs data plane only (no istiod)
  Remote Envoys connect to primary cluster's istiod
  Remote cluster API Server watched by primary istiod
  Use when: you want single control plane managing multiple clusters
  Limitation: primary failure = all remote clusters lose control plane

MULTI-PRIMARY (high availability):
  Each cluster runs its own istiod
  Each istiod manages its own cluster's data plane
  Cross-cluster service discovery: shared API Server or endpoint syncing
  Use when: strong isolation between clusters needed, multi-region HA
  Benefit: each cluster independent, istiod failure isolated

SINGLE NETWORK vs MULTIPLE NETWORKS:
  Single network:  pod IPs routable across clusters (flat network)
                   Envoys connect directly to remote pod IPs
  Multiple networks: separate pod CIDRs, not directly routable
                     East-west gateways bridge the networks
                     Cross-cluster traffic goes: pod → EW gateway → pod

EAST-WEST GATEWAY:
  A dedicated Istio Gateway for cross-cluster traffic
  Terminates inbound mTLS from other clusters
  Routes to local services
  kubectl get svc -n istio-system istio-eastwestgateway
  # EXTERNAL-IP: a1b2.elb.amazonaws.com  ← other clusters connect here
```

---

### Multi-Primary Setup

```bash
# MULTI-PRIMARY SETUP (two clusters, flat network)

# Step 1: Configure cluster names
kubectl config use-context cluster1
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
  --from-file=ca-cert.pem \
  --from-file=ca-key.pem \
  --from-file=root-cert.pem \
  --from-file=cert-chain.pem
# Shared root CA: both clusters must trust each other's workload certs
# They share the same root CA but have different intermediate CAs

# Step 2: Install Istio on cluster1
helm install istiod istio/istiod \
  --namespace istio-system \
  --set global.meshID=mesh1 \
  --set global.multiCluster.clusterName=cluster1 \
  --set global.network=network1

# Step 3: Install Istio on cluster2
kubectl config use-context cluster2
helm install istiod istio/istiod \
  --namespace istio-system \
  --set global.meshID=mesh1 \      # SAME mesh ID
  --set global.multiCluster.clusterName=cluster2 \
  --set global.network=network1    # SAME network (flat routing)

# Step 4: Enable endpoint discovery between clusters
# Cluster1 needs to watch cluster2's API Server for endpoints
istioctl x create-remote-secret \
  --context=cluster2 \
  --name=cluster2 | \
kubectl apply -f - --context=cluster1

# Reverse: cluster2 watches cluster1
istioctl x create-remote-secret \
  --context=cluster1 \
  --name=cluster1 | \
kubectl apply -f - --context=cluster2

# Step 5: Services automatically discoverable across clusters
# Service "api" in cluster1 = discoverable from cluster2
# Load balanced across endpoints from BOTH clusters
```

---

### Cross-Cluster Traffic

```yaml
# Cross-cluster service: locality-aware load balancing
# Prefer local cluster, fail over to remote cluster
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-dr-multicluster
  namespace: production
spec:
  host: api.production.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http2MaxRequests: 1000
    outlierDetection:
      consecutiveGatewayErrors: 3
      interval: 30s
    # Locality-weighted load balancing:
    # local region/zone preferred, remote as failover
    loadBalancer:
      localityLbSetting:
        enabled: true
        failover:
        - from: us-east1      # if us-east1 pods unavailable
          to: us-west1        # fail over to us-west1
        distribute:
        - from: "us-east1/*"
          to:
            "us-east1/*": 80  # 80% to local zone
            "us-west1/*": 20  # 20% to remote for warm standby
```

---

---

# 11.17 Circuit Breaking & Outlier Detection in Production

## 🔴 Advanced

### Production Circuit Breaking Configuration

```yaml
# PRODUCTION-GRADE DESTINATIONRULE
# Combines connection pool + outlier detection for resilience
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-production-dr
  namespace: production
spec:
  host: api-service
  trafficPolicy:
    # CONNECTION POOL: prevent overload (proactive)
    connectionPool:
      http:
        # H2: max concurrent requests per connection
        http2MaxRequests: 500
        # H1: max queued requests when all connections busy
        http1MaxPendingRequests: 50
        # How many retries can be outstanding at once
        maxRetries: 3
        # If connection idle this long, close it
        idleTimeout: 60s
      tcp:
        # Max connections to this service
        maxConnections: 200
        # TCP connect timeout
        connectTimeout: 5s

    # OUTLIER DETECTION: eject failing pods (reactive)
    outlierDetection:
      # Consecutive error threshold before ejection
      consecutive5xxErrors: 5        # 5 consecutive 5xx
      consecutiveGatewayErrors: 3    # OR 3 consecutive gateway errors (502/503/504)
      # How often to scan for unhealthy pods
      interval: 10s
      # How long to eject a pod (doubles on repeated ejection)
      baseEjectionTime: 30s
      # Never eject more than this % of pods simultaneously
      # Prevents removing all healthy pods from pool
      maxEjectionPercent: 50
      # Don't eject if healthy pool would drop below this %
      minHealthPercent: 30
      # Also analyze: split external (5xx) from local (connect/timeout) errors
      splitExternalLocalOriginErrors: true
      consecutiveLocalOriginFailures: 3   # local connection failures

  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

---

### Understanding the Three Circuit States

```
CIRCUIT BREAKING STATES
═══════════════════════════════════════════════════════════════

STATE 1: CLOSED (normal operation)
  All requests flow to all endpoints
  Outlier detection tracking error rates per pod

STATE 2: OPEN (pod ejected from pool)
  Pod returned 5 consecutive errors → EJECTED
  Envoy stops sending new requests to this pod
  Ejection duration: baseEjectionTime × (ejection count)
    1st ejection: 30s
    2nd ejection: 60s
    3rd ejection: 120s
    ...

STATE 3: HALF-OPEN (probe after ejection timeout)
  Ejection duration expires
  Envoy sends ONE probe request to ejected pod
  Success: pod re-added to pool (CLOSED for this pod)
  Failure: pod ejected again for 2× duration

PENDING REQUEST OVERFLOW (connection pool):
  Distinct from per-pod ejection — applies to entire service
  If pending requests > http1MaxPendingRequests:
  → New requests immediately return 503
  → overflow counter incremented (visible in Envoy stats)
  → Upstream not even contacted for these requests
  Response flag: UO (upstream overflow)

DEBUGGING EJECTED PODS:
istioctl proxy-config endpoint my-pod.production \
  --cluster "outbound|8080||api-service.production.svc.cluster.local"
# ENDPOINT         STATUS     OUTLIER CHECK
# 10.244.1.5:8080  HEALTHY    OK
# 10.244.2.3:8080  UNHEALTHY  FAILED (5 GW errors, ejected 60s ago)
# 10.244.3.7:8080  HEALTHY    OK

# Prometheus: ejection rate
rate(envoy_cluster_outlier_detection_ejections_active[5m])
# If this is constantly > 0: service has persistent health issues
```

---

---

# 11.18 Istio Performance Tuning & Resource Overhead

## 🔴 Advanced

### Resource Overhead

```
ISTIO RESOURCE OVERHEAD — REAL NUMBERS
═══════════════════════════════════════════════════════════════

PER-POD SIDECAR OVERHEAD (istio-proxy container):
  Memory (typical production):
    Idle pod: ~60-80MB
    Active pod (100 req/s): ~100-150MB
    High-traffic (1000 req/s): ~200-300MB
  CPU (typical production):
    Idle: <1m
    Active: ~50-100m per 1,000 req/s
    High: ~200-500m per 10,000 req/s

LATENCY OVERHEAD:
  Same-node pod-to-pod: +0.2-0.5ms (iptables + Envoy local processing)
  Cross-node pod-to-pod: +0.5-1ms (both Envoys in path)
  With L7 processing: +1-2ms additional (header processing, routing)
  Total typical: +1-3ms per hop
  Impact: a 10-hop microservice chain: +10-30ms total

ISTIOD OVERHEAD:
  Memory: 200-500MB (scales with service count)
  CPU: 0.1-1 vCPU (scales with config change rate)
  At scale (10,000 pods, 1,000 services): istiod ~1-2 GB RAM, 1-2 CPU

CONTROL PLANE LATENCY:
  Config change → Envoy updated: typically 1-5 seconds
  Large clusters (10,000 pods): up to 10-30 seconds
  Delta xDS reduces this significantly
```

---

### Performance Tuning

```yaml
# ── TUNE SIDECAR RESOURCES PER NAMESPACE ─────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: production
spec:
  egress:
  # CRITICAL OPTIMIZATION: by default Envoy knows about ALL services in cluster
  # This creates thousands of clusters in every Envoy (even if never used)
  # Limit egress to only what this namespace needs:
  - hosts:
    - "production/*"          # all services in own namespace
    - "istio-system/*"        # istiod, ingress gateway
    - "monitoring/prometheus.monitoring.svc.cluster.local"  # specific services

# IMPACT: limits CDS/EDS entries in Envoy
# Before: 10,000 services × 5 endpoints = 50,000 EDS entries per Envoy
# After:  50 services × 5 endpoints = 250 EDS entries per Envoy
# Memory savings: ~40-60% reduction in sidecar memory
# xDS push time reduction: dramatically faster pushes to each Envoy

---
# ── TUNE ISTIOD RESOURCES ─────────────────────────────────────────
# helm values or IstioOperator:
pilot:
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  # Scale replicas for HA + throughput
  replicaCount: 3
  # Enable horizontal autoscaling
  autoscaleEnabled: true
  autoscaleMin: 2
  autoscaleMax: 5

# ── CONCURRENCY TUNING ────────────────────────────────────────────
# Worker threads in Envoy (default: matches CPU count)
metadata:
  annotations:
    proxy.istio.io/config: |
      concurrency: 2   # limit to 2 worker threads if pod has 2 CPUs
                       # prevents Envoy stealing all CPUs from app

# ── TRACING SAMPLE RATE ───────────────────────────────────────────
# meshConfig:
#   defaultConfig:
#     tracing:
#       sampling: 1.0   # 1% sampling (not 100%!)
#                       # 100% adds 5-10% CPU overhead on Envoy
```

---

### Common Performance Issues and Fixes

```
PERFORMANCE TROUBLESHOOTING
═══════════════════════════════════════════════════════════════

ISSUE 1: High Envoy memory usage (>300MB per sidecar)
  Cause: Envoy knows about all services in cluster (no Sidecar resource)
  Fix:  Create Sidecar resource limiting egress hosts per namespace
  
ISSUE 2: Slow xDS pushes (>5s after config change)
  Cause: Large cluster, full xDS mode, every change pushes full config
  Fix: Enable delta xDS (default in Istio 1.12+)
       Check: pilot_xds_push_time metric
       Scale istiod replicas
  
ISSUE 3: High latency with Istio (+10ms observed)
  Cause: Envoy doing complex L7 processing on every request
         OR: Envoy doing full xDS resync frequently (causing GC pauses)
  Fix: Profile with Envoy pprof
       Check connection pool settings (ensure HTTP/2 multiplexing)
       Ensure keep-alive is enabled (not reconnecting per request)

ISSUE 4: istiod OOM crash in large cluster
  Cause: Many services × endpoints × proxies = large in-memory state
  Fix: Increase istiod memory limit
       Enable Sidecar resource per namespace (limits state per Envoy)
       Use delta xDS to reduce memory for push buffers

ISSUE 5: Pod CrashLoopBackOff after istio injection
  Cause: sidecar OOM (low memory limit)
         OR: app starts before sidecar ready (DNS failure)
  Fix: Increase sidecar memory annotations on pod
       Add: holdApplicationUntilProxyStarts: true in meshConfig

ISSUE 6: 503 errors after deploying new version
  Cause: old pods removed before new pods healthy (connection draining)
  Fix: Add preStop hook to app pods (sleep 5s before shutdown)
       OR: configure VirtualService retries for 503 during rolling update
```

---

---

# Quick Reference — Category 11 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **Service mesh purpose** | Move mTLS, retries, circuit breaking, tracing out of app code into infrastructure layer |
| **Istio data plane** | Envoy sidecar per pod. Intercepts all traffic via iptables |
| **Istio control plane** | Istiod = Pilot + Citadel + Galley in one binary. In istio-system |
| **Istiod does NOT** | Handle data-plane traffic. If down: existing traffic keeps flowing |
| **Injection** | Label ns: `istio-injection=enabled`. MutatingWebhook injects istio-proxy + istio-init |
| **Injection gotcha** | Existing pods NOT re-injected. Must rollout restart after labeling namespace |
| **kube-system** | NEVER label for injection — causes bootstrap loops |
| **iptables redirect** | Outbound → port 15001, Inbound → port 15006. Envoy UID 1337 bypasses |
| **VirtualService** | L7 routing rules: weight, header, URI path, retries, timeouts, fault injection |
| **VS match: AND vs OR** | Same list item (no dash between) = AND. Separate items (dash) = OR |
| **DestinationRule** | Defines subsets (label-based pod groups) + LB policy + connection pool + outlier detection |
| **VS + DR together** | VS picks destination subset, DR defines what that subset is |
| **VS without DR subset** | Traffic to undefined subset is DROPPED |
| **Gateway** | L4/TLS config for ingress. Paired with VirtualService for L7 routing |
| **mTLS modes** | PERMISSIVE (plaintext allowed, migration), STRICT (mTLS only, production) |
| **PeerAuthentication scope** | istio-system = mesh-wide, namespace = that namespace, selector = specific pods |
| **SPIFFE ID** | `spiffe://cluster.local/ns/{ns}/sa/{sa}` — cryptographic service identity |
| **Cert rotation** | 24h TTL by default. Envoy requests new cert 12h before expiry. Automatic |
| **AuthorizationPolicy** | No policy = ALLOW all. ANY allow policy = whitelist. DENY evaluated first |
| **AuthorizationPolicy scope** | Same as PeerAuthentication: istio-system > namespace > workload selector |
| **L7 policy** | Methods, paths, JWT claims — NetworkPolicy cannot do this |
| **Canary with Istio** | Weight in VirtualService, not pod count. Change weights to shift traffic |
| **Blue/green** | Two complete deployments, weight 100/0 → instant switch → rollback by reversing |
| **Traffic mirroring** | Async copy to v2, v2 response discarded. Zero user impact |
| **ServiceEntry** | Register external services in mesh for policy, metrics, TLS origination |
| **REGISTRY_ONLY** | Block all external traffic not in a ServiceEntry (egress control) |
| **xDS APIs** | LDS (listeners), RDS (routes), CDS (clusters), EDS (endpoints), SDS (secrets) |
| **xDS ACK/NACK** | NACK = Envoy rejected config (config error). Check `istioctl proxy-status` |
| **Ambient mesh** | Sidecarless. ztunnel per node for L4/mTLS. Waypoint proxy per ns for L7 |
| **Ambient memory savings** | ~60-80% vs sidecar model at scale |
| **Multi-cluster primary-remote** | One istiod for all clusters. Remote fails if primary istiod fails |
| **Multi-cluster multi-primary** | Each cluster has istiod. Independent. Shared root CA required |
| **Circuit breaker: pool** | connectionPool.http.http1MaxPendingRequests. Response flag: UO |
| **Circuit breaker: outlier** | consecutiveErrors → baseEjectionTime (doubles each ejection). Response flag: UH |
| **Sidecar resource** | Limit egress hosts per namespace. Reduces Envoy memory 40-60% |
| **Latency overhead** | ~1-3ms per hop. 10-hop chain = +10-30ms |
| **Memory per sidecar** | ~60-80MB idle, 100-150MB active |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| Envoy admin port | 15000 |
| Envoy outbound capture port | 15001 |
| Envoy inbound capture port | 15006 |
| Envoy health check port | 15021 |
| Envoy Prometheus metrics | 15090 |
| Istiod xDS port (secured) | 15012 |
| Istiod debug/metrics | 15014 |
| Default mTLS cert TTL | 24 hours |
| Cert rotation trigger | 12 hours before expiry |
| Default tracing sample rate | 1% (not 100%) |
| Sidecar idle memory | ~60-80MB |
| Sidecar active memory | ~100-150MB (100 req/s) |
| Latency overhead per hop | ~1-3ms |
| Outlier detection default baseEjectionTime | 30s (doubles each ejection) |
| Max concurrent retries (default) | 3 per cluster |
| xDS push time target (p99) | <1 second |
| Istiod memory at scale (10K pods) | ~1-2 GB |
| Ambient ztunnel per node memory | ~50MB |

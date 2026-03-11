# Kubernetes Interview Mastery
# CATEGORY 3: NETWORKING

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Gotchas → Interview Q&A.
> ⚠️ = High priority, frequently asked in interviews.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 3.1 | Kubernetes networking model — flat network, every pod gets an IP | 🟢 Beginner |
| 3.2 | Services — ClusterIP, NodePort, LoadBalancer, ExternalName | 🟢 Beginner |
| 3.3 | DNS in Kubernetes (CoreDNS) ⚠️ | 🟢 Beginner |
| 3.4 | Ingress — HTTP routing into the cluster | 🟢 Beginner |
| 3.5 | CNI plugins — Flannel, Calico, Cilium | 🟡 Intermediate |
| 3.6 | Network Policies — L3/L4 microsegmentation ⚠️ | 🟡 Intermediate |
| 3.7 | kube-proxy modes — iptables vs IPVS ⚠️ | 🟡 Intermediate |
| 3.8 | Service topology & traffic routing | 🟡 Intermediate |
| 3.9 | EndpointSlices vs Endpoints | 🟡 Intermediate |
| 3.10 | Headless services — when and why | 🟡 Intermediate |
| 3.11 | Cilium eBPF — replacing kube-proxy entirely | 🔴 Advanced |
| 3.12 | Multi-cluster networking (Submariner, Cilium Cluster Mesh) | 🔴 Advanced |
| 3.13 | Ingress vs Gateway API ⚠️ | 🔴 Advanced |
| 3.14 | Pod-to-pod encryption (WireGuard, IPsec via Cilium) | 🔴 Advanced |

---

# 3.1 Kubernetes Networking Model — Flat Network, Every Pod Gets an IP

## 🟢 Beginner

### What it is in simple terms

Kubernetes enforces a **flat networking model** with four fundamental rules. Every pod gets its own routable IP address. Pods on any node can reach pods on any other node without NAT. This is radically different from Docker's default bridge networking where containers need port mapping to communicate.

---

### The Four Fundamental Rules

```
KUBERNETES NETWORKING CONTRACT
═══════════════════════════════════════════════════════════════

RULE 1: Every Pod gets its own unique IP address
  No port sharing between pods on the same node.
  Pod A and Pod B can both listen on port 8080 — different IPs.

RULE 2: All Pods can communicate with all other Pods
  Pod on Node 1 → Pod on Node 2: direct, no NAT.
  No iptables masquerade between pods.
  Source IP is preserved end-to-end.

RULE 3: All Nodes can communicate with all Pods
  Nodes can reach pod IPs directly.
  Used by: kubelet (health probes), kubectl exec port-forward.

RULE 4: Pods see their own IP (not a NAT-translated address)
  Inside a pod: ip addr → shows the pod's real cluster IP.
  No double-NAT confusion.

THESE RULES ARE CONTRACT, NOT MECHANISM:
  Kubernetes defines WHAT must work.
  CNI plugins define HOW it's implemented.
  Flannel, Calico, Cilium, aws-vpc-cni all satisfy these rules differently.
```

---

### Address Ranges

```
KUBERNETES IP SPACES (three separate, non-overlapping ranges)
═══════════════════════════════════════════════════════════════

NODE IPs (host network):
  Range: your VPC/subnet CIDR
  Example: 10.0.1.0/24 (EC2 instance IPs on EKS)
  Assigned by: cloud provider / DHCP
  Who has: every node (worker and control plane)

POD IPs (pod network):
  Range: cluster pod CIDR (configured at cluster creation)
  Example: 10.244.0.0/16 (Flannel default)
           192.168.0.0/16 (Calico default)
           172.16.0.0/16  (user configured)
  Assigned by: CNI plugin (one /24 subnet per node typically)
  Who has: every pod (ephemeral — changes when pod restarts)
  EKS: 10.0.0.0/8 or VPC range (real VPC IPs — no overlay)

SERVICE IPs (virtual IPs):
  Range: cluster service CIDR
  Example: 10.96.0.0/12 (Kubernetes default)
  Assigned by: API Server (from service-cluster-ip-range)
  Who has: every Service object
  VIRTUAL: no network interface has this IP
  Only exists in iptables/IPVS rules on every node
  kube-proxy makes these IPs reachable

IMPORTANT: All three ranges must be non-overlapping.
EKS default:
  Node IPs:    VPC CIDR (10.0.0.0/16)
  Pod IPs:     VPC CIDR (same — VPC CNI gives real VPC IPs to pods)
  Service IPs: 10.100.0.0/16 or 172.20.0.0/16

CHECK YOUR CLUSTER'S CIDRs:
kubectl cluster-info dump | grep -E "cluster-cidr|service-cluster-ip"
kubectl get configmap kubeadm-config -n kube-system -o yaml | grep -A3 "networking"
```

---

### How Pod-to-Pod Traffic Flows

```
TRAFFIC FLOW: Pod A (Node 1) → Pod B (Node 2)
═══════════════════════════════════════════════════════════════

SAME NODE (trivial):
  Pod A (10.244.1.5) → Pod B (10.244.1.8)
  Both on same bridge (cbr0/docker0/cni0)
  Traffic stays on bridge — never leaves node
  Latency: microseconds

DIFFERENT NODES (requires CNI):
  Pod A (10.244.1.5, Node 1)  →  Pod B (10.244.2.5, Node 2)

  Method 1: OVERLAY (Flannel VXLAN, Calico IPIP)
    Pod A → cbr0 (bridge) → flannel0 (tunnel interface)
    → VXLAN encapsulation (outer IP: Node1→Node2)
    → Node 2 physical NIC → decapsulate → cbr0 → Pod B
    Extra overhead: ~50-100 bytes per packet for tunnel header

  Method 2: BGP routing (Calico native, no overlay)
    Each node advertises its pod CIDR via BGP to router
    Router knows: 10.244.1.0/24 is on Node 1
                  10.244.2.0/24 is on Node 2
    Pod A → cbr0 → Node 1 NIC → router → Node 2 NIC → cbr0 → Pod B
    No encapsulation: fastest option

  Method 3: VPC-native (EKS VPC CNI)
    Pods get real VPC IPs (not a separate pod CIDR)
    AWS VPC routing handles everything natively
    Pod A (10.0.1.5) → Node 1 → VPC router → Node 2 → Pod B (10.0.2.8)
    No tunnel, no BGP, uses ENI secondary IP addresses
```

---

### 🎤 Short Crisp Interview Answer

> *"Kubernetes networking has four rules: every pod gets its own unique IP, pods can communicate with any other pod without NAT, nodes can reach pod IPs directly, and pods see their own real IP. These are the contract — CNI plugins implement them. The cluster has three non-overlapping IP ranges: node IPs (VPC addresses), pod IPs (assigned per-pod by the CNI), and Service IPs (virtual IPs that only exist in kube-proxy rules). On EKS, the VPC CNI gives pods real VPC IP addresses directly, so there's no overlay tunnel — pod traffic is native VPC routing."*

---

---

# 3.2 Services — ClusterIP, NodePort, LoadBalancer, ExternalName

## 🟢 Beginner

### What it is in simple terms

A **Service** is a stable virtual IP address with a DNS name that load-balances traffic across a dynamic set of pods selected by label. Pods come and go (restarts, rolling updates, scaling) — Services provide a permanent, stable endpoint that stays constant regardless of which pods are behind it.

---

### Why Services Are Necessary

```
THE PROBLEM WITHOUT SERVICES
═══════════════════════════════════════════════════════════════

Pod IPs are EPHEMERAL:
  Pod crash → restart → NEW IP address
  Rolling update → old pods deleted, new pods created → NEW IPs
  Scaling up → new pods → NEW IPs

Without Service: every client needs to track pod IPs dynamically.
With Service:    clients talk to Service IP/DNS — stable forever.

Service acts as:
  Load balancer (round-robin across healthy pods)
  Service discovery endpoint (DNS name)
  Health-checking layer (only Ready pods receive traffic)
  Abstract boundary (pods can be replaced without client disruption)
```

---

### Service Types

```
SERVICE TYPE COMPARISON
═══════════════════════════════════════════════════════════════

ClusterIP (default):
  Virtual IP only reachable INSIDE the cluster
  Use: internal microservice-to-microservice communication
  Access: other pods, kubectl port-forward, kubectl exec curl
  DNS: <name>.<namespace>.svc.cluster.local

NodePort:
  ClusterIP + opens a static port on EVERY node (30000-32767)
  Access from outside: <any-node-ip>:<nodeport>
  Use: dev/test, on-premise without cloud LB, bare metal clusters
  Problems: exposes node IPs, non-standard ports, no load balancing across nodes

LoadBalancer:
  NodePort + provisions a cloud load balancer automatically
  AWS: creates Network Load Balancer (NLB) or Classic LB
  GCP: creates Google Cloud Load Balancer
  Azure: creates Azure Load Balancer
  Access from outside: LB public DNS/IP → node → pod
  Use: production external services
  Cost: each LoadBalancer Service = one cloud LB (expensive at scale)
  Better at scale: use one Ingress controller instead of many LBs

ExternalName:
  DNS CNAME alias — no proxying, no virtual IP
  Use: reference external services by internal DNS name
  Example: map "database" → "mydb.rds.amazonaws.com"
  Pods use: curl http://database → resolves to RDS endpoint
  Enables: easy migration (swap ExternalName for real pods later)
```

---

### Service YAML — All Types

```yaml
# ── ClusterIP (default) ──────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: ClusterIP              # default — omittable
  selector:
    app: my-api                # must match pod labels EXACTLY
    version: stable            # multiple selectors = AND condition
  ports:
  - name: http
    port: 80                   # Service port (what clients call)
    targetPort: 8080           # Pod port (what container listens on)
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
  # clusterIP: 10.96.45.23    # auto-assigned, or set to "None" for headless

# ── NodePort ─────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: api-nodeport
spec:
  type: NodePort
  selector:
    app: my-api
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080            # static port on every node (30000-32767)
    # omit nodePort to get auto-assigned port

# ── LoadBalancer (AWS NLB via AWS Load Balancer Controller) ───────
apiVersion: v1
kind: Service
metadata:
  name: api-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
  - port: 80
    targetPort: 8080
  externalTrafficPolicy: Local  # preserve client IP, avoid extra hop

# ── ExternalName ─────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.cluster-abc123.us-east-1.rds.amazonaws.com
  # No selector, no ports, no virtual IP
  # DNS: database.production.svc.cluster.local → CNAME → RDS endpoint
```

---

### Service Internals — How ClusterIP Works

```
HOW A SERVICE VIRTUAL IP ACTUALLY WORKS
═══════════════════════════════════════════════════════════════

SETUP (when you kubectl apply a Service):
  1. API Server assigns ClusterIP (e.g., 10.96.45.23)
  2. kube-proxy on every node watches for Service + Endpoint changes
  3. kube-proxy writes iptables/IPVS rules on EVERY node:

     iptables -A KUBE-SERVICES -d 10.96.45.23/32 -p tcp --dport 80
       -j KUBE-SVC-ABCDEF123456

     iptables -A KUBE-SVC-ABCDEF123456 -m statistic --mode random
       --probability 0.33 -j KUBE-SEP-POD1          ← 33% to pod 1
     iptables -A KUBE-SVC-ABCDEF123456 -m statistic --mode random
       --probability 0.50 -j KUBE-SEP-POD2          ← 50% to pod 2
     iptables -A KUBE-SVC-ABCDEF123456
       -j KUBE-SEP-POD3                              ← 100% of remainder to pod 3

     iptables -A KUBE-SEP-POD1 -p tcp -j DNAT
       --to-destination 10.244.1.5:8080              ← actual pod IP:port

RUNTIME (when a pod sends traffic to the Service):
  Pod A sends packet: src=10.244.1.10, dst=10.96.45.23:80
  → iptables PREROUTING chain intercepts (before routing)
  → Matches KUBE-SERVICES rule
  → Random selection: picks pod 2 (33% probability)
  → DNAT: rewrites dst to 10.244.2.8:8080
  → Packet now routed to actual pod IP
  → Pod 2 receives: src=10.244.1.10, dst=10.244.2.8:8080

SERVICE IP NEVER EXISTS ON ANY INTERFACE:
  ip addr show               → no interface has 10.96.45.23
  ping 10.96.45.23           → works! (iptables intercepts ICMP too)
  The IP is a "virtual" that only exists in iptables rules
```

---

### Service Commands

```bash
# Create service imperatively
kubectl expose deployment my-api \
  --type=ClusterIP \
  --port=80 \
  --target-port=8080 \
  --name=api-service \
  -n production

# Inspect service
kubectl get svc -n production
# NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# api-service   ClusterIP   10.96.45.23    <none>        80/TCP    5d
# api-lb        LoadBalancer 10.96.45.99   a1b2c3d4.elb.amazonaws.com 80:30211/TCP 2d

kubectl describe svc api-service -n production
# Shows: Selector, Endpoints, Port mappings

# Check endpoints (what pods are behind the service)
kubectl get endpoints api-service -n production
# NAME          ENDPOINTS                           AGE
# api-service   10.244.1.5:8080,10.244.2.3:8080    5d

# No endpoints? → selector mismatch or all pods NotReady
kubectl get pods -n production -l app=my-api --show-labels

# Test service from within cluster
kubectl run test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://api-service.production.svc.cluster.local/health

# Port forward service for local testing
kubectl port-forward svc/api-service 8080:80 -n production
```

---

### externalTrafficPolicy — Client IP Preservation

```
EXTERNALTRAFFICPOLICY: Cluster vs Local
═══════════════════════════════════════════════════════════════

Cluster (default):
  Any node receives external traffic and forwards to any pod
  (even on a different node)
  Extra hop: Node1 receives → routes to Pod on Node2
  Source IP: SNAT'd to node IP (original client IP LOST)
  Pro: load balanced evenly across all pods

Local:
  Node only routes to pods on THAT node
  No extra hop, no SNAT
  Source IP: preserved (original client IP visible to app)
  Pro: client IP visible for logging, geo-routing
  Con: uneven load if pods aren't evenly distributed across nodes
  Important: pods must be spread across nodes (use TopologySpreadConstraints)

  spec:
    externalTrafficPolicy: Local
    # Health check via NodePort to probe which nodes have local pods
    # AWS NLB sends health checks → only routes to nodes with Ready local pods
```

---

### 🎤 Short Crisp Interview Answer

> *"Services provide stable virtual IPs and DNS names for dynamic pod sets. ClusterIP is internal-only — it's a virtual IP that only exists in kube-proxy iptables rules, with DNAT redirecting traffic to one of the backing pods. NodePort opens a static port on every node for external access. LoadBalancer provisions a cloud load balancer and points it at the NodePort. ExternalName is a DNS CNAME with no proxying — useful for referencing external databases by an internal service name. The critical operational point: if a Service has no endpoints (kubectl get endpoints shows `<none>`), either the selector doesn't match any pod labels, or all matching pods are NotReady."*

---

### ⚠️ Gotchas

1. **Selector is AND logic** — all labels in the selector must match. `app=my-api, version=stable` only selects pods with BOTH labels.
2. **targetPort can be a named port** — `targetPort: http` where `http` refers to the named port in the pod spec. This is more maintainable than hardcoding port numbers.
3. **LoadBalancer creates NodePort too** — a LoadBalancer service also creates an underlying NodePort. The cloud LB points at those NodePorts.
4. **Session affinity** — default is None (random per connection). Set `sessionAffinity: ClientIP` to route the same client IP to the same pod (uses iptables recent module).

---

---

# ⚠️ 3.3 DNS in Kubernetes (CoreDNS)

## 🟢 Beginner — HIGH PRIORITY

### What it is in simple terms

**CoreDNS** is the cluster DNS server. It runs as a Deployment in `kube-system` and has a ClusterIP (typically `10.96.0.10`). Every pod is configured to use CoreDNS as its DNS resolver. CoreDNS knows about all Services and (optionally) pods and answers DNS queries for cluster-internal names.

---

### DNS Name Structure

```
KUBERNETES DNS NAMING SCHEME
═══════════════════════════════════════════════════════════════

SERVICE DNS:
  <service-name>.<namespace>.svc.<cluster-domain>
  cluster-domain is usually: cluster.local

  Examples:
  api-service.production.svc.cluster.local    → full FQDN
  api-service.production                       → works within cluster
  api-service                                  → works within same namespace

  From pod in "production" namespace:
  curl http://api-service              → resolves (same namespace)
  curl http://api-service.production   → resolves (explicit namespace)
  curl http://api-service.staging      → resolves (cross-namespace!)
  curl http://api-service.production.svc.cluster.local  → always works

POD DNS:
  <pod-ip-dashes>.<namespace>.pod.<cluster-domain>
  10-244-1-5.production.pod.cluster.local
  (dots in IP replaced with dashes)
  Rarely used — Service DNS is preferred

HEADLESS SERVICE WITH StatefulSet:
  Individual pod DNS (stable across restarts):
  <pod-name>.<service-name>.<namespace>.svc.cluster.local
  postgres-0.postgres-headless.production.svc.cluster.local
  postgres-1.postgres-headless.production.svc.cluster.local

EXTERNAL NAMES:
  CoreDNS forwards unknown names to upstream (configured in Corefile)
  api.external-company.com → forwarded to node's /etc/resolv.conf upstream
```

---

### CoreDNS Architecture

```
COREDNS INTERNALS
═══════════════════════════════════════════════════════════════

DEPLOYMENT:
  Namespace: kube-system
  Replicas: 2 (default on EKS — always run multiple for HA)
  Service: kube-dns (ClusterIP: 10.96.0.10)
  Resource: Corefile ConfigMap (configuration)

HOW PODS FIND COREDNS:
  kubelet writes /etc/resolv.conf in every pod:
  nameserver 10.96.0.10      ← CoreDNS ClusterIP
  search production.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5

DNS QUERY FLOW:
  Pod needs to resolve "api-service":
  1. Check /etc/hosts (always checked first — for localhost etc.)
  2. Send UDP query to 10.96.0.10:53 (CoreDNS)
  3. CoreDNS receives query: api-service
  4. ndots:5 applies (see below): api-service has 0 dots < 5
     → Append search domain #1: api-service.production.svc.cluster.local
     → CoreDNS checks its in-memory cache of Services
     → Found! Returns ClusterIP: 10.96.45.23
  5. Pod receives answer: api-service = 10.96.45.23
  6. Connection proceeds

FOR EXTERNAL NAMES (e.g., api.stripe.com):
  CoreDNS cache miss → forwards to upstream (/etc/resolv.conf on nodes)
  → AWS VPC DNS resolver (10.0.0.2 on EKS) → answers from Route53/internet
```

---

### The ndots:5 Gotcha

```
NDOTS:5 — THE MOST COMMON DNS PERFORMANCE ISSUE
═══════════════════════════════════════════════════════════════

ndots:5 means: if a name has fewer than 5 dots, try all search
domains BEFORE trying the name as an absolute FQDN.

INTERNAL NAME (0 dots) — fast:
  Query: "api-service"
  → try: api-service.production.svc.cluster.local → HIT! (1 query)

INTERNAL FQDN (5+ dots) — fast:
  Query: "api-service.production.svc.cluster.local."  (trailing dot)
  → 5 dots = absolute → no search domains appended → 1 query

EXTERNAL NAME with few dots — SLOW:
  Query: "stripe.com"  (1 dot < 5)
  → try: stripe.com.production.svc.cluster.local → NXDOMAIN
  → try: stripe.com.svc.cluster.local            → NXDOMAIN
  → try: stripe.com.cluster.local               → NXDOMAIN
  → try: stripe.com.                             → HIT! (4 queries total)
  Adds 3 unnecessary failed DNS lookups before external name resolves!

EXTERNAL NAME with many dots — still SLOW:
  Query: "api.payments.stripe.com"  (3 dots < 5)
  → 3 failed queries before resolving external name

FIXES:
  Option 1: Use FQDN with trailing dot
    curl http://stripe.com./api    → absolute, no search domains
    curl http://api.payments.stripe.com./charge

  Option 2: Lower ndots in pod spec
    spec:
      dnsConfig:
        options:
        - name: ndots
          value: "2"    ← only names with < 2 dots get search domain treatment

  Option 3: Add to /etc/hosts via hostAliases
    spec:
      hostAliases:
      - ip: "10.96.45.23"
        hostnames: ["api-service"]

  Option 4: CoreDNS autopath plugin (reduces lookup count for internal names)
```

---

### CoreDNS Configuration

```yaml
# CoreDNS ConfigMap (view with: kubectl get configmap coredns -n kube-system -o yaml)
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors                     # log errors
        health {                   # /health endpoint for liveness probe
           lameduck 5s
        }
        ready                      # /ready endpoint for readiness probe
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure           # enable pod DNS records (insecure = no verification)
           fallthrough in-addr.arpa ip6.arpa
           ttl 30                  # DNS record TTL (seconds)
        }
        prometheus :9153           # metrics endpoint
        forward . /etc/resolv.conf {  # forward non-cluster queries to node's DNS
           max_concurrent 1000
        }
        cache 30                   # cache responses for 30 seconds
        loop                       # detect and abort if forwarding loop
        reload                     # auto-reload config on ConfigMap change
        loadbalance                # round-robin DNS load balancing
    }
```

---

```bash
# CoreDNS debugging commands
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system deployment/coredns

# Enable debug logging (edit configmap)
kubectl edit configmap coredns -n kube-system
# Add "log" plugin under "errors":
#   .:53 {
#       errors
#       log          ← add this line to log all DNS queries

# Restart CoreDNS after config change
kubectl rollout restart deployment/coredns -n kube-system

# Test DNS from inside a pod
kubectl run dnstest --image=nicolaka/netshoot --rm -it --restart=Never -- bash
# Inside:
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
nslookup api-service.production.svc.cluster.local
dig api-service.production.svc.cluster.local
nslookup external-service.com

# Check CoreDNS metrics
kubectl port-forward -n kube-system deployment/coredns 9153:9153
curl http://localhost:9153/metrics | grep coredns_dns_request

# Verify CoreDNS ClusterIP
kubectl get svc -n kube-system kube-dns
# NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
# kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   30d
```

---

### 🎤 Short Crisp Interview Answer

> *"CoreDNS runs as a Deployment in kube-system with a stable ClusterIP. Every pod's /etc/resolv.conf is set by kubelet to use that ClusterIP as the nameserver, with search domains appended for the pod's namespace and cluster. Service DNS format is service.namespace.svc.cluster.local — pods in the same namespace can use just the service name, cross-namespace needs the namespace suffix. The critical gotcha is ndots:5: any name with fewer than 5 dots gets all search domains appended before being tried as an absolute name. This means external hostnames like stripe.com (1 dot) generate 3 failed DNS queries before resolving, adding 10-30ms latency per external call. Fix with a trailing dot (stripe.com.) for absolute resolution, or lower ndots to 2 in pod dnsConfig."*

---

---

# 3.4 Ingress — HTTP Routing into the Cluster

## 🟢 Beginner

### What it is in simple terms

**Ingress** is a Kubernetes object that defines **HTTP/HTTPS routing rules** — mapping incoming requests by hostname and path to backend Services. A single Ingress controller (like nginx or AWS ALB) handles all these rules, replacing dozens of individual LoadBalancer Services.

---

### Ingress Architecture

```
INGRESS ARCHITECTURE
═══════════════════════════════════════════════════════════════

WITHOUT INGRESS:
  api.company.com     → LoadBalancer Service A → pod A
  admin.company.com   → LoadBalancer Service B → pod B
  pay.company.com     → LoadBalancer Service C → pod C
  Each = separate cloud load balancer = expensive!

WITH INGRESS:
  All traffic → ONE cloud load balancer → Ingress Controller (pods)
                                               │
                              routes based on Host + Path:
                              api.company.com/  → Service A → pod A
                              admin.company.com → Service B → pod B
                              pay.company.com   → Service C → pod C

COMPONENTS:
  Ingress object:     Kubernetes resource defining routing rules (YAML)
  Ingress Controller: Actual reverse proxy implementation (pods + LB)
                      nginx-ingress, AWS ALB Ingress Controller, Traefik, etc.
  Ingress Class:      Links Ingress objects to a specific controller
                      (allows multiple controllers in same cluster)

RELATIONSHIP:
  Ingress object = declares intent (rules)
  Ingress Controller = implements it (actual traffic routing)
  Without a controller: Ingress objects have no effect!
```

---

### Ingress YAML

```yaml
# ── Basic Ingress ────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: production
  annotations:
    # Controller-specific annotations (nginx example)
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    # AWS ALB Controller annotations
    # alb.ingress.kubernetes.io/scheme: internet-facing
    # alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: nginx          # which Ingress controller handles this

  # TLS configuration
  tls:
  - hosts:
    - api.company.com
    - admin.company.com
    secretName: company-tls-secret  # Secret of type kubernetes.io/tls
                                    # contains tls.crt and tls.key

  rules:
  # ── Host-based routing ──
  - host: api.company.com
    http:
      paths:
      - path: /v1
        pathType: Prefix             # Prefix, Exact, or ImplementationSpecific
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
      - path: /                     # catch-all for this host
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

  - host: admin.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80

  # ── Default backend (no host match) ──
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80

---
# TLS Secret (usually managed by cert-manager)
apiVersion: v1
kind: Secret
metadata:
  name: company-tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

---

### Path Types

```
INGRESS PATH TYPES
═══════════════════════════════════════════════════════════════

Exact:
  path: /api/v1
  Matches ONLY: /api/v1 (not /api/v1/, not /api/v1/users)

Prefix:
  path: /api/v1
  Matches: /api/v1, /api/v1/, /api/v1/users, /api/v1/orders
  Does NOT match: /api/v10, /api/v11

ImplementationSpecific:
  Behavior defined by the Ingress controller (nginx, ALB, etc.)
  nginx: supports regex patterns with ~ prefix
    nginx.ingress.kubernetes.io/use-regex: "true"
    path: /api/(v[0-9]+)/.*

PATH PRECEDENCE:
  More specific paths take priority over less specific.
  Exact > Prefix (longer prefix wins among prefixes).
  /api/v1/users (longer prefix) wins over /api/v1 (shorter prefix).
```

---

### AWS ALB Ingress Controller

```yaml
# AWS-specific: ALB Ingress Controller creates an Application Load Balancer
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing       # or internal
    alb.ingress.kubernetes.io/target-type: ip               # ip or instance
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/ssl-redirect: "443"           # redirect HTTP→HTTPS
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/group.name: production        # share ALB across Ingresses
spec:
  ingressClassName: alb
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

# target-type: ip vs instance:
#   ip:       ALB routes directly to pod IPs (requires VPC CNI)
#             Skips NodePort overhead, better performance
#   instance: ALB routes to Node IPs + NodePort
#             Works with any CNI
```

---

### 🎤 Short Crisp Interview Answer

> *"Ingress defines HTTP routing rules — host and path based — and maps them to backend Services. The Ingress Controller (nginx, AWS ALB Controller, Traefik) actually implements those rules as a reverse proxy. The key value prop is cost efficiency: instead of one cloud load balancer per service, you have one load balancer pointing at the Ingress Controller which routes to many services. On EKS I use the AWS Load Balancer Controller with target-type: ip, which routes directly to pod IPs via the VPC CNI — better performance than going through NodePorts. TLS termination happens at the Ingress or at the ALB, with cert-manager handling certificate renewal automatically."*

---

---

# 3.5 CNI Plugins — What They Are, How They Work

## 🟡 Intermediate

### What CNI is

```
CNI (CONTAINER NETWORK INTERFACE) OVERVIEW
═══════════════════════════════════════════════════════════════

CNI is a SPECIFICATION, not a product.
Defines: a standard interface between container runtime and network plugins.

WHEN CNI IS CALLED:
  Pod created → kubelet calls CRI (containerd) → containerd calls CNI plugin
  Two calls:
    ADD:    set up networking when pod starts (assign IP, configure routes)
    DEL:    tear down networking when pod stops (release IP, clean routes)

CNI PLUGIN RESPONSIBILITIES:
  1. Assign a unique IP address to the pod
  2. Configure the pod's network interface (veth pair)
  3. Connect pod to node's network (bridge, BGP routes, VXLAN, etc.)
  4. Ensure pod-to-pod routing works (K8s networking contract)
  5. (Optional) Enforce NetworkPolicies

CNI PLUGIN LOCATION:
  Binary: /opt/cni/bin/flannel, /opt/cni/bin/calico, etc.
  Config: /etc/cni/net.d/10-<plugin>.conf

KUBELET CNI INTEGRATION:
  kubelet arg: --cni-bin-dir=/opt/cni/bin --cni-conf-dir=/etc/cni/net.d
  On pod creation: kubelet calls the CNI binary with pod namespace info
  CNI binary sets up networking and returns the pod's IP
```

---

### Flannel — Simplest Overlay

```
FLANNEL
═══════════════════════════════════════════════════════════════

Model:     VXLAN overlay (default) or host-gw
Pod CIDR:  10.244.0.0/16 (each node gets a /24 subnet)
Pros:      Simple, easy to install, low resource usage
Cons:      No NetworkPolicy support (needs separate Calico/Cilium for policies)
           VXLAN overhead on pod-to-pod traffic
           No encryption

VXLAN TUNNEL:
  Each node runs flanneld daemon
  Allocates /24 subnet from 10.244.0.0/16 per node
  VXLAN device (flannel.1) encapsulates packets between nodes

  Node 1 pods: 10.244.1.0/24
  Node 2 pods: 10.244.2.0/24

  Pod on Node 1 → Pod on Node 2:
  Original: src=10.244.1.5, dst=10.244.2.8
  After VXLAN: src=NodeIP1, dst=NodeIP2 (outer header)
               inner: src=10.244.1.5, dst=10.244.2.8
  Node 2: decapsulates → delivers to pod

HOST-GW MODE (no overlay, better performance):
  Requires: nodes on same L2 network (same subnet/VLAN)
  Each node installs host routes for other nodes' pod CIDRs
  No tunneling: full native routing speed
  Limitation: can't span L3 boundaries (multiple subnets/VPCs)

USE FLANNEL WHEN:
  Simple clusters, no NetworkPolicy needed
  Low resource environments
  On-premise with L2 connectivity between nodes
```

---

### Calico — Networking + NetworkPolicy

```
CALICO
═══════════════════════════════════════════════════════════════

Model:     BGP routing (no overlay by default) or VXLAN overlay
Pod CIDR:  192.168.0.0/16 (default)
Pros:      Full NetworkPolicy support (L3/L4 + Calico extended policies)
           BGP mode: native routing, no encapsulation overhead
           Can peer with external BGP routers
           WireGuard encryption support
Cons:      BGP can be complex to operate
           Requires L3 connectivity or VXLAN for multi-subnet deployments
           More resource usage than Flannel

BGP MODE (preferred for performance):
  Each node runs a BGP daemon (BIRD)
  Advertises its pod CIDR (/26 blocks by default) via BGP
  Other nodes learn routes via BGP
  Pod traffic: native IP routing, no encapsulation

  On-node routing table:
  10.244.1.0/24 via 10.0.1.5 dev eth0   ← route to Node 1's pods
  10.244.2.0/24 via 10.0.1.6 dev eth0   ← route to Node 2's pods

CALICO ON EKS:
  aws-vpc-cni handles IP assignment (VPC native IPs)
  Calico ONLY provides NetworkPolicy enforcement (no IP management)
  Combined: VPC CNI for IPs + Calico for policies
  OR: use Calico for everything (replaces VPC CNI — loses ENI advantage)

CALICO NETWORKPOLICY (extended):
  Supports L7 (application layer) rules in enterprise version
  GlobalNetworkPolicy: cluster-wide policies
  HostEndpoint: policies for node traffic (not just pods)
  Ordered rules (unlike K8s NetworkPolicy which is additive)

USE CALICO WHEN:
  NetworkPolicy enforcement needed
  Large clusters needing BGP routing efficiency
  Need to peer with external network infrastructure
  Fine-grained policy control required
```

---

### Cilium — eBPF-Based Next Generation

```
CILIUM
═══════════════════════════════════════════════════════════════

Model:     eBPF programs in kernel (no kube-proxy needed!)
Pod CIDR:  configurable
Pros:      Best performance (eBPF: kernel bypass, no iptables)
           L7 NetworkPolicy (HTTP path, gRPC method, Kafka topic)
           Built-in Hubble observability (every flow visible)
           WireGuard encryption
           Can replace kube-proxy entirely
           Best observability tooling of any CNI
Cons:      Requires Linux kernel 5.4+ (4.9+ for basic features)
           More complex to operate
           Higher initial learning curve

EBPF VS IPTABLES:
  iptables (traditional):
    Rules loaded into kernel netfilter tables
    Every packet traverses ALL rules O(n) — slow at scale
    10,000 Services = 10,000+ iptables rules to scan per packet

  eBPF (Cilium):
    Small programs compiled into kernel bytecode
    Runs in kernel with O(1) hash lookups
    No iptables, no kube-proxy daemon needed
    Service lookup: hash map → nanoseconds regardless of Service count

CILIUM FEATURES:
  Hubble:           real-time network flow visibility
  Cluster Mesh:     multi-cluster service discovery
  BGP Control Plane: peer with external BGP routers
  Bandwidth Manager: per-pod bandwidth limits
  L7 policies:      allow GET /api/* but deny DELETE /admin/*
  Transparent encryption: WireGuard or IPsec, zero config

USE CILIUM WHEN:
  Performance is critical (large clusters, latency-sensitive)
  Need L7 NetworkPolicy (HTTP-aware)
  Need observability without service mesh overhead
  Kernel 5.4+ available (Amazon Linux 2023, Ubuntu 22.04)
  EKS: Amazon Linux 2023 nodes + Cilium = excellent combination
```

---

### CNI Comparison Table

```
CNI FEATURE MATRIX
═══════════════════════════════════════════════════════════════

Feature              Flannel    Calico     Cilium     VPC CNI
─────────────────────────────────────────────────────────────
NetworkPolicy         No         Yes        Yes        No*
L7 Policy             No         Enterprise Yes        No
eBPF dataplane        No         Optional   Yes        No
kube-proxy replace    No         No         Yes        No
Encryption            No         WireGuard  WireGuard  No
                                            IPsec
Overlay               VXLAN      Optional   Optional   No
Native routing        host-gw    BGP        BGP        VPC
Performance           Medium     High       Highest    High
Observability         Basic      Good       Excellent  AWS VPC
Complexity            Low        Medium     Medium     Low
EKS compatible        Yes        Policy-only Yes       Native

*VPC CNI: use with Calico or Cilium for NetworkPolicy enforcement
```

---

### 🎤 Short Crisp Interview Answer

> *"CNI is a spec defining how container runtimes call network plugins. On pod creation, kubelet calls the CNI binary to assign an IP and set up routing; on deletion it calls DEL to clean up. Flannel is simplest — VXLAN overlay, no NetworkPolicy. Calico adds BGP routing (no encapsulation overhead) and full NetworkPolicy support, commonly used on EKS as a policy engine alongside the VPC CNI. Cilium is the modern choice — eBPF programs in the kernel replace both the CNI dataplane and kube-proxy, with O(1) Service lookups instead of O(n) iptables rule traversal. Cilium also enables L7 NetworkPolicies (HTTP-aware) and Hubble for real-time flow observability. On EKS, the VPC CNI is the default — it assigns real VPC IP addresses to pods via ENI secondary IPs, making pod traffic indistinguishable from regular EC2 traffic to the VPC."*

---

---

# ⚠️ 3.6 Network Policies — L3/L4 Microsegmentation

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

**NetworkPolicy** is a Kubernetes resource that acts as a **firewall for pods** — specifying which pods can send traffic to which other pods, on which ports, using which protocols. By default, all traffic is allowed. Applying a NetworkPolicy to a pod makes it selective — only traffic matching the policy's rules is allowed.

---

### Default Behavior and the Whitelist Model

```
NETWORKPOLICY ENFORCEMENT MODEL
═══════════════════════════════════════════════════════════════

NO POLICIES APPLIED:
  All ingress:  allowed (any pod can receive from anywhere)
  All egress:   allowed (any pod can send to anywhere)
  Open by default — not secure!

POLICY APPLIED TO A POD (whitelist model):
  The moment ANY NetworkPolicy selects a pod:
  ALL traffic in the selected direction is DENIED by default
  ONLY traffic matching a rule in ANY selecting policy is ALLOWED

  Note: multiple policies that select the same pod are ADDITIVE (OR logic)
  Policy A allows traffic from app=frontend
  Policy B allows traffic from app=monitoring
  Combined: pod accepts from frontend OR monitoring

BEST PRACTICE — ZERO TRUST:
  1. Apply default-deny for ingress AND egress
  2. Explicitly allow only required traffic
  3. Always allow DNS egress (port 53) or nothing works!
```

---

### Network Policy YAML — Complete Reference

```yaml
# ── STEP 1: Default deny all in namespace (always start here) ─────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}              # {} = selects ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
  # No ingress/egress rules = deny all selected traffic

---
# ── STEP 2: Allow DNS egress (CRITICAL — without this nothing resolves) ──
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
    # No 'to' field = allows to ANY destination on these ports

---
# ── STEP 3: Allow specific ingress to api pods ────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api                 # this policy applies TO api pods

  policyTypes:
  - Ingress

  ingress:
  - from:
    - podSelector:             # FROM pods with app=frontend
        matchLabels:
          app: frontend
      namespaceSelector:       # AND in namespace with env=production
        matchLabels:           # ← SAME item (no dash) = AND condition
          environment: production

    ports:
    - port: 8080
      protocol: TCP

---
# ── STEP 4: Allow egress from api to database ─────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api

  policyTypes:
  - Egress

  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - port: 5432
      protocol: TCP

---
# ── Cross-namespace traffic ───────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production
spec:
  podSelector: {}              # all pods in production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:       # from monitoring namespace
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:             # AND specifically prometheus pods
        matchLabels:
          app: prometheus
    ports:
    - port: 9090               # metrics port

---
# ── Allow to CIDR range (external traffic) ────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0        # any IP
        except:
        - 10.0.0.0/8           # except private networks (stay internal)
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - port: 443
      protocol: TCP
```

---

### ⚠️ The AND vs OR Gotcha

```
THE MOST COMMON NETWORKPOLICY MISTAKE
═══════════════════════════════════════════════════════════════

QUESTION: Does this allow from "monitoring namespace pods with app=prometheus"
          OR from "any pod with app=prometheus in any namespace"?

WRONG INTERPRETATION:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    - podSelector:              ← SEPARATE DASH = SEPARATE RULE = OR!
        matchLabels:
          app: prometheus

  This means:
  RULE 1: from ANY pod in namespace "monitoring" (any pod, not just prometheus)
  RULE 2: from pod with app=prometheus in ANY namespace
  Combined: EITHER is allowed — very broad!

CORRECT (AND — same list item, no dash between selectors):
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:              ← NO DASH = same item = AND
        matchLabels:
          app: prometheus
  
  This means:
  RULE 1: pod with app=prometheus AND in monitoring namespace
  Combined: only prometheus pod in monitoring namespace — narrow, correct!

MEMORY AID:
  - from:
    - selector1            ← item 1 (dash)
    - selector2            ← item 2 (dash) = OR between items
  
  - from:
    - selector1            ← item 1 (dash)
      selector2            ← part of item 1 (no dash) = AND within item
```

---

### NetworkPolicy Debugging

```bash
# ─── IS A POLICY BLOCKING TRAFFIC? ────────────────────────────────
# Test connectivity from pod A to pod B
kubectl exec pod-a -n production -- nc -zv 10.244.2.5 8080
# Connection timeout = NetworkPolicy blocking
# Connection refused = pod B not listening on 8080
# Connection succeeded = traffic allowed

# What policies apply to a pod?
kubectl get networkpolicy -n production -o yaml | \
  grep -A 10 "podSelector:"
# Check which policies' podSelector matches your pod's labels

# Test with netshoot (has all network tools)
kubectl run debug --image=nicolaka/netshoot --rm -it \
  --restart=Never -n production -- bash
curl -v http://api-service:8080/health
nslookup api-service

# Cilium: see DROPPED flows in real time
hubble observe --namespace production --verdict DROPPED
# Flow: production/frontend → production/api: DROPPED (policy denied)
# Shows EXACTLY which policy is dropping the packet

# Calico: check policy evaluation
calicoctl get networkpolicy -n production
calicoctl policy status

# ─── COMMON MISTAKES CHECKLIST ─────────────────────────────────────
# 1. Forgot DNS egress → pods can't resolve service names
# 2. Wrong AND/OR (separate dashes = OR, same item = AND)
# 3. Namespace not labeled → namespaceSelector never matches
kubectl get ns production --show-labels   # check label exists
kubectl label ns production kubernetes.io/metadata.name=production

# 4. NetworkPolicy requires CNI support (Flannel alone: no support!)
kubectl get pods -n kube-system | grep -E "calico|cilium|weave"
# If nothing: NetworkPolicies are IGNORED!

# 5. egress to ClusterIP needs rules for pod IPs (not Service IP)
# NetworkPolicy operates at pod IP level, not Service virtual IP
# If egress allowed to 10.244.2.5:8080 but Service is 10.96.x.x —
# kube-proxy DNAT happens BEFORE NetworkPolicy evaluation on egress
# → Allow the pod IPs (or allow all egress to the port)
```

---

### 🎤 Short Crisp Interview Answer

> *"NetworkPolicy is a pod-level firewall. Without any policy, all traffic flows freely. The moment a NetworkPolicy selects a pod, all unmatched traffic in the selected direction is denied — it's a whitelist model. My production pattern: start with default-deny for all pods, then allow-dns-egress (UDP/TCP 53) because without it nothing resolves, then specific rules for each communication path. The critical AND vs OR gotcha: two selectors as separate list items (separate dashes) are OR — either one matches. Two selectors as the same list item (no dash between them) are AND — both must match. Getting this wrong creates overly permissive rules. NetworkPolicy only works if the CNI supports it — Flannel doesn't, you need Calico, Cilium, or another policy-aware CNI."*

---

---

# ⚠️ 3.7 kube-proxy Modes — iptables vs IPVS

## 🟡 Intermediate — HIGH PRIORITY

### What kube-proxy does

```
KUBE-PROXY ROLE
═══════════════════════════════════════════════════════════════

kube-proxy is a DaemonSet running on every node.
Responsibility: implement Service virtual IP routing.
Watches: API Server for Service and Endpoint/EndpointSlice changes.
Action:  writes rules to kernel to make Service IPs work.

Without kube-proxy: Service ClusterIPs are unreachable.
With kube-proxy:    packets to 10.96.45.23:80 → DNAT to pod IPs.

THREE MODES:
  userspace   deprecated, never use
  iptables    default (most clusters), O(n) per-packet lookup
  ipvs        O(1) lookups, better performance, more LB algorithms
```

---

### iptables Mode — How It Works

```
IPTABLES MODE INTERNALS
═══════════════════════════════════════════════════════════════

KERNEL CHAINS USED:
  PREROUTING → KUBE-SERVICES → KUBE-SVC-xxx → KUBE-SEP-xxx → DNAT
  OUTPUT     → KUBE-SERVICES (for pod-to-service traffic)

FOR EACH SERVICE:
  KUBE-SERVICES chain: one rule per Service ClusterIP:port
  KUBE-SVC-xxx chain:  N rules for N endpoints (probabilistic selection)
  KUBE-SEP-xxx chain:  one rule per endpoint with DNAT target

PROBABILISTIC LOAD BALANCING:
  3 pods behind service:
  Rule 1: 33.3% → DNAT to pod 1
  Rule 2: 50%   → DNAT to pod 2  (50% of remaining 66.7% = 33.3%)
  Rule 3: 100%  → DNAT to pod 3  (100% of remaining 33.3% = 33.3%)
  Result: equal distribution across 3 pods

PERFORMANCE PROBLEM AT SCALE:
  1,000 Services × 5 pods each = 5,000 iptables rules
  Every packet scans ALL rules linearly → O(n)
  10,000 Services: 50,000 rules → meaningful CPU overhead on every packet
  iptables rule updates: the entire table must be re-written atomically
  100 Services added → full iptables-restore of all 5,000 rules

CHECKING IPTABLES RULES:
  sudo iptables-save | grep KUBE | wc -l      ← count K8s rules
  sudo iptables-save | grep 10.96.45.23       ← rules for specific Service
  sudo iptables -t nat -L KUBE-SERVICES -n    ← view chain
```

---

### IPVS Mode — Better Performance

```
IPVS MODE INTERNALS
═══════════════════════════════════════════════════════════════

IPVS = IP Virtual Server, Linux kernel's L4 load balancer
Uses kernel hash tables instead of iptables linear scan.

ARCHITECTURE:
  kube-proxy creates virtual servers in IPVS for each Service ClusterIP
  kube-proxy also creates dummy interface kube-ipvs0 with all Service IPs
    (so kernel considers traffic to Service IPs as "local" → processes it)
  IPVS: O(1) lookup regardless of service count

PERFORMANCE COMPARISON:
  Cluster size    iptables CPU    IPVS CPU    Diff
  100 services    baseline        baseline    ~same
  1,000 services  2x baseline     baseline    2x faster
  10,000 services 10x baseline    1.1x base   ~10x faster
  Rule updates:   full table      incremental much faster

LOAD BALANCING ALGORITHMS (IPVS supports more than iptables):
  rr     Round Robin (iptables only approximation)
  lc     Least Connection (new in IPVS!)
  dh     Destination Hash
  sh     Source Hash (session affinity by source IP)
  sed    Shortest Expected Delay
  nq     Never Queue
  Default: rr

ENABLE IPVS:
  kube-proxy ConfigMap:
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration
  mode: "ipvs"
  ipvs:
    scheduler: "lc"          # least-connection algorithm

  Requires: ip_vs, ip_vs_rr, ip_vs_wrr, ip_vs_sh kernel modules
  sudo modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh

CHECKING IPVS RULES:
  sudo ipvsadm -L -n
  # IP Virtual Server version 1.2.1
  # Prot LocalAddress:Port Scheduler Flags
  #   -> RemoteAddress:Port Forward Weight ActiveConn InActConn
  # TCP  10.96.45.23:80 rr
  #   -> 10.244.1.5:8080  Masq    1      5          0
  #   -> 10.244.2.3:8080  Masq    1      3          0
  #   -> 10.244.3.1:8080  Masq    1      4          0
```

---

```bash
# Check which mode kube-proxy is running
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
# mode: "ipvs"  OR  mode: "iptables"  OR  mode: ""  (empty = iptables default)

# Check kube-proxy logs for mode
kubectl logs -n kube-system daemonset/kube-proxy | grep "Using"
# Using iptables Proxier.
# OR: Using ipvs Proxier.

# EKS: check kube-proxy version
kubectl get daemonset kube-proxy -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Count iptables rules for Kubernetes
sudo iptables-save | grep -c "KUBE"
# 3842  ← 3842 K8s-related iptables rules

# IPVS view
sudo ipvsadm -L -n | head -30

# Note on Cilium:
# If using Cilium with kube-proxy replacement, kube-proxy DaemonSet may not exist
kubectl get daemonset kube-proxy -n kube-system
# Error from server (NotFound) ← Cilium replaced it entirely
```

---

### 🎤 Short Crisp Interview Answer

> *"kube-proxy runs on every node and writes the kernel rules that make Service virtual IPs work. In iptables mode, it writes PREROUTING and OUTPUT chain rules for every Service and endpoint — the packet does a linear scan through all rules, which is O(n) in the number of Services. At 10,000 Services with 5 pods each, that's 50,000 iptables rules scanned per packet. IPVS mode uses the kernel's IP Virtual Server with hash table lookups — O(1) regardless of Service count. IPVS also supports more load-balancing algorithms including least-connection, which iptables can't do at all. For clusters with more than ~1,000 Services, IPVS is strongly preferred. Cilium can replace kube-proxy entirely with eBPF programs that are even faster than IPVS."*

---

---

# 3.8 Service Topology & Traffic Routing

## 🟡 Intermediate

### The Problem: Cross-Zone Traffic Cost and Latency

```
WHY TOPOLOGY-AWARE ROUTING EXISTS
═══════════════════════════════════════════════════════════════

CLUSTER TOPOLOGY (typical EKS multi-AZ):
  us-east-1a: Node 1 (pods: api-abc, worker-xyz)
  us-east-1b: Node 2 (pods: api-def, worker-uvw)
  us-east-1c: Node 3 (pods: api-ghi, worker-rst)

WITHOUT TOPOLOGY AWARENESS:
  Pod on Node 1 → Service → random endpoint selection
  → 33% chance: lands on Node 1 (same AZ — free)
  → 33% chance: lands on Node 2 (cross-AZ — $0.01/GB)
  → 33% chance: lands on Node 3 (cross-AZ — $0.01/GB)
  66% of traffic crosses AZ boundaries

  At 10TB/month cross-AZ: ~$100/month just in data transfer fees
  Plus latency: same-AZ ~0.5ms, cross-AZ ~2-5ms

WITH TOPOLOGY-AWARE ROUTING:
  Pod on Node 1 (us-east-1a) → Service
  → Prefers endpoints in us-east-1a
  → Falls back to other AZs only if no local endpoints exist
  → ~90%+ same-AZ traffic → cost savings + latency reduction
```

---

### Topology Aware Hints (current mechanism)

```yaml
# Topology Aware Routing — enabled per-Service via annotation
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
  annotations:
    service.kubernetes.io/topology-mode: "auto"
    # "auto" = enable topology hints when it makes sense
    # Kubernetes calculates hint assignments automatically
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

```
HOW TOPOLOGY AWARE HINTS WORK
═══════════════════════════════════════════════════════════════

CONTROL PLANE (EndpointSlice controller):
  Knows: number of endpoints per zone, nodes per zone
  Calculates: how many endpoints each zone should consume
  Writes: hints on each EndpointSlice endpoint:
    hints:
      forZones:
      - name: us-east-1a   ← this endpoint is "for" us-east-1a

KUBE-PROXY ON EACH NODE:
  Reads its node's zone label (topology.kubernetes.io/zone)
  Only programs routes to endpoints with hints matching its zone
  Falls back to all endpoints if:
    - No hints available
    - Local zone has no endpoints
    - Endpoints < 3 (too few to distribute meaningfully)

WHEN "auto" DOES NOT ACTIVATE:
  Endpoints < 3
  Traffic is imbalanced (one zone gets >20% more traffic)
  Any endpoint is not Ready
  Multiple services using same endpoint set
  → Falls back to uniform distribution (all endpoints)

VERIFY HINTS ARE SET:
kubectl get endpointslices -n production \
  -l kubernetes.io/service-name=api-service \
  -o yaml | grep -A 5 "hints:"
# hints:
#   forZones:
#   - name: us-east-1a   ← hint set on this endpoint
```

---

### InternalTrafficPolicy — Node-Local Routing

```yaml
# InternalTrafficPolicy: Local — route to pods on SAME NODE only
# Use case: DaemonSet sidecars, node-local caches, log agents
apiVersion: v1
kind: Service
metadata:
  name: node-local-cache
  namespace: production
spec:
  selector:
    app: redis-cache          # DaemonSet redis on every node
  internalTrafficPolicy: Local
  # Cluster (default): route to any endpoint
  # Local: only route to endpoints on the SAME node as the caller
  ports:
  - port: 6379
    targetPort: 6379
```

```
INTERNALTRAFICPOLICY: LOCAL — USE CASES
═══════════════════════════════════════════════════════════════

Log aggregator DaemonSet:
  Every node has fluentd/filebeat pod.
  App pods send logs to "log-service".
  With Local: each app pod goes to its node's fluentd (no network hop).
  Without Local: might cross nodes unnecessarily.

Node-local cache:
  DaemonSet Redis on every node as L1 cache.
  App pods access "cache-service" → always hits local Redis.
  Cache entries are node-local (this is intentional — L1 cache).

Metrics collection:
  DaemonSet node exporter on every node.
  Prometheus uses Local service to always scrape same-node exporter.

IMPORTANT: If no local endpoints exist → traffic is DROPPED (not rerouted)
  Use only when you're certain a pod exists on every relevant node.
  DaemonSets satisfy this (one pod per node).
```

---

### ExternalTrafficPolicy Recap

```yaml
# ExternalTrafficPolicy: Local — for LoadBalancer/NodePort
# Preserves client source IP, avoids extra hop
apiVersion: v1
kind: Service
metadata:
  name: api-lb
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  # Local: node only routes to pods on THIS node
  #        no SNAT → source IP preserved
  #        cloud LB health-checks nodes → only routes to nodes with local pods
  # Cluster (default): any node can forward to any pod
  #                    extra hop possible, SNAT applied → source IP lost
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

---

### 🎤 Short Crisp Interview Answer

> *"Topology-aware routing solves cross-AZ traffic cost. Without it, a pod in us-east-1a hitting a Service randomly hits pods in other AZs 66% of the time — that's real cross-AZ data transfer costs. The annotation service.kubernetes.io/topology-mode: auto tells the EndpointSlice controller to add zone hints to endpoints, and kube-proxy on each node then prefers endpoints with a matching zone hint. It falls back to uniform distribution if there are fewer than 3 endpoints or imbalanced traffic. InternalTrafficPolicy: Local is different — it routes to pods only on the same node, not same zone. It's ideal for DaemonSet sidecars like log agents, ensuring app pods always hit their node-local sidecar with zero network hop."*

---

---

# 3.9 EndpointSlices vs Endpoints

## 🟡 Intermediate

### Why EndpointSlices were created

```
THE PROBLEM WITH ENDPOINTS (classic API object)
═══════════════════════════════════════════════════════════════

Classic Endpoints object:
  One Endpoints object per Service
  Contains ALL pods' IPs in a single object

SCALE PROBLEM:
  Service with 1,000 pods:
  → Single Endpoints object: 1,000 IP entries
  → Object size: ~100KB
  → Any pod added/removed/becomes unready:
    → Entire 100KB object updated in etcd
    → Entire 100KB object sent to EVERY kube-proxy on EVERY node
    → 100 nodes × 100KB = 10MB of API traffic per endpoint change!
  
  High-churn environment (rolling updates, auto-scaling):
  Endpoint object constantly changing at 100KB per update
  → API Server flooded with large object updates
  → kube-proxy CPU spikes processing large updates
  → etcd write amplification

ENDPOINTSLICE SOLUTION:
  Instead of one large object: many small objects (max 100 endpoints each)
  1,000 pods → 10 EndpointSlice objects (100 each)
  
  One pod changes: only ONE EndpointSlice updated (containing that pod)
  Propagation: that small EndpointSlice sent to all nodes
  → 10KB instead of 100KB per event
  → 10x less data transferred per endpoint change
  → 10x less etcd churn
```

---

### EndpointSlice vs Endpoints Comparison

```
ENDPOINTSLICE vs ENDPOINTS — FEATURE COMPARISON
═══════════════════════════════════════════════════════════════

Feature              Endpoints          EndpointSlice
─────────────────────────────────────────────────────────────
API version          v1                 discovery.k8s.io/v1
Created since        K8s 1.0            K8s 1.17 (GA: 1.21)
Default since        always             K8s 1.21+
Max entries per obj  unlimited          100 (configurable)
Object per service   1                  N (ceil(pods/100))
Update granularity   whole service      per-slice (100 pods)
Topology hints       No                 Yes (forZones)
Dual-stack (IPv4+6)  Separate objects   Single object, two addr types
Address types        IP only            IPv4, IPv6, FQDN
Node name tracking   No                 Yes (nodeName per endpoint)
Zone tracking        No                 Yes (zone per endpoint)
Serving/Terminating  No (just Ready)    Yes (3 conditions)

ENDPOINT CONDITIONS IN ENDPOINTSLICE:
  ready:       pod passed readiness probe → should receive traffic
  serving:     pod is serving traffic (may still be true during termination)
  terminating: pod received SIGTERM, graceful shutdown in progress
  → "serving but terminating" = still accepts connections during drain
  → kube-proxy can send new connections until serving=false
  → Enables proper connection draining during pod shutdown
```

---

### Inspecting EndpointSlices

```bash
# List EndpointSlices for a service
kubectl get endpointslices -n production \
  -l kubernetes.io/service-name=api-service

# NAME                   ADDRESSTYPE   PORTS   ENDPOINTS   AGE
# api-service-abc12      IPv4          8080    10.244.1.5,10.244.2.3   5d
# api-service-def34      IPv4          8080    10.244.3.7,10.244.4.1   5d

# Detailed view with all conditions
kubectl get endpointslices -n production \
  -l kubernetes.io/service-name=api-service \
  -o yaml

# What the YAML shows per endpoint:
# addresses:
# - 10.244.1.5
# conditions:
#   ready: true
#   serving: true
#   terminating: false
# hints:
#   forZones:
#   - name: us-east-1a
# nodeName: worker-1
# targetRef:
#   kind: Pod
#   name: api-abc123
#   namespace: production
# topology:
#   kubernetes.io/hostname: worker-1
#   topology.kubernetes.io/zone: us-east-1a

# Compare with classic Endpoints
kubectl get endpoints api-service -n production
# NAME          ENDPOINTS                               AGE
# api-service   10.244.1.5:8080,10.244.2.3:8080,...    5d

# Count how many EndpointSlices exist per service (for large deployments)
kubectl get endpointslices -A \
  -o custom-columns=\
'SERVICE:.metadata.labels.kubernetes\.io/service-name,\
ENDPOINTS:.endpoints[*].addresses[0]' | head -20

# See terminating endpoints (during rolling update)
kubectl get endpointslices -n production \
  -l kubernetes.io/service-name=api-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" terminating="}{.conditions.terminating}{"\n"}{end}'
```

---

### 🎤 Short Crisp Interview Answer

> *"Endpoints is the classic API object with all pod IPs in a single object — fine for small services, but at 1,000 pods that's a 100KB object that gets fully rewritten on every pod change, sent to every kube-proxy on every node. At 100 nodes that's 10MB of API traffic per endpoint event. EndpointSlices cap at 100 entries per object — a 1,000-pod service gets 10 slices. When one pod changes, only the slice containing it is updated: 10KB instead of 100KB, sent to all nodes. EndpointSlices also add topology information (zone, nodeName), three-state conditions (ready/serving/terminating) for proper connection draining, and native dual-stack support. They've been the default since Kubernetes 1.21."*

---

---

# 3.10 Headless Services — When and Why

## 🟡 Intermediate

### What Headless Services are

```
HEADLESS SERVICE — DEFINITION
═══════════════════════════════════════════════════════════════

A headless service is created by setting clusterIP: None.

NORMAL SERVICE:
  DNS query: api-service.production.svc.cluster.local
  → Returns: 10.96.45.23  (single ClusterIP virtual IP)
  → kube-proxy DNAT picks one pod
  → Client has NO idea which pod it's talking to

HEADLESS SERVICE:
  DNS query: api-service.production.svc.cluster.local
  → Returns: 10.244.1.5, 10.244.2.3, 10.244.3.7  (ALL pod IPs directly!)
  → No load balancing by kube-proxy (no ClusterIP, no iptables rules)
  → Client receives all pod IPs and picks one itself
  → Client KNOWS which pod it's talking to

WHY THIS MATTERS:
  Some distributed systems need to connect to SPECIFIC peers:
  - Kafka brokers: consumers need to know all broker addresses
  - Cassandra: peer discovery requires all node addresses
  - StatefulSets: postgres-0 is the primary, postgres-1/2 are replicas
    → replica must connect to primary specifically, not any random pod
  - Elasticsearch: nodes need to form a cluster with each other

  Headless = "I'll handle the discovery myself, give me all IPs"
  Normal   = "Pick one for me and route to it" (load balancer)
```

---

### Headless Service with StatefulSet

```yaml
# ── Headless Service for StatefulSet (most common use) ───────────
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None            # ← THIS makes it headless
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # ← links StatefulSet to headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
```

```
STATEFULSET DNS — STABLE POD ADDRESSES
═══════════════════════════════════════════════════════════════

With serviceName: postgres-headless, each pod gets a stable DNS name:

  postgres-0.postgres-headless.production.svc.cluster.local → 10.244.1.5
  postgres-1.postgres-headless.production.svc.cluster.local → 10.244.2.3
  postgres-2.postgres-headless.production.svc.cluster.local → 10.244.3.7

These DNS names are STABLE even if the pod restarts:
  postgres-0 crashes → new pod created → SAME NAME postgres-0
  → DNS still resolves to whatever IP the new postgres-0 gets
  → Other pods configured to reach postgres-0 still work

WITHOUT HEADLESS SERVICE:
  Pod IP changes on restart
  No stable per-pod DNS
  Cannot implement primary/replica logic

QUERYING HEADLESS SERVICE DNS (returns all pod IPs):
kubectl run test --image=nicolaka/netshoot --rm -it --restart=Never -- bash
# Inside:
nslookup postgres-headless.production.svc.cluster.local
# Server: 10.96.0.10
# Name:   postgres-headless.production.svc.cluster.local
# Address: 10.244.1.5   ← pod 0 IP
# Address: 10.244.2.3   ← pod 1 IP
# Address: 10.244.3.7   ← pod 2 IP
# (all three returned — client picks)

nslookup postgres-0.postgres-headless.production.svc.cluster.local
# Address: 10.244.1.5   ← specific pod 0 only
```

---

### Headless Service Without Selector

```yaml
# Headless service WITHOUT selector — manual endpoint management
# Use case: external services that need DNS discovery
apiVersion: v1
kind: Service
metadata:
  name: external-kafka
  namespace: production
spec:
  clusterIP: None
  ports:
  - port: 9092

---
# Manually create EndpointSlice pointing to external Kafka brokers
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-kafka-1
  namespace: production
  labels:
    kubernetes.io/service-name: external-kafka
addressType: IPv4
endpoints:
- addresses: ["10.1.2.3"]     # Kafka broker 1 (external)
- addresses: ["10.1.2.4"]     # Kafka broker 2 (external)
- addresses: ["10.1.2.5"]     # Kafka broker 3 (external)
ports:
- port: 9092

# DNS query for external-kafka.production.svc.cluster.local
# → Returns: 10.1.2.3, 10.1.2.4, 10.1.2.5
# App gets all broker IPs — real Kafka bootstrap behavior
```

---

### Headless vs Normal vs ExternalName

```
CHOOSING THE RIGHT SERVICE TYPE
═══════════════════════════════════════════════════════════════

Normal ClusterIP:
  You want: one stable IP, load balanced across pods
  Client knows: nothing about individual pods
  Use: stateless microservices (API servers, web apps)

Headless (clusterIP: None):
  You want: stable per-pod DNS, or all pod IPs for client-side LB
  Client knows: all pod IPs and/or specific pod identities
  Use: StatefulSets (databases, Kafka, Cassandra, Elasticsearch)
       Peer discovery in distributed systems
       Client-side load balancing (gRPC, Kafka consumers)

ExternalName:
  You want: internal DNS alias for an external service
  No pod involvement: pure DNS CNAME
  Use: RDS database DNS, external API endpoints
       Migration: start external, later replace with real pods

ClusterIP + multiple ports:
  Use: expose metrics on separate port from app traffic
  Both ports behind same virtual IP
```

---

### 🎤 Short Crisp Interview Answer

> *"A headless service has clusterIP: None. Where a normal service returns a single virtual ClusterIP, a headless service's DNS returns all backing pod IPs directly. kube-proxy creates no iptables rules for it — there's no virtual IP to proxy. The primary use case is StatefulSets: when serviceName references a headless service, each pod gets a stable DNS name like postgres-0.postgres-headless.namespace.svc.cluster.local that survives pod restarts. This enables distributed systems like Postgres, Kafka, and Cassandra to discover and connect to specific peers — the primary replica, or all Kafka broker addresses for bootstrap."*

---

---

# 3.11 Cilium eBPF — Replacing kube-proxy Entirely

## 🔴 Advanced

### eBPF Architecture

```
EBPF (EXTENDED BERKELEY PACKET FILTER) IN LINUX
═══════════════════════════════════════════════════════════════

eBPF lets you run SANDBOXED PROGRAMS in the Linux kernel
without modifying kernel source or loading kernel modules.

TRADITIONAL KERNEL EXTENSION:
  Write kernel module → load into kernel → kernel panic risk
  No safety guarantees, full kernel access

EBPF:
  Write eBPF program → verifier checks safety → JIT compile
  → Run in kernel at hook points (network, syscalls, tracing)
  Verifier guarantees: no infinite loops, no out-of-bounds, no crashes

EBPF HOOK POINTS CILIUM USES:
  XDP (eXpress Data Path):   runs on NIC driver, before kernel networking stack
  TC (Traffic Control):      runs in kernel networking after XDP
  Socket-level hooks:        intercept connect(), sendmsg() calls
  cgroup hooks:              per-cgroup (per-container) policies

PACKET FLOW WITHOUT CILIUM (traditional):
  NIC → kernel network stack → iptables PREROUTING → routing
  → iptables FORWARD → iptables OUTPUT → POSTROUTING → NIC
  Every packet: multiple iptables chain traversals

PACKET FLOW WITH CILIUM EBPF:
  NIC → XDP/TC eBPF program → direct action (forward/drop/redirect)
  → Destination (possibly bypassing most kernel network stack)
  Service lookup: eBPF map (hash) O(1) vs iptables O(n)
  Encapsulation: handled in eBPF, not userspace
```

---

### Cilium as kube-proxy Replacement

```yaml
# Install Cilium WITHOUT kube-proxy (Helm)
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443

# What Cilium does when replacing kube-proxy:
# 1. Watches API Server for Service/EndpointSlice changes (same as kube-proxy)
# 2. Writes eBPF maps instead of iptables rules
# 3. eBPF programs on each node do Service DNAT in kernel
# 4. kube-proxy DaemonSet can be removed entirely

# EKS: disable kube-proxy addon before installing Cilium
aws eks delete-addon \
  --cluster-name my-cluster \
  --addon-name kube-proxy
```

---

### Cilium eBPF Performance Advantages

```
EBPF vs IPTABLES vs IPVS — PERFORMANCE
═══════════════════════════════════════════════════════════════

SERVICE LOOKUP COMPLEXITY:
  iptables:   O(n) linear scan through all rules
  IPVS:       O(1) hash table (kernel LVS)
  eBPF:       O(1) hash map (eBPF map, even faster than IPVS)

CONNTRACK BYPASS:
  iptables/IPVS: require conntrack to track NATed connections
  eBPF:          can bypass conntrack for trusted pod traffic
  → Less memory, less CPU per connection

SOCKET-LEVEL ACCELERATION:
  Traditional: Pod → veth → bridge → iptables DNAT → route → veth → Pod
  Cilium:      Pod connect() → eBPF intercepts at socket level
               → Rewrites destination to pod IP before packet is formed
               → No actual packet NAT needed!
               → Bypasses kernel network stack for same-node traffic

BENCHMARK NUMBERS (approximate, varies by workload):
  Same-node pod-to-pod:
    iptables:   ~1Gbps, ~100μs p99
    eBPF:       ~10Gbps, ~10μs p99 (socket-level bypass)
  
  Service lookup at 10,000 services:
    iptables: measurable CPU overhead per packet
    eBPF:     flat (hash table constant time)

HUBBLE — BUILT-IN OBSERVABILITY:
  Every network flow visible without overhead:
  hubble observe --namespace production
  # Jan 15 10:00:01.234  production/frontend → production/api:8080  FORWARDED
  # Jan 15 10:00:01.235  production/api → production/postgres:5432  FORWARDED
  # Jan 15 10:00:01.240  production/scanner → production/api:8080   DROPPED (policy)
  
  No sidecar required (unlike Istio for observability)
  Flows captured at kernel level — zero overhead on application
```

---

### Cilium L7 NetworkPolicy

```yaml
# L7 (HTTP-aware) NetworkPolicy — only Cilium supports this natively
# Allow frontend to call GET /api/* but not DELETE /admin/*
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api

  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/api/.*"      # allow GET on /api/* paths
        - method: POST
          path: "/api/.*"      # allow POST on /api/* paths
        # DELETE, PUT to /admin/* = implicitly denied
```

---

### 🎤 Short Crisp Interview Answer

> *"Cilium uses eBPF — sandboxed kernel programs loaded at network hook points — to replace both kube-proxy and traditional CNI dataplane overhead. Where iptables does O(n) linear scans through rules, eBPF maps are O(1) hash lookups regardless of cluster size. Cilium can do socket-level acceleration for same-node traffic: intercepting the connect() syscall and rewriting the destination before a packet is even formed, eliminating veth hops and kernel NAT entirely. Hubble runs alongside Cilium and provides per-flow visibility for every network connection at kernel level with no sidecar overhead — you can see exactly which policy dropped a packet in real time. Cilium also adds L7 NetworkPolicy: HTTP method and path-aware rules that standard Kubernetes NetworkPolicy can't express."*

---

---

# 3.12 Multi-Cluster Networking

## 🔴 Advanced

### Why Multi-Cluster Networking

```
WHY YOU NEED MULTI-CLUSTER NETWORKING
═══════════════════════════════════════════════════════════════

SCENARIOS:
  Disaster Recovery:
    Active cluster (us-east-1) + standby cluster (us-west-2)
    Failover: redirect traffic to us-west-2 if us-east-1 fails

  Geographic Distribution:
    EU cluster (compliance) + US cluster + APAC cluster
    Services in each region, some shared across regions

  Environment Separation:
    Production cluster + shared services cluster (CI/CD, monitoring)
    Prod pods need to reach shared Prometheus, Vault, Artifactory

  Scale Beyond One Cluster:
    Very large workloads split across clusters
    Kubernetes cluster limits (~5,000 nodes, ~150,000 pods)

  Team Isolation:
    Different teams own different clusters
    Need service-to-service calls across cluster boundaries

PROBLEM: Kubernetes Service DNS only works within one cluster.
  api-service.production.svc.cluster.local → only resolves in THAT cluster
  Cross-cluster service discovery requires additional tools.
```

---

### Cilium Cluster Mesh

```
CILIUM CLUSTER MESH ARCHITECTURE
═══════════════════════════════════════════════════════════════

Each cluster runs Cilium.
A Clustermesh API Server runs in each cluster.
Clusters share endpoint information via etcd federation.

SETUP:
  cilium clustermesh enable --context cluster-us-east
  cilium clustermesh enable --context cluster-us-west
  cilium clustermesh connect \
    --destination-context cluster-us-west \
    --source-context cluster-us-east

GLOBAL SERVICE:
  Annotate a Service to be accessible across clusters:
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/shared: "true"

  Now:
  api-service.production.svc.cluster.local
  → Resolves in cluster A: returns pods from cluster A AND cluster B
  → DNS returns pod IPs from both clusters
  → kube-proxy/Cilium load balances across cluster boundaries

FAILOVER:
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/affinity: "local"   # prefer local cluster
  → Sends traffic to local cluster first
  → Falls back to remote cluster if local pods are unavailable
  → Active-passive DR pattern without DNS TTL delays
```

---

### Submariner

```yaml
# Submariner — CNI-agnostic multi-cluster networking
# Works with: Calico, Flannel, OVN-Kubernetes, Canal, etc.
# (Not tied to Cilium)

# Architecture:
# Each cluster runs a Submariner Gateway node
# Gateway nodes form encrypted tunnels (WireGuard or IPsec)
# Route table entries added for remote cluster pod/service CIDRs

# REQUIREMENTS:
#   Clusters must have non-overlapping pod and service CIDRs
#   Cluster A pods: 10.244.0.0/16  Cluster B pods: 10.245.0.0/16
#   (NOT: both 10.244.0.0/16 — conflicts!)

# INSTALLATION (via subctl CLI)
subctl deploy-broker --kubeconfig cluster-a.yaml
subctl join broker-info.subm \
  --kubeconfig cluster-a.yaml \
  --clusterid cluster-a
subctl join broker-info.subm \
  --kubeconfig cluster-b.yaml \
  --clusterid cluster-b

# SERVICE EXPORT — make a service available across clusters
# Using ServiceExport (Kubernetes MCS API):
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: api-service
  namespace: production
# Now api-service.production.svc.clusterset.local resolves
# across all clusters that have joined the broker

# SUBMARINER FEATURES:
# - Encrypted tunnels (WireGuard default, IPsec optional)
# - Service discovery (Lighthouse DNS plugin)
# - Network diagnostics (subctl diagnose)
# - CNI agnostic (works with any CNI)
# - Broker pattern: one cluster hosts broker, others join
```

---

### Multi-Cluster DNS Patterns

```
DNS PATTERNS FOR MULTI-CLUSTER
═══════════════════════════════════════════════════════════════

PATTERN 1: Local + Global DNS (Cilium Cluster Mesh)
  api-service.production.svc.cluster.local   → local cluster only
  api-service.production.svc.clusterset.local → all clusters (global)

PATTERN 2: External DNS federation
  Use ExternalDNS to write Service records to Route53/CloudFlare
  api.us-east.company.internal → NLB in us-east
  api.us-west.company.internal → NLB in us-west
  api.company.internal         → Route53 latency routing → nearest

PATTERN 3: Service Mesh federation (Istio)
  Istio ServiceEntry references services in other clusters
  mTLS tunneled across cluster boundary
  Full observability and traffic management

PATTERN 4: Gateway federation
  Expose cross-cluster services via dedicated Gateway pods
  Other clusters treat remote services as external endpoints
  Simple but adds a hop
```

---

### 🎤 Short Crisp Interview Answer

> *"Multi-cluster networking is needed for DR, geo-distribution, and scale beyond a single cluster. Cilium Cluster Mesh is the most elegant solution if you're already on Cilium — clusters share endpoint information via clustermesh API servers, and annotating a service as global: true makes it DNS-resolvable across all clusters with automatic failover. Submariner is the CNI-agnostic option — it creates encrypted WireGuard tunnels between cluster gateways and enables cross-cluster service discovery via the Kubernetes MCS API (ServiceExport/ServiceImport). The critical prerequisite for Submariner: all clusters must have non-overlapping pod CIDRs — you can't merge two clusters where both use 10.244.0.0/16."*

---

---

# ⚠️ 3.13 Ingress vs Gateway API

## 🔴 Advanced — HIGH PRIORITY

### Why Gateway API was created

```
PROBLEMS WITH INGRESS
═══════════════════════════════════════════════════════════════

PROBLEM 1: Too many annotations
  Every Ingress controller uses different annotations for same features:
  nginx:  nginx.ingress.kubernetes.io/rate-limit: "100"
  traefik: traefik.ingress.kubernetes.io/rate-limit: "100"
  ALB:    alb.ingress.kubernetes.io/wafv2-acl-arn: "arn:..."
  Non-portable: YAML tied to specific controller

PROBLEM 2: No role separation
  Single Ingress resource configures:
    - Infrastructure (which LB, TLS certs) — ops team responsibility
    - Routing rules (which path to which service) — dev team responsibility
  No way to separate concerns — everything in one object

PROBLEM 3: Limited expressiveness
  No traffic weighting (canary 10%/90% split) in standard spec
  No header-based routing in standard spec
  No TCP/UDP routing (HTTP only)
  All "advanced" features need controller-specific annotations

PROBLEM 4: Single namespace scope
  Ingress routes to Services in the same namespace only
  Multi-team: each team needs their own Ingress controller
  OR: platform team manages all routing for all teams

GATEWAY API DESIGN PRINCIPLES:
  Role-oriented: 3 separate resources for 3 different personas
  Portable: standard spec, no annotation extensions needed
  Expressive: traffic weighting, header matching, URL rewrites built-in
  Extensible: well-defined extension points (filters, policies)
  Cross-namespace: routes can target backends in other namespaces
```

---

### Gateway API Objects — Role Separation

```
GATEWAY API ROLE MODEL
═══════════════════════════════════════════════════════════════

PERSONA 1: Infrastructure Provider (cloud/platform team)
  Object: GatewayClass
  Responsibility: define which controller/LB implementation to use
  Like: IngressClass but richer

PERSONA 2: Cluster Operator (platform/ops team)
  Object: Gateway
  Responsibility: provision an actual LB instance, configure TLS
  Binds to a GatewayClass
  Defines: which listeners (ports, protocols, TLS)
  Controls: which namespaces can attach routes to this gateway

PERSONA 3: Application Developer (dev team)
  Objects: HTTPRoute, TCPRoute, GRPCRoute, TLSRoute
  Responsibility: define routing rules for their app
  Attaches to: a Gateway (that the ops team owns)
  Cannot change: LB config, TLS certs (ops controls that)

SEPARATION OF CONCERNS:
  Ops creates ONE Gateway → devs attach many Routes
  Dev deploys new service → creates HTTPRoute pointing at their Service
  Dev cannot modify the Gateway (no RBAC permission)
  Ops can change LB config without touching dev's routing rules
  Perfect GitOps: ops repo has GatewayClass + Gateway
                  dev repo has HTTPRoute
```

---

### Gateway API YAML

```yaml
# ── GatewayClass (infrastructure provider) ───────────────────────
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: aws-nlb
spec:
  controllerName: gateway.k8s.aws/nlb    # AWS LBC controller
  description: "AWS Network Load Balancer"

---
# ── Gateway (ops/platform team) ──────────────────────────────────
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: gateway-infra              # ops namespace
spec:
  gatewayClassName: aws-nlb
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: production-tls
        namespace: gateway-infra
    allowedRoutes:
      namespaces:
        from: Selector                  # only specific namespaces
        selector:
          matchLabels:
            gateway-access: "true"      # namespace must have this label
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All                       # allow any namespace for HTTP

---
# ── HTTPRoute (dev team — in their own namespace) ─────────────────
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-routes
  namespace: production                 # dev namespace
spec:
  parentRefs:
  - name: production-gateway
    namespace: gateway-infra            # references ops-owned Gateway
    sectionName: https                  # attach to HTTPS listener only

  hostnames:
  - api.company.com

  rules:
  # ── Traffic splitting (canary) ──
  - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: api-v2-service
      port: 80
      weight: 10                        # 10% to v2
    - name: api-v1-service
      port: 80
      weight: 90                        # 90% to v1 (canary rollout)

  # ── Header-based routing ──
  - matches:
    - headers:
      - name: X-Beta-User
        value: "true"
    backendRefs:
    - name: api-beta-service
      port: 80

  # ── URL rewrite ──
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /         # strip /api prefix
    backendRefs:
    - name: api-service
      port: 80

  # ── Request redirect ──
  - matches:
    - path:
        type: PathPrefix
        value: /old-path
    filters:
    - type: RequestRedirect
      requestRedirect:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /new-path
        statusCode: 301

---
# ── GRPCRoute (gRPC services) ─────────────────────────────────────
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-route
  namespace: production
spec:
  parentRefs:
  - name: production-gateway
    namespace: gateway-infra
  hostnames:
  - grpc.company.com
  rules:
  - matches:
    - method:
        service: com.company.UserService
        method: GetUser                 # route specific gRPC method
    backendRefs:
    - name: user-service
      port: 9090

---
# ── ReferenceGrant — allow cross-namespace backend reference ──────
# Dev team's HTTPRoute references a Service in another namespace?
# Must have a ReferenceGrant from that namespace
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-route-from-production
  namespace: shared-services           # the namespace BEING referenced
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: production              # allows routes FROM production
  to:
  - group: ""
    kind: Service                      # to reach Services here
```

---

### Ingress vs Gateway API Comparison

```
FEATURE COMPARISON
═══════════════════════════════════════════════════════════════

Feature               Ingress              Gateway API
─────────────────────────────────────────────────────────────
API version           networking.k8s.io/v1 gateway.networking.k8s.io/v1
GA since              K8s 1.19             K8s 1.28 (v1)
Role separation       No (1 object)        Yes (3 objects)
Traffic weighting     Annotation-only      Native (weight field)
Header routing        Annotation-only      Native (matches.headers)
URL rewrite           Annotation-only      Native (URLRewrite filter)
gRPC routing          No                   Yes (GRPCRoute)
TCP routing           No                   Yes (TCPRoute)
Cross-namespace       Same namespace only  Yes (with ReferenceGrant)
Protocol support      HTTP/HTTPS           HTTP, HTTPS, TCP, UDP, TLS, gRPC
Portability           Annotation-coupled   Controller-independent
Extension model       Annotations          Policy Attachment API
Maturity              Stable (years)       Growing ecosystem

WHEN TO USE INGRESS:
  Simple HTTP/HTTPS routing
  Already heavily invested in nginx/ALB Ingress
  Team not ready for new abstraction
  Features needed are available in existing controller

WHEN TO USE GATEWAY API:
  Multi-team cluster: clear role separation needed
  Canary/weighted routing required (without annotation hacks)
  gRPC or TCP/UDP routing needed
  New cluster: start with the better long-term foundation
  GitOps: clean separation of infra vs app routing config

MIGRATION:
  Many controllers now support BOTH Ingress and Gateway API
  nginx-gateway-fabric: Gateway API implementation for nginx
  AWS Load Balancer Controller: supports both
  Traefik 3.x: full Gateway API support
  Can run both in parallel during migration
```

---

### 🎤 Short Crisp Interview Answer

> *"Ingress has three major problems: features require controller-specific annotations making YAML non-portable, everything is in one resource so ops and dev can't have separate concerns, and it only supports HTTP. Gateway API addresses all of these with three role-oriented resources: GatewayClass (infra provider defines the implementation), Gateway (ops team provisions an LB instance with listeners and TLS), and HTTPRoute/GRPCRoute (dev teams attach their routing rules). Traffic weighting for canary deployments, header-based routing, and URL rewrites are all native fields — no annotations needed. ReferenceGrant enables cross-namespace routing with explicit permission. Gateway API reached v1 GA in Kubernetes 1.28 and is the forward direction — I'd start all new clusters on it."*

---

---

# 3.14 Pod-to-Pod Encryption (WireGuard, IPsec via Cilium)

## 🔴 Advanced

### Why Encrypt Pod Traffic

```
WHY POD-TO-POD ENCRYPTION MATTERS
═══════════════════════════════════════════════════════════════

DEFAULT STATE: all pod-to-pod traffic is UNENCRYPTED
  Pod on Node 1 → Pod on Node 2: plaintext on the wire
  Even in a VPC: other EC2 instances on the same host could sniff
  Shared tenancy on cloud hypervisors: network traffic potentially visible
  Insider threat: compromised node can see all pod traffic passing through

COMPLIANCE DRIVERS:
  PCI-DSS:     encrypt cardholder data in transit (Requirement 4)
  HIPAA:       encrypt PHI in transit
  SOC 2:       encryption of data in transit
  ISO 27001:   information security in transmission
  Without encryption: difficult to pass security audits

WITHOUT ENCRYPTION (what you're relying on):
  Cloud provider hypervisor isolation
  VPC security group rules (L4 only, not encryption)
  TLS at the application layer (requires every app to implement it)

WITH TRANSPARENT ENCRYPTION:
  Automatic: no app changes needed
  Zero-trust: even cluster admins can't see pod traffic on the wire
  Compliance: checkmark for "data in transit encrypted"

TWO MAIN OPTIONS IN KUBERNETES:
  WireGuard:  modern, fast, simple key management, built into Linux 5.6+
  IPsec:      older standard, compatible with more hardware/firmware
              required in some regulated environments (FIPS compliant variants)
```

---

### WireGuard via Cilium

```
WIREGUARD ARCHITECTURE WITH CILIUM
═══════════════════════════════════════════════════════════════

WireGuard is a modern VPN protocol built into the Linux kernel.
Key properties:
  - Elliptic curve cryptography (Curve25519 for key exchange)
  - ChaCha20-Poly1305 for symmetric encryption
  - ~4,000 lines of code (IPsec is ~400,000) — simpler, more auditable
  - Kernel native (Linux 5.6+): no userspace daemon, minimal overhead
  - Perfect forward secrecy (keys rotated automatically)

CILIUM WIREGUARD MODE:
  Each Cilium node generates a WireGuard keypair at startup
  Public keys stored in CiliumNode objects (visible to all nodes)
  All nodes establish WireGuard tunnels to each other (full mesh)
  All pod-to-pod traffic routed through WireGuard tunnels
  Node-to-node traffic: encrypted automatically, transparent to pods

ENABLING:
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard \
  --set encryption.wireguard.persistentKeypair=true
  # persistentKeypair: keys survive pod restart (avoids re-keying)

VERIFY ENCRYPTION:
kubectl -n kube-system exec -it ds/cilium -- cilium-dbg encrypt status
# Encryption: Wireguard
# Keys in use: 1
# Peers:
#   Node 10.0.1.5: node-key-abc123 (us-east-1a)
#   Node 10.0.1.6: node-key-def456 (us-east-1b)
# Encrypted packets: 12847593
# Decrypted packets: 11923847

# Verify traffic is encrypted (tcpdump should show gibberish)
# On node: tcpdump -i eth0 -nn -X host 10.0.1.6
# Should see WireGuard UDP packets (port 51871), not HTTP content
```

---

### IPsec via Cilium

```bash
# IPSEC MODE — for compliance environments requiring FIPS/NIST standards

# Create IPsec pre-shared key secret
kubectl create secret generic cilium-ipsec-keys \
  --namespace kube-system \
  --from-literal=keys="3 rfc4106(gcm(aes)) \
    $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64) \
    128"

# Enable IPsec in Cilium
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=ipsec

# IPSEC vs WIREGUARD COMPARISON:
# WireGuard:
#   + Simpler (4K lines vs 400K)
#   + Faster (ChaCha20 is hardware-accelerated on modern CPUs)
#   + Lower latency
#   - Not FIPS 140-2 certified (ChaCha20 not in FIPS)
#   - Newer, less battle-tested in regulated environments

# IPsec:
#   + FIPS 140-2 compliant (AES-GCM is FIPS-approved)
#   + Industry standard (hardware offload available)
#   + Required in some DoD/government environments
#   - More complex key management
#   - Higher overhead than WireGuard
#   - Manual key rotation (Cilium handles this, but more moving parts)

# WHEN TO USE EACH:
# WireGuard: most production clusters, performance-sensitive, modern Linux
# IPsec:     regulated environments (FedRAMP, DoD IL2/4, FIPS compliance)
```

---

### Calico WireGuard

```bash
# Calico also supports WireGuard encryption
# Independent of Cilium — use if already on Calico

# Enable WireGuard on all nodes via FelixConfiguration
kubectl patch felixconfiguration default \
  --patch '{"spec":{"wireguardEnabled":true}}'

# Verify on a node
kubectl exec -it -n kube-system ds/calico-node -- \
  wg show
# interface: wireguard.cali
#   public key: abc123def456...
#   private key: (hidden)
#   listening port: 51820
#
# peer: ghi789jkl012...
#   endpoint: 10.0.1.6:51820
#   allowed ips: 10.244.2.0/24
#   latest handshake: 5 seconds ago
#   transfer: 14.25 MiB received, 12.87 MiB sent

# Calico WireGuard works at node-to-node level:
# All traffic between node CIDRs is encrypted
# Transparent to pods — no app changes needed
```

---

### Encryption Overhead

```
ENCRYPTION PERFORMANCE IMPACT
═══════════════════════════════════════════════════════════════

WIREGUARD OVERHEAD (approximate):
  Throughput reduction: 5-15% depending on packet size
  Latency addition:     ~10-50 microseconds per packet
  CPU overhead:         minimal on modern CPUs with AES-NI/ChaCha20 instructions
  Memory overhead:      ~minimal (kernel-native, no userspace daemon)

IPSEC OVERHEAD:
  Throughput reduction: 10-25%
  Latency addition:     20-100 microseconds
  With hardware offload (supported by some NICs): ~3-5% overhead

OPTIMIZATION STRATEGIES:
  1. Use instances with AES-NI support (AWS: all modern instance types)
     → Hardware-accelerated AES → near-zero overhead for IPsec AES-GCM
  2. WireGuard on nodes with ChaCha20 hardware acceleration
  3. Use MTU-sized packets (avoid fragmentation — encrypt once per MTU)
  4. Encrypt only sensitive namespaces:
     # Cilium node-to-node encryption can be namespace-selective
     # With Calico: FelixConfiguration.wireguardEnabled per node pool

WHEN ENCRYPTION IS FREE (practical cases):
  If workload is already TLS at app layer (most HTTP services):
    Double encryption = overhead at both CNI and app level
    Consider: node-to-node encryption OR app-level TLS, rarely both
  Latency-sensitive (sub-millisecond) intra-cluster traffic:
    Measure before/after — may not need encryption for internal calls
    Encrypt inter-cluster or public-facing traffic first
```

---

### 🎤 Short Crisp Interview Answer

> *"Pod-to-pod traffic is plaintext by default — compliance frameworks like PCI-DSS and HIPAA require encryption in transit. WireGuard is the modern choice: it's built into the Linux kernel, uses Curve25519 and ChaCha20 for excellent performance with ~5-15% overhead, and Cilium manages the full mesh keypair distribution via CiliumNode objects. IPsec is the alternative for regulated environments needing FIPS 140-2 compliance — AES-GCM is FIPS-approved while ChaCha20 isn't. Both options are transparent to applications — no code changes needed. Calico also supports WireGuard via FelixConfiguration if you're not on Cilium. The practical consideration is that adding node-level encryption on top of existing application-level TLS is double encryption — I'd pick one layer, usually transparent WireGuard for zero-change rollout."*

---

---

# Quick Reference — Category 3 Networking Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **Networking contract** | Every pod unique IP; pod-to-pod no NAT; nodes reach pods; pods see own real IP |
| **3 IP ranges** | Node IPs (VPC), Pod IPs (CNI assigned), Service IPs (virtual, kube-proxy only) |
| **EKS VPC CNI** | Pods get real VPC IPs via ENI secondary IPs — no overlay needed |
| **ClusterIP** | Virtual IP + DNS. Exists only in iptables/IPVS rules. DNAT to pod IPs |
| **NodePort** | Opens static port 30000-32767 on EVERY node |
| **LoadBalancer** | NodePort + cloud LB provisioned. One LB per Service = expensive at scale |
| **ExternalName** | DNS CNAME only. No virtual IP. No kube-proxy rules |
| **externalTrafficPolicy: Local** | Preserve source IP + no extra hop. Pods must be spread across nodes |
| **CoreDNS** | kube-system Deployment. ClusterIP is nameserver in every pod's /etc/resolv.conf |
| **Service DNS format** | `<svc>.<ns>.svc.cluster.local` |
| **ndots:5 gotcha** | Names with <5 dots: search domains tried first → 3 extra failed queries for external names |
| **Ingress** | HTTP routing rules. Needs a Controller. One LB for many Services |
| **Ingress pathType** | Exact (only that path), Prefix (path + children), ImplementationSpecific |
| **Flannel** | VXLAN overlay. No NetworkPolicy. Simple, low resource |
| **Calico** | BGP routing (no overlay). Full NetworkPolicy. Widely used on EKS |
| **Cilium** | eBPF dataplane. Replaces kube-proxy. L7 policy. Hubble observability |
| **VPC CNI** | EKS native. Real VPC IPs. Add Calico/Cilium for NetworkPolicy |
| **NetworkPolicy** | Whitelist model. First policy = all unmatched traffic denied |
| **AND vs OR gotcha** | Same list item (no dash between) = AND. Separate items (dash) = OR |
| **DNS always needed** | Allow egress port 53 UDP+TCP before any other policy — or nothing resolves |
| **iptables mode** | O(n) rule scan. Rule update = full table rewrite |
| **IPVS mode** | O(1) hash table. More LB algorithms. Better at >1000 services |
| **Topology hints** | `service.kubernetes.io/topology-mode: auto`. Prefers same-zone endpoints |
| **InternalTrafficPolicy: Local** | Route to pods on same node only. Drops if none. For DaemonSet sidecars |
| **EndpointSlice** | Max 100 entries. Incremental updates. Topology + terminating condition. Default K8s 1.21+ |
| **Headless service** | `clusterIP: None`. DNS returns all pod IPs. No kube-proxy rules. StatefulSet use |
| **StatefulSet stable DNS** | `pod-n.svc.ns.svc.cluster.local` via headless service + serviceName |
| **eBPF advantage** | O(1) service lookup. Socket-level bypass for same-node. ~10x vs iptables at scale |
| **Cilium L7 policy** | HTTP method + path matching. gRPC service + method. Kafka topic |
| **Cluster Mesh** | Cilium multi-cluster. `service.cilium.io/global: "true"` for cross-cluster service |
| **Submariner** | CNI-agnostic multi-cluster. WireGuard tunnels. Non-overlapping pod CIDRs required |
| **Gateway API** | 3 objects: GatewayClass (infra) + Gateway (ops) + HTTPRoute (dev) |
| **Gateway API features** | Traffic weighting, header routing, URL rewrite, gRPC — all native, no annotations |
| **ReferenceGrant** | Allows HTTPRoute in ns-A to target Service in ns-B |
| **WireGuard** | Linux 5.6+. Curve25519 + ChaCha20. 5-15% overhead. Cilium/Calico support |
| **IPsec** | FIPS 140-2 compliant. AES-GCM. Higher overhead. Regulated environments |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| Default service CIDR | 10.96.0.0/12 |
| Default Flannel pod CIDR | 10.244.0.0/16 |
| CoreDNS default ClusterIP | 10.96.0.10 |
| DNS TTL in CoreDNS | 30 seconds |
| ndots setting | 5 (pods: fewer than 5 dots → search domains first) |
| NodePort range | 30000–32767 |
| iptables performance | O(n) per packet |
| IPVS performance | O(1) per lookup |
| EndpointSlice max entries | 100 per slice |
| Topology hints min endpoints | 3 (fewer = disabled) |
| WireGuard port | 51820–51871 (UDP) |
| WireGuard overhead | ~5–15% throughput |
| IPsec overhead | ~10–25% throughput |
| Gateway API GA | Kubernetes 1.28 |
| Cilium min kernel | 4.9 (basic), 5.4 (full features), 5.10 (WireGuard) |
| kube-proxy rule update | Full iptables-restore (all rules, not incremental) |

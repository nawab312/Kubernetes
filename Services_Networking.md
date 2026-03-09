**Why Services Exist**
```bash
Pods are temporary — they die and get replaced constantly
Every time a Pod restarts, it gets a NEW IP address

Without Services:
  Pod A wants to talk to Pod B
  Pod B IP = 10.0.0.5
  Pod B crashes, restarts
  Pod B new IP = 10.0.0.8
  Pod A still trying 10.0.0.5 → connection fails

With Services:
  Service gets a stable IP that NEVER changes
  Pod A talks to Service IP
  Service forwards to whatever Pods are currently healthy
  Pod restarts? Service automatically updates 
```

### ClusterIP, NodePort, LoadBalancer ###

**ClusterIP — Internal Only**
- ClusterIP gives your Service a *stable internal IP* that only works inside the cluster.
```bash
┌─────────────────────────────────────────┐
│           Kubernetes Cluster            │
│                                         │
│  Pod A  ──→  Service (ClusterIP)  ──→  Pod B │
│              10.96.0.1:80               │
│                                         │
│  ← nothing outside can reach this →    │
└─────────────────────────────────────────┘
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP        # default — you can omit this line
  selector:
    app: backend         # forwards to pods with this label
  ports:
    - port: 80           # port the service listens on
      targetPort: 8080   # port the pod actually runs on
```
- When to use ClusterIP:
  - Backend APIs that only frontend pods need to reach
  - Databases — only your app should talk to them
  - Internal microservice communication
  - Anything that should NOT be exposed outside the cluster
 
**NodePort — Exposes on Every Node**
- NodePort opens a port on every single node in your cluster and forwards traffic to your Service.
```bash
┌─────────────────────────────────────────────────┐
│                Kubernetes Cluster               │
│                                                 │
│  Node 1 (IP: 10.0.0.1)                         │
│    port 30080 ──→  Service  ──→  Pod            │
│                                                 │
│  Node 2 (IP: 10.0.0.2)                         │
│    port 30080 ──→  Service  ──→  Pod            │
│                                                 │
│  Node 3 (IP: 10.0.0.3)                         │
│    port 30080 ──→  Service  ──→  Pod            │
└─────────────────────────────────────────────────┘
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80           # ClusterIP port (internal)
      targetPort: 3000   # Pod port
      nodePort: 30080    # external port on every node (30000-32767)
```
- Problems with NodePort:
  - You expose node IPs directly — nodes should not be public
  - Port range limited to 30000-32767 — ugly URLs
  - If a node dies, that IP stops working
  - No load balancing across nodes
- When to use NodePort:
  - Local development or testing
  - On-premise clusters with external load balancer in front
  - Quick debugging — never in production on AWS
 
**LoadBalancer — Production External Access**
- LoadBalancer tells your cloud provider to create an actual load balancer in front of your Service.
- On AWS EKS this creates an AWS ELB or NLB automatically.
```bash
Internet
    ↓
AWS Load Balancer (ELB/NLB)
    ↓
NodePort on each node
    ↓
ClusterIP Service
    ↓
Pods
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  annotations:
    # tells AWS to create NLB instead of classic ELB
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
```
- After applying this, AWS creates an NLB and gives you a DNS name: `a1b2c3d4e5f6.elb.us-east-1.amazonaws.com`
- Problem with LoadBalancer per Service:
  - Every service gets its OWN load balancer
  - 10 services = 10 AWS load balancers = 10 × $18/month = $180/month
  - This is why Ingress exists — one load balancer for all services
 
---
---

### How kube-proxy Works ###
kube-proxy runs on every node in the cluster. It is responsible for making Services actually work by setting up network rules.

**The Problem kube-proxy Solves**
- You create a Service with ClusterIP 10.96.0.1
- But 10.96.0.1 is a VIRTUAL IP — No actual *Network Interface* has this IP
- Something needs to intercept traffic to 10.96.0.1 and redirect it to a real Pod IP
- That something is kube-proxy

**How kube-proxy works — iptables mode (default)**
```bash
Service created: ClusterIP = 10.96.0.1:80
Pods behind it:  10.0.0.5:8080, 10.0.0.6:8080, 10.0.0.7:8080
        ↓
kube-proxy watches Kubernetes API for Service/Endpoint changes
        ↓
kube-proxy writes iptables rules on every node:

  "Any traffic going to 10.96.0.1:80 →
   randomly forward to one of:
   10.0.0.5:8080 (33% chance)
   10.0.0.6:8080 (33% chance)
   10.0.0.7:8080 (33% chance)"
        ↓
Pod dies → Endpoint removed → kube-proxy updates iptables rules
New Pod starts → Endpoint added → kube-proxy updates iptables rules
```

**kube-proxy modes**
- iptables mode (default):
  - Uses Linux iptables rules
  - Random load balancing across pods
  - Rules evaluated linearly — slow with 1000s of services
- ipvs mode (better performance):
  - Uses Linux IPVS (IP Virtual Server)
  - Better load balancing algorithms (round-robin, least-connections)
  - O(1) rule lookup — much faster with many services
- eBPF mode (modern — used by Cilium):
  - Bypasses iptables entirely
  - Highest performance
  - Used when Cilium CNI replaces kube-proxy completely
 
---
---

### DNS in Kubernetes ###
Every Service and Pod gets a DNS nameautomatically. This is how microservices find each other by name instead of IP.

**How it works**
- CoreDNS runs as a Pod inside kube-system namespace. It is the DNS server for the entire cluster.
- Every Pod is automatically configured to use CoreDNS
```bash
/etc/resolv.conf inside every pod:
nameserver 10.96.0.10   ← CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
```

**Service DNS Format**
- Full DNS name of a Service: `<service-name>.<namespace>.svc.cluster.local`

**How a Pod resolves a Service name**
- Pod A (in default namespace) wants to reach backend-service
  - Step 1: Pod does DNS lookup for "backend-service"
  - Step 2: CoreDNS gets the query
  - Step 3: CoreDNS checks the search domains: `backend-service.default.svc.cluster.local`
  - Step 4: CoreDNS returns ClusterIP: 10.96.0.1
  - Step 5: Pod connects to 10.96.0.1
  - Step 6: kube-proxy iptables rules forward to actual Pod
 
**Pod DNS names**
- Pods also get DNS names but they are rarely used directly: `<pod-ip-dashes>.<namespace>.pod.cluster.local`
- Example:
```bash
Pod IP: 10.0.0.5
DNS:    10-0-0-5.default.pod.cluster.local
```
- StatefulSet Pods get predictable DNS names — this is one reason StatefulSets exist ->  `StatefulSet: mysql, namespace: default, service: mysql`
  ```bash
  mysql-0.mysql.default.svc.cluster.local
  mysql-1.mysql.default.svc.cluster.local
  mysql-2.mysql.default.svc.cluster.local
  ```
  - Each pod is individually addressable
  - Critical for databases where you need to talk to a specific replica

---
---

### Endpoints and EndpointSlices ###
When you create a Service, Kubernetes automatically creates an Endpoints objectthat tracks the IPs of all healthy Pods behind it.

**Endpoints**
```bash
Service: backend-service (selector: app=backend)
        ↓
Kubernetes finds all pods with label app=backend
        ↓
Creates Endpoints object:

Name: backend-service
Addresses:
  - 10.0.0.5:8080   (pod-1, healthy)
  - 10.0.0.6:8080   (pod-2, healthy)
  - 10.0.0.7:8080   (pod-3, healthy)
```
```bash
kubectl get endpoints backend-service

NAME               ENDPOINTS                                      AGE
backend-service    10.0.0.5:8080,10.0.0.6:8080,10.0.0.7:8080      5d
```
- When a Pod fails its readiness probe:
```bash
Pod 10.0.0.6 fails readiness probe
        ↓
Kubernetes removes it from Endpoints
        ↓
kube-proxy updates iptables — no more traffic to 10.0.0.6
        ↓
Pod recovers, passes readiness probe
        ↓
Kubernetes adds it back to Endpoints
```

**EndpointSlices — Why they replaced Endpoints**
- Problem with Endpoints:
  - Service has 1000 pods
  - One pod changes → Entire Endpoints object updated and sent to every node
  - With 100 nodes × 1000 pods = Massive network traffic just for updates
- EndpointSlices solution:
  - Endpoints split into slices of 100 pods each
  - One pod changes → Only ONE slice updated and sent to affected nodes
  - Much less network traffic

---
---

### Headless Services ###
A Headless Service is a Service with no ClusterIP. Instead of a virtual IP, DNS returns the actual Pod IPs directly.

**Regular Service vs Headless Service**
- Regular Service DNS lookup: `nslookup backend-service`
  - Returns 10.96.0.1 (single ClusterIP)
  - kube-proxy load balances to pods
  - Client never knows pod IPs
- Headless Service DNS lookup: `nslookup backend-service`
  - Returns 10.0.0.5, 10.0.0.6, 10.0.0.7 (actual pod IPs)
  - Client connects directly to pods
  - No kube-proxy involvement
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None      # ← this makes it headless
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

**Why Headless Services matter — StatefulSets**
- The main use case is StatefulSets with databases:
- MySQL StatefulSet with 3 replicas:
  - mysql-0 → Primary (Accepts Writes)
  - mysql-1 → Replica (Read Only)
  - mysql-2 → Replica (Read Only)
- With regular Service:
  - DNS returns one ClusterIP
  - Load balanced randomly across all 3 pods
  - Your app might send WRITES to a replica
- With Headless Service:
  - mysql-0.mysql.default.svc.cluster.local → 10.0.0.5 (Primary)
  - mysql-1.mysql.default.svc.cluster.local → 10.0.0.6 (Replica)
  - mysql-2.mysql.default.svc.cluster.local → 10.0.0.7 (Replica)
 
---
---

### Network Policies ###
By default all Pods can talk to all other Pods in a Kubernetes cluster. Network Policies let you restrict this.

**Network Policy concepts**
```bash
podSelector:       Which pods this policy applies TO
ingress:           Who is ALLOWED to send traffic IN
egress:            Who is ALLOWED to send traffic OUT
namespaceSelector: Allow traffic from a specific namespace
ipBlock:           Allow traffic from specific IP ranges
```
- Example 1 — Only allow frontend to talk to backend
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend-only
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend         # this policy applies to backend pods

  policyTypes:
    - Ingress              # restricting incoming traffic

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend   # ONLY frontend pods can send traffic in
      ports:
        - protocol: TCP
          port: 8080
```
- Example 2 — Deny ALL traffic then selectively allow. Best practice is default deny everything then open only what you need:
```yaml
# Step 1 — deny all ingress to all pods in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}      # applies to ALL pods
  policyTypes:
    - Ingress
    - Egress
```
```yaml
# Step 2 — selectively allow what you need
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

**Important — Network Policies need a CNI that supports them**
- Network Policies are just Kubernetes objects — rules on paper. The actual enforcement is done by the CNI plugin
- CNI plugins that support Network Policies:
  - Calico: Most feature-rich
  - Cilium: eBPF-based, very powerful
- CNI plugins that do NOT enforce Network Policies:
  - Flannel: Policies exist but are not enforced
- On AWS EKS:
  - Default CNI is AWS VPC CNI — does NOT enforce Network Policies
  - You need to either:
    - Enable Network Policy support in AWS VPC CNI (added recently)
    - Or install Calico alongside VPC CNI
    - Or replace with Cilium
   
---
---

### Pod-to-Pod Communication Across Nodes ###

**The Core Rule — Kubernetes Networking Model**
- Every Pod gets a unique IP address
- Pod can reach any other Pod using its IP directly
- NO NAT (Network Address Translation) between pods

**Same Node Communication**
```bash
Pod A (10.0.0.5) and Pod B (10.0.0.6) on Node 1

Pod A → Virtual Ethernet (veth) → Linux bridge (cbr0) → Pod B
All inside the node — Fast, Simple
```

**Cross Node Communication — How it works on AWS EKS**
- AWS EKS uses the AWS VPC CNI plugin which works differently from most CNI plugins:
- Standard CNI (Flannel, Calico):
  - Pods get IPs from a separate overlay network
  - Traffic between nodes is encapsulated (VXLAN tunnel)
  - Extra overhead for encapsulation/decapsulation
- AWS VPC CNI:
  - Pods get REAL AWS VPC IP addresses
  - Pod IPs are directly routable in the VPC
  - No encapsulation needed — native VPC routing
  - Lower latency, higher throughput
 
**How AWS VPC CNI assigns IPs**
```bash
Node starts up
        ↓
VPC CNI requests secondary IPs from AWS for the node's ENI
(Elastic Network Interface)
        ↓
AWS assigns real VPC IPs to the ENI
        ↓
These IPs are assigned to Pods as they start
        ↓
Pod on Node 1 (10.0.1.5) talks to Pod on Node 2 (10.0.2.8)
        ↓
Traffic goes through VPC routing directly
No tunnel, no encapsulation
```

**Full Journey — Pod A talks to Pod B on different node**
- Pod A on Node 1 (10.0.1.5) → Pod B on Node 2 (10.0.2.8)
  - Step 1: Pod A sends packet to 10.0.2.8
  - Step 2: Packet hits Node 1 network stack
  - Step 3: AWS VPC routing table knows 10.0.2.8 is on Node 2
  - Step 4: Packet sent directly to Node 2 via VPC
  - Step 5: Node 2 receives packet
  - Step 6: VPC CNI delivers to Pod B (10.0.2.8)
  - Step 7: Pod B receives packet
  - No tunnels, no NAT, direct VPC routing


  

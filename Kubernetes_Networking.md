**Service Discovery**
- Kubernetes provides service discovery through its *DNS (Domain Name System)* and *Kubernetes Services* mechanism.

*DNS-Based Discovery*
- Each Kubernetes Service gets a DNS name within the cluster (e.g., `my-service.default.svc.cluster.local`).
- When a pod wants to communicate with a service, it simply uses the service's DNS name, and Kubernetes resolves this to the appropriate set of pods backing the service.
- The DNS server within the cluster is managed by *CoreDNS* (or kube-dns in earlier versions), which provides internal name resolution.

*Service Abstraction:*
- A Kubernetes Service acts as a stable endpoint for a set of pods, abstracting away the actual pod IPs (which can change over time due to scaling or pod restarts).
- Kubernetes keeps track of the available pods for each service through labels and selectors, ensuring the service always points to the right set of pods.

**Internal Load Balancing**
- Kubernetes Service (ClusterIP) uses *iptables* or *IPVS* (depending on configuration) for load balancing traffic among the pods that back the service.
- The traffic is evenly distributed using methods like round-robin or least connections (depending on the implementation).
- *kube-proxy* manages the *iptables rules* that handle routing traffic to the appropriate pod, ensuring load balancing at the internal service level.

How does Kubernetes handle pod-to-pod communication across nodes
- **CNI (Container Network Interface)** – How Kubernetes assigns IPs to pods and enables networking across nodes.
- **Kube-Proxy** – How Kubernetes manages services and routes traffic efficiently.
- **Kubernetes DNS** – How internal pod-to-pod and service discovery happens dynamically.

By combining *CNI for pod IP allocation*, *kube-proxy for service routing*, and *Kubernetes DNS for discovery*, Kubernetes ensures that pod-to-pod communication remains seamless, even when pods are rescheduled or replaced.

**CNI (Container Network Interface) – How Pods Get IPs and Communicate Across Nodes**
- Kubernetes does not have a built-in networking solution; instead, it relies on CNI plugins to handle pod networking. Each pod gets a unique IP address from the CNI plugin, making inter-pod communication
- How CNI Works:
  - When a pod is created, the CNI plugin assigns it an IP address from the cluster’s network.
  - The plugin ensures that all pods, across all nodes, can communicate without NAT (Network Address Translation). This is achieved through Overlay Networking (e.g., Flannel, Calico VXLAN) or Routing-based Networking (e.g., Calico BGP, Cilium).
  - CNI sets up the necessary network routes on each node so that traffic can reach any pod, regardless of which node it runs on.

**Kube-Proxy – How Kubernetes Manages Traffic for Services**
- While CNI handles raw pod IP communication, kube-proxy ensures traffic routing and load balancing for Kubernetes Services.
- How kube-proxy Works:
  - Each Kubernetes Service gets a ClusterIP (virtual IP).
  - kube-proxy updates iptables (or IPVS rules) to forward service traffic to the correct pod.
  - When a pod behind a service dies and is replaced, kube-proxy updates the routing rules dynamically.

**Kubernetes DNS – How Pods Discover Other Pods and Services**
- Pods don’t usually communicate directly via IP addresses. Instead, they use Kubernetes DNS for service discovery.
- How Kubernetes DNS Works:
  - Each Kubernetes Service gets a DNS name: `service-name.namespace.svc.cluster.local`
  - When a pod queries this DNS name, it gets the ClusterIP of the service.
  - The service forwards traffic to healthy pods.

**Pod-to-Pod Communication Across Nodes – Step-by-Step Flow**
- *Pod Creation & IP Assignment (CNI)*
  - A new Pod A is scheduled on Node 1.
  - The CNI plugin assigns Pod A a unique IP.
  - Networking routes are updated to ensure Pod A can communicate with other pods.
- *Pod-to-Pod Communication (CNI Routing)*
  - Pod A (Node 1) wants to talk to Pod B (Node 2).
  - The CNI plugin on Node 1 routes or encapsulates the packet and forwards it to Node 2.
  - The CNI plugin on Node 2 decapsulates and delivers the packet to Pod B.
- *Service Communication & Load Balancing (kube-proxy)*
  - If Pod A contacts Service B, it sends traffic to Service B’s ClusterIP.
  - kube-proxy intercepts and forwards traffic to a healthy pod behind Service B.
- *Service Discovery (Kubernetes DNS)*
  - Instead of IPs, Pod A uses DNS (e.g., `service-b.default.svc.cluster.local`).
  - Kubernetes DNS resolves it to Service B’s ClusterIP.
  - Traffic flows through kube-proxy to reach an available Pod B.

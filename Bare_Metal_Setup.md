<img width="344" height="131" alt="image" src="https://github.com/user-attachments/assets/946ea29c-903b-40f0-a2d5-229b319cf6d7" />- Kubernetes is a container orchestration platform that automates deployment, scaling, and management of containerized applications.
- On bare metal, Kubernetes is used to provide the same benefits we get in the cloud—such as high availability, self-healing, and declarative deployments—but without relying on cloud-managed services.
- Since bare metal doesn’t provide built-in load balancers, storage, or networking, Kubernetes on bare metal requires configuring components like a CNI for networking, MetalLB or external load balancers for traffic, and a storage solution like NFS or Ceph

**Challenge My Thinking**
- “Kubernetes manages everything”: False. Kubernetes depends on external systems (networking, storage).
- “Bare metal is just cheaper cloud”: Not always. Ops cost can be higher.
- “Same setup as cloud”: Wrong. Load balancing & storage are DIY.

---

**How do you set up Kubernetes on bare metal?**
- To set up Kubernetes on bare metal, I first prepare the servers by configuring the OS, disabling swap, and installing a container runtime like containerd.
- Then I install Kubernetes components using kubeadm, initializing the control plane and joining worker nodes to the cluster.
- After the cluster is up, I configure networking using a CNI plugin such as Calico to enable pod-to-pod communication.
- Since bare metal does not provide a native load balancer, I configure an external load balancer or MetalLB for exposing services.
- Finally, I set up storage using solutions like NFS or distributed storage

*FOLLOW UP QUESTIONS*

**Why do we disable swap in Kubernetes?**
- Swap is disabled in Kubernetes because the scheduler and kubelet rely on accurate memory usage to make scheduling and eviction decisions.
- If swap is enabled, a node may appear to have free memory while it is actually swapping, which can lead to unpredictable performance and delayed Out-Of-Memory handling.
- Kubernetes expects memory pressure to be handled through OOM kills rather than swapping

**Why do you use kubeadm instead of a manual Kubernetes installation?**
- Kubeadm is preferred over manual installation because it provides a standardized and production-ready way to bootstrap a Kubernetes cluster with correct defaults.
- It automates complex tasks such as certificate generation, kubeconfig creation, control-plane setup, and etcd configuration, which are error-prone when done manually.

**How does traffic enter a Kubernetes cluster, especially on bare metal?**
- Traffic enters a Kubernetes cluster through Services and Ingress.
- External traffic first reaches a load balancer or external IP, which forwards the request to a Kubernetes Service.
- On bare metal, since there is no cloud load balancer, tools like MetalLB or an external load balancer are used to provide an external IP.
- The Service routes traffic to the appropriate pods using kube-proxy, and if Ingress is used, the Ingress controller handles HTTP routing to backend services.

<img width="344" height="131" alt="image" src="https://github.com/user-attachments/assets/e5cc383b-f6bc-433a-9f69-ea04dcd5825c" />

*Ingress is Layer 7, not a replacement for Services*
- Layer 4 (Transport) → TCP / UDP
- Layer 7 (Application) → HTTP / HTTPS, paths, hosts, headers
- What a Service actually does (Layer 4)
  - Exposes pods on a stable IP and port
  - Load-balances TCP/UDP
  - Has no idea about: URLs `(/api)`, Hostnames `(app.example.com)`, HTTP headers
  - Service types: ClusterIP, NodePort, LoadBalancer
- What Ingress does (Layer 7)
  - Works only with HTTP/HTTPS
  - Routes traffic based on:
    - Host `(app.example.com)`
    - Path `(/login, /api)`
  - Requires an Ingress Controller (NGINX, Traefik, HAProxy)
  - Ingress does NOT: Give pods an IP, Load-balance raw TCP by default, Replace Services
 
<img width="281" height="121" alt="image" src="https://github.com/user-attachments/assets/e260d3ad-a857-4fe0-abf0-e252747ce7e6" />

*Bare-metal specific nuance (important) On bare metal: Ingress still needs: NodePort or LoadBalancer Service MetalLB or external LB*

Step-by-step: how traffic really enters on bare metal
- Client sends request
  - `curl http://app.example.com`
  -  Where does this packet go?: It must hit a real IP address
- You need an external IP (this is the missing piece)
  - On cloud: Cloud provider auto-creates a LoadBalancer
  - On bare metal:
    - Kubernetes does nothing by default
    - Must provide one of these: MetalLB (software LB), External LB (HAProxy, F5), NodePort (exposes node IPs directly)
    - Without this → traffic dies before Ingress.
- Ingress Controller Service (this is critical)
  - Ingress controller is exposed via a Service:
    - NodePort OR
    - LoadBalancer (with MetalLB)
  ```yaml
  Ingress Controller
  Service type: LoadBalancer
  External IP: 192.168.1.100
  ```
  - Now traffic can enter the cluster.
- Ingress Controller (Layer 7)
  - Now—and only now—Ingress rules apply:
    - Host-based routing
    - Path-based routing
    - TLS termination
  - Ingress decides which Service to forward to.

<img width="292" height="158" alt="image" src="https://github.com/user-attachments/assets/f080bf8e-7ae0-4e23-b489-941dc2a1b6d7" />

*Can I run Ingress without MetalLB on bare metal?*
- Yes, using NodePort
- But it exposes node IPs directly and is usually not ideal for production

---
---


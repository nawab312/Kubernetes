- Kubernetes is a container orchestration platform that automates deployment, scaling, and management of containerized applications.
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

---

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

**What is etcd, and what happens if etcd goes down in a bare-metal cluster?**
- etcd is a distributed, strongly consistent key-value store used by Kubernetes to store the entire cluster state, including configuration, metadata, and desired state.
- All control-plane components such as the API server, scheduler, and controller manager depend on etcd to read and write cluster data.
- If etcd goes down or loses quorum in a bare-metal cluster, the Kubernetes API becomes unavailable for writes, meaning no new pods can be scheduled and no configuration changes can be made.

**Pressure-test My Understanding**
- If etcd is down, the whole cluster goes down: This is partially wrong.
- What actually happens:

<img width="576" height="302" alt="image" src="https://github.com/user-attachments/assets/ecf2daf5-6ab8-4e70-9270-3704cc750ff5" />

**What “quorum” actually means**
- Quorum means more than half of etcd nodes must be available; if that majority is lost, etcd stops accepting changes to protect cluster consistency.

---

**What happens if one worker node goes down in a Kubernetes cluster?**
- When a worker node goes down, the kubelet on that node stops reporting to the control plane, and the node is marked as NotReady.
- Pods running on that node are considered lost, and after a grace period, Kubernetes reschedules those pods onto healthy worker nodes if they are part of a ReplicaSet, Deployment, or StatefulSet with more than one replica.
- Stateless applications recover automatically, while stateful workloads depend on the underlying storage configuration.

*Important nuance*
- Does Kubernetes instantly reschedule?
  - No. Default grace period ≈ 40 seconds

*Stateful vs Stateless*
- Where does the application store its data?
  - Stateless Applications
    - The app does not store important data inside the pod
    - If the pod dies, nothing valuable is lost
    - Examples: Web frontend, API server, Auth service (tokens stored elsewhere), Reverse proxy (NGINX)
    - What happens when a node dies? Pod disappears, Kubernetes creates a new pod on another node, App continues normally
    - Why “no data loss”? Because: Data is in DB, cache, or external service.
  - Stateful applications
    - The app stores important data
    - Data must survive pod restarts
    - Examples: Databases (MySQL, PostgreSQL), Kafka, Redis (When Persistence Enabled)

- Two storage scenarios (this is the key)
  - Case 1: Local disk storage (Dangerous)
    - Pod stores data on: `/var/lib/app-data  (Node Disk)`
    - If the node dies: Disk is gone, Data is gone, New pod has empty data (Result: Data Loss)
  - Case 2: Network / Persistent storage (Safe)
    - Pod uses: NFS, Ceph, Cloud Disk (in Cloud)
    - If the node dies: Pod is recreated on another node. Same volume is reattached. Data is still there
    - Result: Recovery Possible
   
<img width="672" height="217" alt="image" src="https://github.com/user-attachments/assets/ebae2178-216f-48d8-8b26-f7fbdeb087ba" />

- Stateless apps don’t store data in the pod, so Kubernetes can freely recreate them, while stateful apps store data and require persistent storage to avoid data loss when pods or nodes fail.

- Pressure-test question
  - If I run a database in a Deployment with a PVC, is it still stateful?
    - Yes, because data exists
    - StatefulSet is recommended, but storage defines state
   
---

**How does storage work in Kubernetes on bare metal?**
- Kubernetes itself does not provide storage; it only provides an abstraction through PersistentVolumes and PersistentVolumeClaims.
- On bare metal, storage must be provisioned using external solutions like NFS, Ceph, Longhorn, or local disks.
- Applications request storage using a PersistentVolumeClaim, which is then bound to a PersistentVolume based on the StorageClass and availability.
- Data persists beyond pod restarts, and recovery from node failure depends on whether the storage is local or network-based.


The 4 building blocks (in order)
- PersistentVolume (PV)
  - Actual storage, Comes from NFS, Ceph, disk, etc., Exists outside pods
  - Think: Real Disk
- PersistentVolumeClaim (PVC)
  - A request for storage
  - Pod never talks to PV directly
  - Think: Storage Request
- StorageClass
  - Defines how storage is created
  - Needed for dynamic provisioning
  - Think: Storage Template
- Volume Mount
  - Attaches storage to the pod filesystem
  - Think: Disk plugged into container


Bare-metal reality (Important)
- Local storage
  - Fast, Node-specific, Node dies → data gone
- Network storage
  - Slower, Node-independent, Node dies → data recoverable


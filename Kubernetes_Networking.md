How does Kubernetes handle pod-to-pod communication across nodes, and what mechanisms ensure that a pod can always reach another pod, even when rescheduled or replaced? Explain in detail how CNI plugins, kube-proxy, and DNS resolution work together to maintain seamless communication.
- **CNI (Container Network Interface)** – How Kubernetes assigns IPs to pods and enables networking across nodes.
- **Kube-Proxy** – How Kubernetes manages services and routes traffic efficiently.
- **Kubernetes DNS** – How internal pod-to-pod and service discovery happens dynamically.
- **Pod Rescheduling & Endpoint Updates** – How networking stays intact even when pods restart or move.

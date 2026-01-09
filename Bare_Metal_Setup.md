- Kubernetes is a container orchestration platform that automates deployment, scaling, and management of containerized applications.
- On bare metal, Kubernetes is used to provide the same benefits we get in the cloud—such as high availability, self-healing, and declarative deployments—but without relying on cloud-managed services.
- Since bare metal doesn’t provide built-in load balancers, storage, or networking, Kubernetes on bare metal requires configuring components like a CNI for networking, MetalLB or external load balancers for traffic, and a storage solution like NFS or Ceph

**Challenge My Thinking**
- “Kubernetes manages everything”: False. Kubernetes depends on external systems (networking, storage).
- 

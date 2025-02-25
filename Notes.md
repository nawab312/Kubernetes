## What is Kubernetes? ##
Kubernetes (often abbreviated as K8s) is an open-source platform for automating the deployment, scaling, and management of containerized applications. It helps you manage applications that are composed of many containers across multiple machines, ensuring that the system is highly available, scalable, and resilient.

### Why to use Kubernetes if we have Docker? ###
- Docker is a powerful tool for creating and running containers, allowing developers to package applications with all their dependencies into lightweight, portable environments. However, Docker alone is not enough when managing multiple containers across different machines in a production environment.
- Kubernetes is a container orchestration platform that automates the deployment, scaling, and management of containerized applications.
- Unlike Docker, which requires manual intervention to start, stop, and scale containers, Kubernetes provides built-in features like automatic scaling, load balancing, self-healing, and rolling updates. It ensures high availability by restarting failed containers and distributing traffic efficiently across multiple instances
- Additionally, Kubernetes makes it easy to manage multi-container applications across multiple servers, ensuring consistency and reliability. In short, while Docker helps in containerizing applications, Kubernetes is essential for managing and orchestrating them at scale in production environments.

## Kubernetes Architecture ##
Kubernetes operates in a master-slave architecture, where the Master controls and manages the state of the cluster, while the Node(s) run the containers.

![Kubernetes Architecture](https://github.com/nawab312/Kubernetes/blob/main/Images/Kubernetes_Architecture.png)

### Master Components ###
The Master node is responsible for for managing the *overall cluster state*, *scheduling applications*, and ensuring that the *desired state of the system* is maintained. The key components of the master node include
- **API Server (kube-apiserver):** is the entry point for all API requests. It exposes Kubernetes APIs and is responsible for handling all the internal and external requests for the cluster. It validates and processes REST requests, updates etcd, and sends commands to the other components.
- **Controller Manager:** runs controllers that regulate the state of the cluster. These controllers ensure that the current state of the system matches the desired state. Examples of controllers are:
    - Replication Controller: Ensures that the desired number of pod replicas are running
    - Node Controller: Manages node status, including adding or removing nodes
    - Job Controller: Ensures that jobs are completed successfully
- **Scheduler (kube-scheduler):** is responsible for assigning Pods to Nodes based on resource availability, constraints, and policies. It monitors the cluster to determine the best nodes for the Pods. If you want to schedule it to a specific node, you can use the `nodeSelector` field.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx 
      image: nginx:1.14.2
  nodeSelector:
    disktype: ssd
```
- **etcd:** is a distributed key-value store used by Kubernetes to store all cluster data. It stores configuration and state data (e.g., which Pods are running, what the cluster's state is, etc.). If etcd fails, the entire cluster's state could be lost

### Node Components ###
A Node is a physical or virtual machine in the Kubernetes cluster that runs the applications (containers) and their workloads. Each Node contains several important components:
- **kubelet:** The kubelet ensures that containers are running in Pods on each node. It listens to the API server and makes sure the node’s containers are running as specified.
    - Pod Management: The Kubelet reads the PodSpecs (configuration for the Pod) and ensures that containers in the Pod are started, running, and healthy. If a container fails or crashes, the Kubelet will attempt to restart it, as per the configuration.
    - Health Checks: It continuously monitors the health of the containers and can perform liveness probes (to check if a container is running as expected) and readiness probes (to check if a container is ready to accept traffic). If a container fails a health check, the Kubelet will report the failure and take appropriate action like restarting the container
    - Node Communication: The Kubelet communicates with the Kubernetes API server to register the status of the node and report on the health of the Pods running on it.
- **kube-proxy:** runs on each node in the cluster and is responsible for managing network traffic to and from the Pods. It plays a critical role in ensuring that traffic is routed to the correct Pods within the cluster.
    - Service Abstraction: Kubernetes services (like ClusterIP, NodePort, LoadBalancer, etc.) abstract away the complexity of communicating with individual Pods by providing a stable IP and DNS name. The Kube-Proxy helps route traffic to the correct Pod(s) behind a Service
    - Network Routing: The Kube-Proxy handles the routing of network requests from external clients or other Pods to the correct destination Pods. It uses iptables or IPVS (depending on the implementation) to manage the network rules and direct traffic
    - Load Balancing: It also performs basic load balancing. When there are multiple Pods behind a Service, the Kube-Proxy will distribute traffic across these Pods in a round-robin fashion.
- **Container Runtime (Docker, containerd):** The container runtime is the software responsible for running the containers. Docker was the initial choice for Kubernetes, but it now supports other runtimes like containerd and CRI-O.

### Key Concepts in Kubernetes ###
- **Cluster:** A Kubernetes Cluster consists of a set of nodes (physical or virtual machines) that run containerized applications. Each cluster has at least one master node (control plane) and one or more worker nodes (where containers are actually run).
- **Node:** A Node is a worker machine within the Kubernetes cluster. It can be either a physical or virtual machine. Each node runs a kubelet, which is responsible for running containers on that node.
- **Pod:** A Pod is the smallest and simplest unit in Kubernetes. It encapsulates one or more containers that share storage and network resources. Containers in the same Pod are *tightly coupled* and run on the *same node*. A Pod may contain
    - Single Container Pod: A single container inside the Pod.
    - Multi-container Pod: Multiple containers inside the same Pod, which are closely related and share the same environment (network, storage).
- **Service:** Service is an abstraction that defines a logical set of Pods and a policy by which to access them. It provides a stable endpoint (usually a DNS name) to access a group of Pods, even though the Pods themselves might be dynamically created, destroyed, or replaced.
- **ReplicaSets:** A ReplicaSet ensures that a specified number of replicas (Pods) of a containerized application are running at all times. If a Pod crashes or is deleted, the ReplicaSet will create a new Pod to replace it, maintaining the desired number of Pod
- **Deployment** is a higher-level object that manages ReplicaSets and Pods. It provides declarative updates to applications, meaning you can define the desired state of your application, and Kubernetes will work to ensure that the system converges to that state. Key Features:
    - Rolling updates: Deployments provide an easy way to update an application without downtime. Kubernetes gradually replaces old Pods with new ones, ensuring that there’s no interruption in service.
    - Rollback: If something goes wrong with the new deployment, you can roll back to a previous version.
    - Scaling: You can easily scale your application by adjusting the number of replicas in the Deployment.





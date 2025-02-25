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


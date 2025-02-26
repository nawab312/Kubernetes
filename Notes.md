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
- **DaemonSet** in Kubernetes ensures that a copy of a specific Pod is running on every node (or a subset of nodes) in the cluster. This is useful for running system-level or utility applications that need to be present on each node, such as logging agents, monitoring agents, or networking tools.
    - One Pod per Node: DaemonSets guarantee that there will be exactly one Pod running on each node in the cluster. If new nodes are added, a Pod will be automatically scheduled to those nodes.
    - Pods in a DaemonSet are not scheduled based on normal Deployment behavior (replica scaling); they are scheduled based on the availability of nodes.
 
**Storage in Kubernetes** https://github.com/nawab312/Kubernetes/blob/main/Storage/Notes.md

**Services** https://github.com/nawab312/Kubernetes/blob/main/Services/Notes.md

### ConfigMaps and Secrets ###

**ConfigMaps**
- A Kubernetes object used to store non-sensitive configuration data in key-value pairs
- Use Cases: Application settings (e.g., environment variables). File-based configurations (e.g., .properties or .json files)
- ConfigMaps are mounted into Pods as: Environment variables, Command-line arguments, Volumes (files in a directory).

**Secrets**
- Kubernetes objects to securely store sensitive information such as passwords, tokens, or certificates. Data is encoded using Base64.
- Types of Secrets:
    - Opaque: Default type for arbitrary data.
    - TLS: Used to store TLS certificates and keys.
- Secrets can be accessed by Pods as: Environment variables, Mounted volumes.


### Application Lifecycle Management ###
Probe (Probe Means: To test Behaviour of System) Mechanisms:
- HTTP Probes: Sends an HTTP GET request to a specified endpoint.
- TCP Probes: Tries to establish a TCP connection.
- Command Probes: Executes a specified command inside the container.

- **Liveness Probe:** Determines if the application inside a Pod is still running. If the probe fails, Kubernetes restarts the container. Common scenarios: Application crashes or enters a deadlock state, Recovering from temporary issues. Liveness probe sends HTTP GET requests to the container's endpoint internally, using the kubelet, which runs on the worker node hosting the container. A 2xx or 3xx HTTP response indicates the container is healthy.
- **Readiness Probe:** Determines if the application inside a Pod is ready to serve traffic. If the probe fails, the Pod is removed from the Service's endpoints. Common scenarios: Application initialization takes time. External dependencies (e.g., database) are not ready yet.

- ProbeParameters:
    - initialDelaySeconds: Time to wait before starting the probe.
    - periodSeconds: Frequency of the probe.
    - failureThreshold: Number of failures before marking the container as unhealthy.
    - timeoutSeconds: Maximum duration to wait for a response.

```yaml
livenessProbe:
    httpGet:
        path: /
        port: 80
    initialDelaySeconds: 5
    periodSeconds: 10
readinessProbe:
    httpGet:
        path: /
        port: 80
    initialDelaySeconds: 5
    periosdSeconds: 5
```

### Kubernetes Scheduling ###

**Resource Requests and Limits**
- Resource Requests:
    - Specify the minimum amount of CPU and memory a Pod needs to run.
    - The scheduler ensures the node has at least the requested resources available
- Resource Limits:
    - Specify the maximum amount of CPU and memory a Pod can use.
    - Prevents a Pod from monopolizing resources.
https://github.com/nawab312/Kubernetes/blob/main/Pods/Pod-1.yaml

**Node Affinity & Anti-Affinity**
- Node Affinity: A way to specify which nodes a Pod prefers to run on based on node labels.
- Node Anti-Affinity: Opposite of affinity; ensures Pods are scheduled on separate nodes, useful for high availability.
https://github.com/nawab312/Kubernetes/blob/main/Pods/Pod-2.yaml

**Taints and Tolerations**
- A taint is a way to mark a node so that only certain Pods can be scheduled on it.
- `kubectl taint nodes node1 gpu=true:NoSchedule` Only Pods that tolerate the gpu=true taint can be scheduled on node1.
- A toleration is a way for a Pod to indicate that it can tolerate (or accept) a specific taint on a node.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myPod
spec:
  tolerations:
    - key: "gpu"
      operator: "equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: tensorflow
      image: tensorflow:latest
```

**Ingress** https://github.com/nawab312/Kubernetes/blob/main/Ingress/Notes.md

**Statefuleset** https://github.com/nawab312/Kubernetes/blob/main/StatefuleSet/Notes.md






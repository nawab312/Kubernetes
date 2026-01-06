**What is Kubernetes?**
- Kubernetes (often abbreviated as K8s) is an open-source platform for automating the deployment, scaling, and management of containerized applications. 
- It helps you manage applications that are composed of many containers across multiple machines, ensuring that the system is highly available, scalable, and resilient

**Why to use Kubernetes if we have Docker?**
- Docker is a powerful tool for creating and running containers, allowing developers to package applications with all their dependencies into lightweight, portable environments. However, Docker alone is not enough when managing multiple containers across different machines in a production environment
- Kubernetes is a container orchestration platform that automates the deployment, scaling, and management of containerized applications.
- Unlike Docker, which requires manual intervention to start, stop, and scale containers, Kubernetes provides built-in features like automatic scaling, load balancing, self-healing, and rolling updates. It ensures high availability by restarting failed containers and distributing traffic efficiently across multiple instances

### Kubernetes Architecture ###
The Master node is responsible for for managing the *overall cluster state*, *scheduling applications*, and ensuring that the *desired state of the system* is maintained. The key components of the master node include

**API Server (kube-apiserver)**
- Entry point for all API requests. It exposes Kubernetes APIs and is responsible for handling all the internal and external requests for the cluster. It validates and processes REST requests, updates etcd, and sends commands to the other components.
- You can interact with the API server directly using `kubectl`. The `-v=8` flag increases verbosity and shows API calls made to the API server.
```bash
kubectl get pods -v=8
```
- How kube-apiserver Handles Requests?
  - Let’s go through an example where a user creates a pod, and the API server processes the request.
  - User Creates a Pod `kubectl run nginx --image=nginx --restart=Never`
  - API Request Flow
    - `kubectl` sends an HTTP POST request to the API server. The request body contains JSON defining the Pod object.
      ```bash
      POST /api/v1/namespaces/default/pods
      ```
    - API Server Validates and Authenticates
      - The API server *validates* the request (ensuring correct syntax and required fields).
      - It *authenticates* the user using Kubernetes RBAC, service accounts, or certificates.
      - It *authorizes* the action using Role-Based Access Control (RBAC).
    - API Server Updates etcd
      - If validation passes, the API server writes the Pod definition into etcd, ensuring it's stored persistently
    - API Server Notifies Other Components
      - *Scheduler*: The API server notifies the scheduler that a new Pod needs a node assignment.
      - *Controller Manager*: The API server updates the ReplicaSet or Deployment controllers if needed.
      - *Kubelet*: Once a node is assigned, the API server sends an update to the kubelet on that node to pull the image and start the container.

**Controller Manager**
- *Replication Controller*: Ensures that the desired number of pod replicas are running
- *Node Controller*: Manages node status, including adding or removing nodes
- *Job Controller*: Ensures that jobs are completed successfully
- How Controllers Work?
  - Let’s go through an example where a *ReplicaSet controller* ensures that the desired number of pods are running.
  - Creating a Deployment: A user runs the following command to create a deployment with *3 replicas*:
    ```bash
    kubectl create deployment nginx --image=nginx --replicas=3
    ```
  - What Happens Internally?
    - kubectl Sends Request to API Server. The API Server validates and stores the Deployment definition in *etcd*.
      ```bash
      POST /apis/apps/v1/namespaces/default/deployments
      ```
    - Deployment Controller in Controller Manager Detects the Change. The *Deployment Controller* notices that the *desired state* is 3 replicas, but currently, there are 0 pods running.
    - The Deployment Controller creates a *ReplicaSet* to manage the pods.
    - *ReplicaSet Controller* Ensures Pod Creation. The ReplicaSet Controller compares the desired state (`replicas=3`) with the current state (`0 pods running`). It creates 3 new pods by sending a request to the API Server.
    - Scheduler Assigns Pods to Nodes. The *API Server* notifies the corresponding kubelet to start the pods.
- What happens if a Kubernetes Node running multiple critical Pods suddenly crashes?
  - Node Failure Detection: The Kubernetes node controller, part of the `kube-controller-manager`, continuously monitors the health of nodes. It does this through heartbeat signals received from the kubelet running on each node.
  - Marking Node as Unreachable: If a node stops sending heartbeats for a specific period (default is 40 seconds), Kubernetes marks it as NotReady.
  - By default, after 5 minutes (300 seconds), the node controller evicts all Pods running on the failed node.
  - The scheduler then attempts to reschedule these Pods onto other healthy nodes in the cluster.
   
**Scheduler (kube-scheduler)**
- Responsible for assigning Pods to Nodes based on resource availability, constraints, and policies. It monitors the cluster to determine the best nodes for the Pods. If you want to schedule it to a specific node, you can use the `nodeSelector` field.
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
- Check label of Nodes: `kubectl get nodes --show-labels`
- Label a Node: `kubectl label node <node-name> disktype=ssd`

**etcd**
- Is a distributed key-value store used by Kubernetes to store all cluster data. 
- It stores configuration and state data (e.g., which Pods are running, what the cluster's state is, etc.). If etcd fails, the entire cluster's state could be lost
```bash
etcdctl --endpoints=https://<etcd-endpoint>:2379 --cert-file=/path/to/cert --key-file=/path/to/key --ca-file=/path/to/ca-cert get "" --prefix --keys-only
```
```bash
# Output
/registry/pods/default/pod-1
/registry/pods/default/pod-2
/registry/pods/default/pod-3
/registry/services/default/my-service
/registry/nodes/node-1
/registry/nodes/node-2
/registry/namespaces/default
/registry/deployments/default/deployment-1
/registry/secrets/default/my-secret
/registry/configmaps/default/my-configmap
/registry/services/endpoints/default/my-service
/registry/replicasets/default/rs-1
/registry/replicasets/default/rs-2
/registry/replicasets/default/rs-3
/registry/statefulsets/default/statefulset-1
/registry/statefulsets/default/statefulset-2
```

```bash
etcdctl --endpoints=https://<etcd-endpoint>:2379 --cert-file=/path/to/cert --key-file=/path/to/key --ca-file=/path/to/ca-cert get "" --prefix
```
```bash
# Output
/registry/pods/default/pod-1
{"kind":"Pod","apiVersion":"v1","metadata":{"name":"pod-1","namespace":"default","uid":"1234567890","labels":{"app":"myapp"}},"spec":{"containers":[{"name":"nginx","image":"nginx:latest"}]}}
/registry/pods/default/pod-2
{"kind":"Pod","apiVersion":"v1","metadata":{"name":"pod-2","namespace":"default","uid":"1234567891","labels":{"app":"myapp"}},"spec":{"containers":[{"name":"nginx","image":"nginx:latest"}]}}
```


A Node is a physical or virtual machine in the Kubernetes cluster that runs the applications (containers) and their workloads. Each Node contains several important components:

**kubelet**
- The kubelet ensures that containers are running in Pods on each node. It listens to the API server and makes sure the node’s containers are running as specified.
- Pod Management: The Kubelet reads the PodSpecs (configuration for the Pod) and ensures that containers in the Pod are started, running, and healthy. If a container fails or crashes, the Kubelet will attempt to restart it, as per the configuration
- Health Checks: It continuously monitors the health of the containers and can perform liveness probes (to check if a container is running as expected) and readiness probes (to check if a container is ready to accept traffic). If a container fails a health check, the Kubelet will report the failure and take appropriate action like restarting the container
- Node Communication: The Kubelet communicates with the Kubernetes API server to register the status of the node and report on the health of the Pods running on it.

**kube-proxy**
- Runs on each node in the cluster and is responsible for managing network traffic to and from the Pods. It plays a critical role in ensuring that traffic is routed to the correct Pods within the cluster.
- Service Abstraction: Kubernetes services (like ClusterIP, NodePort, LoadBalancer, etc.) abstract away the complexity of communicating with individual Pods by providing a stable IP and DNS name. The Kube-Proxy helps route traffic to the correct Pod(s) behind a Service
- Network Routing: The Kube-Proxy handles the routing of network requests from external clients or other Pods to the correct destination Pods. It uses iptables or IPVS (depending on the implementation) to manage the network rules and direct traffic
- Load Balancing: It also performs basic load balancing. When there are multiple Pods behind a Service, the Kube-Proxy will distribute traffic across these Pods in a round-robin fashion.

**Container Runtime (Docker, containerd)** 
- The container runtime is the software responsible for running the containers. Docker was the initial choice for Kubernetes, but it now supports other runtimes like containerd and CRI-O.

      

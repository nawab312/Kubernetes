### What is the Kubernetes Metrics Server? ###
The **Metrics Server** is a cluster-wide aggregator of resource usage data, such as CPU and memory, in a Kubernetes cluster. It collects real-time metrics from the **kubelet** on each node and makes them available via the **Metrics API**.

### How Does It Work? ###
- **Kubelet exposes metrics:** Each node’s kubelet collects CPU and memory stats using the *cAdvisor (Container Advisor)* and exposes them through the `metrics-server` API.
- **Metrics Server scrapes the data:** The Metrics Server queries each node’s kubelet on the `/metrics/resource` endpoint.
- **Aggregation and Storage:** It aggregates this data and exposes it through the *Metrics API* (`/apis/metrics.k8s.io`).
- **HPA and Other Components Use the Metrics:** The Kubernetes Horizontal Pod Autoscaler (HPA) and kubectl top command use this API to make scaling decisions.

### Architecture of Metrics Server ###
**Data Collection Layer (Kubelet & cAdvisor)**
- Each *Kubernetes node* runs a *kubelet*, which is responsible for managing containers.
- The kubelet uses *cAdvisor (Container Advisor)* to collect real-time *CPU & memory usage* of all pods and nodes.
- The kubelet exposes this data via the `/metrics/resource` endpoint on each node.

**Aggregation & Processing Layer (Metrics Server)**
- The *Metrics Server* is a *Deployment* running in the `kube-system` namespace.
- It queries *each node's kubelet* for CPU and memory usage using the `kubelet summary API`.
- It then *aggregates* the collected data from multiple nodes.
- The aggregated data is *temporarily stored* in memory (not a database).

**API Exposure Layer (Metrics API)**
- The Metrics Server exposes the aggregated metrics via the *Kubernetes Metrics API* (`/apis/metrics.k8s.io`).
- The API follows the *Custom Resource API pattern*.
- Kubernetes components like Horizontal Pod Autoscaler (HPA) and kubectl top use this API to fetch real-time metrics.

### How Metrics Server Fetches Metrics from the Kubelet’s /metrics/resource Endpoint? ###
Before collecting metrics, the Metrics Server must *discover the kubelet endpoints.*

**Querying the API Server for Node List**
- The Metrics Server sends an authenticated request to the Kubernetes API Server to get the list of nodes. How Metrics Server Authenticates Requests to API Server
   - Metrics Server Uses a *ServiceAccount* Token
     - When the Metrics Server Pod starts, it runs under the `metrics-server` ServiceAccount (in the `kube-system` namespace).
     - Kubernetes automatically *mounts a token file* inside the pod at:
       ```bash
       /var/run/secrets/kubernetes.io/serviceaccount/token
       ```
     - This token is a JWT (JSON Web Token) signed by Kubernetes.
     - Example: The Metrics Server Sends an API Request Like This
       ```bash
       curl -k -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/apis/metrics.k8s.io/v1beta1/nodes
       ```
     - The Bearer Token is used to authenticate with the API Server.
  - API Server Verifies the Token
    - The Kubernetes API Server verifies the JWT *against the cluster’s built-in OIDC token issuer*.
    - If valid, the API Server checks the RBAC (Role-Based Access Control) permissions for the `metrics-server` ServiceAccount.
  - RBAC Controls What Metrics Server Can Access
    - The `metrics-server` ServiceAccount must have the right *ClusterRole and ClusterRoleBinding* to read metrics.
    - Metrics Server requires the following RBAC permissions:
      ```yaml
      apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRole
      metadata:
        name: system:metrics-server
      rules:
      - apiGroups: [""]
        resources: ["nodes", "nodes/stats", "pods", "namespaces"]
        verbs: ["get", "list", "watch"]
      ```
    - Bind the ClusterRole to a *metrics-server* ServiceAccount
      ```yaml
      apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRoleBinding
      metadata:
        name: metrics-server-binding
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: system:metrics-server
      subjects:
      - kind: ServiceAccount
        name: metrics-server
        namespace: kube-system
      ```
  - Metrics Server Uses TLS for Secure Communication
    - The Metrics Server uses *TLS certificates* to ensure *secure communication* with the API Server.
    - It typically runs with the `--kubelet-insecure-tls` flag to *skip* certificate validation for the Kubelet.

**How Metrics Server Fetches Metrics from Kubelet’s /metrics/resource?**
- Metrics Server Constructs an HTTPS Request to Kubelet
  Once Metrics Server gets the node IP addresses, it sends an HTTPS request to the Kubelet running on each node.
  ```bash
  curl -k --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://192.168.49.2:10250/metrics/resource
  ```

**Example Kubelet Response**

```bash
# HELP node_cpu_usage_seconds_total CPU usage in seconds
node_cpu_usage_seconds_total{container="nginx",pod="nginx-pod",namespace="default"} 12.5

# HELP node_memory_working_set_bytes Memory usage in bytes
node_memory_working_set_bytes{container="nginx",pod="nginx-pod",namespace="default"} 536870912
```

### How Metrics Server Exposes Aggregated Metrics via the Metrics API (metrics.k8s.io)? ###
After collecting raw metrics from all nodes' kubelets, **Metrics Server** makes them available to Kubernetes components (like HPA and `kubectl top`) through the **Kubernetes Metrics API** (`/apis/metrics.k8s.io`).

**How Metrics Server Exposes Metrics via Metrics API?**
- API Registration as an Aggregated API
  - The *Metrics API (`metrics.k8s.io`) is not built into the Kubernetes API Server*.
  - Instead, it is an *Aggregated API* that *extends* the Kubernetes API.
  - The API Aggregation Layer in the Kubernetes API Server forwards requests to the Metrics Server.
- Defining the API in APIService
  - To register itself as the provider of `metrics.k8s.io`, the Metrics Server *creates an APIService object*:
   ```yaml
   apiVersion: apiregistration.k8s.io/v1
   kind: APIService
   metadata:
     name: v1beta1.metrics.k8s.io
   spec:
     group: metrics.k8s.io
     version: v1beta1
     service:
       name: metrics-server
       namespace: kube-system
     groupPriorityMinimum: 100
     versionPriority: 100
   ```
   - This tells Kubernetes: *"For requests to `metrics.k8s.io`, forward them to the Metrics Server."*
   - The Metrics Server runs as a Service (`metrics-server.kube-system`).
   - The *Kubernetes API Server acts as a proxy*, forwarding requests to the Metrics Server.
- What Endpoints Does Metrics Server Provide?
  - Node Metrics
  ```bash
  GET /apis/metrics.k8s.io/v1beta1/nodes
  ```
  ```json
  #Output
   {
     "kind": "NodeMetricsList",
     "apiVersion": "metrics.k8s.io/v1beta1",
     "items": [
       {
         "metadata": { "name": "node-1" },
         "timestamp": "2025-03-18T10:00:00Z",
         "usage": {
           "cpu": "500m",
           "memory": "1024Mi"
         }
       }
     ]
   }
  ```
  - Pod Metrics
  ```bash
  GET /apis/metrics.k8s.io/v1beta1/namespaces/default/pods
  ```
  ```json
   {
     "kind": "PodMetricsList",
     "apiVersion": "metrics.k8s.io/v1beta1",
     "items": [
       {
         "metadata": { "name": "nginx-pod", "namespace": "default" },
         "timestamp": "2025-03-18T10:00:00Z",
         "containers": [
           {
             "name": "nginx",
             "usage": {
               "cpu": "200m",
               "memory": "256Mi"
             }
           }
         ]
       }
     ]
   }
  ```

**Who Uses This API?**

Now that Metrics Server exposes the API, various components in Kubernetes can consume it.
- Horizontal Pod Autoscaler (HPA)
  - HPA automatically scales pods based on CPU/memory usage.
  - It queries the *Metrics API* every *30 seconds*:
    ```bash
    GET /apis/metrics.k8s.io/v1beta1/namespaces/default/pods/nginx-pod
    ```
- kubectl top Command
  ```bash
  kubectl top nodes
  ```
  - The command queries:
    ```bash
    GET /apis/metrics.k8s.io/v1beta1/nodes
    ```
    ```bash
    #Output
    NAME     CPU(cores)   MEMORY(bytes)
    sid-cluster   500m         1024Mi
    ```



  


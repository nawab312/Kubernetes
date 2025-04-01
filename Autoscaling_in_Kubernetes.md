- **HPA** Automatically scales the number of Pods in a Deployment, ReplicaSet, or StatefulSet based on observed metrics like CPU/memory usage or custom application metrics.
- **VPA** Automatically adjusts *resource requests and limits* for containers in a Pod to optimize resource usage. How It Works:
  - Monitors actual resource usage over time
  - Recommends or directly applies changes to resource requests and limits.
  - Can operate in three modes: 
    - *Off*: Only Provide Recommendations
    - *Auto*: Automatically update resource requests and limits
    - *Initial*: Set the requests and limits at pod creation
   
**How Does HPA Work?**

HPA continuously monitors the resource utilization of pods and adjusts the number of replicas accordingly. It uses the *Kubernetes Metrics API* to fetch resource utilization data. HPA Workflow
- Monitor Metrics: HPA collects metrics (like CPU or memory usage) using the Metrics Server or custom metric providers (like Prometheus).
- Compare Against Target: It compares the current resource utilization with the target threshold set in the HPA configuration.
- HPA Relies on Resource Requests
  - HPA scales pods based on CPU and memory utilization. However, for it to calculate utilization properly, each pod must have resource requests defined.
  - Kubernetes calculates utilization using the formula:

    ![image](https://github.com/user-attachments/assets/ceaefd38-e447-4e38-8241-78d5d723a394)

  - If requests are not defined, Kubernetes does not have a baseline to compare against, making autoscaling ineffective.
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 2
      template:
        spec:
          containers:
          - name: my-app
            image: my-app:latest
            resources:
              requests:
                cpu: "500m"
                memory: "256Mi"
              limits:
                cpu: "1000m"
                memory: "512Mi"
      ```
    
- Scale Up or Down:
  - Scale Up: If the resource utilization exceeds the threshold, HPA increases the number of pod replicas.
  - Scale Down: If the resource utilization is below the threshold, HPA decreases the number of pod replicas.
- Apply Changes: Kubernetes adjusts the pod count in the deployment or stateful set.

![image](https://github.com/user-attachments/assets/d54f1832-3ee5-4f9f-bc99-805f773341c5)


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

**Basic HPA Example (CPU-based)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

**Scaling Based on Memory**
```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 70
```

**Scale-up and Scale-down behavior of the Horizontal Pod Autoscaler (HPA)**
- The scaleUp behavior defines how quickly the HPA will increase the number of pods when the observed metric exceeds the desired target.
- The scaleDown behavior defines how quickly the HPA will decrease the number of pods when the observed metric falls below the desired target.
- `stabilizationWindowSeconds`: The time period during which HPA will prevent further scaling actions after a scale-up/down event, allowing the system to stabilize before making another scaling decision.
- `selectPolicy`: Specifies the policy for scaling (e.g., scaling based on the most recent metric or the average).
- `policies`: Defines the scaling policy, including the amount of pods to scale up/down at once and how fast it can increase the number of pods.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: AverageValue
          averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 300  # Prevents further scaling up within 5 minutes after a scale-up
      selectPolicy: MaxChangePolicy  # Select the most significant scaling change
      policies:
        - type: Percent
          value: 50  # Scale up by 50% of the current replica count
          periodSeconds: 60  # Apply this policy every minute
    scaleDown:
      stabilizationWindowSeconds: 300  # Prevents scaling down too quickly after scaling up
      selectPolicy: MaxChangePolicy  # Select the most significant scaling change
      policies:
        - type: Percent
          value: 50  # Scale down by 50% of the current replica count
          periodSeconds: 60  # Apply this policy every minute
```




**How Does HPA Work with Prometheus?**
- Deploy Prometheus & Prometheus Adapter. Kubernetes does not understand Prometheus metrics by default. **Prometheus Adapter** translates these metrics into *Kubernetes Custom Metrics API*.
  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
  helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring
  ```

- Verify Prometheus Has CPU & Memory Metrics. Prometheus collects CPU & memory metrics using kube-state-metrics and node-exporter. To check if the metrics exist in Prometheus:
  ```bash
  kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090
  ```
- Go to `http://localhost:9090` and run:
  ```bash
  rate(container_cpu_usage_seconds_total[5m])
  container_memory_usage_bytes
  ```
  - These metrics will be used by Prometheus Adapter.

- Configure Prometheus Adapter to Serve CPU & Memory Metrics
  ```bash
  kubectl edit configmap prometheus-adapter -n monitoring`
  ```
  Inside the ConfigMap, find the rules section and modify it like this:
  ```yaml
  rules:
  - seriesQuery: 'rate(container_cpu_usage_seconds_total{namespace!="", pod!=""}[2m])'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "container_cpu_usage_seconds_total"
      as: "cpu_usage"
    metricsQuery: 'rate(container_cpu_usage_seconds_total{pod!="",namespace!=""}[2m]) * 1000'

  - seriesQuery: 'container_memory_usage_bytes{namespace!="", pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "container_memory_usage_bytes"
      as: "memory_usage"
    metricsQuery: 'container_memory_usage_bytes{pod!="",namespace!=""}'
  ```
  After editing the ConfigMap, restart the Prometheus Adapter:
  ```bash
  kubeclt delete pod -l app=prometheus-adapter -n monitoring
  ```

- Create HPA Using Prometheus CPU & Memory Metrics
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: my-app-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: my-app
    minReplicas: 2
    maxReplicas: 20
    metrics:
    - type: Pods
      pods:
        metric:
          name: cpu_usage  # Custom Prometheus CPU metric
        target:
          type: AverageValue
          averageValue: 500m  # Scale if CPU > 500m (0.5 core)

    - type: Pods
      pods:
        metric:
          name: memory_usage  # Custom Prometheus Memory metric
        target:
          type: AverageValue
          averageValue: 500Mi  # Scale if memory > 500Mi
    ```

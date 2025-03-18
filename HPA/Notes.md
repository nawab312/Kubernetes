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
- Scale Up or Down:
  - Scale Up: If the resource utilization exceeds the threshold, HPA increases the number of pod replicas.
  - Scale Down: If the resource utilization is below the threshold, HPA decreases the number of pod replicas.
- Apply Changes: Kubernetes adjusts the pod count in the deployment or stateful set.

**What is Metrics Server?**

Metrics Server is a *lightweight in-cluster* aggregator that collects CPU and memory usage metrics from the *Kubelet* on each node and exposes them through the Kubernetes Metrics API.

### How Does Metrics Server Collect and Expose Data? ###
Metrics Server works by communicating with the **Kubelet** on each node, collecting CPU & memory usage data, and exposing it through the **Kubernetes Metrics API**.

**Collecting Metrics from Nodes**
- Who Provides the Data?
  - Each Kubernetes *node* runs a *Kubelet*, which continuously monitors the resource usage of all pods running on that node.
-  How Does Metrics Server Get This Data?
  - Metrics Server sends HTTPS requests to Kubelet's `/stats/summary` API endpoint to fetch CPU & memory usage for pods and nodes. 

**Installing Metrics Server**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get apiservices | grep metrics
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

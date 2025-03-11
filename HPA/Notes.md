- **HPA** Automatically scales the number of Pods in a Deployment, ReplicaSet, or StatefulSet based on observed metrics like CPU/memory usage or custom application metrics.
- **VPA** Automatically adjusts *resource requests and limits* for containers in a Pod to optimize resource usage. How It Works:
  - Monitors actual resource usage over time
  - Recommends or directly applies changes to resource requests and limits.
  - Can operate in three modes: 
    - *Off*: Only Provide Recommendations
    - *Auto*: Automatically update resource requests and limits
    - *Initial*: Set the requests and limits at pod creation

**How Does HPA Work with Prometheus?**
- Deploy Prometheus & Prometheus Adapter. Kubernetes does not understand Prometheus metrics by default. **Prometheus Adapter** translates these metrics into Kubernetes Custom Metrics API.
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

### Kubernetes Scheduling ###
**Resource Requests and Limits**
- Resource Requests:
    - Specify the minimum amount of CPU and memory a Pod needs to run.
    - The scheduler ensures the node has at least the requested resources available
- Resource Limits:
    - Specify the maximum amount of CPU and memory a Pod can use.
    - Prevents a Pod from monopolizing resources.

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
![image](https://github.com/user-attachments/assets/6466be37-1dea-4296-aac6-2bcab920e105)

Probe (Probe Means: To test Behaviour of System) Mechanisms:
- HTTP Probes: Sends an HTTP GET request to a specified endpoint.
- TCP Probes: Tries to establish a TCP connection.
- Command Probes: Executes a specified command inside the container.

- **Liveness Probe:** Determines if the application inside a Pod is still running. If the probe fails, Kubernetes restarts the container. Common scenarios: Application crashes or enters a deadlock state, Recovering from temporary issues. Liveness probe sends HTTP GET requests to the container's endpoint internally, using the *kubelet*, which runs on the worker node hosting the container. A 2xx or 3xx HTTP response indicates the container is healthy.
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
- What will happen if a Pod has a Readiness Probe that always fails, but its liveness probe always passes?
    - Pod Will Never Receive Traffic: If Readiness Probe always fails, the Pod is never added to the *Endpoints* list of its Service.
    - Pod Will Keep Running: Since Liveness Probe always passes, Kubernetes assumes the container is healthy and does not restart it.
    - Rolling Update: If this Pod is part of a Deployment, the rolling update may get stuck because Kubernetes waits for the new Pod to become "Ready" before terminating the old one.
 
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
- Metrics Server is a *lightweight in-cluster* aggregator that collects CPU and memory usage metrics from the *Kubelet* on each node and exposes them through the Kubernetes Metrics API.
- https://github.com/nawab312/Kubernetes/blob/main/HPA/Metrics_Server.md

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
 
### Resource Managerment ###
**What happens if there are not enough resources to schedule a pod?**
- *Pod is Unschedulable* The Kubernetes scheduler will attempt to find a node with enough resources (CPU, memory, etc.) to accommodate the Pod. If no suitable node is found, the Pod remains in the "Pending" state.
- *Retry Mechanism* Kubernetes continuously retries scheduling the Pod, waiting for resources to free up or for new nodes to be added to the cluster.
- *Cluster Autoscaler* If Cluster Autoscaler is configured, Kubernetes can automatically provision new nodes to handle the resource demand. Once new nodes are available, the Pod is scheduled.

In a Kubernetes cluster, you delete a Kibana pod, but it keeps getting recreated with the status Init:0/1. How would you permanently delete the Kibana Pod, and what underlying Kubernetes concepts explain this behavior
- The pod is likely managed by a higher-level controller (e.g., Deployment, StatefulSet, DaemonSet, or Helm Release).
- `kubectl get all | grep kibana` This will reveal whether Kibana is controlled by a Job, Deployment, StatefulSet, or Helm Release.
```bash
kubectl get all | grep kibana
pod/post-delete-kibana-kibana-4cbkf   0/2     Init:0/1   0          5m31s
job.batch/post-delete-kibana-kibana   Running   0/1           10m        10m
```
- Delete the Underlying Controller, Here it is the Job
```bash
kubectl delete job post-delete-kibana-kibana
```


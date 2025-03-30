### Kubernetes Scheduling ###
Kubernetes scheduling is the process of assigning Pods to Nodes within a cluster based on various constraints and policies. The *kube-scheduler* is the default Kubernetes scheduler responsible for making these decisions.

**Key Components of Kubernetes Scheduling**

*kube-scheduler*
- The default scheduler in Kubernetes.
- Watches for unscheduled Pods and assigns them to appropriate Nodes based on constraints and policies.

*Scheduler Workflow*
- Filtering (Predicates): Determines eligible nodes based on constraints (e.g., resource availability, node selectors, taints, tolerations).
- Scoring (Prioritization): Ranks eligible nodes based on scoring functions (e.g., least resource usage, affinity rules).
- Binding: Assigns the Pod to the best-ranked node.

**Scheduling Constraints**

*Node Selectors (nodeSelector)*
- Assigns a Pod to a Node with a specific label.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  nodeSelector:
    disktype: ssd
```

*Node Affinity*
- More flexible than nodeSelector, supporting `required (hard)` rules and `preferred (soft)` rules.
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values:
          - us-east-1a
```

*Taints and Tolerations*
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

*Pod Affinity and Anti-Affinity*
- Affinity: Prefer placing Pods on the same node/zone.
- Anti-Affinity: Avoid placing Pods on the same node/zone.
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      labelSelector:
        matchLabels:
          app: frontend
      topologyKey: "kubernetes.io/hostname"
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      weight: 1
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: backend
        topologyKey: "kubernetes.io/zone"
```

**Custom Scheduling**

*Assigning Pods Manually*  
- Using nodeName directly in the Pod spec:
```yaml
spec:
  nodeName: worker-node-1
```

**Resource Requests and Limits**
- Resource Requests:
    - Specify the minimum amount of CPU and memory a Pod needs to run.
    - The scheduler ensures the node has at least the requested resources available
- Resource Limits:
    - Specify the maximum amount of CPU and memory a Pod can use.
    - Prevents a Pod from monopolizing resources.


https://github.com/nawab312/Kubernetes/blob/main/Pods/Pod-2.yaml


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
 
### Resource Managerment ###
**What happens if there are not enough resources to schedule a pod?**
- *Pod is Unschedulable* The Kubernetes scheduler will attempt to find a node with enough resources (CPU, memory, etc.) to accommodate the Pod. If no suitable node is found, the Pod remains in the "Pending" state.
- *Retry Mechanism* Kubernetes continuously retries scheduling the Pod, waiting for resources to free up or for new nodes to be added to the cluster.
- *Cluster Autoscaler* If Cluster Autoscaler is configured, Kubernetes can automatically provision new nodes to handle the resource demand. Once new nodes are available, the Pod is scheduled.

**When a node experiences resource pressure (CPU, memory, or disk pressure), Kubernetes evicts lower-priority Pods first to free up resources.**

To protect critical application Pods from eviction, you should:
- Use Priority Classes (`priorityClassName`)
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-app-priority
value: 100000  # Higher value = Higher priority
globalDefault: false
description: "Priority for critical application pods"
```
- Assign this priority to critical Pods:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: critical-app-priority
  containers:
    - name: my-container
      image: my-app
```

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


**ImagePullBackOff / ErrImagePull**
```bash
kubectl get pods
NAME         READY   STATUS             RESTARTS   AGE
my-app-xyz   0/1     ImagePullBackOff   0          1m
```
- ErrImagePull: This error happens immediately when Kubernetes fails to pull an image from a container registry.
- ImagePullBackOff: This happens after multiple ErrImagePull failures. Kubernetes starts delaying retries exponentially (BackOff).

Reason
- Kubernetes cannot pull the container image from the registry.
- Possible causes:
  - Wrong image name or tag.
  - Private registry authentication failure.
  - Image does not exist.

*BackOff* means that the system is gradually increasing the delay before retrying a failed operation. In the case of ImagePullBackOff or ErrImagePull, it specifically refers to progressively longer wait times before attempting to pull the container image again. How BackOff Works in ImagePullBackOff
- First attempt → Kubernetes tries to pull the image. If it fails, it retries immediately.
- Subsequent failures → Kubernetes increases the delay before the next retry (e.g., 10s, 20s, 40s, etc.).
- Exponential BackOff → The wait time grows exponentially to prevent unnecessary retries from overloading the system.

---
 
### CrashLoopBackOff ###
```bash
kubectl get pods
NAME         READY   STATUS             RESTARTS   AGE
my-app-xyz   0/1     CrashLoopBackOff   5          2m
```

**Reason**
- The container inside the pod keeps crashing and restarting.
- Possible causes:
  - Application code issue (crashing on startup).
  - Missing dependencies (e.g., DB connection not available).
  - Insufficient resources (out of memory or CPU).
  - Incorrect startup command.

*Check Container Exit Code:*
```bash
kubectl get pod my-app-xyz -o jsonpath="{.status.containerStatuses[*].state.waiting.reason}"
```
  - Exit Code 1 → Application crashed due to a code error.
  - Exit Code 137 → Pod killed due to Out of Memory (OOM).
  - Exit Code 126/127 → Wrong command in command or entrypoint.

### OOMKilled (Out of Memory) ###
```bash
kubectl describe pod my-app-xyz
State:       Terminated
Reason:      OOMKilled
Exit Code:   137
```

**Reason**
- The container exceeded the allocated memory and was killed.
- Possible causes:
  - Memory leak in the application.
  - Resource limits too low.
 
- Check Logs Before Crash:
  ```bash
  kubectl logs my-app-xyz --previous
  ```
- Increase Memory Limits
- Optimize Application Memory Usage:
  - Use JVM heap limits for Java apps:
    ```yaml
    command: ["java", "-Xmx512m", "-jar", "app.jar"]
    ```

### Pending ###
```bash
kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
my-app-xyz   0/1     Pending   0          3m
```

**Reason**
- The pod is not scheduled to any node.
- Possible causes:
  - No available nodes (resource constraints)
  - Pod nodeSelector or affinity rules mismatch.
 
- Check Node Status: `kubectl get nodes` If nodes are `NotReady`, check node logs.
- Check Taints & Tolerations: If the node is tainted, the pod needs a toleration:

### Evicted ###
```bash
kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
my-app-xyz   0/1     Evicted   0          5m
```

**Reason**
- Kubernetes killed the pod due to resource pressure.
- Happens when a node runs out of memory or disk space.

---

- When a Pod is in the "ContainerCreating" state for an extended period, it usually means Kubernetes is unable to pull the container image.
- Running `kubectl describe pod <pod-name>` and seeing the error "Failed to pull image" strongly suggests that the image name, tag, or repository path is incorrect.

---

**You have a Kubernetes cluster with a Deployment running a mission-critical application. However, the Pods are frequently restarting. There are no errors in the application logs. How would you debug this?**   

Debugging Frequent Pod Restarts in a Kubernetes Cluster
- Resource Limits (OOMKills)
- Readiness/Liveness Probe Failures
- Node-Level Issues
- Container-Level Issues
- Networking or Persistent Volume Problems

Step 1: Inspect Pod Events and Status
```bash
kubectl describe pod <pod-name> -n <namespace>
```
This provides insights into: OOMKills, Probe failures, Node-related issues

Step 2: Check for Out-of-Memory (OOMKilled) Issues
- If the pod was OOMKilled, confirm using:
```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].state.terminated.reason}'
```
Possible outputs: 
- OOMKilled → Memory limit exceeded.
- Completed → The container exited successfully.
- Error → The container crashed.

Further Investigation:
- Check previous container logs for memory spikes:
  ```bash
  kubectl logs <pod-name> --previous
  ```
- Check node and pod memory usage:
  ```bash
  kubectl top node
  kubectl top pod -n <namespace>
  ```
- Adjust Resource Limits (if needed) in the Deployment YAML:
  ```yaml
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  ```

Step 3: Debug Liveness/Readiness Probe Failures
- If the pod is not OOMKilled but is still restarting, check liveness probe failures:
```bash
kubectl describe pod <pod-name> -n <namespace>
```
- If probe failures are found, inspect the probe configuration:
  ```yaml
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
  ```

Step 4: Investigate Node-Level Issues
- If the pod is not OOMKilled and probes are fine, check node health:
  ```bash
  kubectl get nodes -o wide
  kubectl describe node <node-name>
  ```
- Additional Node Debugging
  - Check system logs for kubelet failures:
    ```bash
    journalctl -u kubelet -f
    ```
  - Look for disk pressure, network issues, or crashes.
- Test if the issue is node-specific
  - Drain the node and move the pod elsewhere:
    ```bash
    kubectl cordon <node-name>
    kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
    ```

Step 5: Check Container Runtime Issues
- If node-level checks do not show issues, investigate container failures:
  - Check container runtime logs (Docker, Containerd, CRI-O):
    ```bash
    docker ps -a | grep <container-id>
    docker logs <container-id>
    ```'

Step 6: Debug Networking Issues
- If the pod relies on external services, test DNS resolution:
  ```bash
  kubectl exec -it <pod-name> -- nslookup <service-name>
  ```
- Check CNI plugin health:
  ```bash
  kubectl get pods -n kube-system | grep cni
  ```

Step 7: Check Persistent Volume Issues (If Applicable)
- If the pod mounts a volume, verify if it's failing:
  ```bash
  kubectl describe pod <pod-name> | grep Mount
  ```
- If volume mounting is failing, check:
  - PersistentVolumeClaim (PVC) status
  - Storage class compatibility
  - Storage class compatibility

---

**Question** In a Kubernetes cluster, a Pod remains in "Terminating" state for an extended period even after issuing kubectl delete pod <pod-name>. What is the most likely reason?

**Answer** The Pod has a *finalizer* preventing deletion until cleanup tasks are completed.
- A Finalizer is a Kubernetes metadata field that prevents an object (such as a Pod, PVC, or CRD) from being deleted until certain pre-deletion cleanup tasks are completed.
- Think of it as a "pre-delete hook" that ensures dependent resources are properly cleaned up before Kubernetes removes the object.
- How Do Finalizers Work?
  - When you delete an object, Kubernetes does not remove it immediately if it has a finalizer.
  - The object enters a "Terminating" state.
  - The finalizer controller runs cleanup tasks (e.g., detaching volumes, deregistering from external services, or notifying cloud providers).
  - Once cleanup is complete, Kubernetes removes the finalizer and then deletes the object.

---

**What are the possible causes and troubleshooting steps for a slow-responding service in Kubernetes?**

If a service in Kubernetes is taking a long time to respond, it could be due to issues at different levels of the system. Below are the key areas to investigate along with their underlying concepts.

*Pod-Level Issues*
- High Resource Utilization: If a pod is running out of CPU or memory, the application might slow down.
  ```bash
  kubectl top pods -n <namespace>
  ```
- Readiness Probe Failures: If the pod is unready, Kubernetes removes it from service endpoints.
  ```bash
  kubectl describe pod <pod-name> -n <namespace>
  ```
- Application-Level Bottlenecks: Inefficient code, slow database queries, or heavy computation may cause latency.

*Service-Level Issues*
- DNS Resolution Delays: CoreDNS issues can delay service discovery.
  ```bash
  kubectl logs -n kube-system -l k8s-app=kube-dns
  ```
- Load Balancing Inefficiencies: Some pods might be slower, causing uneven request distribution.
  ```bash
  kubectl get endpoints <service-name> -n <namespace>
  ```

*Network Issues*
- Network Latency: Slow communication between pods due to congestion or misconfigured network policies.
  ```bash
  kubectl get networkpolicies -n <namespace>
  ```
- Ingress Controller Overload: If using an Ingress, an overloaded controller may slow requests.
  ```bash
  kubectl logs -n <namespace> -l app=ingress-nginx
  ```

*Node-Level Issues*
- High CPU/Memory Usage: If the node hosting the pod is overloaded, response times increase.
  ```bash
  kubectl top nodes
  ```
- Disk I/O Bottlenecks: If an application is disk-intensive, slow storage can cause delays.

*External Dependencies*
- Database Slowness: Queries taking too long to execute will impact response times.
- Third-Party API Latency: External API dependencies may introduce delays if they are slow.

---

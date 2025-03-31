### ImagePullBackOff / ErrImagePull ###
```bash
kubectl get pods
NAME         READY   STATUS             RESTARTS   AGE
my-app-xyz   0/1     ImagePullBackOff   0          1m
```
- ErrImagePull: This error happens immediately when Kubernetes fails to pull an image from a container registry.
- ImagePullBackOff: This happens after multiple ErrImagePull failures. Kubernetes starts delaying retries exponentially (BackOff).

**Reason**
- Kubernetes cannot pull the container image from the registry.
- Possible causes:
  - Wrong image name or tag.
  - Private registry authentication failure.
  - Image does not exist.

*BackOff* means that the system is gradually increasing the delay before retrying a failed operation. In the case of ImagePullBackOff or ErrImagePull, it specifically refers to progressively longer wait times before attempting to pull the container image again. How BackOff Works in ImagePullBackOff
- First attempt → Kubernetes tries to pull the image. If it fails, it retries immediately.
- Subsequent failures → Kubernetes increases the delay before the next retry (e.g., 10s, 20s, 40s, etc.).
- Exponential BackOff → The wait time grows exponentially to prevent unnecessary retries from overloading the system.
 
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

Since the application logs don’t show any errors, the issue might be related to resource limits, OOM kills, readiness/liveness probes, or underlying node issues
- I would start by inspecting the pod events and status using:
  - `kubectl describe pod <pod-name> -n <namespace>`. This will show me reasons for restarts, such as OOM kills, failed probes, or node issues.
- If the pod was OOMKilled, I’d check the container exit reason using:
  - `kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].state.terminated.reason}'`. If the container has terminated, it prints the reason (e.g., `Completed`, `Error`, `OOMKilled`, `ContainerCannotRun`). If the container is still running or not yet terminated, it returns an empty string.
  - I’d check the pod logs from the previous run to see if there were memory spikes before termination: `kubectl logs <pod-name> --previous`
  - To confirm system-level memory issues, I’d check the node's memory usage:
    ```bash
    kubectl top node
    kubectl top pod -n <namespace>
    ```
- If the pod is not OOMKilled but is restarting due to liveness probe failures
  - `kubectl describe pod <pod-name> -n <namespace>`. I’d specifically look at the `Liveness probe failed` messages. If the probe is failing, I’d check the probe configuration in the Deployment YAML
    ```yaml
     livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    ```
- If probes are fine, but the pod is still restarting
  - Node-Level Issues:
    - Check node health and availability:
      ```bash
      kubectl get nodes -o wide
      kubectl describe node <node-name>
      ```
    - journalctl logs on the node to check for kernel crashes, disk pressure, or kubelet errors: `journalctl -u kubelet -f`
  - Container-Level Issues:
    - Check Docker/container runtime logs:
      ```bash
      docker ps -a | grep <container-id>
      docker logs <container-id>
      ```
    - Verify if the container is exiting due to CrashLoopBackOff:
      ```bash
      kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].state.waiting.reason}'
      ```
    - Networking Issues:
      - If the pod depends on external services, I’d check DNS resolution: `kubectl exec -it <pod-name> -- nslookup <service-name>`
      - Check CNI plugin status: `kubectl get pods -n kube-system | grep cni`
    - Persistent Volume Issues (if applicable):
      - Check if the volume mount is failing: `kubectl describe pod <pod-name> | grep Mount`
- If everything seems fine but the issue persists, I’d:
  - Enable detailed logging on the kubelet and check `/var/log/kubelet.log`.
  - Check if the cluster has any active policies (e.g., `LimitRange` or `PodSecurityPolicy`) affecting pod scheduling.
  - Consider moving the pod to another node using `kubectl cordon` and `kubectl drain` to see if the issue is node-specific.
 
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

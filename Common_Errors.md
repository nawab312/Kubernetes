### ImagePullBackOff / ErrImagePull ###
```bash
kubectl get pods
NAME         READY   STATUS             RESTARTS   AGE
my-app-xyz   0/1     ImagePullBackOff   0          1m
```

**Reason**
- Kubernetes cannot pull the container image from the registry.
- Possible causes:
  - Wrong image name or tag.
  - Private registry authentication failure.
  - Image does not exist.
 
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

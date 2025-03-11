**RestartPolicy** field in a Pod specification controls how the *kubelet* restarts containers within the pod when they exit. The possible values for restartPolicy are:
- **Always (default)**
  - The container is always restarted if it fails or exits. Used for long-running applications like web servers.
- **OnFailure**
  - The container is restarted only if it exits with a non-zero status. Suitable for batch jobs that should only restart on failure.
- **Never**
  - The container is never restarted, even if it fails. Suitable for one-time execution tasks.
- `restartPolicy` applies to the *Pod* level, not individual containers.
- If using *Jobs* or *CronJobs*, the default restartPolicy must be *OnFailure* or *Never* (not Always).
- For *Deployments*, *StatefulSets*, and *DaemonSets*, the restartPolicy is always set to *Always*.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  restartPolicy: OnFailure
  containers:
  - name: my-container
    image: my-batch-job
```

- What happens if there are not enough resources to schedule a pod?
  - Pod is Unschedulable: The Kubernetes scheduler will attempt to find a node with enough resources (CPU, memory, etc.) to accommodate the Pod. If no suitable node is found, the Pod remains in the "Pending" state.
  - Retry Mechanism: Kubernetes continuously retries scheduling the Pod, waiting for resources to free up or for new nodes to be added to the cluster.
  - Cluster Autoscaler (if enabled): If Cluster Autoscaler is configured, Kubernetes can automatically provision new nodes to handle the resource demand. Once new nodes are available, the Pod is scheduled.

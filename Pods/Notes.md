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

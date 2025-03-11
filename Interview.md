**How do you handle pod restarts when thery crash**
- Define a RestartPolicy: By default, Kubernetes uses a RestartPolicy of `Always` for Pods, which automatically restarts containers when they fail. You can also set it to `OnFailure` or `Never` depending on your needs.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  restartPolicy: OnFailure  # Only restart if the Pod fails
  containers:
    - name: my-container
      image: my-app-image
      ports:
        - containerPort: 80
```
- Use a Liveness Probe: Kubernetes liveness probes detect unhealthy containers and restart them if needed.
- Use Deployments or StatefulSets: For workloads managed by a Deployment or StatefulSet, Kubernetes automatically ensures the desired number of Pods are running. If a Pod crashes, the controller will recreate it.
- Check logs and events: Use kubectl logs and kubectl describe pod to understand the cause of crashes and fix the underlying issue

**How does DaemonSet scheduling work?**

*DaemonSet Controller*: Instead of relying entirely on the default Kubernetes scheduler to decide where the pods should go, a DaemonSet controller manages the scheduling of DaemonSet pods. The DaemonSet controller runs a pod on every eligible node in the cluster, or in nodes that match certain criteria (such as labels or taints). It will ensure that the correct number of pods are running, even if nodes are added or removed. For example, if a new node is added to the cluster, the DaemonSet controller will automatically schedule a pod on that node.

**For a service, when we use nodeport, EVERY node does what?**

- Suppose your service is named  `my-service` and has a `NodePort` of `30001`. You send a request to `http://<node-ip>:30001`.
- Kubernetes' `kube-proxy` will proxy that request to the pods associated with my-service based on the service's selector.

**A PV with the Retain reclaim policy is deleted after being bound to a PVC. What happens to the data in the underlying storage?**
- The **Retain reclaim policy** ensures that the underlying storage is not automatically deleted or recycled when the PV is deleted. Instead, the data remains intact and accessible.
- The PV enters a **Released state**, indicating that it is no longer bound to the PVC.
- However, the underlying storage is not available for re-binding to other PVCs unless an administrator manually intervenes. To make the storage reusable, an administrator must manually clean up or prepare the storage (if needed) and re-create the PV to bind it to a new PVC.




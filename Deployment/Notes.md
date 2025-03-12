**Question** What happens if a Kubernetes Deployment has more replicas than the number of available worker nodes?
- Kubernetes Will Try to Schedule All Replicas:
  - The *scheduler* will attempt to place all replicas on available nodes.
  - If there aren’t enough nodes, some Pods will remain in a *Pending* state.
- Pods Might Be Distributed Unevenly:
  - If some nodes have more capacity, multiple Pods might get scheduled on a few nodes.
  - Others will remain unscheduled until resources become available.
- Cluster Autoscaler (If Enabled) Can Add Nodes:
  - If a cloud-based cluster has autoscaling enabled, Kubernetes might provision new nodes.

- Cluster Autoscaler (CA) automatically adjusts the number of worker nodes in a cluster.
- It does not scale pods. It scales nodes.

**What Problem It Solves**
- Kubernetes scheduler places pods on nodes.
- If: Pods can’t be scheduled due to insufficient resources → They remain `Pending`.
- Cluster Autoscaler detects those unschedulable pods and decides: *Would adding a node make this pod schedulable?*
- If yes → it adds a node. If no → it does nothing.

**What it Actually Watches**
- Cluster Autoscaler monitors: Pods stuck in Pending, Scheduling failure reasons, Node group templates, Allocatable resources
- It does not watch: CPU usage, Memory percentage, Pod metrics, HPA metrics. This belongs to HPA

---

Cluster Autoscaler will NOT scale if pods are pending due to:
- Taints without tolerations
- NodeSelector mismatch
- Affinity rules not satisfiable
- Unbound PVC
- Topology constraints
- Pod too large for any node type

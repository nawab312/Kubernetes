**The Four Layers of EKS Cost**
- Layer 1 — EC2 Nodes (biggest cost)
  - The actual servers your pods run on. Wrong instance type or size = Wasted money
- Layer 2 — Pod Resources (directly affects Layer 1)
  - Over-requesting CPU/memory wastes node capacity. Means you need MORE nodes than necessary
- Layer 3 — Idle Resources
  - Namespaces, Deployments, PVCs nobody uses. Still charging you money silently
- Layer 4 — AWS Services
  - Load balancers, EBS volumes, Data transfer. Each LoadBalancer service = ~$18/month

### Right-Sizing Requests and Limits ###
This is the foundation of all cost optimization. Everything else builds on this.

**The Problem — Over-requesting Resources**
- Developer sets resource requests out of caution:
  ```yaml
  requests:
        cpu: 2000m (2 cores)
        memory: 4Gi
  ```
- Actual usage:
  - cpu: 200m (0.2 cores — using 10% of request)
  - memory: 512Mi (using 12% of request)
- What this means for your cluster:
  - 10 pods × 2 CPU requested = 20 CPU reserved on nodes
  - Actual usage = 2 CPU
  - 18 CPU wasted = Nodes reserved but doing nothing
  - Kubernetes scheduler sees node as "full" at 20 CPU. Even though actual usage is only 2 CPU. You need 10x more nodes than necessary

**The Problem — Under-requesting Resources**
- Developer sets requests too low to save money:
  ```yaml
  requests:
        cpu:    10m
        memory: 64Mi
  ```
- Actual usage:
  - cpu: 500m
  - memory: 1Gi
- What happens:
  - Pod scheduled on node (looks tiny to scheduler)
  - Node becomes overcommitted — Too many pods
  - All pods competing for CPU → Throttling
  - Pod hits memory limit → OOMKilled
 
**The Right Approach — Data-Driven Right-Sizing**
- Never guess resource requests. Measure first.
```yaml
# Step 1 — see current usage
kubectl top pods --all-namespaces --sort-by=cpu
kubectl top pods --all-namespaces --sort-by=memory

# Step 2 — compare usage vs requests
kubectl describe pod myapp-pod | grep -A10 "Requests"
kubectl top pod myapp-pod

# Example output:
# Requests:
#   cpu:    2000m   ← requested
#   memory: 4Gi     ← requested
#
# kubectl top:
# myapp-pod   150m   512Mi  ← actual usage
#
# 2000m requested but only 150m used → massively over-requested
```

**Using VPA in Recommendation Mode**
- VPA's safest mode — Just tells you what to set, does not change anything:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
  updatePolicy:
    updateMode: "Off"     # recommendation only — no changes made
```
```bash
# check recommendations
kubectl describe vpa myapp-vpa

# Output:
# Recommendation:
#   Container Recommendations:
#     Container Name: myapp
#     Lower Bound:    cpu: 50m,   memory: 128Mi
#     Target:         cpu: 150m,  memory: 256Mi  ← set this
#     Upper Bound:    cpu: 400m,  memory: 512Mi
#
# You were requesting 2000m — VPA says 150m is enough
# Reduce requests → pack more pods per node → need fewer nodes
```

**Right-Sizing Formula**
- Set requests based on:
  - CPU request = Average CPU usage
  - Memory request = Average memory usage + 20% buffer
- Set limits based on:
  - CPU limit = Peak CPU usage × 1.5 (or no CPU limit — CPU throttling is better than OOMKill)
  - Memory limit = Peak memory usage × 1.5 (Always set memory limit — Prevents runaway memory usage)

---
---

### Spot Instances for Non-Critical Workloads ###
Spot instances are spare AWS EC2 capacity available at up to 90% discount compared to on-demand pricing.

**Which Workloads Are Safe on Spot**
- Safe for Spot (Stateless, fault-tolerant):
  - Web servers and APIs (multiple replicas)
  - Batch processing jobs
  - CI/CD build agents (Jenkins, GitHub Actions runners)
  - Data processing pipelines
  - Machine learning training jobs
  - Dev and staging environments
  - Worker queues
- NOT safe for Spot:
  - Databases (single instance, data loss risk)
  - Stateful apps with no replication
  - Production systems where ANY downtime is unacceptable
  - Long-running jobs that cannot be restarted
 
***How to Use Spot on EKS — Node Labels**
```yaml
# Node group or Karpenter NodePool for spot
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-pool
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]

        # IMPORTANT: multiple instance types
        # reduces interruption risk
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m5", "m6i", "m5a", "m6a", "c5", "c6i"]
```
```yaml
# Tell your deployment to prefer spot nodes
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: karpenter.sh/capacity-type
                    operator: In
                    values: ["spot"]
```

**Spot Interruption Handling**
```bash
AWS sends interruption notice 2 minutes before termination
        ↓
AWS Node Termination Handler (NTH) DaemonSet detects notice
        ↓
NTH cordons the node — No new pods scheduled
        ↓
NTH drains the node — Pods evicted gracefully
        ↓
Pods reschedule onto other nodes 
        ↓
EC2 instance terminated by AWS
```
- Install Node Termination Handler:
```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-node-termination-handler \
    eks/aws-node-termination-handler \
    --namespace kube-system
```
- Note: If using Karpenter, it handles spot interruption natively — NTH not needed.

**Spot Best Practices**
- Always use multiple instance types. Never pin to one instance type. If m5.xlarge spot is unavailable → Use m6i.xlarge or c5.2xlarge
- Always have multiple replicas. 1 replica on spot = Single point of failure. 3 replicas across multiple AZs = Safe
- Use Pod Disruption Budgets. Ensures minimum replicas stay running during spot interruption
- Spread across AZs. Spot capacity varies by AZ. Use topology spread constraints

---
---

### Karpenter with Spot + On-Demand Mix ###
The smartest cost strategy — use spot for most workloads, on-demand for critical ones, and let Karpenter manage the mix automatically.

**Strategy — Prioritize Spot, Fall Back to On-Demand**
- Normal operation: Karpenter provisions spot instances (cheap)
- Spot capacity unavailable or interrupted:
  - Karpenter falls back to on-demand automatically
  - No manual intervention needed
- Result:
  - 70-80% of workloads run on spot
  - 20-30% on on-demand (critical or when spot unavailable)
  - Average savings: 50-60% vs all on-demand
 
**NodePool Configuration for Mixed Capacity**
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: mixed-capacity
spec:
  template:
    spec:
      requirements:
        # prefer spot, allow on-demand as fallback
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]

        # many instance types → better spot availability
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m5", "m6i", "m5a", "m6a", "c5", "c6i", "r5", "r6i"]

        # minimum 2 vCPUs, max 16
        - key: karpenter.k8s.aws/instance-cpu
          operator: In
          values: ["2", "4", "8", "16"]

  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 1m
```

**Separate NodePools for Different Workloads**
```yaml
# NodePool 1 — Spot for non-critical workloads
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    metadata:
      labels:
        workload-type: non-critical
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
  weight: 100    # prefer this pool first

---
# NodePool 2 — On-demand for critical workloads
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: ondemand-critical
spec:
  template:
    metadata:
      labels:
        workload-type: critical
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
  weight: 10     # use this pool only when needed
```
```yaml
# Critical deployment — pin to on-demand nodes
spec:
  template:
    spec:
      nodeSelector:
        workload-type: critical

# Non-critical deployment — use spot nodes
spec:
  template:
    spec:
      nodeSelector:
        workload-type: non-critical
```

**Karpenter Consolidation for Cost Savings**
- Consolidation constantly looks for savings:
- Scenario:
  - 5 nodes running. After traffic drops:
  - Node 1: 80% utilized
  - Node 2: 80% utilized
  - Node 3: 20% utilized ← Underutilized
  - Node 4: 15% utilized ← Underutilized
  - Node 5: 10% utilized ← Underutilized
- Karpenter consolidation: Can pods from nodes 3, 4, 5 fit on nodes 1 and 2. Yes → Evict pods → Reschedule on nodes 1 and 2. Terminate nodes 3, 4, 5. 5 nodes → 2 nodes = 60% cost reduction
- Also replaces expensive on-demand with spot when available:
  - On-demand node running but spot now available and cheaper
  - Karpenter replaces on-demand with spot → Immediate savings
 
---
---

### Cluster Autoscaler Scale-Down Behaviour ###
If you are using Cluster Autoscaler instead of Karpenter, understanding its scale-down behaviour is important for cost optimization.

**How Scale-Down Works**
```bash
CA continuously evaluates every node
        ↓
For each node it asks:
  1. Has this node been underutilized for 10 minutes?
     (less than 50% CPU requested AND less than 50% memory requested)
  2. Can all pods on this node fit on other nodes?
  3. Are there any blockers preventing scale-down?
        ↓
All conditions met → CA cordons and drains the node
        ↓
Pods reschedule → Node removed from ASG
```

**Scale-Down Blockers — Why Nodes Get Stuck**
- This is why your cluster might not scale down even when idle:
- Blocker 1 — Pod with no controller:
  - A standalone Pod (not managed by Deployment, DaemonSet etc.)
  - CA will NOT evict it → Node stays
  - Fix: always use Deployments, Never standalone Pods
- Blocker 2 — Pod with local storage (emptyDir):
  - Pod uses emptyDir volume
  - CA assumes data cannot be moved → Skips node
  - Fix: Annotate pod if data loss is acceptable `cluster-autoscaler.kubernetes.io/safe-to-evict: "true"`
- Blocker 3 — PodDisruptionBudget violated:
  - Evicting pod would violate PDB minAvailable
  - CA waits → Node never freed
  - Fix: Ensure PDB allows at least 1 disruption
- Blocker 4 — Node has system pods:
  - kube-system pods that cannot be evicted. CA skips the node
- Blocker 5 — Pod with restrictive node affinity:
  - Evicted pod can ONLY run on THIS node
  - CA will not evict because pod has nowhere to go
 
**Tuning Scale-Down for Better Cost Savings**
```yaml
# cluster-autoscaler deployment args
command:
  - ./cluster-autoscaler
  - --cloud-provider=aws
  - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled
  - --scale-down-enabled=true

  # how long node must be underutilized before scale-down
  - --scale-down-unneeded-time=5m        # default 10m, reduce for faster savings

  # utilization threshold for scale-down
  - --scale-down-utilization-threshold=0.5   # default 0.5 (50%)

  # how long to wait after scale-up before scale-down
  - --scale-down-delay-after-add=5m      # default 10m

  # max nodes to remove at once
  - --max-empty-bulk-delete=10
```

---
---

### Fargate Cost vs Managed Nodes ###
Fargate is AWS's serverless compute for containers — no EC2 nodes to manage.

**How Fargate Works on EKS**
- Managed Nodes:
  - You provision EC2 instances (node groups)
  - Pods run on these EC2 instances
  - You pay for EC2 instance whether pods use it or not
  - You manage node upgrades, patches, scaling
- Fargate:
  - No EC2 instances to manage
  - Each Pod gets its own isolated compute environment
  - You pay ONLY for pod CPU and memory
  - AWS manages all the underlying infrastructure
 
**When Fargate is Cheapers**
- Fargate wins when:
  - Bursty, Unpredictable workloads
  - Many small short-lived pods (batch jobs)
  - Low utilization workloads
  - You want zero node management overhead
  - Example: 10 pods running 8 hours/day (not 24/7). Managed nodes: Pay for EC2 24/7 = Full cost. Fargate: Pay only for 8 hours = 67% savings
- Managed nodes win when:
  - High, Consistent utilization
  - Large pods that fill nodes efficiently
  - Need GPU instances (Fargate does not support GPU)
  - Need faster startup (Fargate cold start is slower)
  - Need DaemonSets (Fargate does not support DaemonSets)
 
---
---

### Idle Namespace and Resource Cleanup ###
Silent money wasters. Resources nobody uses but you are still paying for.

**Types of Idle Resources**
- Idle Namespaces
  - Dev namespace created for a feature that shipped 6 months ago. Still running 3 deployments, 2 services, 1 PVC. Nobody uses it — Developer forgot it exists.
  ```bash
  # find namespaces with no recent activity
  kubectl get namespaces
  
  # check last time pods ran in each namespace
  kubectl get pods -n old-feature-dev
  kubectl get events -n old-feature-dev \
      --sort-by='.lastTimestamp' | tail -5
  
  # if nothing happened in 30+ days → candidate for deletion
  ```
- Orphaned PersistentVolumeClaims
  - Pod deleted → PVC stays behind (PVCs are not deleted with pods). EBS volume still exists and charging you. $0.10/GB/month × 100GB × 10 orphaned PVCs = $100/month wasted
  ```bash
  # find PVCs not mounted by any pod
  kubectl get pvc --all-namespaces
  
  # check which PVCs have no associated pod
  kubectl get pvc -A -o json | \
      jq '.items[] | select(.status.phase=="Bound") |
      select(.metadata.annotations."volume.kubernetes.io/selected-node" != null) |
      .metadata.name'
  
  # or simply look for PVCs in namespaces with no pods
  kubectl get pods -n myapp-dev
  # No resources found
  kubectl get pvc -n myapp-dev
  # myapp-data   Bound   10Gi   ← orphaned!
  ```
- LoadBalancer Services Nobody Uses
  - Each LoadBalancer type Service = AWS ELB = ~$18/month minimum. 10 forgotten LoadBalancer services = $180/month wasted
  ```bash
  # find all LoadBalancer services
  kubectl get svc -A | grep LoadBalancer
  
  # check if any traffic is hitting them
  # look at CloudWatch metrics for the ELB
  # zero requests in 30 days → safe to delete
  ```
- Completed Jobs and Their Pod
  - CronJob runs every hour. Completed pods pile up — 720 pods after a month. Each pod object uses etcd storage. Does not cost money directly but adds API server load
  ```bash
  # find completed and failed pods
  kubectl get pods -A | grep -E "Completed|Error"
  
  # delete completed pods
  kubectl delete pods -A \
      --field-selector=status.phase==Succeeded
  
  # configure TTL on Jobs so they auto-clean
  spec:
    ttlSecondsAfterFinished: 3600    # delete job 1 hour after completion
  ```

**Automated Cleanup Tools**
- Doing this manually is not scalable. Use tools:
- Kubernetes Resource Report:
  ```bash
  # install kube-resource-report
  # shows unused resources, their costs, and recommendations
  
  helm install kube-resource-report \
      codeberg.org/hjacobs/kube-resource-report
  ```
- Goldilocks — Right-Sizing Dashboard:
  ```bash
  # installs VPA in recommendation mode for ALL deployments
  # shows nice dashboard of what requests should be
  
  helm repo add fairwinds-stable https://charts.fairwinds.com/stable
  helm install goldilocks fairwinds-stable/goldilocks \
      --namespace goldilocks \
      --create-namespace
  
  # label a namespace to enable
  kubectl label namespace myapp \
      goldilocks.fairwinds.com/enabled=true
  
  # view recommendations
  kubectl port-forward svc/goldilocks-dashboard 8080:80 -n goldilocks
  # open browser at localhost:8080
  ```
- Popeye — Cluster Sanitizer:
  ```bash
  # scans cluster for misconfigurations and waste
  brew install derailed/popeye/popeye
  popeye --context my-eks-cluster
  
  # reports:
  #   Unused ConfigMaps
  #   Unused Secrets
  #   Pods with no resource limits
  #   Services with no endpoints
  #   Dead namespaces
  ```

**Cost Monitoring — AWS Cost Explorer + Kubecost**
- AWS Cost Explorer:
  - Shows EC2 spend by instance type
  - Shows EBS volume spend
  - Shows ELB spend
  - Tells you WHERE money goes at AWS level
- Kubecost:
  - Shows cost per namespace
  - Shows cost per deployment
  - Shows cost per team (label-based)

**Why Auto Scaling Matters**
- Without auto scaling:
  - Traffic spike hits your app: Pods are overwhelmed → Slow responses → Timeouts
  - Traffic drops at night: Pods sitting idle → Wasting money
  - All nodes are full: New pods cannot schedule → Stuck Pending
- With auto scaling:
  - Traffic spike → HPA adds more pods automatically
  - Traffic drops → HPA removes pods → Saves money
  - Nodes full → Karpenter adds new nodes in seconds
  - Nodes empty → Karpenter removes them → saves money

**The Three Layers of Auto Scaling**
- Layer 1 — Pod level (what runs inside nodes)
  - HPA → Adds/Removes pod replicas based on CPU/memory/custom metrics
  - VPA → Increases/Decreases CPU/memory of existing pods
  - KEDA → Adds/Removes pods based on external events (SQS, Kafka)
- Layer 2 — Node level (the machines themselves)
  - Cluster Autoscaler → Adds/Removes EC2 nodes on EKS
  - Karpenter → Smarter, faster AWS-native node provisioning
- They work together:
  - HPA needs more pods → Nodes are full → Karpenter adds a node
  - Traffic drops → HPA removes pods → Nodes empty → Karpenter removes node
 
### HPA — Horizontal Pod Autoscaler ###
HPA adds or removes pod replicas based on observed metrics. Horizontal = More pods (scale out) or Fewer pods (scale in).

**How HPA Works — Internals**
```bash
HPA controller runs in the control plane
Checks metrics every 15 seconds (default)
        ↓
Fetches current metric value from Metrics Server
(CPU usage of all pods in the deployment)
        ↓
Compares against target value you configured
        ↓
Calculates desired replica count:

desiredReplicas = currentReplicas × (currentMetricValue / targetMetricValue)

Example:
  currentReplicas  = 2
  currentCPU       = 80% average across pods
  targetCPU        = 50%

  desiredReplicas  = 2 × (80 / 50) = 3.2 → Rounds up to 4

HPA scales from 2 → 4 pods 
```

**Prerequisites — Metrics Server**
- HPA needs Metrics Server installed to get CPU and memory data.
```bash
# check if metrics server is running
kubectl get pods -n kube-system | grep metrics-server

# install on EKS
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# verify it works
kubectl top pods
kubectl top nodes
```

**Basic HPA — CPU Based**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment      # which deployment to scale

  minReplicas: 2                # never go below 2
  maxReplicas: 10               # never go above 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50    # target 50% CPU across all pods
```

**HPA with Memory**
```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi     # target 512MB average per pod
```

**HPA with Multiple Metrics**
- HPA evaluates ALL metrics and uses the one that requires the most replicas.
```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50

  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
```
- CPU says: Scale to 4 pods
- Memory says: Scale to 6 pods
- HPA chooses: 6 pods (Highest wins)

**HPA Scaling Behaviour — Cooldown**
- By default HPA has cooldown periods to prevent thrashing (constantly scaling up and down):
```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # wait 60s before scaling up again
      policies:
        - type: Pods
          value: 4                       # add at most 4 pods per minute
          periodSeconds: 60

    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 minutes before scaling down
      policies:
        - type: Pods
          value: 2                       # remove at most 2 pods per minute
          periodSeconds: 60
```
- Why scale down is slower than scale up:
  - Scale up fast → Users need capacity NOW
  - Scale down slow → Traffic might spike again soon. Dont remove capacity too aggressively

***Check HPA Status**
```bash
kubectl get hpa

# NAME        REFERENCE             TARGETS   MINPODS  MAXPODS  REPLICAS
# myapp-hpa   Deployment/myapp      48%/50%   2        10       4

kubectl describe hpa myapp-hpa
# shows scaling events and decisions
```

**Important — Pods MUST Have Resource Requests**
```yaml
# HPA cannot work without resource requests because it calculates utilization as:
# (actual usage / requested amount) × 100

# HPA will not work
containers:
  - name: myapp
    image: myapp:1.0
    # no resources defined

# HPA works correctly
containers:
  - name: myapp
    image: myapp:1.0
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
```

---
---

### VPA — Vertical Pod Autoscaler ###
- VPA adjusts the CPU and memory requests/limits of existing pods instead of adding more pods.
- Vertical = Bigger pods (Scale up) or Smaller pods (Scale down).

**VPA vs HPA**
- HPA:
  - Your app gets more traffic
  - Add MORE pods to share the load
  - Horizontal scaling — More instances
  - Good for: Stateless apps, Web servers, APIs
- VPA:
  - Your app needs more memory
  - Give EXISTING pods more memory
  - Vertical scaling — Bigger instances
  - Good for: Single-instance apps, Databases, Apps that cannot run multiple replicas
 
**How VPA Works**
- VPA has three components:
  - Recommender
    - Watches pod resource usage continuously
    - Builds recommendation: "this pod should have cpu: 500m, memory: 1Gi"
  - Updater
    - Checks running pods against recommendations
    - If pod is too far from recommendation → Evicts the pod
  - Admission Controller
    - When evicted pod restarts
    - Intercepts the pod creation
    - Injects the new resource requests before pod starts
- VPA flow:
  - Pod running with cpu request: 100m
  - Actual usage: 800m consistently
  - VPA Recommender: "this pod needs cpu: 900m"
  - VPA Updater: Evicts the pod
  - VPA Admission Controller: New pod starts with cpu: 900m
 
**VPA Modes**
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
    updateMode: "Auto" 
```
- Off:
  - VPA only calculates recommendations
  - Does NOT apply them
  - Use to see what VPA would recommend without changing anything
  - Good for: Learning what your app actually needs
- Initial:
  - VPA applies recommendations only when pod is FIRST created
  - Does not touch running pods
  - Safer than Auto
- Auto:
  - VPA evicts and restarts pods to apply new recommendations
  - Most powerful but causes pod restarts
- Recreate:
  - Same as Auto — Evicts pods when recommendations change

**Checking VPA Recommendations**
```bash
kubectl describe vpa myapp-vpa

# Recommendation:
#   Container Recommendations:
#     Container Name: myapp
#     Lower Bound:    cpu: 100m, memory: 256Mi
#     Target:         cpu: 500m, memory: 512Mi   ← Apply this
#     Upper Bound:    cpu: 1000m, memory: 1Gi
```

**Risks of Using VPA**
- Risk 1 — Pod restarts cause downtime:
  - VPA evicts pods to apply new resource values. If you only have 1 replica → Downtime during eviction. Always use minReplicas: 2+ with VPA Auto mode
- Risk 2 — VPA and HPA conflict:
  - VPA changes resource requests. HPA uses resource requests to calculate utilization. Both trying to control the same pods → Unpredictable behaviour
  - Safe combination: VPA manages memory (HPA does not scale on memory). HPA manages CPU scaling. Do NOT use both on same metric
- Risk 3 — Sudden large resource changes:
  - VPA might recommend a huge jump: 100m → 2000m CPU. Node might not have capacity for the new pod size. Pod stays Pending after eviction
- Risk 4 — Recommendation based on history:
  - VPA looks at past usage. If your app has not yet seen peak traffic. VPA underestimates → Sets limits too low

---
---

### Cluster Autoscaler on EKS ###
Cluster Autoscaler (CA) adds or removes EC2 nodes in your EKS node groups when pods cannot be scheduled or nodes are underutilized.

**How Cluster Autoscaler Works**
```bash
Scale UP:
  New pod created
  Scheduler cannot find a node with enough CPU/Memory
  Pod stays Pending
        ↓
  Cluster Autoscaler detects Pending pod
  Checks: "if I add a node, would this pod fit?"
  Yes → Tells AWS Auto Scaling Group to increase desired count
        ↓
  AWS launches new EC2 instance
  EC2 joins cluster as a node
  Pod scheduled onto new node 

Scale DOWN:
  Node has been underutilized for 10 minutes (default)
  All pods on it can fit on other nodes
        ↓
  Cluster Autoscaler cordons the node
  Evicts all pods (respecting PDBs)
  Pods reschedule onto other nodes
        ↓
  Tells ASG to decrease desired count
  EC2 instance terminated 
```

**Installing Cluster Autoscaler on EKS**
```bash
# Step 1 — IAM policy for Cluster Autoscaler
# CA needs permission to modify Auto Scaling Groups

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeInstanceTypes",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}

# Step 2 — attach policy to node group IAM role or use IRSA for the CA service account

# Step 3 — deploy Cluster Autoscaler
kubectl apply -f cluster-autoscaler-deployment.yaml
```

**Node Group Tags — Required**
- Cluster Autoscaler discovers node groups via AWS tags:
```bash
Tag these on your Auto Scaling Group:
  k8s.io/cluster-autoscaler/enabled          = true
  k8s.io/cluster-autoscaler/<cluster-name>   = owned
```

**Cluster Autoscaler Limitations**
- Problem 1 — Slow to provision:
  - CA decides to add a node
  - AWS launches EC2 instance
  - Instance bootstraps, joins cluster
  - Total time: 3-5 minutes
  - Pods stuck Pending during this time
- Problem 2 — Node group aware only:
  - CA can only scale node groups you pre-configured
  - Want a different instance type? → Need a new node group
  - Inflexible for mixed workloads
- Problem 3 — Scale down is conservative:
  - Node must be underutilized for 10 minutes
  - Has many reasons to NOT scale down (default behavior)
  - Often leaves idle nodes running → Wastes money
- Problem 4 — One node at a time:
  - Adds nodes one by one
  - Burst traffic needs 10 nodes fast? CA adds them slowly
 
---
---

### KEDA — Event-Driven Autoscaling ###
- HPA limitation:
  - HPA only scales on CPU, Memory, or Custom metrics
  - What if you want to scale based on:
    - SQS queue depth? → HPA cannot do this natively
    - Kafka consumer lag? → HPA cannot do this natively
    - RabbitMQ queue length? → HPA cannot do this natively
    - Cron schedule? → HPA cannot do this natively
- KEDA solves all of these

**How KEDA Works**
- KEDA has two components:
  - KEDA Operator
    - Watches ScaledObject resources you create
    - Connects to external event sources (SQS, Kafka etc.)
    - Reads current metric value (queue depth, lag etc.)
    - Tells HPA what to scale to
  - Metrics Adapter
    - Exposes external metrics to Kubernetes metrics API
    - HPA reads these metrics and scales accordingly
- Flow:
  - SQS queue has 1000 messages
  - KEDA reads queue depth = 1000
  - KEDA tells HPA: scale to 10 pods
  - HPA scales deployment to 10 pods
  - Queue drains to 0 messages
  - KEDA tells HPA: Scale to 0 pods ← Scale to ZERO (HPA cannot do this)
 
**Scale to Zero — KEDA's Superpower**
- HPA minimum is 1 replica — Cannot scale to zero
- KEDA can scale to ZERO replicas when there is no work
- Example:
  - Batch job processor
  - No messages in SQS queue → 0 pods running → Zero cost 
  - Messages arrive in queue → KEDA scales from 0 → N pods 
  - Queue empties → Scales back to 0
 
**KEDA with AWS SQS**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-consumer-scaler
spec:
  scaleTargetRef:
    name: sqs-consumer-deployment

  minReplicaCount: 0          # scale to zero when queue empty
  maxReplicaCount: 20         # max 20 consumers

  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
        queueLength: "10"     # 1 pod per 10 messages in queue
        awsRegion: us-east-1
      authenticationRef:
        name: keda-aws-creds  # IRSA or access key reference
```

**KEDA with Kafka**
```yaml
triggers:
  - type: apache-kafka
    metadata:
      bootstrapServers: kafka-broker:9092
      consumerGroup: myapp-consumer-group
      topic: orders
      lagThreshold: "100"     # 1 pod per 100 messages of lag
```

**KEDA with Cron Schedule**
```yaml
triggers:
  - type: cron
    metadata:
      timezone: Asia/Kolkata
      start: 30 8 * * 1-5      # 8:30 AM weekdays
      end: 30 18 * * 1-5        # 6:30 PM weekdays
      desiredReplicas: "10"     # run 10 replicas during business hours
```

**KEDA Scalers Available**
- AWS: SQS Queue depth, CloudWatch metrics, DynamoDB streams, Kinesis stream lag
- Messaging: Kafka consumer lag, RabbitMQ queue length
- Databases: PostgreSQL query result, MySQL query result, Redis list length
- Other: Prometheus metrics, Cron schedule, HTTP request rate (HTTP Add-on)

---
---

### Karpenter — AWS-Native Node Autoscaler ###
Karpenter is a modern node provisioner built by AWS that replaces Cluster Autoscaler for EKS. It provisions nodes much faster and more intelligently.

**Karpenter vs Cluster Autoscaler — The Core Difference**
- Cluster Autoscaler:
  - Works with pre-configured node groups
  - Scales existing ASGs up or down
  - Limited to instance types you pre-configured
  - Slow — Goes through ASG lifecycle
- Karpenter:
  - Bypasses node groups entirely
  - Creates EC2 instances directly via AWS APIs
  - Can pick the BEST instance type for each workload
  - Fast — Provisions nodes in ~60 seconds vs 3-5 minutes

**How Karpenter Works**
```bash
Pod is Pending (no node has capacity)
        ↓
Karpenter detects the Pending pod immediately
        ↓
Reads pod requirements:
  cpu: 4 cores
  memory: 8Gi
  nodeSelector: zone=us-east-1a
  spot: preferred
        ↓
Karpenter finds the cheapest/best matching EC2 instance
(Evaluates hundreds of instance types in real time)
        ↓
Launches EC2 instance DIRECTLY via AWS API
        ↓
Node joins cluster in ~60 seconds
        ↓
Pod scheduled 

No node group configuration needed 
No ASG to manage 
```

**NodePool — Karpenter Configuration**
- NodePool defines the constraints and preferences for nodes Karpenter can provision:
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        # allow any of these instance families
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m5", "m6i", "c5", "c6i", "r5"]

        # allow both spot and on-demand
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]

        # only us-east-1 AZs
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-east-1a", "us-east-1b", "us-east-1c"]

      nodeClassRef:
        name: default    # EC2NodeClass with AMI, security groups etc.

  # scale down empty nodes after 30 minutes
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30m

  # maximum resources this NodePool can provision
  limits:
    cpu: 1000
    memory: 1000Gi
```

**Karpenter Consolidation — Smart Scale Down**
- Consolidation is Karpenter's scale-down strategy
- Scenario: 10 nodes running. Traffic drops. Pods now only need 3 nodes. 7 nodes are underutilized
- Karpenter consolidation: Evaluates if pods can fit on fewer nodes. Identifies the 7 underutilized nodes. Gracefully evicts pods off them. Pods reschedule onto remaining 3 nodes. 7 nodes terminated
- Two consolidation policies:
  - WhenEmpty: Only remove nodes with NO pods running
  - WhenUnderutilized: Remove and Repack nodes with few pods

**Spot Instances with Karpenter**
- One of Karpenter's biggest advantages — Intelligent spot instance handling:
```bash
Karpenter provisions a spot instance
        ↓
AWS sends spot interruption notice (2 minutes warning)
        ↓
Karpenter receives the interruption notice
        ↓
Immediately starts provisioning a replacement node
        ↓
Gracefully drains the spot node
        ↓
Replacement node ready — Pods rescheduled 
Minimal disruption 
```

**Before diving into components, understand what Kubernetes actually is:**
- You have an application to run. You have multiple servers (nodes) to run it on
- Without Kubernetes:
  - You manually decide which server runs what
  - You manually restart apps when they crash
  - You manually handle scaling
  - You manually manage networking between apps
- With Kubernetes:
  - You say WHAT you want (3 replicas of myapp)
  - Kubernetes figures out HOW and WHERE to run it
  - Kubernetes heals it when it crashes
  - Kubernetes scales it when traffic spikes
  - You focus on your app, not the infrastructure

<img width="742" height="470" alt="image" src="https://github.com/user-attachments/assets/5533b5b9-469e-49f0-b1f9-1b76354baae2" />

### Control Plane Components ###
The control plane is the brain of the cluster. It makes all decisions — where to run pods, when to scale, when to restart. On EKS, AWS manages the control plane for you. You never SSH into control plane nodes.

**API Server — The Front Door**
- The API Server is the single entry point for everything in Kubernetes. Every request — from kubectl, from controllers, from kubelets — goes through the API Server.
- Nothing in Kubernetes talks directly to anything else. Everything goes through the API Server
- kubectl → API Server → etcd
- kubelet → API Server → Gets pod assignments
- Controller → API Server → Watches for changes
- Scheduler → API Server → Gets pending pods, writes node assignments
- What it does:
  - Authentication
    - Who are you? Validates your certificate, token, or IAM identity (EKS)
  - Authorization
    - Are you allowed to do this? Checks RBAC rules — Can this user create pods in this namespace?
  - Admission Control
    - Should this request be allowed? Runs admission webhooks — Enforces policies, Injects sidecars, Validates resource limits
  - Validation
    - Is this request valid Kubernetes syntax? Checks the object schema
  - Persistence
    - Writes the accepted object to etcd. Returns success to the caller
- Real flow when you run `kubectl apply`:
  ```bash
  kubectl apply -f deployment.yaml
          ↓
  API Server receives request
          ↓
  Authenticates your kubeconfig credentials
          ↓
  Authorizes: Can you create Deployments in default namespace?
          ↓
  Admission controllers run:
    LimitRanger adds default resource limits if missing
    PodSecurity checks security policy
    MutatingWebhook injects Istio sidecar if configured
          ↓
  Validates Deployment schema is correct
          ↓
  Writes Deployment object to etcd
          ↓
  Returns 200 OK to kubectl
          ↓
  kubectl prints: deployment.apps/myapp created
  ```

**etcd — The Database**
- etcd is the only persistent storage in Kubernetes. Everything Kubernetes knows about your cluster lives in etcd.
- etcd stores:
  - All Kubernetes objects: Deployments, Pods, Services, ConfigMaps, Secrets, Namespaces, RBAC rules, Node registrations, Everything
  - Current state:
    - Which pods are running on which nodes
    - Which nodes are healthy
    - What resources are allocated
  - Desired state:
    - What you asked Kubernetes to run. Replica counts, images, configs
- Key characteristics:
  - Distributed key-value store:
    - Not a Relational database — No tables, No SQL
    - Simple Key → Value storage
    - `Key: /registry/deployments/default/myapp`, `Value: Full JSON of the Deployment object`
  - Strongly consistent:
    - Uses Raft consensus algorithm
    - Every read returns the latest write
    - Critical for correctness — No stale data
  - HA setup — Odd number of nodes:
    - 1 Node → No HA (single point of failure)
    - 3 Nodes → Can tolerate 1 failure
    - 5 Nodes → Can tolerate 2 failures
    - Always odd number — Needs majority (quorum) to elect leader
    - 3 Nodes → quorum = 2 → Can lose 1
    - 5 Nodes → quorum = 3 → Can lose 2
- Why etcd matters so much:
  - etcd is destroyed → Your entire cluster state is gone. Kubernetes does not know what was running. Does not know your Deployments, Services, Secrets
  - This is why:
    - On EKS — AWS backs up etcd automatically
    - On-prem — YOU must back up etcd regularly
    - etcd backup = velero snapshots or etcdctl snapshot save
- What is NOT stored in etcd:
  - Container logs → Stored on node disk
  - Metrics data → Stored in Prometheus/CloudWatch
  - Container images → Stored in container registry
  - Application data → Stored in PersistentVolumes (EBS, EFS)

**Scheduler — The Placement Engine**
- The Scheduler watches for Pending pods and decides which node each pod should run on.
  ```bash
  New pod created → Pod has no node assigned (Pending)
          ↓
  Scheduler notices the unscheduled pod
          ↓
  Runs scheduling algorithm:
  
    Phase 1 — Filtering (find valid nodes)
      Goes through every node and asks:
      "Can this pod even run on this node?"
  
      Filters out nodes that:
        Don't have enough CPU or memory
        Don't match the pod's nodeSelector
        Have taints the pod doesn't tolerate
        Are in the wrong zone
        Don't have the required port available
        Are not Ready
  
    Phase 2 — Scoring (find the best node)
      For remaining valid nodes, scores each one:
  
      LeastRequestedPriority:
        Prefer nodes with more free resources
        Spreads pods across nodes
  
      NodeAffinityPriority:
        Prefer nodes matching node affinity rules
  
      InterPodAffinityPriority:
        Prefer nodes where related pods already run
  
      ImageLocalityPriority:
        Prefer nodes that already have the image cached
  
          ↓
    Picks highest scoring node
          ↓
    Writes node name to pod spec in etcd
    (called "binding" — Pod is now bound to the node)
          ↓
    kubelet on that node sees the assignmentand starts the container
  ```

  **Controller Manager — The Reconciliation Engine**
  - The Controller Manager runs controllers — loops that watch the current state of the cluster and make it match the desired state.
  - The core concept — Reconciliation loop:
    - Desired state (what you want): 3 replicas of myapp
    - Current state (what exists): 2 replicas running
    - Action: Create 1 more pod
  - This loop runs constantly — Every few seconds. *This is why Kubernetes is self-healing*
  - Controllers running inside Controller Manager:
    - Deployment Controller: Watches Deployment objects. Creates/manages ReplicaSets. Handles rolling updates
    - ReplicaSet Controller: Watches ReplicaSet objects. Ensures correct number of pods. Creates pods when count is low. Deletes pods when count is high
    - Node Controller: Watches node health. Marks nodes as NotReady when they stop reporting. Evicts pods from unhealthy nodes after timeout
    - Job Controller: Watches Job objects. Creates pods to run the job. Tracks completions and failures
    - ServiceAccount Controller: Creates default ServiceAccount in every new namespace. Ensures tokens exist for service accounts
    - Namespace Controller: Cleans up all resources when namespace is deleted
    - EndpointSlice Controller: Watches Services and Pods. Keeps EndpointSlices updated with healthy pod IPs

---
---

### Worker Node Components ###
Worker nodes are where your actual application containers run. Every worker node has three components.

**kubelet — The Node Agent**
- kubelet is an agent that runs on every worker node. It is the bridge between the control plane and the containers on that node.
- kubelet does two things:
  - Watches API Server for pod assignments. "Has any pod been assigned to MY node?". Polls API Server constantly. New pod Assigned → kubelet takes action
  - Reports node and pod status back to API Server. "Here is my CPU/memory usage". "Here are the pods I am running and their states". "My liveness probe for pod X is failing". Sends heartbeat every few seconds
- Full pod lifecycle managed by kubelet:
  ```bash
  API Server assigns pod to this node
          ↓
  kubelet reads the pod spec
          ↓
  kubelet calls container runtime:
    "Pull this image: myapp:1.0"
    "Create this container with these env vars and volumes"
          ↓
  kubelet sets up volumes:
    Mounts ConfigMaps and Secrets
    Creates emptyDir volumes
    Attaches PersistentVolumes (EBS)
          ↓
  kubelet sets up networking:
    Calls CNI plugin to set up pod networking
    Assigns pod IP address
          ↓
  kubelet starts the container
          ↓
  kubelet runs liveness and readiness probes
          ↓
  kubelet reports pod status to API Server:
    Running, Ready, ContainerStatuses
          ↓
  Pod deleted → kubelet sends SIGTERM → waits → sends SIGKILL → cleans up
  ```

**kube-proxy — The Network Rules Manager**
- kube-proxy runs on every node and implements Services — it makes the virtual Service IPs actually work.
- You create a Service: ClusterIP 10.96.0.1:80. This IP does not actually exist on any network interface. It is virtual — something needs to intercept traffic to it
- kube-proxy writes iptables rules on every node:
  ```bash
  Rule: "Any packet going to 10.96.0.1:80 → randomly forward to one of:
           10.0.0.5:8080  (pod 1)
           10.0.0.6:8080  (pod 2)
           10.0.0.7:8080  (pod 3)"
  ```
- kube-proxy watches API Server for Service/Endpoint changes
- Pod dies → Endpoints updated → kube-proxy updates iptables
- New pod starts → Endpoints updated → kube-proxy adds new rule
- kube-proxy modes:
  - iptables mode (default):
    - Linux iptables rules
    - Random load balancing
    - Works fine for most clusters
  - ipvs mode (better for large clusters):
    - Linux IPVS (IP Virtual Server)
    - More efficient rule lookup
    - Better load balancing algorithms
    - Better for 1000+ services
  - eBPF mode (Cilium replaces kube-proxy entirely):
  - Highest performance
  - Bypasses iptables completely
  - Used with Cilium CNI
 
**Container Runtime — The Container Engine**
- The container runtime is the software that actually runs containers on the node.
- kubelet says: "Run this container with this image"
  - Container runtime does the actual work: Pulls the image from registry. Creates the container. Starts and stops the container. Manages container lifecycle
- Container runtimes:
  - containerd (Most common today): Lightweight, Production-grade. Default on EKS. Directly implements CRI (Container Runtime Interface)
  - CRI-O: Lightweight alternative. Built specifically for Kubernetes. Used in OpenShift
  - Docker Engine: Was used historically. Kubernetes removed Docker support in v1.24. containerd (which Docker uses internally) took over directly
  - How they relate:
    - Docker (old way): kubectl → kubelet → dockershim → Docker → containerd → container
    - containerd (current way): kubectl → kubelet → containerd → container
   
---
---

###  How a Pod Gets Scheduled — Full Request Flow ###
Full Flow — kubectl apply -f deployment.yaml

**Step 1: kubectl sends request to API Server**
- You run: kubectl apply -f deployment.yaml
- kubectl reads your kubeconfig (~/.kube/config)
- Gets API Server URL and your credentials
- Sends HTTP POST to API Server: POST /apis/apps/v1/namespaces/default/deployments

**Step 2: API Server processes the request**
- Authenticates your credentials
- Authorizes: Can you create Deployments?
- Admission controllers run
- Validates Deployment schema
- Writes Deployment object to etcd
- Returns 201 Created to kubectl

**Step 3: Deployment Controller notices**
- Deployment Controller is watching API Server
- "New Deployment appeared — default/myapp, replicas: 3"
- Creates a ReplicaSet object. Writes ReplicaSet to etcd via API Server

**Step 4: ReplicaSet Controller notices**
- ReplicaSet Controller is watching API Server
- "New ReplicaSet appeared — wants 3 pods, 0 exist"
- Creates 3 Pod objects
- Pods have no node assigned yet — status: Pending
- Writes 3 Pod objects to etcd via API Server

**Step 5: Scheduler notices**
- Scheduler watches API Server for Pending pods. Sees 3 new pods with no node assigned
- For each pod: Filters nodes (finds valid nodes). Scores valid nodes (finds best node). Picks winner
- Writes node name into pod spec:
  - pod-1 → node-1
  - pod-2 → node-2
  - pod-3 → node-1
- Updates pod objects in etcd via API Server

**Step 6: kubelet on each node notices**
- kubelet on node-1 watches API Server: "Two pods assigned to ME — pod-1 and pod-3"
- For each pod:
  - Calls CNI plugin → Sets up networking → Assigns pod IP
  - Calls container runtime (containerd): Pulls image myapp:1.0 from registry. Creates container with env vars, volumes. Starts container
  - Mounts volumes (ConfigMaps, Secrets, PVCs)
  - Starts liveness and readiness probes

**Step 7: kubelet reports status back**
- kubelet updates pod status to API Server:
  - pod-1: Running, Ready
  - pod-3: Running, Ready
- API Server writes updated status to etcd

**Step 8: Endpoints updated**
- EndpointSlice Controller sees pods are Ready
- Adds pod IPs to Service Endpoints
- kube-proxy updates iptables rules on all nodes
- Traffic can now reach the pods

---
---

###  etcd — Deep Dive ###

**How etcd Stores Kubernetes Objects**
```bash
etcd is a key-value store

Key format:
  /registry/<resource-type>/<namespace>/<name>

Examples:
  /registry/deployments/default/myapp
  /registry/pods/default/myapp-7d9f8b-xkzp2
  /registry/services/default/myapp-service
  /registry/secrets/default/db-secret
  /registry/configmaps/production/app-config
  /registry/nodes/node-1

Value:
  Full JSON/protobuf encoded Kubernetes object
```

**The Watch Mechanism — How Controllers Get Notified**
- This is the key to understanding how Kubernetes works internally
- All controllers use WATCH — Not polling
- Instead of: "Check every second if anything changed" ← Polling, Wasteful
- They do: "Tell me immediately when anything in /registry/pods changes" ← Watch, Efficient
- etcd supports long-lived watch connections
- Controller opens watch stream to API Server.
- API Server opens watch stream to etcd
- Any change → etcd notifies API Server → API Server notifies all watchers
- This is how:
  - Scheduler instantly knows about new Pending pods
  - Deployment Controller instantly knows about new Deployments
  - kubelet instantly knows about pods assigned to its node
 
**etcd Backup and Restore — Critical for On-Prem**
```bash
# backup etcd
ETCDCTL_API=3 etcdctl snapshot save backup.db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key

# verify backup
ETCDCTL_API=3 etcdctl snapshot status backup.db

# restore etcd from backup
ETCDCTL_API=3 etcdctl snapshot restore backup.db \
    --data-dir=/var/lib/etcd-restored
```

**etcd HA — Raft Consensus**
- 3-node etcd cluster:
  - Leader → Handles all writes
  - Follower → Replicates data from leader
  - Follower → Replicates data from leader
- Write request comes in:
  - Client sends write to Leader
  - Leader sends write to all Followers
  - Majority confirm (quorum) → Write committed
  - Leader responds success
- Leader fails:
  - Remaining nodes hold election. New leader elected in second. Cluster continues serving
 
---
---

### How kubectl Communicates with the Cluster ###

**kubeconfig File**
- kubectl uses a config file at `~/.kube/config` to know how to connect:
```yaml
apiVersion: v1
kind: Config

clusters:
  - name: my-eks-cluster
    cluster:
      server: https://ABC123.gr7.us-east-1.eks.amazonaws.com
      certificate-authority-data: <base64 CA cert>
      # this is the API Server URL

users:
  - name: my-aws-user
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args:
          - eks
          - get-token
          - --cluster-name
          - my-eks-cluster
        # on EKS: runs aws cli to get a token
        # token proves your IAM identity to EKS

contexts:
  - name: my-eks-cluster
    context:
      cluster: my-eks-cluster
      user: my-aws-user
      namespace: default
      # context = cluster + user + namespace

current-context: my-eks-cluster
```

**How kubectl Makes API Calls**
- Every kubectl command is just an HTTP request to the API Server:
```bash
kubectl get pods
  → GET /api/v1/namespaces/default/pods

kubectl apply -f deployment.yaml
  → POST /apis/apps/v1/namespaces/default/deployments

kubectl delete pod myapp-pod
  → DELETE /api/v1/namespaces/default/pods/myapp-pod

kubectl logs myapp-pod
  → GET /api/v1/namespaces/default/pods/myapp-pod/log

kubectl exec -it myapp-pod -- bash
  → POST /api/v1/namespaces/default/pods/myapp-pod/exec
  → WebSocket connection for interactive terminal
```

**How EKS Authentication Works**
- On EKS, kubectl authentication is tied to AWS IAM:
```bash
kubectl get pods
        ↓
kubectl runs: aws eks get-token --cluster-name my-cluster
        ↓
AWS returns a short-lived token (valid 15 minutes)
Token encodes your IAM identity
        ↓
kubectl sends request to EKS API Server with token in Authorization header
        ↓
EKS API Server validates token with AWS IAM
"Is this token valid? Who is this?"
AWS confirms: "This is arn:aws:iam::123456789:user/john"
        ↓
EKS checks aws-auth ConfigMap (or access entries):
"What Kubernetes permissions does this IAM identity have?"
        ↓
RBAC authorization:
"Can this user get pods in default namespace?"
        ↓
Request allowed → Returns pod list 
```

**kubectl useful flags for context management**
```bash
# see current context
kubectl config current-context

# list all contexts
kubectl config get-contexts

# switch context (switch clusters)
kubectl config use-context my-prod-cluster

# run one command against different cluster
kubectl get pods --context=my-staging-cluster

# set default namespace for current context
kubectl config set-context --current --namespace=myapp

# view full kubeconfig
kubectl config view

# merge multiple kubeconfigs
KUBECONFIG=~/.kube/config:~/.kube/eks-config \
    kubectl config view --flatten > ~/.kube/merged-config
```

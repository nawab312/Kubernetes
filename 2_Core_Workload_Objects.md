### Pod — The Smallest Unit ###
- A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers and gives them shared networking and storage.
```bash
Pod
├── Container 1 (main app)
├── Container 2 (sidecar — optional)
└── Shared:
      - same IP address
      - same localhost
      - same volumes
```

**Pod Lifecycle — Full Journey**
```bash
kubectl apply -f pod.yaml
        ↓
API Server accepts the Pod spec
        ↓
Scheduler finds the best node for the Pod
        ↓
kubelet on that node picks it up
        ↓
Container runtime pulls the image
        ↓
Containers start
        ↓
Pod runs
        ↓
Pod dies (crash, OOM, node failure)
        ↓
Pod is gone forever — Pods are never restarted by themselves
A controller (Deployment, ReplicaSet) creates a NEW Pod
```

**Pod Phases**
- These are the phases you see in kubectl get pods under STATUS column:
- Pending:
  - Pod is accepted by Kubernetes but not running yet
  - Reasons:
    - Scheduler is finding a node
    - Image is being pulled
    - Not enough CPU/memory on any node
    - PVC not bound yet
  - How to debug:
    - `kubectl describe pod <pod-name>`
    - Look at Events section at the bottom
- Running:
  - Pod is scheduled on a node
  - At least one container is running
  - Note:
    - Running does NOT mean your app is healthy
    - It just means the container process started
    - Your app inside might still be crashing 
- Succeeded:
  - All containers in the Pod exited with code 0
  - Pod completed its work successfully
  - Seen in: Jobs and CronJobs
  - Not seen in: long-running apps (Deployments)
- Failed:
  - All containers have stopped
  - At least one container exited with non-zero code
  - Means: Something went wrong — App crashed or errored
- Unknown:
  - Kubernetes cannot get the status of the Pod
  - Reason:
    - Usually the node is unreachable
    - kubelet on that node stopped reporting
- CrashLoopBackOff (not an official phase but very important):
  - Container keeps crashing and restarting repeatedly
  - Kubernetes adds increasing delay between restarts
    - Restart 1 → Wait 10s
    - Restart 2 → Wait 20s
    - Restart 3 → Wait 40s ... up to 5 minutes
  - Common causes:
    - App crashes on startup (config error, missing env var)
    - App runs out of memory immediately
    - Wrong command in Dockerfile
    - Database not reachable and app exits instead of retrying
  - How to debug:
    - `kubectl logs <pod-name> --previous` (Shows logs from the PREVIOUS crashed container)

**Pod Restart Policies**
- Restart policy tells Kubernetes what to do when a container inside a Pod exits.
- Always (default):
  - Container exits for ANY reason → Kubernetes restarts it
  - Even if it exited successfully (code 0)
  - Use for: Long-running apps — Web servers, APIs, Workers. Deployment, ReplicaSet, DaemonSet use this
  ```yaml
  spec:
    restartPolicy: Always
  ```
- OnFailure
  - Container exits with error (Non-zero code) → Restart
  - Container exits successfully (Code 0) → Do NOT restart
  - Use for: Batch jobs that should run once and finish. Jobs use this
- Never
  - Container exits for ANY reason → Do NOT restart
  - Pod stays in Failed or Succeeded state
  - Use for: One-Shot tasks where you want to inspect the result. Debugging scenarios

**Pod Spec — Real Example with Everything**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  restartPolicy: Always

  initContainers:
    # runs BEFORE main container starts
    # used for setup tasks — wait for DB, run migrations
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z mysql 3306; do sleep 2; done']

  containers:
    - name: myapp
      image: myapp:1.0
      ports:
        - containerPort: 8080

      # environment variables
      env:
        - name: DB_HOST
          value: mysql-service
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password

      # resource requests and limits
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"

      # health checks
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10

      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
```

**Init Containers — Important to Know**
- Init containers run before the main container starts. They must complete successfully before the main container starts.
- Use cases:
  - Wait for a database to be ready before app starts
  - Run database migrations before app starts
  - Download config files before app starts
  - Set up permissions on volumes
- Flow:
  - initContainer 1 runs → Exits 0
  - initContainer 2 runs → Exits 0
  - Main container starts
- If any init container fails → Main container never starts
- Init container is retried until it succeeds

---
---

### ReplicaSet — Keep N Pods Running ###
- A ReplicaSet ensures a specified number of Pod replicas are always running.
```bash
You say: "I want 3 replicas of myapp"
        ↓
ReplicaSet watches the cluster
        ↓
Counts pods with label app=myapp
        ↓
Always makes sure count = 3
```
- Pod dies → ReplicaSet creates new one immediately. Extra pod appears → ReplicaSet deletes it
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp        # manages pods with this label
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
```

**Why You Never Use ReplicaSet Directly**
- ReplicaSet problem — No update strategy:
  - You have ReplicaSet running myapp:1.0
  - You want to update to myapp:2.0
  - You edit the ReplicaSet image
  - What happens?
    - Existing pods are NOT updated
    - Only NEW pods (when old ones die) get the new image
    - You have no control over how the update happens
    - No Rollback mechanism
- Deployment solves all of this.
- Deployment manages ReplicaSets for you

---
---

### Deployment — The Most Used Object ###
Deployment is what you use for every stateless application. It manages ReplicaSets and gives you controlled updates and rollbacks.
```bash
Deployment
    └── manages → ReplicaSet v1 (myapp:1.0) → Pod, Pod, Pod
                  ReplicaSet v2 (myapp:2.0) → Pod, Pod, Pod  (during update)
```

**Basic Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8080
```

**Rolling Updates**
- Rolling update replaces Pods gradually so your app stays available during updates.

**Recreate Strategy**
- Kills ALL old pods first then creates new ones. Causes downtime — only use when you cannot run two versions simultaneously.

**Rollback**
- Every update creates a new ReplicaSet. Old ReplicaSets are kept for rollback.
```bash
# check rollout status
kubectl rollout status deployment/myapp-deployment

# see revision history
kubectl rollout history deployment/myapp-deployment

# REVISION  CHANGE-CAUSE
# 1         initial deployment
# 2         updated to myapp:2.0
# 3         updated to myapp:3.0

# rollback to previous version
kubectl rollout undo deployment/myapp-deployment

# rollback to specific revision
kubectl rollout undo deployment/myapp-deployment --to-revision=1
```

**Revision History**
- By default Kubernetes keeps 10 old ReplicaSets for rollback. Control this with:
```yaml
spec:
  revisionHistoryLimit: 5    # keep only 5 old ReplicaSets
```

---
---

### StatefulSet — For Stateful Applications ###
- StatefulSet is like a Deployment but for applications that need stable identity and persistent storage — Databases, Message queues, Distributed systems.

**Key Differences from Deployment**
- Deployment:
  - Pods are Identical and Interchangeable
  - Pod names are random: myapp-7d9f8b-xkzp2
  - Pods share one PVC or use no storage
  - All pods start and stop simultaneously
  - Good for: Stateless apps — Web servers, APIs
- StatefulSet:
  - Pods have stable, predictable names: mysql-0, mysql-1, mysql-2
  - Each Pod gets its OWN PVC (own storage)
  - Pods start in order: 0, then 1, then 2
  - Pods stop in reverse order: 2, then 1, then 0
  - Good for: Databases, Kafka, Zookeeper, Elasticsearch
 
**Why Order Matters for Databases**
- MySQL cluster with Primary and Replicas:
  - mysql-0 starts first → Becomes PRIMARY
  - mysql-1 starts after → Connects to mysql-0 to replicate
  - mysql-2 starts after → Connects to mysql-0 to replicate
  - If they all started randomly at the same time: No clear primary. Replicas don't know who to connect to
 
**StatefulSet Example — MySQL**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql        # must match headless service name
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql    # each pod mounts its OWN volume

  # each pod gets its own PVC automatically
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp2             # AWS EBS
        resources:
          requests:
            storage: 20Gi
```
- This creates:
  - mysql-0 → PVC: mysql-data-mysql-0 → EBS volume (20Gi)
  - mysql-1 → PVC: mysql-data-mysql-1 → EBS volume (20Gi)
  - mysql-2 → PVC: mysql-data-mysql-2 → EBS volume (20Gi)
  - Pod restarts → Same PVC reattaches → Data preserved
  - Pod reschedules to different node → Same PVC reattaches

**



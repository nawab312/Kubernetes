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
```bash
Pod is accepted by Kubernetes but not running yet

Reasons:
  - Scheduler is finding a node
  - Image is being pulled
  - Not enough CPU/memory on any node
  - PVC not bound yet

How to debug:
  kubectl describe pod <pod-name>
  Look at Events section at the bottom
```
- Running:
```bash
Pod is scheduled on a node
At least one container is running

Note:
  Running does NOT mean your app is healthy
  It just means the container process started
  Your app inside might still be crashing 
```
- Succeeded:
```bash
All containers in the Pod exited with code 0
Pod completed its work successfully

Seen in: Jobs and CronJobs
Not seen in: long-running apps (Deployments)
```
- Failed:
```bash
All containers have stopped
At least one container exited with non-zero code

Means: something went wrong — app crashed or errored
```
- Unknown:
```bash
Kubernetes cannot get the status of the Pod

Reason:
  Usually the node is unreachable
  kubelet on that node stopped reporting
```
- CrashLoopBackOff (not an official phase but very important):
```bash
Container keeps crashing and restarting repeatedly
Kubernetes adds increasing delay between restarts
  restart 1 → wait 10s
  restart 2 → wait 20s
  restart 3 → wait 40s ... up to 5 minutes

Common causes:
  - App crashes on startup (config error, missing env var)
  - App runs out of memory immediately
  - Wrong command in Dockerfile
  - Database not reachable and app exits instead of retrying

How to debug:
  kubectl logs <pod-name> --previous
  (shows logs from the PREVIOUS crashed container)
```

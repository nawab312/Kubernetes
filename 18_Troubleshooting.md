**The Golden Rule of Kubernetes Troubleshooting**
- Before diving in, always follow this order:
```bash
Step 1: kubectl get pods          → What is the current state?
Step 2: kubectl describe pod      → What events happened?
Step 3: kubectl logs              → What did the app say?
Step 4: kubectl exec              → Can I get inside and check?
Step 5: kubectl get events        → What is happening cluster-wide?
```

### Pod Stuck in Pending ###
Pending means Kubernetes accepted the pod but has not scheduled it onto a node yet. The pod has not started. No container is running. Something is preventing scheduling.

**Always Start Here**
```bash
kubectl describe pod myapp-pod

# scroll to the bottom — look at Events section
# this tells you EXACTLY why it is pending
```

**Cause 1 — Not Enough Resources on Any Node***
```bash
Events:
Warning FailedScheduling 0/3 nodes are available: 3 Insufficient cpu.
```
- What happened:
  - Pod requests cpu: 4000m (4 cores)
  - Every node only has 2 cores available
  - No node can fit this pod
  - Pod stays Pending forever
- How to verify:
  ```bash
  kubectl describe nodes | grep -A5 "Allocated resources"
  
  Node 1: CPU 1800m/2000m used   (90% full)
  Node 2: CPU 1900m/2000m used   (95% full)
  Node 3: CPU 2000m/2000m used   (100% full)
  ```
- How to fix:
  - Option 1 — Reduce pod resource requests. Edit deployment and lower cpu/memory requests
  - Option 2 — Add more nodes to cluster. On EKS — Increase node group desired capacity
  ```bash
  aws autoscaling set-desired-capacity \
      --auto-scaling-group-name my-node-group \
      --desired-capacity 5
  ```
  - Option 3 — Check if any pods are wasting resources
  ```bash
  kubectl top pods --all-namespaces
  # find pods using much less than they requested
  # reduce their requests to free up space
  ```

**Cause 2 — No Nodes Match the NodeSelector or Affinity**
```bash
Events:
Warning FailedScheduling 0/3 nodes are available:
3 node(s) didn't match Pod's node affinity/selector.
```
- What happened:
  - Pod has nodeSelector: disktype=ssd
  - No nodes have this label
- How to fix:
  ```bash
  # Check what labels your nodes have
  kubectl get nodes --show-labels
  
  # Check what your pod is asking for
  kubectl describe pod myapp-pod | grep -A5 "Node-Selectors"
  ```
  - Option 1 — Add label to a node
  ```bash
  kubectl label node node-1 disktype=ssd
  ```
  - Option 2 — Fix the nodeSelector in your deployment. Remove it or change to a label that exists
 
**Cause 3 — Taints and Tolerations Mismatch**
```bash
Events:
Warning  FailedScheduling  0/3 nodes are available:
3 node(s) had untolerated taint {dedicated: gpu}
```
- What happened:
  - All nodes have taint: dedicated=gpu:NoSchedule
  - Pod has no toleration for this taint
- How to fix:
  ```bash
  # Check node taints
  kubectl describe node node-1 | grep Taints
  ```
  - Option 1 — Add toleration to your pod
  ```yaml
  spec:
    tolerations:
      - key: dedicated
        value: gpu
        effect: NoSchedule
  ```
  - Option 2 — Remove taint from nod
  ```bash
  kubectl taint node node-1 dedicated=gpu:NoSchedule-
  ```

  **Cause 4 — PVC Not Bound**
  ```bash
  Events:
  Warning  FailedScheduling  0/3 nodes are available:
  pod has unbound immediate PersistentVolumeClaims
  ```
  - What happened:
    - Pod needs a PVC
    - PVC is in Pending state — No PV available to bind to
    - Pod cannot start without its storage
  - How to fix:
  ```bash
  # check PVC status
  kubectl get pvc
  
  NAME          STATUS    VOLUME   CAPACITY   STORAGECLASS
  myapp-data    Pending                       gp2          ← stuck
  
  # describe PVC to see why
  kubectl describe pvc myapp-data
  
  # Common reasons:
  # 1. StorageClass does not exist
  kubectl get storageclass
  
  # 2. On EKS — EBS CSI driver not installed
  kubectl get pods -n kube-system | grep ebs-csi
  
  # 3. No PV matches the PVC request (static provisioning)
  kubectl get pv
  ```

**Cause 5 — ImagePullBackOff During Scheduling**
- Sometimes pod is briefly Pending while image is being pulled. If it stays Pending, check image issues.

---
---

### Pod Stuck in CrashLoopBackOff ###
CrashLoopBackOff means the container keeps crashing and Kubernetes keeps restarting it with increasing delays between restarts.
```bash
Restart 1 → wait 10s
Restart 2 → wait 20s
Restart 3 → wait 40s
Restart 4 → wait 80s
...up to 5 minutes between restarts
```

**Always Start Here**
```bash
# current logs (if container is running right now)
kubectl logs myapp-pod

# PREVIOUS container logs (most useful — shows why it crashed)
kubectl logs myapp-pod --previous

# if multiple containers in pod
kubectl logs myapp-pod -c container-name --previous
```

**Cause 1 — Application Error on Startup**
```bash
Logs:
  Error: Cannot find module '/app/index.js'
  Error: ENOENT no such file or directory

  or

  Exception in thread "main" java.lang.ClassNotFoundException
  Could not find or load main class com.myapp.Main
```
- What happened:
  - Wrong WORKDIR in Dockerfile
  - Wrong CMD or ENTRYPOINT
  - Missing files in the image
  - Wrong jar name
- How to fix:
  ```bash
  # get inside the container when it is briefly running
  kubectl exec -it myapp-pod -- sh
  
  # or run the image locally to debug
  docker run -it myapp:1.0 sh
  ls /app    # check if files exist
  ```

**Cause 2 — Missing Environment Variable or Config**
```bash
Logs:
  Error: DATABASE_URL is not defined
  Fatal: Required environment variable DB_HOST missing
  panic: runtime error: invalid memory address
         (nil pointer — config not loaded)
```
- How to fix:
  ```bash
  # check what env vars the pod has
  kubectl exec myapp-pod -- env | grep DB
  
  # check if secret or configmap exists
  kubectl get secret db-secret
  kubectl get configmap app-config
  
  # check if they are correctly mounted
  kubectl describe pod myapp-pod | grep -A20 "Environment"
  ```

**Cause 3 — Cannot Connect to Database on Startup**
```bash
Logs:
  Error: connect ECONNREFUSED 10.96.0.5:5432
  Fatal: database connection failed after 3 retries — exiting
```
- What happened:
  - App tries to connect to DB on startup
  - DB not ready yet or wrong hostname
  - App exits instead of retrying → CrashLoopBackOff
- Best fix: App should retry with backoff, not exit OR use an init container to wait for DB
  ```yaml
  # init container waits for DB before app starts
  initContainers:
    - name: wait-for-db
      image: busybox
      command:
        - sh
        - -c
        - |
          until nc -z postgres-service 5432; do
            echo "waiting for database..."
            sleep 2
          done
          echo "database is ready"
  ```

**Cause 4 — OOMKilled Immediately**
```bash
kubectl describe pod myapp-pod

# Last State:
#   Reason: OOMKilled
#   Exit Code: 137
```
- What happened:
  - Memory limit too low
  - App exceeds limit on startup → kernel kills it → Restarts → Repeat
- How to fix:
  ```bash
  # check actual memory usage
  kubectl top pod myapp-pod
  
  # increase memory limit in deployment
  resources:
    limits:
      memory: "1Gi"    # increase from 256Mi
  ```

**Cause 5 — Wrong Command or Entrypoint**
```bash
Logs:
  (empty — container exits immediately with code 0 or 1)
```
- What happened:
  - CMD in Dockerfile is wrong
  - Script exits immediately
  - Container has nothing to do → Exits → Restart loop
- How to fix:
  ```bash
  # check what command the container is running
  kubectl describe pod myapp-pod | grep -A3 "Command"
  
  # run interactively to debug
  kubectl run debug --image=myapp:1.0 -it --rm -- sh
  # manually run your start command and see what happens
  ```

---
---

### Pod Stuck in ImagePullBackOff ###
ImagePullBackOff means Kubernetes cannot pull the container image from the registry.

**Always Start Here**
```bash
kubectl describe pod myapp-pod

# Events:
#   Warning  Failed     Failed to pull image "myapp:latest":
#            rpc error: code = Unknown
#            desc = failed to pull and unpack image:
#            401 Unauthorized
```

**Cause 1 — Image Does Not Exist**
```bash
Events:
  Failed to pull image "myorg/myapp:v2.5":
  Repository does not exist or may require authorization
```
- What happened:
  - Wrong image name or tag in deployment
  - Tag was never pushed to registry
  - Typo in image name
- How to fix:
  ```bash
  # check what image the pod is trying to pull
  kubectl describe pod myapp-pod | grep Image
  
  # verify image exists in registry
  aws ecr describe-images \
      --repository-name myapp \
      --region us-east-1
  
  # fix image tag in deployment
  kubectl set image deployment/myapp \
      myapp=myorg/myapp:v2.4    # correct tag
  ```

**Cause 2 — Private Registry — No Pull Secret**
```bash
Events:
  Failed to pull image: 401 Unauthorized
  or
  Failed to pull image: no basic auth credentials
```
- What happened:
  - Image is in a private registry
  - Pod has no credentials to pull from it
- How to fix:
  ```bash
  # create docker registry secret
  kubectl create secret docker-registry regcred \
      --docker-server=myregistry.com \
      --docker-username=myuser \
      --docker-password=mypassword \
      --docker-email=me@company.com
  
  # add imagePullSecrets to deployment
  spec:
    imagePullSecrets:
      - name: regcred
    containers:
      - name: myapp
        image: myregistry.com/myapp:1.0
  ```

**Cause 3 — ECR on EKS — IAM Permission Missing**
- On EKS pulling from ECR, no pull secret is needed — but the node IAM role needs ECR permissions.
```bash
Events:
  Failed to pull image: 403 Forbidden
  no basic auth credentials
```
- What happened:
  - Node IAM role missing ECR pull permissions
  - Nodes cannot authenticate to ECR
- How to fix:
  ```bash
  # check node IAM role
  aws iam list-attached-role-policies \
      --role-name eks-node-role
  
  # attach ECR read policy
  aws iam attach-role-policy \
      --role-name eks-node-role \
      --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
  ```

**Cause 4 — Image Too Large / Node Disk Full**
```bash
Events:
  Failed to pull image: write /var/lib/docker/overlay2/...:
  no space left on device
```
- How to fix:
  ```bash
  # check node disk usage
  kubectl describe node node-1 | grep -A5 "Conditions"
  # look for DiskPressure condition
  
  # clean up unused images on nodes
  # use a DaemonSet with image cleanup job
  # or add larger disk to nodes in EKS node group
  ```

---
---

### OOMKilled — Out of Memory ###
OOMKilled means the Linux kernel killed your container because it exceeded its memory limit.

**How to Identify**
```bash
kubectl describe pod myapp-pod

# Last State:
#   Terminated
#     Reason:    OOMKilled     ← clear signal
#     Exit Code: 137           ← 128 + 9 (SIGKILL)
#     Started:   Mon, 09 Mar 2026 10:00:00
#     Finished:  Mon, 09 Mar 2026 10:02:34
```

**Three Scenarios**
- Scenario 1 — Memory limit too low
  - App genuinely needs 1GB. Memory limit set to 256Mi. App killed every time it reaches 256Mi
  - Fix: increase memory limit
  ```yaml
  resources:
    limits:
      memory: "1Gi"
  ```
- Scenario 2 — Memory leak in application
  - App starts at 200MB. Grows slowly over hours. Hits limit → OOMKilled → Restarts → Grows again → Repeat
  - How to identify: `kubectl top pod myapp-pod --watch`. Watch memory grow over time
  - Fix:
    - Short term: Increase limit to buy time
    - Long term: Fix memory leak in application code
- Scenario 3 — Node memory pressure — no limit se
  - Pod has NO memory limit set. Uses as much as it wants
  - Node runs out of memory. OOM killer picks a process to kill
  - Often kills YOUR pod
  - Fix: Always set memory requests AND limits
 
**How to Right-size Memory**
```bash
# watch live memory usage
kubectl top pod myapp-pod

# check historical usage in CloudWatch (EKS)
# or Prometheus query:
container_memory_working_set_bytes{
    pod="myapp-pod",
    container="myapp"
}

# Rule of thumb:
# requests = average usage
# limits   = peak usage × 1.5
# gives headroom without being wasteful
```

---
---

### Service Not Reachable — Debugging Network Step by Step ###
This is one of the most complex troubleshooting scenarios. Follow these steps in order.

**Step 1 — Are the Pods actually running?**
```bash
kubectl get pods -l app=myapp

# if pods are not running → fix pod issues first
# service cannot route to dead pods
```

**Step 2 — Does the Service exist and have correct selector?**
```bash
kubectl get service myapp-service

kubectl describe service myapp-service

# look at:
#   Selector:  app=myapp     ← must match pod labels
#   Endpoints: 10.0.0.5:8080, 10.0.0.6:8080  ← must not be empty
```

**Step 3 — Are Endpoints populated?**
```bash
kubectl get endpoints myapp-service

# Good:
# NAME            ENDPOINTS                     AGE
# myapp-service   10.0.0.5:8080,10.0.0.6:8080  5d

# Bad:
# NAME            ENDPOINTS   AGE
# myapp-service   <none>      5d   ← no pods matched the selector
```
- If endpoints are empty:
```bash
# check pod labels
kubectl get pods --show-labels | grep myapp

# check service selector
kubectl describe service myapp-service | grep Selector

# they must match exactly
# pod label:        app=myapp
# service selector: app=myapp  

# pod label:        app=my-app    (hyphen)
# service selector: app=myapp    (no hyphen)  mismatch
```

**Step 4 — Can you reach the Service from inside the cluster?**
```bash
# spin up a debug pod
kubectl run debug \
    --image=nicolaka/netshoot \
    -it --rm -- bash

# test DNS resolution
nslookup myapp-service
nslookup myapp-service.default.svc.cluster.local

# test HTTP connection
curl http://myapp-service:80/health
curl http://myapp-service.default.svc.cluster.local:80

# test TCP connection
nc -zv myapp-service 80
```

**Step 5 — Can you reach the Pod directly (bypass Service)?**
```bash
# get pod IP
kubectl get pod myapp-pod -o wide
# NAME        READY  STATUS   IP
# myapp-pod   1/1    Running  10.0.0.5

# from debug pod — hit pod IP directly
curl http://10.0.0.5:8080/health

# if pod IP works but service does not →
# problem is with service selector or kube-proxy
```

**Step 6 — Check if app is listening on correct port**
```bash
# get inside the pod
kubectl exec -it myapp-pod -- sh

# check what ports are listening
netstat -tlnp
# or
ss -tlnp

# app should be listening on the containerPort
# tcp 0.0.0.0:8080    ← correct 
# tcp 0.0.0.0:3000    ← but service targets 8080  mismatch
```

**Step 7 — Check Network Policy**
```bash
# check if any network policy is blocking traffic
kubectl get networkpolicy -n default

kubectl describe networkpolicy myapp-policy

# check if ingress rules allow traffic from
# the source namespace/pod to your service
```

**Service Debugging Flowchart**
```bash
Service not reachable
      ↓
Pods running?               → No → Fix pod issues first
      ↓
Endpoints populated?        → No → Fix selector label mismatch
      ↓
DNS resolves?               → No → CoreDNS issue. kubectl get pods -n kube-system                                  
      ↓
Pod IP reachable directly?  → No → CNI issue or pod not listening
      ↓
Service IP works?           → No → kube-proxy issue
      ↓
Network policy blocking?    → Yes → Update network policy rules
```

---
---

### Node NotReady ###
Node NotReady means the kubelet on that node stopped reporting to the control plane. Pods on that node may be evicted or stuck.

**Always Start Here**
```bash
kubectl get nodes

# NAME     STATUS     ROLES    AGE
# node-1   Ready      worker   30d
# node-2   NotReady   worker   30d   ← Problem here
# node-3   Ready      worker   30d

kubectl describe node node-2

# look at Conditions section:
# Type              Status
# MemoryPressure    False
# DiskPressure      True    ← Disk is full 
# PIDPressure       False
# Ready             False   ← Node is not ready
```

**Cause 1 — Disk Pressure**
- What happened:
  - Node disk is full
  - Kubelet cannot write logs or pull images
  - Node marks itself NotReady
- Common cause on EKS:
  - Too many container images cached on node
  - Log files filling up /var/log
  - Large emptyDir volumes
- How to fix:
  ```bash
  # SSH into the node (EKS)
  # find and remove unused docker images
  docker image prune -a
  
  # check disk usage
  df -h
  du -sh /var/log/containers/*  | sort -rh | head -20
  
  # long term: add larger EBS volume to node group
  # or configure log rotation
  ```

**Cause 2 — Memory Pressure**
- What happened:
  - Node is running out of memory
  - Kubelet starts evicting pods to free memory
  - Eventually marks node NotReady
- How to fix:
  ```bash
  # check memory usage on node
  kubectl top node node-2
  
  # find which pods are using most memory
  kubectl top pods --all-namespaces \
      --field-selector spec.nodeName=node-2 \
      --sort-by=memory
  
  # evict or delete the biggest memory consumers
  # or add more nodes and redistribute pods
  ```

**Cause 3 — Kubelet Stopped / Crashed**
```bash
Conditions:
  Ready: False
  Reason: KubeletNotReady
  Message: kubelet stopped posting node status
```
- What happened:
  - kubelet process crashed on the node
  - Node cannot communicate with control plane
- How to fix:
  ```bash
  # SSH into the node
  # check kubelet status
  sudo systemctl status kubelet
  
  # check kubelet logs
  sudo journalctl -u kubelet -n 100 --no-pager
  
  # restart kubelet
  sudo systemctl restart kubelet
  
  # on EKS — if node is unrecoverable, terminate it
  # auto scaling group will replace it automatically
  aws ec2 terminate-instances --instance-ids i-1234567890
  ```

**Cause 4 — Node Network Issue**
- Node is running fine but cannot reach the API server
- Heartbeats lost → Control plane marks node NotReady
- How to fix:
  - Check security groups on EKS node. Node must be able to reach API server on port 443
  - Check VPC routing. Check if node ENI is healthy in AWS console
 
**What Happens to Pods on NotReady Node**
```bash
Node goes NotReady
        ↓
Kubernetes waits 5 minutes (pod-eviction-timeout)
        ↓
Pods on that node marked as Unknown/Terminating
        ↓
New pods scheduled on healthy nodes
        ↓
Old pods stuck in Terminating on dead node (cannot be cleaned up because node is unreachable)
        ↓
Node comes back → Pods cleaned up automatically 
Node never comes back → Force delete stuck pods -> kubectl delete pod myapp-pod --force --grace-period=0 (acceptable here because node is confirmed dead)
```

---
---

### kubectl Commands for Debugging ###

**kubectl describe**
- The most useful command. Shows full object state and events.
```bash
# describe a pod — shows events, state, probe results
kubectl describe pod myapp-pod

# describe a node — shows conditions, capacity, allocated resources
kubectl describe node node-1

# describe a service — shows selector, endpoints, ports
kubectl describe service myapp-service

# describe a deployment — shows rollout status, conditions
kubectl describe deployment myapp-deployment
```

**kubectl logs**
```bash
# current logs
kubectl logs myapp-pod

# previous container logs (after crash)
kubectl logs myapp-pod --previous

# specific container in multi-container pod
kubectl logs myapp-pod -c sidecar-container

# follow logs in real time
kubectl logs myapp-pod -f

# last 100 lines only
kubectl logs myapp-pod --tail=100

# logs since last 1 hour
kubectl logs myapp-pod --since=1h

# all pods matching a label
kubectl logs -l app=myapp --all-containers
```

**kubectl exec**
```bash
# open interactive shell inside pod
kubectl exec -it myapp-pod -- bash
kubectl exec -it myapp-pod -- sh    # if bash not available

# run a single command
kubectl exec myapp-pod -- env
kubectl exec myapp-pod -- cat /etc/config/app.conf
kubectl exec myapp-pod -- ps aux
kubectl exec myapp-pod -- netstat -tlnp

# specific container in multi-container pod
kubectl exec -it myapp-pod -c sidecar -- sh
```

**kubectl get with useful flags**
```bash
# show pod IPs and which node they run on
kubectl get pods -o wide

# watch pods in real time
kubectl get pods -w

# show all pods across all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A              # shorthand

# get pods with specific label
kubectl get pods -l app=myapp

# get output as yaml (see full spec)
kubectl get pod myapp-pod -o yaml

# get specific field using jsonpath
kubectl get pod myapp-pod \
    -o jsonpath='{.status.podIP}'

# sort pods by restart count
kubectl get pods \
    --sort-by='.status.containerStatuses[0].restartCount'
```

**kubectl get events**
```bash
# all events in namespace sorted by time
kubectl get events \
    --sort-by='.lastTimestamp'

# watch events in real time
kubectl get events -w

# events for specific pod
kubectl get events \
    --field-selector involvedObject.name=myapp-pod

# events across all namespaces
kubectl get events -A \
    --sort-by='.lastTimestamp'

# only warning events
kubectl get events \
    --field-selector type=Warning
```

**kubectl top — Resource Usage**
```bash
# pod CPU and memory usage
kubectl top pods

# all namespaces
kubectl top pods -A

# sort by memory
kubectl top pods --sort-by=memory

# node CPU and memory usage
kubectl top nodes
```

**kubectl rollout — Deployment Management**
```bash
# check rollout status
kubectl rollout status deployment/myapp

# see revision history
kubectl rollout history deployment/myapp

# rollback to previous version
kubectl rollout undo deployment/myapp

# rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=3

# pause a rollout (stop mid-update)
kubectl rollout pause deployment/myapp

# resume a paused rollout
kubectl rollout resume deployment/myapp
```

**kubectl debug — Advanced Debugging**
```bash
# create a debug container alongside running pod
# useful when main container has no shell (distroless images)
kubectl debug -it myapp-pod \
    --image=nicolaka/netshoot \
    --target=myapp-container

# debug a node directly
kubectl debug node/node-1 \
    -it --image=ubuntu
```

**Why This Topic Matters**
- Without health checks and reliability configs:
  - App crashes internally but container is still running -> Kubernetes keeps sending traffic to broken pod
  - New pod starts but app not ready yet -> Kubernetes sends traffic before app is ready
  - Rolling update kills too many pods at once -> Users see downtime during deploy
  - Pod gets killed mid-request -> Users get 502 errors
- With proper health checks and reliability:
  - Broken pod detected → Removed from traffic immediately
  - Pod not ready → No traffic until it is
  - Rolling update → Zero downtime
  - Pod shutdown → All requests finish before pod dies
 
### Liveness Probe ###
Liveness probe answers the question:
- Is this container alive and functioning?
- Or is it stuck/deadlocked and needs a restart?
- If liveness probe fails → Kubernetes restarts the container.

**The Problem it Solves**
- Your Java app starts fine. Runs for 6 hours. Gets into a deadlock — All threads stuck. Container is still RUNNING — Process has not crashed. Kubernetes has no idea anything is wrong. Users get no response — Requests hang forever
- Without liveness probe → Pod stays broken until someone notices
- With liveness probe → Kubernetes detects deadlock → Restarts pod

**Three Types of Liveness Probes**
- HTTP GET — Most common
  - Kubernetes makes an HTTP GET request to your app. Any response 200-399 = Healthy. Anything else = Unhealthy.
  ```yaml
  livenessProbe:
    httpGet:
      path: /health/live     # endpoint your app exposes
      port: 8080
    initialDelaySeconds: 30  # wait 30s before first probe
                             # gives app time to start up
    periodSeconds: 10        # check every 10 seconds
    failureThreshold: 3      # fail 3 times in a row before restart
    successThreshold: 1      # 1 success = healthy
    timeoutSeconds: 5        # probe times out after 5s
  ```
  - What your /health/live endpoint should return:
  ```js
  // Node.js example
  app.get('/health/live', (req, res) => {
      // check if app is functioning
      // NOT checking DB or external services here
      // just: is THIS app alive and not deadlocked?
  
      res.status(200).json({
          status: 'alive',
          uptime: process.uptime()
      });
  });
  ```
- TCP Socket
  - Kubernetes tries to open a TCP connection to a port. If connection succeeds = Healthy.
  ```yaml
  livenessProbe:
    tcpSocket:
      port: 3306             # just checks if port is open
    initialDelaySeconds: 15
    periodSeconds: 10
  ```
  - Good for: Databases, Message queues, any TCP service where HTTP is not available
- Exec Command
  - Kubernetes runs a command inside the container. Exit code 0 = Healthy. Anything else = unhealthy.
  ```yaml
  livenessProbe:
    exec:
      command:
        - sh
        - -c
        - "redis-cli ping | grep PONG"   # redis-specific health check
    initialDelaySeconds: 10
    periodSeconds: 10
  ```
  - Good for: Apps that don't expose HTTP. Custom health check scripts. Checking internal state files
 
***Liveness Probe Timing — Getting it Right**
- initialDelaySeconds: Very important to get right
  - Too low:
    - App needs 60s to start
    - Probe starts at 5s
    - App not ready → Probe fails → Kubernetes restarts
    - App never gets a chance to start → Restart loop
  - Too high:
    - App is deadlocked at minute 2
    - Probe does not start until minute 5
    - 3 minutes of broken pod serving traffic
  - Right value:
    - Measure your app startup time
    - Set initialDelaySeconds = Startup time + Buffer
    - Or better: Use startupProbe

**What Liveness Probe Should NOT Check**
- Do NOT check database connectivity in liveness probe
- Do NOT check external API availability
- Do NOT check Redis connection
- Why?
  - Database goes down temporarily. Liveness probe fails
  - Kubernetes restarts ALL your pods
  - Pods come back → DB still down → All pods restart again
  - Infinite restart loop
  - This takes down your entire app because of a DB blip

- *Liveness probe should ONLY check if THIS app process is healthy*
- *Is the app deadlocked? Is it responsive? That is all.*

---
---

### Readiness Probe ###
Readiness probe answers the question:
- Is this container ready to serve traffic?
- Should Kubernetes send requests to this pod?"
- If readiness probe fails → Kubernetes removes pod from Service endpoints (no traffic). Container is NOT restarted.

**The Problem it Solves**
- New pod starts during rolling update. App process starts in 2 seconds
- But app needs 30 seconds to:
  - Load ML model into memory
  - Warm up cache
  - Build connection pool to DB
- Without readiness probe:
  - Pod starts → Kubernetes immediately sends traffic
  - App not warmed up → Slow responses or errors
- With readiness probe:
  - With readiness probe:
    - Pod starts → Probe fails (app not warm yet)
    - Kubernetes keeps pod out of Service endpoints
    - After 30 seconds → App warmed up → Probe passes
    - Kubernetes adds pod to Service → Traffic flows
   
**Readiness Probe Example**
```yaml
readinessProbe:
  httpGet:
    path: /health/ready    # different endpoint from liveness
    port: 8080
  initialDelaySeconds: 5   # start checking quickly
  periodSeconds: 5         # check every 5 seconds
  failureThreshold: 3      # 3 failures = remove from endpoints
  successThreshold: 1      # 1 success = add back to endpoints
```
- What your /health/ready endpoint should check:
```js
app.get('/health/ready', async (req, res) => {
    try {
        // DO check dependencies here — is app ready to serve?
        await db.query('SELECT 1');           // DB reachable?
        await redis.ping();                   // cache reachable?

        const cacheLoaded = cache.isWarmedUp();  // cache loaded?

        if (!cacheLoaded) {
            return res.status(503).json({
                status: 'not ready',
                reason: 'cache not warmed up'
            });
        }

        res.status(200).json({ status: 'ready' });

    } catch (err) {
        res.status(503).json({
            status: 'not ready',
            reason: err.message
        });
    }
});
```

**Readiness During Ongoing Operation**
- Readiness probe runs throughout the pod's entire lifetime — not just at startup.
```bash
Pod running fine for 2 hours
Database goes down temporarily
        ↓
Readiness probe hits /health/ready
DB check fails → 503
        ↓
Kubernetes removes pod from Service endpoints
No traffic sent to this pod
        ↓
DB recovers
Readiness probe passes again
        ↓
Kubernetes adds pod back to Service endpoints

Pod was never restarted — just temporarily removed from traffic
This is the correct behaviour
```

**Liveness + Readiness Together — Full Example**
```yaml
containers:
  - name: myapp
    image: myapp:1.0

    livenessProbe:
      httpGet:
        path: /health/live    # is app alive? (simple check)
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /health/ready   # is app ready? (deeper check)
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```
- /health/live → Simple check, just return 200 if process is running
- /health/ready → Deeper check, verify DB, Cache, Dependencies

---
---

### Startup Probe ###
Has the application finished starting up yet?
- While startup probe is running, liveness and readiness probes are disabled. This prevents liveness probe from restarting a slow-starting app.

**The Problem it Solves**
- Legacy Java app takes 3 minutes to start(loading Spring context, building caches, warming up)
- Without startup probe:
  - Set initialDelaySeconds: 180 on liveness probe
    - App crashes at minute 4 → liveness probe does not detect it until minute 4 + (failureThreshold × periodSeconds)
    - Very slow to detect real failures. OR
  - Set initialDelaySeconds: 30 → Probe starts too early
    - App still starting → Probe fails → Restart → Never starts
- With startup probe:
  - Startup probe runs during startup phase
  - Liveness probe disabled until startup probe passes
  - Startup probe gives app up to 5 min to start (generous)
  - After startup complete → liveness probe takes over (strict)
 
**Startup Probe Example**
```yaml
containers:
  - name: legacy-java-app
    image: legacyapp:1.0

    startupProbe:
      httpGet:
        path: /health/live
        port: 8080
      failureThreshold: 30    # allow up to 30 failures
      periodSeconds: 10       # check every 10 seconds
                              # 30 × 10 = 300 seconds = 5 minutes max startup time

    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      periodSeconds: 10
      failureThreshold: 3     # strict — 3 failures = restart

    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 3
```

**All Three Probes Together — Timeline**
```bash
Pod starts
│
├── 0s    Startup probe begins (liveness + readiness disabled) checking every 10s
│          
│
├── 120s  App finishes starting → startup probe passes. Startup probe stops
│          
│
├── 120s  Liveness + Readiness probes both activate
│
├── 125s  Readiness passes → pod added to Service endpoints. Traffic flows in 
│          
│
├── ...   App runs normally. Liveness checks every 10s. Readiness checks every 5s
│        
│          
│
├── 6h    App deadlocks. Liveness probe fails 3 times. Kubernetes restarts container
│          
│          
│
└── 6h+  Container restarts, startup probe runs again
```

---
---

### Pod Disruption Budgets ###
PDB answers the question: During voluntary disruptions, how many pods of this app can be taken down at the same time?

**Voluntary vs Involuntary Disruptions**
- Voluntary (PDB protects against these):
  - Node drain for maintenance    → kubectl drain node
  - Cluster upgrade               → Nodes drained one by one
  - Scaling down node group       → Nodes removed
  - Admin manually deletes pod    → kubectl delete pod
- Involuntary (PDB cannot help):
  - Node hardware failure         → Sudden loss
  - OOM kill on node              → kernel kills process
  - Kernel panic                  → Node crashes
 
**PDB with minAvailable**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2          # at least 2 pods must always be running
  selector:
    matchLabels:
      app: myapp
```
- Can also use percentage: `minAvailable: "75%"`

**PDB with maxUnavailable**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  maxUnavailable: 1        # at most 1 pod can be unavailable
  selector:
    matchLabels:
      app: myapp
```

**PDB for Critical Systems — Zero Disruption**
```yaml
spec:
  minAvailable: "100%"     # no pods can be evicted voluntarily

# This means:
# kubectl drain will be blocked
# Cluster upgrade will be stuck
# Use carefully — only for truly critical systems
# You must manually handle disruptions
```

**Check PDB Status**
```bash
kubectl get pdb

NAME         MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
myapp-pdb    2               N/A               1

# ALLOWED DISRUPTIONS = 1
# means right now, 1 pod can be safely evicted
```

---
---

### Graceful Shutdown ###
Graceful shutdown ensures your app finishes handling current requests before the pod is killed.

**Full Termination Flow — What Actually Happens**
```bash
kubectl delete pod myapp (or rolling update evicts pod)
        ↓
Two things happen SIMULTANEOUSLY:

Thing 1: Pod removed from Service endpoints
kube-proxy updates iptables
No NEW requests sent to this pod (takes a few seconds to propagate)

Thing 2: preStop hook runs (if configured)
        ↓
preStop hook finishes
        ↓
SIGTERM sent to container (PID 1)
        ↓
terminationGracePeriodSeconds countdown starts (30s default)
        ↓
App handles SIGTERM — finishes in-flight requests, closes connections
        ↓
App exits cleanly (exit 0)
        ↓
Pod removed 

OR

30 seconds pass and app has NOT exited
        ↓
Kubernetes sends SIGKILL — immediate death 
```

**The Timing Problem — Why preStop Sleep Matters**
- Problem:
  - Pod removed from endpoints (Thing 1) takes 2-5 seconds to propagate
  - SIGTERM sent immediately (Thing 2)
  - App starts shutting down immediately
  - But load balancer still sending requests for 2-5 seconds
  - Those requests hit a shutting-down app → Errors
- Solution: `preStop` sleep
  - preStop hook: sleep 5
  - This delays SIGTERM by 5 seconds
  - Gives kube-proxy time to update iptables first
  - By the time SIGTERM arrives, no new requests coming in
 
**preStop Hook — The Right Way**
```yaml
spec:
  terminationGracePeriodSeconds: 60    # total time allowed

  containers:
    - name: myapp
      image: myapp:1.0

      lifecycle:
        preStop:
          exec:
            command:
              - sh
              - -c
              - |
                sleep 5
                # wait for load balancer to stop sending traffic
                # then app receives SIGTERM and starts cleanup
```

**Full Graceful Shutdown Configuration**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      # total time Kubernetes waits before SIGKILL
      terminationGracePeriodSeconds: 60

      containers:
        - name: myapp
          image: myapp:1.0

          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 5
            failureThreshold: 3
```
- App code handling SIGTERM:
```js
let isShuttingDown = false;
const activeRequests = new Set();

// track active requests
app.use((req, res, next) => {
    activeRequests.add(req);
    res.on('finish', () => activeRequests.delete(req));
    next();
});

// readiness endpoint
app.get('/health/ready', (req, res) => {
    if (isShuttingDown) {
        return res.status(503).json({ status: 'shutting down' });
    }
    res.status(200).json({ status: 'ready' });
});

process.on('SIGTERM', () => {
    console.log('SIGTERM received — starting graceful shutdown');
    isShuttingDown = true;

    // stop accepting new connections
    server.close(() => {
        console.log('All connections closed — exiting');
        process.exit(0);
    });

    // force exit after 50 seconds
    // (10 second buffer before Kubernetes SIGKILL at 60s)
    setTimeout(() => {
        console.log('Forcing exit after timeout');
        process.exit(1);
    }, 50000);
});
```

**Timing Budget — How to Think About It**
```bash
terminationGracePeriodSeconds: 60

0s    → preStop hook starts (sleep 5)
5s    → SIGTERM sent to app
5s    → app starts graceful shutdown finish in-flight requests, close DB connections, deregister from service mesh
55s   → app should exit by now (50s after SIGTERM)
60s   → Kubernetes sends SIGKILL (hard limit)

Buffer: Always exit 5-10 seconds BEFORE the grace period ends in case cleanup takes slightly longer than expected
```

---
---

### Deployment Strategies ###
Deployment strategies control how pods are replaced during an update.

**RollingUpdate — Default Strategy**
- Replaces pods gradually — always keeping some pods running.
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # max pods that can be down during update
      maxSurge: 1          # max extra pods that can exist during update
```
- maxUnavailable — How many pods can be below desired count during the update
  - maxUnavailable: 0 -> You must always have full desired capacity running. No old pod dies until a new one is ready.
  - maxUnavailable: 1 -> 1 pod can be killed before a replacement is ready.
- maxSurge — How many extra pods can exist above desired count during the update.
  - maxSurge: 1 -> Kubernetes can create 1 extra pod temporarily (beyond your desired replica count
  - maxSurge: 0 -> No extra pods allowed. Must kill one before creating one.

<img width="642" height="387" alt="image" src="https://github.com/user-attachments/assets/02b023a5-aff4-43cc-97fd-c3a81c3021b5" />

- You have a Kubernetes Deployment with rollingUpdate strategy — maxSurge: 1 and maxUnavailable: 0. During a rollout, you notice the deployment is completely stuck — no new pods are being scheduled, no old pods are being terminated, and the rollout has been in this state for 20+ minutes. The cluster has sufficient node capacity. What is the most likely cause, and how do you diagnose it?
```bash
Kubernetes creates new pod (surge)
         ↓
Waits for new pod to become READY
         ↓
New pod NEVER becomes ready (bad readiness probe)
         ↓
Kubernetes won't kill old pod (maxUnavailable: 0)
         ↓
Can't create another new pod (already at maxSurge)
         ↓
   STUCK FOREVER
```

**Recreate Strategy**
- Kills ALL old pods first, then creates new ones.
```yaml
spec:
  strategy:
    type: Recreate
```
- When to use Recreate:
  - App cannot run two versions simultaneously (DB schema change that is not backward compatible)
  - App uses a file lock or singleton resource (only one instance can run at a time)
  - Development environment (downtime acceptable, want clean restart)

**Blue-Green Deployment (via Argo Rollouts or manual)**
- Not a built-in Deployment strategy but very commonly asked.
- Blue environment: Current production (myapp:1.0)
- Green environment: New version (myapp:2.0)
- Step 1: Deploy green alongside blue
  - Blue: 3 pods (myapp:1.0) ← Getting 100% traffic
  - Green: 3 pods (myapp:2.0) ← Getting 0% traffic
- Step 2: Test green thoroughly
- Step 3: Switch traffic instantly
  - Blue:  3 pods (myapp:1.0) ← 0% traffic
  - Green: 3 pods (myapp:2.0) ← 100% traffi
- Step 4: Keep blue running for instant rollback
- Step 5: Delete blue after confidence builds
- Pros: Instant cutover. Instant rollback. Green fully tested before traffic switch
- Cons: Double the resources during transition. Database migrations are tricky

**Canary Deployment (via Argo Rollouts)**
- Send a small percentage of traffic to new version first. Gradually increase if healthy.
- Step 1: Deploy canary (5% traffic)
  - Stable: 19 pods (myapp:1.0) ← 95% traffic
  - Canary:  1 pod  (myapp:2.0) ← 5% traffic
- Step 2: Monitor error rates, latency for 10 minutes. All good? → increase to 20%
- Step 3: Gradually increase. 20% → 50% → 80% → 100%

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

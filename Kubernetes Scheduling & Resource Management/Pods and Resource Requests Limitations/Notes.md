Probe (Probe Means: To test Behaviour of System) Mechanisms:
- HTTP Probes: Sends an HTTP GET request to a specified endpoint.
- TCP Probes: Tries to establish a TCP connection.
- Command Probes: Executes a specified command inside the container.

- **Liveness Probe:** Determines if the application inside a Pod is still running. If the probe fails, Kubernetes restarts the container. Common scenarios: Application crashes or enters a deadlock state, Recovering from temporary issues. Liveness probe sends HTTP GET requests to the container's endpoint internally, using the *kubelet*, which runs on the worker node hosting the container. A 2xx or 3xx HTTP response indicates the container is healthy.
- **Readiness Probe:** Determines if the application inside a Pod is ready to serve traffic. If the probe fails, the Pod is removed from the Service's endpoints. Common scenarios: Application initialization takes time. External dependencies (e.g., database) are not ready yet.

- ProbeParameters:
    - initialDelaySeconds: Time to wait before starting the probe.
    - periodSeconds: Frequency of the probe.
    - failureThreshold: Number of failures before marking the container as unhealthy.
    - timeoutSeconds: Maximum duration to wait for a response.

```yaml
livenessProbe:
    httpGet:
        path: /
        port: 80
    initialDelaySeconds: 5
    periodSeconds: 10
readinessProbe:
    httpGet:
        path: /
        port: 80
    initialDelaySeconds: 5
    periosdSeconds: 5
```
- What will happen if a Pod has a Readiness Probe that always fails, but its liveness probe always passes?
    - Pod Will Never Receive Traffic: If Readiness Probe always fails, the Pod is never added to the *Endpoints* list of its Service.
    - Pod Will Keep Running: Since Liveness Probe always passes, Kubernetes assumes the container is healthy and does not restart it.
    - Rolling Update: If this Pod is part of a Deployment, the rolling update may get stuck because Kubernetes waits for the new Pod to become "Ready" before terminating the old one.
 
**You have a Kubernetes cluster with a Deployment running a mission-critical application. However, the Pods are frequently restarting. There are no errors in the application logs. How would you debug this?**    

Since the application logs don’t show any errors, the issue might be related to resource limits, OOM kills, readiness/liveness probes, or underlying node issues
- I would start by inspecting the pod events and status using:
  - `kubectl describe pod <pod-name> -n <namespace>`. This will show me reasons for restarts, such as OOM kills, failed probes, or node issues.
- If the pod was OOMKilled, I’d check the container exit reason using:
  - `kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].state.terminated.reason}'`
  - I’d check the pod logs from the previous run to see if there were memory spikes before termination: `kubectl logs <pod-name> --previous`
  - To confirm system-level memory issues, I’d check the node's memory usage:
    ```bash
    kubectl top node
    kubectl top pod -n <namespace>
    ```
- If the pod is not OOMKilled but is restarting due to liveness probe failures
  - `kubectl describe pod <pod-name> -n <namespace>`. I’d specifically look at the `Liveness probe failed` messages. If the probe is failing, I’d check the probe configuration in the Deployment YAML
    ```yaml
     livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    ```
- If probes are fine, but the pod is still restarting
  - Node-Level Issues:
    - Check node health and availability:
      ```bash
      kubectl get nodes -o wide
      kubectl describe node <node-name>
      ```
    - journalctl logs on the node to check for kernel crashes, disk pressure, or kubelet errors: `journalctl -u kubelet -f`
  - Container-Level Issues:
    - Check Docker/container runtime logs:
      ```bash
      docker ps -a | grep <container-id>
      docker logs <container-id>
      ```
    - Verify if the container is exiting due to CrashLoopBackOff:
      ```bash
      kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].state.waiting.reason}'
      ```
    - Networking Issues:
      - If the pod depends on external services, I’d check DNS resolution: `kubectl exec -it <pod-name> -- nslookup <service-name>`
      - Check CNI plugin status: `kubectl get pods -n kube-system | grep cni`
    - Persistent Volume Issues (if applicable):
      - Check if the volume mount is failing: `kubectl describe pod <pod-name> | grep Mount`
- If everything seems fine but the issue persists, I’d:
  - Enable detailed logging on the kubelet and check `/var/log/kubelet.log`.
  - Check if the cluster has any active policies (e.g., `LimitRange` or `PodSecurityPolicy`) affecting pod scheduling.
  - Consider moving the pod to another node using `kubectl cordon` and `kubectl drain` to see if the issue is node-specific.

        


    
    


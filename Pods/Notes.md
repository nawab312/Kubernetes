**RestartPolicy** field in a Pod specification controls how the *kubelet* restarts containers within the pod when they exit. The possible values for restartPolicy are:
- **Always (default)**
  - The container is always restarted if it fails or exits. Used for long-running applications like web servers.
- **OnFailure**
  - The container is restarted only if it exits with a non-zero status. Suitable for batch jobs that should only restart on failure.
- **Never**
  - The container is never restarted, even if it fails. Suitable for one-time execution tasks.
- `restartPolicy` applies to the *Pod* level, not individual containers.
- If using *Jobs* or *CronJobs*, the default restartPolicy must be *OnFailure* or *Never* (not Always).
- For *Deployments*, *StatefulSets*, and *DaemonSets*, the restartPolicy is always set to *Always*.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  restartPolicy: OnFailure
  containers:
  - name: my-container
    image: my-batch-job
```

- What happens if there are not enough resources to schedule a pod?
  - Pod is Unschedulable: The Kubernetes scheduler will attempt to find a node with enough resources (CPU, memory, etc.) to accommodate the Pod. If no suitable node is found, the Pod remains in the "Pending" state.
  - Retry Mechanism: Kubernetes continuously retries scheduling the Pod, waiting for resources to free up or for new nodes to be added to the cluster.
  - Cluster Autoscaler (if enabled): If Cluster Autoscaler is configured, Kubernetes can automatically provision new nodes to handle the resource demand. Once new nodes are available, the Pod is scheduled.
 
- In a Kubernetes cluster, you delete a Kibana pod, but it keeps getting recreated with the status Init:0/1. How would you permanently delete the Kibana Pod, and what underlying Kubernetes concepts explain this behavior
  - The pod is likely managed by a higher-level controller (e.g., Deployment, StatefulSet, DaemonSet, or Helm Release).
  - `kubectl get all | grep kibana`. This will reveal whether Kibana is controlled by a Job, Deployment, StatefulSet, or Helm Release.
  - ```bash
    kubectl get all | grep kibana
    pod/post-delete-kibana-kibana-4cbkf   0/2     Init:0/1   0          5m31s
    job.batch/post-delete-kibana-kibana   Running   0/1           10m        10m
    ```
  - Delete the Underlying Controller, Here it is the *Job*
    - Kubernetes Jobs are designed to run to completion.
    - Some Helm charts use pre-delete or post-delete hooks to run cleanup jobs.
    - The presence of `post-delete-kibana-kibana` suggests that it is part of a Helm hook.
  - Delete the Running Job `kubectl delete job post-delete-kibana-kibana`
 
  **There are three main types of containers used in a Pod**
  - **Main (or Application) Container**
    - This is the primary container that runs the main application logic.
    - Typically, a Pod will have one main container, but multi-container Pods are possible.
    - Example: A container running a web server like NGINX or Spring Boot application.
  - **Init Containers**
    - These are special containers that *run before the main container starts*.
    - They are used for initialization tasks such as:
      - Setting up configurations.
      - Ensuring dependencies are available.
      - Performing database migrations.
    - Each Init container must complete successfully before the main container starts.
    - Example: A container that waits for a database to become available before starting the application.
  - **Sidecar Containers**
    - These containers *run alongside the main container* and provide additional functionality.
    - Common use cases:
      - Logging and monitoring (e.g., Fluentd, Logstash).
      - Service mesh proxies (e.g., Envoy for Istio).
      - Security (e.g., authentication sidecars).
    - Example: A Fluentd container that collects logs from the main application container and ships them to Elasticsearch.
   
**Pod to Pod Communication Across Nodes** https://github.com/nawab312/Kubernetes/blob/main/Pods/Questions/Question1.md


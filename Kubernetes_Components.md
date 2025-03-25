### Pod ###
A Pod is the smallest and simplest unit in Kubernetes. It encapsulates one or more containers that share storage and network resources. Containers in the same Pod are tightly coupled and run on the same node. A Pod may contain 
- Single Container Pod: A single container inside the Pod.
- Multi-container Pod: Multiple containers inside the same Pod, which are closely related and share the same environment (network, storage).

**RestartPolicy** field in a Pod specification controls how the kubelet restarts containers within the pod when they exit. `restartPolicy` applies to the Pod level, not individual containers.The possible values for restartPolicy are:
- *Always (default)* The container is always restarted if it fails or exits. Used for long-running applications like web servers.
- *OnFailure* The container is restarted only if it exits with a non-zero status. Suitable for batch jobs that should only restart on failure.
- *Never* The container is never restarted, even if it fails. Suitable for one-time execution tasks.
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

**There are three main types of containers used in a Pod**
- *Main (or Application) Container*
  - This is the primary container that runs the main application logic.
  - Typically, a Pod will have one main container, but multi-container Pods are possible.
  - Example: A container running a web server like NGINX or Spring Boot application.
- *Init Containers*
  - These are special containers that run before the main container starts.
  - They are used for initialization tasks such as:
    - Setting up configurations.
    - Ensuring dependencies are available.
    - Performing database migrations.
  - Each Init container must complete successfully before the main container starts.
- *Sidecar Containers*
  - These containers run alongside the main container and provide additional functionality.
  - Common use cases:
    - Logging and monitoring (e.g., Fluentd, Logstash).
    - Service mesh proxies (e.g., Envoy for Istio).
    - Security (e.g., authentication sidecars).

  

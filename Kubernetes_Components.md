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
   
### Service ###
- Service is an abstraction that defines a logical set of Pods and a policy by which to access them. It provides a stable endpoint (usually a DNS name) to access a group of Pods, even though the Pods themselves might be dynamically created, destroyed, or replaced. We cannot scale a Kubernetes Service. 
- **Endpoint** is an object that represents the IP addresses (Including Port) of the Pods that are dynamically assigned to a Service. It helps in routing traffic to the correct Pod(s) backing a Service.
`SERVICE ---> ENDPOINT (automaticlaly created) --> POD'S IP and PORT`

**Why Services are Needed**
- Pod IPs Are Ephemeral: Pods are dynamic, and their IP addresses can change when they restart. A Service ensures a stable endpoint to communicate with the Pods.
- Load Balancing: Services distribute traffic across all Pods that match the selector, enabling better resource utilization and reliability

**ClusterIP**
- Exposes the Service only within the cluster. Pods within the same cluster can access it via its ClusterIP.
- Use Case: Backend services like databases that don’t need external exposure.
```yaml
apiVersion: v1 
kind: Service
metadata:
  name: my-clusterip-service
spec:
  selector:
    app: my-app 
  ports:
    - protocol: TCP
      port: 80 #PORT EXPOSED BY SERVICE
      targetPort: 8080 #PORT EXPOSED IN THE POD/CONTAINER
```
- How does network flow when a client hits clusterIP Service?
  - Client Request: The client sends a request to the ClusterIP of the Service. This is a virtual IP assigned by Kubernetes, which abstracts the underlying Pods.
  - *kube-proxy*: The kube-proxy running on each node intercepts the request. It uses `iptables` or `IPVS rules` to forward the request to one of the Pods that match the Service's label selector.
  - Pod Selection: kube-proxy performs load balancing and chooses a Pod based on round-robin (or another policy). The request is routed to the selected Pod's IP and port.
  - Pod Response: The Pod processes the request and sends the response back to the client. The response flows through the same network path, using the Pod's IP to communicate back.


**NodePort**
- Exposes the Service on a static port `(30000-32767)` on each Node’s IP. External traffic can access the application via `NodeIP>:<NodePort>`
- Use Case: Debugging or exposing an application during development.

![NodePort](https://github.com/nawab312/Kubernetes/blob/main/Images/NodePort.png)

```yaml
apiVersion: v1 
kind: Service 
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30007
```

**LoadBalancer**
- Automatically provisions an external load balancer (cloud-provider specific). Exposes the Service to the internet with a public IP. 
- Use Case: Production services that need external access, such as websites or APIs
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort:8080
```

### ReplicaSets ###
A ReplicaSet ensures that a specified number of replicas (Pods) of a containerized application are running at all times. If a Pod crashes or is deleted, the ReplicaSet will create a new Pod to replace it, maintaining the desired number of Pod

### Deployment ###
Deployment is a higher-level object that manages ReplicaSets and Pods. It provides declarative updates to applications, meaning you can define the desired state of your application, and Kubernetes will work to ensure that the system converges to that state.

### Daemonset ###
DaemonSet in Kubernetes ensures that a copy of a specific Pod is running on every node (or a subset of nodes) in the cluster. This is useful for running system-level or utility applications that need to be present on each node, such as logging agents, monitoring agents, or networking tools.
- One Pod per Node: DaemonSets guarantee that there will be exactly one Pod running on each node in the cluster. If new nodes are added, a Pod will be automatically scheduled to those nodes.
- Pods in a DaemonSet are not scheduled based on normal Deployment behavior (replica scaling); they are scheduled based on the availability of nodes.

### ConfigMaps and Secrets ###
**ConfigMaps**
- A Kubernetes object used to store non-sensitive configuration data in key-value pairs
- Use Cases: Application settings (e.g., environment variables). File-based configurations (e.g., .properties or .json files)
- ConfigMaps are mounted into Pods as: Environment variables, Command-line arguments, Volumes (files in a directory).

**Secrets**
- Kubernetes objects to securely store sensitive information such as passwords, tokens, or certificates. Data is encoded using Base64 *(not encrypted by default)*.
- Types of Secrets:
    - Opaque: Default type for arbitrary data.
    - TLS: Used to store TLS certificates and keys.
- Secrets can be accessed by Pods as: Environment variables, Mounted volumes.
- Kubernetes Secrets are stored in etcd, but by default, they are only base64-encoded, not encrypted. This means anyone with access to etcd storage can read sensitive data like passwords and API keys. To enhance security, Kubernetes provides Encryption at Rest, which encrypts Secrets before storing them in etcd.

Example
- How to Create a Secret?
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: dXNlcm5hbWU=  # Base64 encoded "username"
  password: cGFzc3dvcmQ=  # Base64 encoded "password"
```
- Use Secret as an Environment Variable in a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: myContainer
      image: nginx
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: username
  restartPolicy: Always
```
- Use Secret as a Mounted Volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: myContainer
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: "/etc/secret-data"
          readOnly: true #readOnly: true → Prevents accidental modification of Secret files.
  volumes:
    - name: secret-volume
      secret:
        secretName: my-secret
  restartPolicy: Always
```

  

- Stateful workloads require *persistent storage* and *stable network* identities
- Stateless apps like web servers don’t maintain persistent data across restarts.
- Stateful apps like databases, caches, or distributed systems require: Persistent storage to retain data.

### Statefulsets ###
- It manages Pods with unique, stable identities and persistent storage.
- *It is required for pods that need to maintain their state across restarts*
- Stable Pod Names: Pods are named with a predictable suffix (e.g., app-0, app-1).
- Stable Network Identities: Each Pod has a consistent DNS name (e.g., `app-0.service-name.namespace.svc.cluster.local`).
- Ordered Pod Deployment and Scaling: Pods are created or deleted in order
- Persistent Storage: Each Pod gets its Persistent Volume (PV), which isn’t deleted even if the Pod is removed.
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-stateful-app
spec:
  serviceName: "my-stateful-service" # Name of the headless service
  replicas: 3 # Number of replicas (pods)
  selector:
    matchLabels:
      app: my-stateful-app
  template:
    metadata:
      labels:
        app: my-stateful-app
    spec:
      containers:
      - name: web-container
        image: nginx:latest
        volumeMounts:
        - name: web-storage
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: web-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### HeadLess Service ###
- A Headless Service manages DNS records for individual Pods *without providing a cluster IP*.
- Pods in a Headless Service are discoverable by their DNS names
- The `clusterIP: None` in the Service definition creates a Headless Service. This means Kubernetes will not allocate an external IP but will resolve DNS queries to the individual Pods in the StatefulSet.

**How the Headless Service Helps the StatefulSet:**
- Stable DNS for StatefulSet Pods:
  - The Pods in a StatefulSet are assigned predictable DNS names, such as `my-stateful-app-0.my-stateful-service`, `my-stateful-app-1.my-stateful-service`, etc
  - This helps when other Pods or external systems need to communicate with a specific Pod in the StatefulSet. For example, a database might need to connect to a specific replica
- No Load Balancing:
  - A Headless Service doesn’t perform load balancing. Instead, it exposes the DNS records of each individual Pod. This is useful when you want direct access to the Pods, which is often required by stateful applications like databases.
 
**How does a StatefulSet ensure that each pod gets a unique Persistent Volume?**
- A StatefulSet defines a `volumeClaimTemplates` section, which acts as a template for creating PVCs dynamically. Each pod in the StatefulSet gets its own PVC created from this template.
- Kubernetes appends the pod's name to the PVC name to ensure uniqueness. The pod name includes the StatefulSet name and an ordinal index (e.g., web-0, web-1), so the PVCs are named like `data-web-0`, `data-web-1`, etc
- Each pod in the StatefulSet is permanently associated with its specific PVC. If the pod is deleted and recreated (e.g., due to failure), it will reuse the same PVC and its data.




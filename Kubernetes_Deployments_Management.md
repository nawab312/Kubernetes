### Kubernetes Deployment Strategies ###
When deploying applications in Kubernetes, different strategies ensure minimal downtime, controlled rollouts, and seamless updates. Below are the primary deployment strategies:

**Rolling Deployment (Default Strategy)**
- Concept: Gradually replaces old pods with new ones
- How it works:
  - Kubernetes incrementally updates a specified number of pods at a time.
  - Ensures zero downtime by keeping some old pods running while bringing up new ones.
- Use Cases:
  - Suitable for most applications.
  - Ideal for non-disruptive updates.
- Pros:
  - Zero downtime.
  - Automatic rollback if failure occurs.
- Cons:
  - Slow deployment if the application is large.
  - Difficult to test new versions before full rollout.
 
**Recreate Deployment**
- Concept: Deletes all existing pods before creating new ones.
- How it works:
  - The entire application goes down before the new version comes up.
- Use Cases:
  - When an application cannot run multiple versions simultaneously.
  - Useful for databases or stateful applications that don't support parallel execution.
- Pros:
  - Simple to implement.
- Cons:
  - Causes downtime.
  - Not suitable for high-availability applications.
 
**Blue-Green Deployment**
- Concept: Two separate environments (Blue = Current, Green = New). Traffic is switched from Blue to Green after successful validation.
- How it works:
  - The new version (Green) is deployed alongside the old version (Blue).
  - Once the new version is validated, the traffic is switched using a LoadBalancer or Service.
  - If issues arise, rollback is instantaneous by switching traffic back to Blue.
- Use Cases:
  - When downtime is unacceptable.
  - Useful for applications that require pre-deployment testing.
- Pros:
  - Instant rollback possible.
  - No downtime.
- Cons:
  - Requires double the resources.
  - Higher cost

**Canary Deployment**
- Concept: A small percentage of traffic is routed to the new version, gradually increasing until full rollout.
- How it works:
  - Deploys a subset of new pods (e.g., 10%) while keeping old pods active.
  - Traffic splitting is handled via Ingress, Service, or Istio.
  - Based on monitoring results, traffic is increased or rolled back.
- Use Cases:
  - When testing new features on a small subset of users.
  - Helps reduce risk in production deployments.
- Pros:
  - Fine-grained control over rollout.
  - Quick rollback if issues are detected.
- Cons:
  - More complex setup.
  - Requires monitoring tools for decision-making.
 
### Kubectl CLI Commands (Common Commands) ###
**Cluster & Context Management**
```bash
kubectl config get-contexts               # List available contexts
kubectl config use-context <context-name> # Switch to a different context
kubectl config current-context            # Show the current context
kubectl cluster-info                      # Display cluster details
kubectl version --short                   # Show client and server versions
```

**Get Information About Resources**
```bash
kubectl get nodes                         # List all nodes in the cluster
kubectl get pods                           # List all pods in the current namespace
kubectl get pods -A                        # List pods in all namespaces
kubectl get deployments                    # List deployments in the current namespace
kubectl get services                       # List services
kubectl get replicasets                    # List replica sets
kubectl get configmap                      # List ConfigMaps
kubectl get secrets                        # List secrets
kubectl get namespaces                     # List all namespaces
kubectl get ingress                        # List all Ingress resources
```

**Describe Resources**
```bash
kubectl describe pod <pod-name>            # Detailed information about a pod
kubectl describe deployment <deployment-name>
kubectl describe node <node-name>
kubectl describe service <service-name>
kubectl describe ingress <ingress-name>
```

**Creating and Managing Resources**
```bash
kubectl create deployment my-app --image=nginx  # Create a deployment
kubectl expose deployment my-app --type=NodePort --port=80  # Expose it as a service
kubectl apply -f my-app.yaml  # Apply configuration from a YAML file
kubectl delete pod my-pod     # Delete a pod
kubectl delete deployment my-app  # Delete a deployment
kubectl delete service my-service  # Delete a service
kubectl delete ingress my-ingress  # Delete an ingress
kubectl delete namespace my-namespace  # Delete a namespace
kubectl delete all --all  # Delete all resources in the current namespace
```

**Pod Management**
```bash
kubectl logs <pod-name>                     # View logs of a pod
kubectl logs <pod-name> -f                  # View logs in real-time (like tail -f)
kubectl logs <pod-name> -c <container-name> # View logs of a specific container in a pod
kubectl exec -it <pod-name> -- bash         # Open a bash shell inside a pod
kubectl port-forward pod/<pod-name> 8080:80 # Forward port 8080 on local machine to 80 on the pod
kubectl top pod                             # View CPU and memory usage of pods
```

**Scale and Rollout Management**
```bash
kubectl scale deployment my-app --replicas=5  # Scale deployment to 5 replicas
kubectl rollout status deployment my-app      # Check rollout status
kubectl rollout history deployment my-app     # View deployment revision history
kubectl rollout undo deployment my-app        # Roll back to the previous deployment
kubectl set image deployment my-app nginx=nginx:latest # Update container image
```

**Namespace Management**
```bash
kubectl get namespaces                     # List all namespaces
kubectl create namespace my-namespace      # Create a new namespace
kubectl delete namespace my-namespace      # Delete a namespace
kubectl config set-context --current --namespace=my-namespace  # Set default namespace
kubectl get pods -n my-namespace           # Get pods in a specific namespace
```

**ConfigMaps & Secrets**
```bash
kubectl create configmap my-config --from-literal=env=prod  # Create ConfigMap from key-value pair
kubectl get configmap my-config -o yaml                      # View ConfigMap
kubectl create secret generic my-secret --from-literal=password=12345  # Create secret
kubectl get secret my-secret -o yaml                         # View secret
kubectl delete configmap my-config                          # Delete ConfigMap
kubectl delete secret my-secret                             # Delete secret
```

**Debugging & Troubleshooting**
```bash
kubectl get events                            # Get cluster events
kubectl get pod <pod-name> -o yaml            # Get detailed YAML output of a pod
kubectl logs <pod-name> -f                    # Stream logs
kubectl describe pod <pod-name>               # Get detailed pod info
kubectl exec -it <pod-name> -- bash           # Enter into a running container
kubectl top pod                                # Show resource usage (CPU & Memory)
kubectl top node                               # Show node resource usage
```

**Node Management**
```bash
kubectl get nodes                            # List all nodes
kubectl describe node <node-name>            # Get detailed node info
kubectl cordon <node-name>                   # Mark node as unschedulable
kubectl drain <node-name> --ignore-daemonsets --force  # Evacuate all pods from a node
kubectl uncordon <node-name>                 # Mark node as schedulable
```

**Autoscaling with HPA (Horizontal Pod Autoscaler)**
```bash
kubectl autoscale deployment my-app --cpu-percent=50 --min=2 --max=10  # Set HPA
kubectl get hpa                     # List all HPAs
kubectl delete hpa my-app            # Delete HPA
```


### Helm Chart ###
Helm Chart is a package for Kubernetes applications that defines all resources needed to deploy an application in a cluster. 
Helm charts provide templated YAML files, making deployments more manageable, reusable, and configurable.
```bash
my-helm-chart/
│── charts/                # (Optional) Subcharts for dependencies
│── templates/             # Kubernetes resource templates
│   ├── deployment.yaml    # Deployment manifest
│   ├── service.yaml       # Service manifest
│   ├── ingress.yaml       # (Optional) Ingress for external access
│   ├── _helpers.tpl       # (Optional) Template helpers
│── values.yaml            # Customizable parameters
│── Chart.yaml             # Chart metadata
│── .helmignore            # Files to exclude from Helm package
```

**Commands**
- `helm list` List all installed charts
- `helm create mychart` Create a new Helm chart
- `helm package mychart` Package a Helm chart
- `helm lint mychart` Lint (validate) a Helm chart
- `helm install myrelease mychart` Install a Helm chart
- `helm install myrelease mychart -f values.yaml` Install a Helm chart with custom values
- `helm install myrelease mychart --set replicaCount=3` Install with --set flag for inline value overrides
- `helm upgrade myrelease mychart -f values.yaml` Upgrade an existing release
- `helm rollback myrelease 1` Rollback to a previous Helm release version
- `helm install myrelease mychart --dry-run --debug` Dry-run and debug a Helm chart (without installing it)





**Key Files in a Helm Chart**
- Chart.yaml – Metadata about the chart (name, version, description)
- values.yaml – Defines customizable values for the deployment (e.g., image name, replicas, environment variables).
- templates/ – Contains Kubernetes YAML files with Helm templating syntax.

**Helm Hooks**
- Helm hooks are special Kubernetes resources that execute at specific points in the Helm release lifecycle.
- They allow you to run jobs, scripts, or custom Kubernetes resources before or after certain Helm operations like installation, upgrade, or deletion.
- Why Use Helm Hooks?
  - Database migrations before deploying an app.
  - Running pre-install validation checks.
  - Cleaning up resources after uninstalling.
  - Executing custom scripts during Helm lifecycle events.
    
![image](https://github.com/user-attachments/assets/a0e0bf62-6e14-46f1-a534-dd03023b5a01)

- Helm Hook: Pre-Install Job for DB Migration
  - `helm.sh/hook: pre-install` → Runs before the main deployment.
  - `helm.sh/hook-delete-policy: hook-succeeded` → Deletes job after success.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migration
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
          image: myrepo/db-migration:latest
          command: ["python", "migrate.py"]
```

**Conditionals in Helm**
- Helm uses `if` statements inside templates with the `values.yaml` file to control resource deployment.
- Conditionally Deploy a ConfigMap
```yaml
#values.yaml
configMap:
  enabled: true
  data:
    APP_ENV: "production"
```
```yaml
#configmap.yaml
{{- if .Values.configMap.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  APP_ENV: {{ .Values.configMap.data.APP_ENV | quote }}
{{- end }}
```

**Helper Functions in Helm (_helpers.tpl)**
- Helm helper functions are reusable templates defined in the `_helpers.tpl` file inside the `templates/ `directory.
- They help eliminate redundancy by allowing you to define and reuse common logic across multiple resources

Custom Helm Helper Function for Standardized Labels. Provides a consistent labeling scheme for all resources.
- Create `_helpers.tpl` with a Custom Label Function
```yaml
{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.Version }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```
- Use the Helper Function in deployment.yaml
  - `nindent 4` & `nindent 8` ensure proper YAML formatting.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        {{- include "mychart.labels" . | nindent 8 }}
    spec:
      containers:
        - name: my-app
          image: nginx
```
- Use the Helper Function in service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**Example**
You need to package and deploy a microservice using Helm by creating a custom Helm chart.
- Create a Helm chart to deploy a Node.js or Python Flask application.
- Parameterize values such as replica count, environment variables, and resource limits using a values.yaml file.
- Configure the Chart.yaml file with metadata and dependencies (if any).
- Deploy the Helm chart to a Kubernetes cluster and verify the deployment.

```bash
helm create flask-microservice-chart
```
```bash
flask-microservice-chart/
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── service.yaml
└── values.yaml
```
```yaml
#Chart.yaml
apiVersion: v2
name: flask-microservice
description: A Helm chart for deploying a Flask microservice
type: application
version: 1.0.0
appVersion: "1.0.0"
maintainers:
  - name: Siddharth Singh
    email: devops@example.com
```
```yaml
#values.yaml
replicaCount: 2

image:
  repository: sid3121997/sample-app
  pullPolicy: IfNotPresent
  tag: latest


service:
  type: ClusterIP
  port: 5000


resources: 
  limits:
    cpu: "500m"
    memory: "256Mi"
  requests:
    cpu: "250m"
    memory: "128Mi"


env:
  APP_ENV: "production"
  DEBUG: "false"

ingress:
  enabled: false
```
```yaml
#deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "{{ .Release.Name }}-deployment"
  labels:
    app: flask-microservice
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: flask-microservice
  template:
    metadata: 
      labels: 
        app: flask-microservice
    spec:
      containers:
        - name: flask-microservice
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 5000
          resources:
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}

```
```yaml
#service.yaml
apiVersion: v1 
kind: Service 
metadata: 
  name: "{{ .Release.Name }}-service"
spec: 
  selector:
    app: flask-microservice
  type: {{ .Values.service.type }}
  ports:
    - protocol: TCP
      containerPort: {{ .Values.service.port }}
      port: 5000
```
```bash
helm install sample-app ./flask-microservice-chart/
```

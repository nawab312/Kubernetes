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

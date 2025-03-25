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

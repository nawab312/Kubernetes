Your application is exposed using a Kubernetes Ingress with an Nginx Ingress Controller. Users report that some requests are randomly failing with 502 Bad Gateway, while others succeed.
Question:
- How would you debug and identify the root cause of these intermittent failures?
- What logs or configurations would you check first?
- Could this be related to backend pod failures, DNS resolution, or ingress misconfiguration? How would you confirm?

- Flask application defines a simple web server with one route (`/`) that returns "Hello from backend" when accessed. The server runs on all available network interfaces (`0.0.0.0`) and listens on port 8080.
```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from backend", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

- Create the DockerFile
```Dockerfile
FROM python:3.9

WORKDIR /app
COPY app.py /app/

RUN pip install flask

CMD ["python", "app.py"]
```

- Build the Image and Push to Repository
```bash
docker build -t sid3121997/simple_app:version1 .
docker push sid3121997/simple_app:version1
```

- Start Minikube and Enable Ingress
```bash
minikube start
minikube addons enable ingress
```

- Write the Manifest Files
```yaml
# backendDeployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: sid3121997/simple_app:version1
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

```yaml
# backendService.yaml

apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

```yaml
#ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: backend.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

```bash
kubectl apply -f backendDeployment.yaml
kubectl apply -f backendService.yaml
kubectl apply -f ingress.yaml
```
```bash
echo "$(minikube ip) backend.local" | sudo tee -a /etc/hosts
```

**TroubleShooting**

- Test the App

![image](https://github.com/user-attachments/assets/1aac1487-5cf1-4853-8759-51a3b00a398c)

- Check the Ingress Logs
```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```
`Error obtaining Endpoints for Service "default/backend-service": no object matching key "default/backend-service" in local store`

- Check Readiness/Liveness Probe Status. You’ll see probe failures causing pod restarts.
```bash
kubectl describe pod -l app=backend
```

- Check Service Endpoints. If it shows None, it means no healthy backend pods exist.
```bash
kubectl get endpoints backend-service
NAME              ENDPOINTS   AGE
backend-service               11m
```

**Fixning the Issue**

- Rebuild the Image by adding `/healthz` route in `app.py`
```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from backend", 200

@app.route("/healthz")
def health_check():
    return "OK", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```





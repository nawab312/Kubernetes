The need for Ingress in Kubernetes arose primarily to address the challenges of managing external access to services running inside a Kubernetes cluster. Before Ingress, there were a few common methods to expose services, like using NodePort or LoadBalancer resources. 
- NodePort: This opens a specific port on each node in the cluster. It means exposing each service on a unique port, which can quickly become messy when there are multiple services, and also isn’t scalable.
- LoadBalancer: This requires setting up a *separate external load balancer for each service*. This approach can be costly, particularly in cloud environments where each load balancer comes with a fee.
![Ingress1](https://github.com/nawab312/Kubernetes/blob/main/Images/Ingress1.png)

- **Ingress** is an API object that manages external HTTP/HTTPS access to services within a cluster
- It provides routing rules to expose multiple services under a single IP or domain.
- An **Ingress Resource** is a Kubernetes object that defines rules for external access to your services in the cluster, such as HTTP and HTTPS traffic.
- **Ingress Controller** is a Kubernetes component responsible for implementing the rules defined in the Ingres. NGINX Ingress Controller, Traefik, HAProxy, Istio.

**How do you set up load balancing with Ingress?**
- Two main steps: deploying an Ingress controller and defining Ingress resources
- Step 1 is to Deploy the Ingress Controller. Ingress controller is a pod or set of pods running in your cluster that takes care of routing traffic based on the Ingress rules.
- Once the Ingress controller is deployed, the next step is to define Ingress resources for your services. These resources specify how incoming traffic should be routed.

*The Ingress controller handles the external routing, while within each service, Kubernetes uses a service to load balance between multiple pods. So, if a service has multiple pods, Kubernetes automatically distributes traffic among them.
For example, if you have a service with three replicas (pods), Kubernetes will round-robin the traffic among those pods. The Ingress controller doesn’t have to worry about that part — it just ensures that the traffic gets directed to the correct service, and Kubernetes handles the internal load balancing within the service.*

**Annotations in Ingress**
- Annotations in an Ingress resource in Kubernetes are used to provide additional configuration or metadata that can control the behavior of the Ingress Controller 
- Unlike regular fields in an Ingress definition (such as `spec.rules` or `spec.backend`), annotations are typically used to specify configuration options that are specific to the Ingress Controller.
- `nginx.ingress.kubernetes.io/ssl-redirect: "true"` Controls whether HTTP traffic is redirected to HTTPS.
- `nginx.ingress.kubernetes.io/ingress.class: "nginx"` Specifies which Ingress Controller should handle this Ingress.
- `nginx.ingress.kubernetes.io/load-balancer-method: least_conn` Specifies the load balancing algorithm to use for requests.

**Let’s say we have an EKS cluster with multiple nodes. In this cluster, we have 20 services, each with 5 pods. Now, for each service, we want to implement a least-connection load balancing algorithm to distribute traffic more efficiently. How would you go about setting this up?**
By default, Kubernetes Services use round-robin load balancing for distributing traffic across pods. However, Kubernetes doesn't provide a built-in option for least-connection load balancing at the Service level.
- Using an External Load Balancer: We could start by using a load balancer outside the Kubernetes cluster (like AWS ALB (Application Load Balancer) or NGINX) that routes traffic to the Ingress controller.
- Choosing the Right Ingress Controller: For least-connection load balancing at the pod level, we should choose an Ingress controller that supports this algorithm. One such controller is NGINX
- First, we need to deploy the NGINX Ingress controller in the cluster. We can do this by using Helm or kubectl. `helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.ingressClass=nginx --set controller.replicaCount=2`
- Once the NGINX Ingress controller is deployed, we can configure it to use the least-connection load balancing algorithm. This can be done by editing the ConfigMap of the NGINX Ingress controller.
- `kubectl edit configmap nginx-ingress-controller -n ingress-nginx`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller
  namespace: nginx-ingress
data:
  enable-ssl-passthrough: "true"
  lb-method: least_conn

#https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/configmap-resource/
```

**Kubernetes does not allow overlapping Ingress rules with the same path (/app) and host (example.com) but different pathType values (Exact vs Prefix).**
```yaml
# Ingress Rule 1: Exact path matching
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-exact
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Exact
        backend:
          service:
            name: exact-service
            port:
              number: 80
```
```yaml
# Ingress Rule 2: Prefix path matching
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-prefix
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: prefix-service
            port:
              number: 80
```
Solution:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-matching-ingress
spec: 
  rules:
    - host: example.com
      http:
        paths:
          - path: /app
            pathType: Exact
            backend:
              service:
                name: exact-service
                port:
                  number: 80

          - path: /app
            pathType: Prefix
            backend:
              service:
                name: prefix-service
                port:
                  number: 80
```

### Scenario ###
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

### Scenario ###
You are tasked with deploying a simple order management service using Kubernetes. The service is built with Node.js and uses Express to handle order data and health checks. The application has a readiness probe configured to check the `/healthz` endpoint to ensure the service is ready to serve traffic. The service is exposed using an Ingress resource at the domain `order.local`.
Given the following setup:
- Deployment YAML for `order-service` with the readiness probe checking `/healthz`.
- Ingress resource to expose the service on the path `/order` under the `order.local` domain

### Issue That I was getting ###
- when doing `curl http://order.local/order` I was getting **Error 404**
- Reason in `ingress.yaml` this was present
```yaml
metadata:
  name: order-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

- **nginx.ingress.kubernetes.io/rewrite-target: /**
  - The annotation `rewrite-target: /` tells NGINX Ingress to remove the `/order` path when forwarding the request.
  - So, when you request `http://order.local/order`, NGINX rewrites it to `/`, but your backend expects `/order` leading to a 404 error.

### Step1) Create the Dockerfile ####

```bash
mkdir order-service
cd order-service
npm init -y
```
- Install Dependencies
```bash
npm install express
```

- Create the `server.js` file
```js
const express = require("express");
const app = express();
const port = 8080;

// Simulated database
let orders = [
    { id: 1, item: "Laptop", status: "Processing" },
    { id: 2, item: "Phone", status: "Shipped" }
];

// Health check endpoint
app.get("/healthz", (req, res) => {
    res.status(200).send("OK");
});

// Order API endpoint
app.get("/order", (req, res) => {
    res.status(200).json(orders);
});

// Simulate slow readiness for Kubernetes
setTimeout(() => {
    console.log("App is ready to receive traffic.");
}, 10000); // Simulate a 10s startup delay

app.listen(port, () => {
    console.log(`Order service is running on port ${port}`);
});
```

- Create the `Dockerfile`
```dockerfile
# Use official Node.js image
FROM node:18

# Set working directory
WORKDIR /app

# Copy package.json and install dependencies
COPY package.json package-lock.json ./
RUN npm install

# Copy source code
COPY . .

# Expose the application port
EXPOSE 8080

# Start the application
CMD ["node", "server.js"]
```

- Build and Push Docker Image
```bash
docker build -t sid3121997/simple_app:nodeJS
docker push sid3121997/simple_app:nodeJS
```

### Step2) Create and Apply the Kubernetes Manifest Files ###

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service-pod
  template:
    metadata:
      labels:
        app: order-service-pod
    spec:
      containers:
        - name: order-app
          image: sid3121997/simple_app:nodeJS # Replace with your image
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

```yaml
# service.yaml

apiVersion: v1
kind: Service
metadata:
  name: order-service-service
spec:
  selector:
    app: order-service-pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

```yaml
# ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-ingress
spec:
  rules:
  - host: order.local
    http:
      paths:
      - path: /order
        pathType: Exact
        backend:
          service:
            name: order-service-service
            port:
              number: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl apply -f ingress.yaml
kubectl apply -f service.yml
```

![image](https://github.com/user-attachments/assets/cb80b424-4612-49c0-b508-2f390d81ad8f)






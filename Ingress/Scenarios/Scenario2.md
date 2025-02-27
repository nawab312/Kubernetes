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

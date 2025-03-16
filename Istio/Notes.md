## Introduction to Istio and Service Mesh ##

### What is Istio? Why Do We Need a Service Mesh? ###
**What is Istio?**
Istio is an *open-source service mesh* that provides *traffic management, security, and observability* for microservices running in Kubernetes. It acts as an *intermediary* between microservices, abstracting networking complexities and enforcing policies.

**Why Do We Need a Service Mesh?**
In a microservices architecture, applications are broken down into smaller, independent services. As the number of services grows, managing them becomes complex:
- Service Discovery – How do services find and communicate with each other?
- Traffic Routing – How do we ensure efficient load balancing, retries, and failovers?
- Security – How do we enforce authentication, authorization, and encryption between services?
- Observability – How do we track and debug requests flowing across multiple services?
A service mesh like Istio helps solve these challenges by automating service-to-service communication, ensuring security, and providing deep observability.

### Istio Components ###
Istio is made up of a *Control Plane* and a *Data Plane*:

**Control Plane (Manages and configures proxies)**
- Pilot – Manages traffic rules, service discovery, and routing.
- Citadel – Provides security, manages mTLS certificates.
- Galley – Validates and distributes configurations (deprecated in Istio 1.6+).

**Data Plane (Handles actual traffic)**
- Envoy Proxy – A lightweight proxy deployed as a *sidecar* to each service.
- Intercepts and manages service-to-service communication.

**How It Works:**
- The control plane tells Envoy proxies how to route traffic, enforce security policies, and collect telemetry data.
- The data plane (Envoy) actually moves the traffic between services based on these rules.

### Control Plane vs. Data Plane ###
![image](https://github.com/user-attachments/assets/65e91756-e28a-4869-a248-55f50fee9156)

### Installing Istio on Kubernetes ###
```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```

```bash
kubectl get pods -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istio-egressgateway-6c7d7d4747-vqmxg   1/1     Running   0          51s
istio-ingressgateway-9c954544f-n6th2   1/1     Running   0          51s
istiod-59c9d5b9b8-9xkmh                1/1     Running   0          73s
```

### Deploy a Microservice App and Enable Istio ###

![image](https://github.com/user-attachments/assets/7243928a-2451-499b-89d7-8db68edcadf8)


**Step 1: Deploy a Sample Microservice (Bookinfo App)**
```bash
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo
```
- Enable Istio Injection: Adding `istio-injection=enabled` label ensures sidecar Envoy proxies are automatically added to pods.

**Step 2: Expose the Application Using Istio Gateway**
```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo
```

**Step 3: Verify Everything is Running**
```bash
kubectl get pods -n istio-demo
kubectl get services -n istio-demo
kubectl get virtualservices -n istio-demo
kubectl get gateways -n istio-demo
```

### Hands-on: Verify Istio Installation ###

**Check Installed Istio Components**
```bash
kubectl get pods -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istio-egressgateway-6c7d7d4747-vqmxg   1/1     Running   0          5m32s
istio-ingressgateway-9c954544f-n6th2   1/1     Running   0          5m32s
istiod-59c9d5b9b8-9xkmh                1/1     Running   0          5m54s
```

## Traffic Management Basics in Istio ##

### Introduction to Istio Traffic Management ###
Istio *decouples traffic control* from your application logic, allowing you to *manage routing, load balancing, retries, and fault tolerance* without changing your microservices' code.
When multiple versions of a service exist, Istio provides *fine-grained traffic control* using its key networking resources:
- **VirtualService** - Defines routing rules.
- **DestinationRule** - Specifies load balancing and subset configurations.
- **Gateway** - Manages ingress/egress traffic.
- **ServiceEntry** - Allows access to external services.

### Key Istio Networking Resources ###

**VirtualService (VS)**
- A `VirtualService` defines *how requests are routed* to different versions of a microservice.
-  This configuration sends 80% of traffic to `reviews:v1` and 20% to `reviews:v2` (canary deployment).
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
```

**DestinationRule (DR)**
- A `DestinationRule` defines *load balancing policies* and *subsets* of a service.
- Defines subsets `v1` and `v2` for the `reviews` service and applies round-robin load balancing
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
  namespace: istio-demo
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```

**Gateway**
- A `Gateway` allows external access to services inside the mesh.
- Exposes HTTP traffic to the mesh through the Istio Ingress Gateway.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: istio-demo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

**ServiceEntry**
- A `ServiceEntry` allows Kubernetes pods to communicate with *external services*.
- Allows internal pods to access `www.google.com`.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-google
  namespace: istio-demo
spec:
  hosts:
  - www.google.com
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: HTTPS
```

### Traffic Splitting, Routing, and Retries ###

**Traffic Splitting (Canary Deployment)**
- Canary deployments gradually shift traffic between *old and new versions* of a service.
-  90% of traffic goes to `reviews:v1`, 10% to `reviews:v2`.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

**Header-Based and Cookie-Based Routing**
- Route requests based on user identity (cookies, headers).
- If `user=test-user`, route to `v2`; otherwise, use `v1`.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        user:
          exact: "test-user"
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

**Retries and Timeouts**
- *Retries*: Ensures Istio retries failed requests.
- *Timeouts*: Avoids infinite waiting.
- Retries up to 3 times if a 5xx error occurs, each attempt waiting up to 2s.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
```

### Hands-on: Deploy an App with Multiple Versions and Configure Routing ###

**Step 1: Deploy Bookinfo App**

**Step 2: Deploy Istio Gateway**

**Step 3: Define Destination Rules**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
  namespace: istio-demo
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```
```bash
kubectl apply -f destinationRule.yaml
```

**Step 4: Apply Traffic Splitting**
- Now, 80% of traffic goes to v1, and 20% to v2.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
```

## Advanced Traffic Routing in Istio ##

### Introduction to Advanced Traffic Routing ###
Now we focus on **fine-grained control** over traffic flow using:
- **Weight-based traffic routing** → Splitting traffic between different service versions.
- **Header-based & cookie-based routing** → Directing users based on headers or cookies.
- **Fault injection (delays & aborts)** → Testing failure scenarios.
- **Real-world traffic scenarios (Blue-Green, A/B Testing)** → Implementing modern deployment strategies.

### Weight-Based Traffic Routing (Canary Deployment) ###
Use Case: Gradually roll out a new version (`v2`) while keeping most traffic on the stable version (`v1`).

**VirtualService for Weight-Based Traffic Splitting**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
```

### Header-Based & Cookie-Based Traffic Routing ###
Use Case: Route traffic based on user identity (e.g., specific users see a beta version).

**Header-Based Routing Example**
-  If `user=test-user` is in the request header, route to `v2`, otherwise route to `v1`.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        user:
          exact: "test-user"
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

**Cookie-Based Routing Example**
Use Case: Sticky sessions (e.g., premium users see a new version).
-  If a user has a beta-user cookie, they get `v2`, otherwise `v1`.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        cookie:
          regex: ".*beta-user.*"
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```
- Test Cookie-Based Routing
```bash
curl --cookie "session=beta-user" http://$INGRESS_IP/productpage
```

### Fault Injection (Testing Failures) ###
Use Case: Simulate delays and errors to test resilience strategies.

**Injecting Delay (Latency)**
- Adds a 5s delay for all traffic to `reviews:v1`.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - fault:
      delay:
        fixedDelay: 5s
        percentage:
          value: 100
    route:
    - destination:
        host: reviews
        subset: v1
```
- Test Delay Injection: The request should take at least 5 seconds.
```bash
time curl http://$INGRESS_IP/productpage
```

**Injecting HTTP Errors (Abort Traffic)**
- Use Case: Simulate service failures to test fallback mechanisms.
- 50% of requests to `reviews:v1` will return a `500 Internal Server Error`.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 50
    route:
    - destination:
        host: reviews
        subset: v1
```
- Test Fault Injection: Around 5 out of 10 requests should return 500.
```bash
for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}\n" http://$INGRESS_IP/productpage; done
```

### Hands-On: Simulating Real-World Traffic Scenarios ###

**Scenario 1: Blue-Green Deployment**
- Route traffic to `v1`, then switch 100% to `v2` once tested.
- All traffic goes to v1.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```
- Now, all traffic is routed to v2.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
```

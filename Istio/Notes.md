# Day 1: Introduction to Istio and Service Mesh

## 1. What is Istio? Why Do We Need a Service Mesh?

### What is Istio?

Istio is an open-source service mesh that provides traffic management, security, and observability for microservices running in Kubernetes. It acts as an intermediary between microservices, abstracting networking complexities and enforcing policies.

### Why Do We Need a Service Mesh?

In a microservices architecture, applications are broken down into smaller, independent services. As the number of services grows, managing them becomes complex:

✅ **Service Discovery** – How do services find and communicate with each other?  
✅ **Traffic Routing** – How do we ensure efficient load balancing, retries, and failovers?  
✅ **Security** – How do we enforce authentication, authorization, and encryption between services?  
✅ **Observability** – How do we track and debug requests flowing across multiple services?  

A service mesh like Istio helps solve these challenges by automating service-to-service communication, ensuring security, and providing deep observability.

---

## 2. Istio Components

Istio is made up of a **Control Plane** and a **Data Plane**:

### Control Plane (Manages and configures proxies)

1️⃣ **Pilot** – Manages traffic rules, service discovery, and routing.  
2️⃣ **Citadel** – Provides security, manages mTLS certificates.  
3️⃣ **Galley** – Validates and distributes configurations (deprecated in Istio 1.6+).  

### Data Plane (Handles actual traffic)

1️⃣ **Envoy Proxy** – A lightweight proxy deployed as a sidecar to each service.  
2️⃣ **Intercepts and manages service-to-service communication.**  

👉 **How It Works:**  

- The control plane tells Envoy proxies how to route traffic, enforce security policies, and collect telemetry data.  
- The data plane (Envoy) actually moves the traffic between services based on these rules.  

---

## 3. Control Plane vs. Data Plane

| Feature | Control Plane (Istio) | Data Plane (Envoy) |
|---------|----------------------|----------------------|
| **Role** | Manages configuration & policy | Handles actual network traffic |
| **Components** | Pilot, Citadel, Galley (deprecated) | Envoy Proxy |
| **Traffic Management** | Defines routing rules | Applies rules & routes traffic |
| **Security** | Issues TLS certificates | Encrypts traffic using mTLS |
| **Observability** | Collects & processes metrics | Sends logs & metrics |

✅ **Analogy:**  

- The **control plane** is like an air traffic control tower giving directions.  
- The **data plane** is like airplanes following those directions.  

---

## 4. Installing Istio on Kubernetes

There are multiple ways to install Istio:

### 1️⃣ Using Istioctl (Recommended)
```sh
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```

### 2️⃣ Using Helm
```sh
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
helm install istio-base istio/base -n istio-system --create-namespace
helm install istiod istio/istiod -n istio-system
```

### 3️⃣ Using Istio Operator
```sh
kubectl apply -f https://github.com/istio/istio/releases/latest/download/istio-operator.yaml
```

✅ **Verify Istio Installation:**
```sh
kubectl get pods -n istio-system
```

---

## 5. Deploy a Microservice App and Enable Istio

### Step 1: Deploy a Sample Microservice (Bookinfo App)
```sh
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo
```

✅ **Enable Istio Injection:** Adding `istio-injection=enabled` label ensures sidecar Envoy proxies are automatically added to pods.

### Step 2: Expose the Application Using Istio Gateway
```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo
```

### Step 3: Verify Everything is Running
```sh
kubectl get pods -n istio-demo
kubectl get services -n istio-demo
kubectl get virtualservices -n istio-demo
kubectl get gateways -n istio-demo
```

---

## 6. Hands-on: Verify Istio Installation

### Check Installed Istio Components
```sh
kubectl get pods -n istio-system
```

🔹 **Expected Output:**
```sh
NAME                                    READY   STATUS    RESTARTS   AGE
istiod-xyz                              1/1     Running   0          5m
istio-ingressgateway-abc                1/1     Running   0          5m
```

✅ **If pods are running, Istio is successfully installed!**

---

## 📌 Day 1 Recap

✅ **Istio is a service mesh that simplifies traffic routing, security, and observability.**  
✅ **It consists of a control plane (Istiod) and a data plane (Envoy Proxy).**  
✅ **We installed Istio and deployed a sample app (Bookinfo) with automatic sidecar injection.**  
✅ **We set up an Istio Gateway to expose the app externally.**  

🚀 **Next Up: Day 2 - Traffic Management Basics!**  

# Day 2: Traffic Management Basics in Istio

## 1. Introduction to Istio Traffic Management

Istio decouples traffic control from your application logic, allowing you to manage routing, load balancing, retries, and fault tolerance without changing your microservices' code.

When multiple versions of a service exist, Istio provides fine-grained traffic control using its key networking resources:

- **VirtualService** - Defines routing rules.
- **DestinationRule** - Specifies load balancing and subset configurations.
- **Gateway** - Manages ingress/egress traffic.
- **ServiceEntry** - Allows access to external services.

## 2. Key Istio Networking Resources

### 🔹 VirtualService (VS)

A VirtualService defines how requests are routed to different versions of a microservice.

Example:

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

🔹 This configuration sends **80%** of traffic to **reviews:v1** and **20%** to **reviews:v2** (canary deployment).

### 🔹 DestinationRule (DR)

A DestinationRule defines load balancing policies and subsets of a service.

Example:

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

🔹 Defines subsets v1 and v2 for the **reviews** service and applies **round-robin** load balancing.

### 🔹 Gateway

A Gateway allows external access to services inside the mesh.

Example:

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

🔹 Exposes HTTP traffic to the mesh through the **Istio Ingress Gateway**.

### 🔹 ServiceEntry

A ServiceEntry allows Kubernetes pods to communicate with external services.

Example:

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

🔹 Allows internal pods to access **www.google.com**.

## 3. Traffic Splitting, Routing, and Retries

### 🔹 Traffic Splitting (Canary Deployment)

Canary deployments gradually shift traffic between old and new versions of a service.

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

🔹 90% of traffic goes to **reviews:v1**, 10% to **reviews:v2**.

### 🔹 Header-Based and Cookie-Based Routing

Route requests based on user identity (cookies, headers).

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

🔹 If `user=test-user`, route to **v2**; otherwise, use **v1**.

### 🔹 Retries and Timeouts

- **Retries**: Ensures Istio retries failed requests.
- **Timeouts**: Avoids infinite waiting.

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

🔹 Retries up to **3 times** if a **5xx** error occurs, each attempt waiting up to **2s**.

## 4. Hands-on: Deploy an App with Multiple Versions and Configure Routing

### Step 1: Deploy Bookinfo App

```sh
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo
```

### Step 2: Deploy Istio Gateway

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo
```

### Step 3: Define Destination Rules

```sh
kubectl apply -f - <<EOF
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
EOF
```

### Step 4: Apply Traffic Splitting

```sh
kubectl apply -f - <<EOF
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
EOF
```

✅ Now, **80% of traffic** goes to **v1**, and **20%** to **v2**.

### Step 5: Verify Traffic Distribution

```sh
for i in {1..10}; do curl http://<INGRESS-IP>/productpage; done
```

🔹 Check responses to see different versions!

## 5. Recap

✅ **VirtualService**: Defines routing rules.  
✅ **DestinationRule**: Defines subsets and load balancing policies.  
✅ **Gateway**: Manages external access.  
✅ **ServiceEntry**: Allows access to external services.  
✅ **Traffic splitting, header-based routing, retries, and timeouts** help optimize traffic flow.  

🚀 **Next Up: Day 3 - Advanced Traffic Routing!**  

# Day 3: Advanced Traffic Routing in Istio

## 1. Introduction to Advanced Traffic Routing

After learning basic traffic management on Day 2, today we focus on fine-grained control over traffic flow using:

- **Weight-based traffic routing** → Splitting traffic between different service versions.
- **Header-based & cookie-based routing** → Directing users based on headers or cookies.
- **Fault injection (delays & aborts)** → Testing failure scenarios.
- **Real-world traffic scenarios (Blue-Green, A/B Testing)** → Implementing modern deployment strategies.

## 2. Weight-Based Traffic Routing (Canary Deployment)

💡 **Use Case:** Gradually roll out a new version (v2) while keeping most traffic on the stable version (v1).

### VirtualService for Weight-Based Traffic Splitting

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

🔹 **80% of traffic** goes to v1, **20% to v2**.  
🔹 **Use this strategy for Canary Deployments.**  

### Verify Weight-Based Routing

```sh
for i in {1..10}; do curl http://$INGRESS_IP/productpage; done
```

💡 **Expected Behavior:** ~8 requests should hit v1, ~2 should hit v2.

## 3. Header-Based & Cookie-Based Traffic Routing

💡 **Use Case:** Route traffic based on user identity (e.g., specific users see a beta version).

### Header-Based Routing Example

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

🔹 If `user=test-user` is in the request header, **route to v2**, otherwise route to v1.

#### Test Header-Based Routing

```sh
curl -H "user: test-user" http://$INGRESS_IP/productpage
```

💡 **Expected Behavior:** The response should come from v2.

### Cookie-Based Routing Example

💡 **Use Case:** Sticky sessions (e.g., premium users see a new version).

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

🔹 If a user has a `beta-user` cookie, they get **v2**, otherwise **v1**.

#### Test Cookie-Based Routing

```sh
curl --cookie "session=beta-user" http://$INGRESS_IP/productpage
```

💡 **Expected Behavior:** If the cookie matches, the request should be routed to v2.

## 4. Fault Injection (Testing Failures)

💡 **Use Case:** Simulate delays and errors to test resilience strategies.

### Injecting Delay (Latency)

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

🔹 Adds a **5s delay** for all traffic to `reviews:v1`.

#### Test Delay Injection

```sh
time curl http://$INGRESS_IP/productpage
```

💡 **Expected Behavior:** The request should take **at least 5 seconds**.

### Injecting HTTP Errors (Abort Traffic)

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

🔹 **50% of requests** to `reviews:v1` will return a **500 Internal Server Error**.

#### Test Fault Injection

```sh
for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}
" http://$INGRESS_IP/productpage; done
```

💡 **Expected Behavior:** Around **5 out of 10 requests** should return **500**.

## 5. Hands-On: Simulating Real-World Traffic Scenarios

### Scenario 1: Blue-Green Deployment

💡 **Goal:** Route traffic to v1, then switch 100% to v2 once tested.

#### Step 1: Deploy Both Versions

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

🔹 **All traffic goes to v1.**

#### Step 2: Switch to v2

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

🔹 **Now, all traffic is routed to v2.**

✅ **Blue-Green Deployment Complete!**

### Scenario 2: A/B Testing

💡 **Goal:** Let **10% of users** try a new feature before full rollout.

#### Step 1: Weight-Based Routing for A/B Test

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
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

🔹 **90% of users** see `v1`, **10% see v2**.

#### Step 2: Validate A/B Test

```sh
for i in {1..10}; do curl http://$INGRESS_IP/productpage; done
```

💡 **Expected Behavior:** Around **1 request in 10** should hit **v2**.

## 6. Recap of Day 3

✅ **Weight-Based Routing** → Gradually shift traffic (**Canary Deployments**).  
✅ **Header-Based & Cookie-Based Routing** → Personalize user experience.  
✅ **Fault Injection** → Simulate failures (**delays, HTTP errors**).  
✅ **Real-World Traffic Strategies** → **Blue-Green, A/B Testing**.  

🚀 **Next Up: Day 4 - Security & Authentication in Istio!** Let me know if you need deeper explanations.

# Day 4: Istio Security (Authentication & Authorization)

Security is a core feature of Istio that helps in securing communication between services inside a Kubernetes cluster.

## What You'll Learn Today

✅ Mutual TLS (mTLS) → Encrypt traffic between microservices.  
✅ Istio Authorization Policies (RBAC) → Control access between services.  
✅ JWT Authentication → Secure services with JSON Web Tokens.  
✅ Hands-on: Implement mTLS and RBAC in Istio.  

---

## 1. Mutual TLS (mTLS) in Istio

### What is mTLS?

🔹 In a typical microservices setup, services communicate over HTTP. But this is insecure!  
🔹 mTLS (Mutual TLS) ensures:

- **Encryption:** Traffic is encrypted between services.
- **Authentication:** Both client and server verify each other.
- **No need for application-level TLS:** Istio handles it automatically.

### How mTLS Works in Istio

1. **Service A (Client) → Service B (Server)**
2. Istio **Envoy Proxies** manage certificates & encryption.
3. Both services verify each other's identity using **TLS certificates**.
4. Communication is **secure & encrypted**.

---

## 2. Enforcing mTLS in Istio

### Step 1: Enable mTLS in Strict Mode

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-demo
spec:
  mtls:
    mode: STRICT
```

🔹 **STRICT Mode:** Services must use **mTLS** (no plain HTTP allowed).  
🔹 **PERMISSIVE Mode:** Accepts both **HTTP & mTLS** (for gradual migration).  

### Step 2: Verify mTLS is Working

Run the following command:

```bash
kubectl get peerauthentication -n istio-demo
```

🔹 If **mTLS is enabled**, all traffic inside the namespace will be encrypted.  

### Step 3: Test HTTP Request Without TLS

```bash
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -v http://reviews:9080
```

💡 **Expected Output**: The request should fail because **HTTP is blocked**.

---

## 3. Istio Authorization Policies (RBAC)

### What is RBAC in Istio?

Role-Based Access Control (**RBAC**) allows you to:

✅ **Limit which services can talk to each other**.  
✅ **Control which users have access to APIs**.  

### RBAC Example: Allow Only `productpage` to Access `reviews`

#### Step 1: Define Authorization Policy

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-demo/sa/productpage"]
```

🔹 This **only allows** `productpage` service to access `reviews`.  
🔹 All **other services** will be **denied** access.  

#### Step 2: Verify RBAC Policy

Run:

```bash
kubectl get authorizationpolicy -n istio-demo
```

💡 **Expected Output:** Shows the **RBAC policy is applied**.  

#### Step 3: Test the Policy

✅ **Test from `productpage` (allowed):**

```bash
kubectl exec -it $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') -c productpage -- curl -v http://reviews:9080
```

💡 **Expected:** Success.  

❌ **Test from `ratings` (denied):**

```bash
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -v http://reviews:9080
```

💡 **Expected:** Access Denied (`403 Forbidden`).

---

## 4. JWT Authentication

### What is JWT?

JWT (**JSON Web Token**) allows users to prove their identity via tokens.  
✅ Used for API security and **authentication**.  
✅ Istio can validate **JWT tokens** automatically.  

### Configuring JWT Authentication in Istio

#### Step 1: Apply JWT Policy

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: productpage
  jwtRules:
  - issuer: "https://example.com"
    jwksUri: "https://example.com/.well-known/jwks.json"
```

🔹 This requires a **valid JWT token** to access `productpage`.  
🔹 JWT tokens must be issued by `"https://example.com"`.  

#### Step 2: Test Authentication

✅ **Access Without JWT (Should Fail):**

```bash
curl -v http://$INGRESS_IP/productpage
```

💡 **Expected:** `401 Unauthorized`.  

✅ **Access With JWT (Should Succeed):**

```bash
TOKEN=$(curl https://example.com/auth -d 'user=admin&pass=secret' | jq -r '.token')
curl -H "Authorization: Bearer $TOKEN" http://$INGRESS_IP/productpage
```

💡 **Expected:** The request succeeds.  

---

## 5. Hands-On: Secure Microservices with Istio

✅ **Step 1:** Enable **mTLS** across services.  
✅ **Step 2:** Restrict access using **RBAC policies**.  
✅ **Step 3:** Implement **JWT authentication** for API security.  
✅ **Step 4:** Test all security policies using `curl` commands.  

---

## 6. Recap of Day 4

🔹 **mTLS** → Encrypts traffic between microservices.  
🔹 **RBAC Policies** → Controls which services can talk to each other.  
🔹 **JWT Authentication** → Requires valid **tokens** for access.  
🔹 **Hands-on Practice** → Applied mTLS, RBAC, and JWT authentication.  

🚀 **Next Up: Day 5 - Observability in Istio!**




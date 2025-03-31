**What is Istio?**

Istio is an open-source service mesh that provides traffic management, security, and observability for microservices running in Kubernetes. It acts as an intermediary between microservices, abstracting networking complexities and enforcing policies.

**Why Do We Need a Service Mesh?**

In a microservices architecture, applications are broken down into smaller, independent services. As the number of services grows, managing them becomes complex:
- Service Discovery – How do services find and communicate with each other?  
- Traffic Routing – How do we ensure efficient load balancing, retries, and failovers?  
- Security – How do we enforce authentication, authorization, and encryption between services?  
- Observability – How do we track and debug requests flowing across multiple services?  

A service mesh like Istio helps solve these challenges by automating service-to-service communication, ensuring security, and providing deep observability.

**Istio Components**

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

**Deploy a Microservice App and Enable Istio**

Step 1: Deploy a Sample Microservice (Bookinfo App)
```sh
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo
```

Enable Istio Injection: Adding `istio-injection=enabled` label ensures sidecar Envoy proxies are automatically added to pods.

Step 2: Expose the Application Using Istio Gateway
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


# Day 5: Ingress and Egress Gateway in Istio

## 1. Understanding Istio Gateway vs Kubernetes Ingress

### Kubernetes Ingress
- **Kubernetes Ingress** is a native Kubernetes resource that manages external access to services in a cluster.
- Uses an **Ingress Controller** (e.g., NGINX, Traefik) to handle HTTP/S traffic.
- Works with standard Kubernetes Services (ClusterIP, NodePort, LoadBalancer).
- Basic routing capabilities:  
  - Path-based routing  
  - Host-based routing  
  - TLS termination  

### Istio Gateway
- **Istio Gateway** is an Istio-managed component that controls incoming and outgoing traffic at the edge of the service mesh.
- Unlike Kubernetes Ingress, **Istio Gateway only configures the L4-L6 networking** (TLS, TCP, WebSocket) and relies on **VirtualServices for HTTP routing**.
- Works with **Istio-proxy (Envoy) as the ingress controller** instead of NGINX/Traefik.
- Offers **more advanced traffic management**, including:  
  - Traffic shaping (timeouts, retries, rate limits).  
  - Security (mTLS, JWT authentication).  
  - Advanced routing (headers, cookies, fault injection).  
  - Fine-grained control over egress traffic.  

### Key Differences Between Kubernetes Ingress and Istio Gateway

| Feature                 | Kubernetes Ingress  | Istio Gateway  |
|-------------------------|--------------------|---------------|
| Ingress Controller      | NGINX, Traefik, etc. | Envoy (Istio Proxy) |
| L7 Routing Control     | Path, Host         | VirtualService |
| Traffic Shaping        | Limited           | Advanced (Timeouts, Retries, Mirroring) |
| mTLS (Mutual TLS)      | Not supported     | Supported |
| JWT Authentication     | Not built-in      | Built-in |
| Fine-grained Egress    | Not available     | Supports Egress Gateway |
| External Service Access | Allowed directly  | Needs ServiceEntry & Egress Gateway |

---

## 2. Configuring Ingress Gateway for External Traffic

💡 **Scenario:** You have a microservice running inside the Istio service mesh, and you want to expose it to the outside world securely.

### Step 1: Deploy an Application in Istio

```sh
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml -n istio-demo
```

### Step 2: Define an Istio Ingress Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
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

Apply it:

```sh
kubectl apply -f httpbin-gateway.yaml
```

### Step 3: Configure a VirtualService

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
  namespace: istio-demo
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    route:
    - destination:
        host: httpbin
        port:
          number: 8000
```

Apply it:

```sh
kubectl apply -f httpbin-virtualservice.yaml
```

### Step 4: Get the External IP and Test

```sh
export INGRESS_IP=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$INGRESS_IP/status/200
```

✅ **Expected Result:** The `httpbin` service should return a `200 OK` response.

---

## 3. Allowing/Blocking External Services Using Egress Gateway

💡 **Scenario:** By default, Istio **blocks all external traffic**. We will allow **specific external services** using a **ServiceEntry** and an **Egress Gateway**.

### Step 1: Block External Traffic

```sh
kubectl exec -it <pod-name> -n istio-demo -- curl -I https://www.google.com
```

🔴 **Expected Result:** Request fails (`connection reset`).

### Step 2: Allow External Traffic Using a ServiceEntry

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: allow-google
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

Apply it:

```sh
kubectl apply -f allow-google.yaml
```

Test again:

```sh
kubectl exec -it <pod-name> -n istio-demo -- curl -I https://www.google.com
```

✅ **Expected Result:** The request should succeed.

### Step 3: Secure External Access via an Egress Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: egress-gateway
  namespace: istio-demo
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - www.google.com
```

Apply it:

```sh
kubectl apply -f egress-gateway.yaml
```

Now, configure the **VirtualService** to route outbound traffic via the **Egress Gateway**.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: google-egress
  namespace: istio-demo
spec:
  hosts:
  - www.google.com
  gateways:
  - egress-gateway
  tcp:
  - match:
    - port: 443
    route:
    - destination:
        host: www.google.com
        port:
          number: 443
```

Apply it:

```sh
kubectl apply -f google-egress-virtualservice.yaml
```

🔹 Now, all traffic to `www.google.com` is routed through the Egress Gateway.

✅ **Final Test:**

```sh
kubectl exec -it <pod-name> -n istio-demo -- curl -I https://www.google.com
```

---

## 4. Recap of Day 5

✅ **Ingress Gateway:** Securely exposes internal services to external clients.  
✅ **VirtualService + Gateway:** Handles external traffic flow.  
✅ **ServiceEntry:** Allows external traffic access.  
✅ **Egress Gateway:** Routes outbound traffic through a controlled proxy.  


# **Day 6: Resilience and Reliability in Istio**  

Modern microservices architectures require robust resilience and reliability strategies to prevent cascading failures. Istio provides built-in features like **circuit breaking, retries, timeouts, and outlier detection** to ensure your services remain reliable under various failure conditions.

---

## **1. Circuit Breaking with Istio**  

### **What is Circuit Breaking?**  
Circuit breaking prevents services from being overwhelmed when there are failures. If a service is experiencing high failure rates, Istio temporarily **blocks** requests to that service instead of retrying indefinitely. This helps avoid **service degradation** and allows the system to recover.  

### **How Istio Implements Circuit Breaking**  
Istio’s **DestinationRule** allows you to define circuit-breaking policies for your microservices.  

### **Example: Basic Circuit Breaker Policy**  

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-circuit-breaker
  namespace: istio-demo
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 5
      http:
        http1MaxPendingRequests: 3
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
```

### **Explanation**  
- **maxConnections: 5** → Allows a max of **5 concurrent TCP connections**.  
- **http1MaxPendingRequests: 3** → Only **3 pending HTTP requests** are allowed.  
- **maxRequestsPerConnection: 1** → Each connection handles only **one request** at a time.  
- **outlierDetection** → If a service instance returns **3 consecutive 5xx errors** within **10 seconds**, it gets removed for **30 seconds**.  

### **Testing Circuit Breaking**  
1. Send multiple concurrent requests:  

   ```bash
   for i in {1..20}; do curl http://$INGRESS_IP/productpage & done
   ```

2. If circuit breaking is working, some requests will **fail fast** instead of hanging indefinitely.  

---

## **2. Retry Policies and Timeouts**  

### **What are Retries and Timeouts?**  
- **Retries** → If a request fails, Istio can retry automatically before reporting failure.  
- **Timeouts** → Prevents infinite waiting by enforcing an upper limit on request duration.  

### **Example: Setting Retries and Timeouts**  

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
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
```

### **Explanation**  
- **retries.attempts: 3** → Retries **3 times** before giving up.  
- **perTryTimeout: 2s** → Each retry attempt waits **up to 2 seconds**.  
- **retryOn: 5xx** → Retries only if the response has a **5xx error (server failure)**.  

### **Testing Retries and Timeouts**  
1. Simulate failures using **fault injection**:  

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
             value: 100
       route:
       - destination:
           host: reviews
           subset: v1
   ```

2. Make a request and observe retry behavior:  

   ```bash
   curl -v http://$INGRESS_IP/productpage
   ```

3. Expected result: Istio should retry **3 times** before failing.

---

## **3. Outlier Detection to Remove Faulty Instances**  

### **What is Outlier Detection?**  
Outlier detection automatically **removes unhealthy instances** from the service pool if they repeatedly fail, preventing cascading failures.  

### **Example: Configure Outlier Detection**  

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-outlier-detection
  namespace: istio-demo
spec:
  host: reviews
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

### **Explanation**  
- **consecutive5xxErrors: 3** → If an instance **fails 3 times** in a row, it is removed.  
- **interval: 10s** → Checks failure every **10 seconds**.  
- **baseEjectionTime: 30s** → Ejected instances remain **out of service for 30 seconds**.  
- **maxEjectionPercent: 50** → At most **50% of instances** can be removed at a time.  

### **Testing Outlier Detection**  
1. Apply the configuration and simulate failures:  

   ```bash
   for i in {1..10}; do curl http://$INGRESS_IP/productpage; done
   ```

2. Check if some service instances get **removed** from the Istio service mesh.  

---

## **4. Hands-on: Implement Circuit Breaker and Observe Failure Handling**  

### **Step 1: Deploy the Bookinfo Application**  

```bash
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo
```

### **Step 2: Apply Circuit Breaking Policy**  

```yaml
# Apply this DestinationRule to enable circuit breaking
```

### **Step 3: Simulate High Load**  

```bash
for i in {1..20}; do curl http://$INGRESS_IP/productpage & done
```

### **Step 4: Observe Failure Handling**  

Check logs:  

```bash
kubectl logs -l app=reviews -n istio-demo
```

Check which instances are ejected:  

```bash
kubectl get pods -n istio-demo
```

---

## **5. Recap of Day 6**  

✅ **Circuit Breaking** → Prevents overloaded services from crashing.  
✅ **Retries & Timeouts** → Ensures services automatically recover from transient failures.  
✅ **Outlier Detection** → Removes failing instances before they cause more issues.  
✅ **Hands-on Exercise** → Implemented circuit breakers and tested failure handling.  


### DAY 8 Multi-Cluster Deployment in Istio ###
Deploying a service mesh across multiple clusters enables high availability, disaster recovery, and traffic management across different regions or cloud providers.

Prerequisites:
- At least two Kubernetes clusters (Cluster A & Cluster B).
- Istio installed on both clusters.
- A single control plane or multi-control plane setup.

*Multi-Cluster Topologies*

There are two common architectures:
- Primary-Remote (Single Control Plane)
  - One cluster (primary) hosts the Istio control plane (istiod).
  - Other clusters (remote) host workloads and connect to the control plane.
  - Easier to manage but introduces latency for control plane operations.
- Multi-Primary (Multi-Control Plane)
  - Each cluster runs its own control plane.
  - Federation is used to synchronize configurations.
  - Better for fault tolerance but requires more configuration.
 
*Deploy Istio in Multi-Cluster Mode*

Example: Primary-Remote Setup
- Install Istio on the Primary Cluster (Cluster A)
  ```bash
  istioctl install --set profile=default -y
  ```
- Enable Multi-Cluster API Server Access
  - Ensure Cluster A's API server is accessible from Cluster B.
  - Generate a service account token in Cluster A and apply it in Cluster B.
- Install Istio in Remote Cluster (Cluster B)
  ```bash
  istioctl install --set values.global.meshID=mesh1 \
    --set values.global.multiCluster.clusterName=cluster-b \
    --set values.global.network=network2 \
    --set profile=remote -y
  ```
- Connect the Clusters
  - Export the primary cluster’s service discovery and credentials.
  - Apply them in the remote cluster.
- Verify Cross-Cluster Communication
  ```bash
  kubectl get svc -n istio-system
  ```



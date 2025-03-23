- A Network Policy is a Kubernetes object that controls traffic flow to/from Pods based on rules.
- By default, all traffic is allowed between Pods in a cluster. Network Policies restrict unwanted traffic.

---

- In the namespace **ckad-netpol**, create a NetworkPolicy named **allow-only-frontend**.
- This policy should allow ingress traffic to pods with the label `role=backend` only from pods with the `label role=frontend` on port 8080.

```
||||||||||||||||||||||                          ||||||||||||||||||||||
|                    |                          |                    |
|  Role = Frontend   | ---------------------->  |  Role = Backend    |
|                    |                          |                    |
||||||||||||||||||||||                          |||||||||||||||||||||| 
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-frontend
  namespace: ckad-netpol
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

Your team has deployed a multi-tenant application on Kubernetes, and each tenant has a separate namespace. 
A security team reports that one tenant's application is trying to access another tenant's database. How would you investigate and prevent this from happening while keeping minimal disruption to the system?

**Network Policy to Isolate Tenants**
- This NetworkPolicy denies all incoming traffic (ingress) to any pod in the `tenant2` namespace(DB Namespace).
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-access-from-other-namespaces
  namespace: tenant2
spec:
  podSelector: {}
  ingress:
    - from: []
  policyTypes:
    - Ingress
```

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

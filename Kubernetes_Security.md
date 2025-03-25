### Network Policy ###
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


**How Encryption at Rest Works**
Kubernetes protects sensitive data, such as Secrets, by encrypting them before storing them in etcd. When a request is made to access a Secret, Kubernetes decrypts it in real-time, ensuring security while maintaining accessibility.

- Create an Encryption Configuration File
  - On the control plane node, create an encryption configuration file at `/etc/kubernetes/encryption-config.yaml`
  - aescbc: Uses AES-CBC encryption to secure Secrets.
  - identity: Leaves Secrets unencrypted for backward compatibility.
  ```yaml
  apiVersion: apiserver.config.k8s.io/v1
  kind: EncryptionConfiguration
  resources:
    - resources:
        - secrets
      providers:
        - aescbc:
            keys:
              - name: key1
                secret: c2VjcmV0LWtleS1oZXJlCg==  # Replace with a real base64-encoded key
        - identity: {}
  ```

  - Configure the API Server to Use Encryption
    - Edit the API server manifest, typically found at: `/etc/kubernetes/manifests/kube-apiserver.yaml`
    - Add the following flag to enable encryption:
    ```yaml
    --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    ```

  - Restart the API Server
    ```bash
    systemctl restart kubelet
    ```

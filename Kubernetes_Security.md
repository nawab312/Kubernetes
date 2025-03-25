### Authentication & Authorization ###
**Role-Based Access Control (RBAC) in Kubernetes**
RBAC is a security mechanism in Kubernetes that controls who (subjects) can perform what actions (verbs) on which resources (objects) in a cluster. Key RBAC Components
- *Role* – Defines permissions for resources within a namespace.
- *ClusterRole* – Similar to a Role but applies to the entire cluster.
- *RoleBinding* – Binds a Role to a User/Group/ServiceAccount within a namespace.
- *ClusterRoleBinding* – Binds a ClusterRole to a User/Group/ServiceAccount at the cluster level.

RBAC Example: Granting Read-Only Access to Pods
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: dev-user  # The user getting the permission
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Service Accounts**
Service Account is a special type of Kubernetes account used by pods to interact with the Kubernetes API server and other resources in a Kubernetes cluster. 
It allows applications running inside the cluster to authenticate to the Kubernetes API and access cluster resources in a controlled and secure manner.
- Default Service Account
  - Every pod automatically gets a default service account from its namespace unless specified otherwise.
    ```bash
    kubectl get serviceaccount default -n my-namespace
    ```
Key Characteristics of Service Accounts in Kubernetes:
- Identity for Pods: Service accounts provide a unique identity for applications (running inside pods) to interact with the Kubernetes API server. This identity can be used to manage access to Kubernetes resources.
- Automatic Token Creation: Kubernetes automatically creates a service account token for each service account. This token is used for authenticating requests from the pods to the Kubernetes API server.
- RBAC (Role-Based Access Control): Service accounts are often assigned roles via RBAC to specify what actions they can perform within the cluster. For example, a service account might be granted permission to read from a Kubernetes config map, create deployments, or access other resourcess
- Namespace Bound: Service accounts are created in specific namespaces within a Kubernetes cluster. When a pod is deployed, it is assigned a service account from the namespace in which it is running, and this service account governs the permissions and access for that pod.

Creating a Custom Service Account & Binding it to a Pod
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: custom-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sa-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: custom-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
Usage in a Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: custom-sa
  containers:
  - name: app-container
    image: my-app:latest
```

**Kubernetes API Authentication Methods**
- Kubernetes supports multiple authentication methods for users and services:

*Certificate-Based Authentication*
- Uses client certificates to authenticate users.
- Certificates are signed by the Kubernetes API server's Certificate Authority (CA).
```bash
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=user1/O=dev-team"
```
- The CN (Common Name) becomes the username, and O (Organization) can be mapped to RBAC groups.

*Token-Based Authentication*
- Service Accounts use tokens to authenticate against the API server.
- Get a token for a service account:
```bash
kubectl get secret $(kubectl get serviceaccount my-sa -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 --decode
```

### Network Security ###
**Network Policy**
- A Network Policy is a Kubernetes object that controls traffic flow to/from Pods based on rules.
- By default, all traffic is allowed between Pods in a cluster. Network Policies restrict unwanted traffic.

---

- In the namespace *ckad-netpol*, create a NetworkPolicy named *allow-only-frontend*.
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

*Network Policy to Isolate Tenants*
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

### Pod & Container Security ###
**Security Contexts & Privileged Containers**
- A security context in Kubernetes defines privilege and access control settings for a pod or container. It prevents privilege escalation and restricts resource access

Setting a Security Context for a Pod: Example of restricting a pod from running as root:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000   # Run as non-root user
    runAsGroup: 3000
    fsGroup: 2000     # Group for file system access
  containers:
  - name: my-app
    image: my-app:latest
    securityContext:
      allowPrivilegeEscalation: false  # Prevent privilege escalation
      readOnlyRootFilesystem: true  # Root filesystem is read-only
```

**Pod Security Policies (PSP) and Pod Security Admission (PSA)**
*Pod Security Policies (PSP)*
- Deprecated in Kubernetes 1.21 and removed in Kubernetes 1.25.

*Pod Security Admission (PSA)*
- It enforces security policies at the namespace level.
- Three security levels:
  - `privileged` → Least restrictive, allows privileged pods.
  - `baseline` → Blocks some unsafe configurations but allows necessary privileges.
  - `restricted` → Most secure, enforces strict security policies.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: "restricted"
```
**Protecting Kubernetes Secrets**
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

### Cluster Security & Best Practices ###
Securing a Kubernetes cluster involves hardening the cluster, securing the etcd datastore, scanning container images, and monitoring runtime security.

**Kubernetes CIS Benchmark Security Hardening**

The CIS Kubernetes Benchmark provides best practices to secure a Kubernetes cluster. Key areas include:

*API Server Security Hardening*
- The Kubernetes API server is the entry point for all cluster operations. Best Practices:
- Enable Authentication & RBAC `--authorization-mode=RBAC`
- Limit anonymous access `--anonymous-auth=false`
- Disable insecure port (Default: 8080) `--insecure-port=0`
- Enable Audit Logging to track API requests `--audit-log-path=/var/log/kubernetes/kube-apiserver-audit.log`

*Kubelet Security Hardening*
- The Kubelet manages pods on nodes. Best Practices:
- Disable anonymous access `--anonymous-auth=false`
- Use client certificates for authentication `--client-ca-file=/etc/kubernetes/pki/ca.crt`
- Restrict access to the kubelet API `--authorization-mode=Webhook`

*Etcd Security Hardening*
- Enable etcd authentication `--client-cert-auth --trusted-ca-file=/etc/etcd/ca.crt`

*Worker Node Security Hardening*
- Use AppArmor, Seccomp, or SELinux to restrict container capabilities.
- Disable root user access in pods using security contexts.
- Use a non-root user for all containers.

**Securing etcd (Kubernetes Data Store)**

*Securing etcd Access*
- Enable authentication & authorization
  ```yaml
  --client-cert-auth=true
  --trusted-ca-file=/etc/kubernetes/pki/ca.crt
  ```
- Restrict access to etcd (only control plane should access it).
- Use a firewall to block unauthorized access to port 2379.






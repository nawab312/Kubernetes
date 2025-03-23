Service Account is a special type of Kubernetes account used by pods to interact with the Kubernetes API server and other resources in a Kubernetes cluster. 
It allows applications running inside the cluster to authenticate to the Kubernetes API and access cluster resources in a controlled and secure manner.

**Key Characteristics of Service Accounts in Kubernetes:**
- Identity for Pods: Service accounts provide a unique identity for applications (running inside pods) to interact with the Kubernetes API server. This identity can be used to manage access to Kubernetes resources.
- Automatic Token Creation: Kubernetes automatically creates a service account token for each service account. This token is used for authenticating requests from the pods to the Kubernetes API server.
- RBAC (Role-Based Access Control): Service accounts are often assigned roles via RBAC to specify what actions they can perform within the cluster. For example, a service account might be granted permission to read from a Kubernetes config map, create deployments, or access other resourcess
- Namespace Bound: Service accounts are created in specific namespaces within a Kubernetes cluster. When a pod is deployed, it is assigned a service account from the namespace in which it is running, and this service account governs the permissions and access for that pod.

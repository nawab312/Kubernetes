### Introduction to Kubernetes ###
- **What is Kubernetes?**
- **Kubernetes Architecture**

### Kubernetes Components ###
- **Pods**
- **ReplicaSets**
- **Deployments**
- **Namespaces**
- **Services**
- **ConfigMaps and Secrets**
- **StatefulSets**
- **DaemonSets**
- **Jobs and CronJobs**

### Kubernetes Networking ###
- **Cluster Networking Model**
- **Service Discovery**
- **Types of Services (ClusterIP, NodePort, LoadBalancer, ExternalName)**
- **Network Policies**



 
**Storage in Kubernetes** https://github.com/nawab312/Kubernetes/blob/main/Storage/Notes.md



### Kubernetes Scheduling & Resource Management ###
- **Scheduler Overview**
- **Pods and Resource Requests/Limitations**
- **Affinity and Taints/Tolerations**
- **Horizontal Pod Autoscaling**
- **Cluster Autoscaler**
- **Resource Quotas**
- https://github.com/nawab312/Kubernetes/blob/main/Kubernetes_Scheduling_Resource_Management.md

### Kubernetes Security ###
- **RBAC (Role-Based Access Control)**
- **Network Policies**
- **Service Accounts**
- **Pod Security Policies (PSP)**
- **Secrets Management**
- **Security Contexts**
- **Kubernetes and CIS Benchmarks**

### Kubernetes Deployments & Management ###
- **Deployment Strategies (Rolling, Recreate, Blue-Green, Canary)**
- **Helm Charts (Introduction, Installation, and Usage)**
- **Kubectl CLI Commands (Common Commands)**
- **Troubleshooting Kubernetes (Logs, Events, Describe, etc.)**
- **Continuous Integration and Continuous Delivery (CI/CD) with Kubernetes**
- https://github.com/nawab312/Kubernetes/blob/main/Kubernetes_Deployments_Management.md

### Kubernetes Troubleshooting & Debugging ###
- **Debugging Pods and Containers**
- **Common Issues and Solutions (CrashLoopBackOff, ImagePullBackOff, etc.)**
- **Using kubectl for Troubleshooting**
- https://github.com/nawab312/Kubernetes/blob/main/Kubernetes_TroubleShooting_Debugging.md

### Kubernetes Scheduling ###

**Resource Requests and Limits**
- Resource Requests:
    - Specify the minimum amount of CPU and memory a Pod needs to run.
    - The scheduler ensures the node has at least the requested resources available
- Resource Limits:
    - Specify the maximum amount of CPU and memory a Pod can use.
    - Prevents a Pod from monopolizing resources.
https://github.com/nawab312/Kubernetes/blob/main/Pods/Pod-1.yaml

**Node Affinity & Anti-Affinity**
- Node Affinity: A way to specify which nodes a Pod prefers to run on based on node labels.
- Node Anti-Affinity: Opposite of affinity; ensures Pods are scheduled on separate nodes, useful for high availability.
https://github.com/nawab312/Kubernetes/blob/main/Pods/Pod-2.yaml

**Taints and Tolerations**
- A taint is a way to mark a node so that only certain Pods can be scheduled on it.
- `kubectl taint nodes node1 gpu=true:NoSchedule` Only Pods that tolerate the gpu=true taint can be scheduled on node1.
- A toleration is a way for a Pod to indicate that it can tolerate (or accept) a specific taint on a node.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myPod
spec:
  tolerations:
    - key: "gpu"
      operator: "equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: tensorflow
      image: tensorflow:latest
```
![image](https://github.com/user-attachments/assets/6466be37-1dea-4296-aac6-2bcab920e105)

### Kubernetes Security ###
- **RBAC (Role-Based Access Control)**
- **Network Policies** https://github.com/nawab312/Kubernetes/blob/main/Security/NetworkPolicy.md
- **ServiceAccount** https://github.com/nawab312/Kubernetes/blob/main/Security/ServiceAccount.md


**Ingress** https://github.com/nawab312/Kubernetes/blob/main/Ingress/Notes.md

**Statefuleset** https://github.com/nawab312/Kubernetes/blob/main/StatefuleSet/Notes.md

**Scaling in Kubernetes** https://github.com/nawab312/Kubernetes/blob/main/HPA/Notes.md

**Istio** https://github.com/nawab312/Kubernetes/blob/main/Istio/Notes.md

**SSL Certificates in Kubernetes** https://github.com/nawab312/Kubernetes/blob/main/SSLCertificate_Kubernetes.md

**Common Errors in Kubernetes** https://github.com/nawab312/Kubernetes/blob/main/Common_Errors.md

- Difference between `kubectl apply`, `kubectl create`, `kubectl replace`
![image](https://github.com/user-attachments/assets/e2771329-33be-446f-ad3e-74de10812595)

**Important Commands** https://github.com/nawab312/Kubernetes/blob/main/Commands.md







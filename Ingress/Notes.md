The need for Ingress in Kubernetes arose primarily to address the challenges of managing external access to services running inside a Kubernetes cluster. Before Ingress, there were a few common methods to expose services, like using NodePort or LoadBalancer resources. 
- NodePort: This opens a specific port on each node in the cluster. It means exposing each service on a unique port, which can quickly become messy when there are multiple services, and also isn’t scalable.
- LoadBalancer: This requires setting up a *separate external load balancer for each service*. This approach can be costly, particularly in cloud environments where each load balancer comes with a fee.
![Ingress1](https://github.com/nawab312/Kubernetes/blob/main/Images/Ingress1.png)

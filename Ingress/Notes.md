The need for Ingress in Kubernetes arose primarily to address the challenges of managing external access to services running inside a Kubernetes cluster. Before Ingress, there were a few common methods to expose services, like using NodePort or LoadBalancer resources. 
- NodePort: This opens a specific port on each node in the cluster. It means exposing each service on a unique port, which can quickly become messy when there are multiple services, and also isn’t scalable.
- LoadBalancer: This requires setting up a *separate external load balancer for each service*. This approach can be costly, particularly in cloud environments where each load balancer comes with a fee.
![Ingress1](https://github.com/nawab312/Kubernetes/blob/main/Images/Ingress1.png)

- **Ingress** is an API object that manages external HTTP/HTTPS access to services within a cluster
- It provides routing rules to expose multiple services under a single IP or domain.
- An **Ingress Resource** is a Kubernetes object that defines rules for external access to your services in the cluster, such as HTTP and HTTPS traffic.
- **Ingress Controller** is a Kubernetes component responsible for implementing the rules defined in the Ingres. NGINX Ingress Controller, Traefik, HAProxy, Istio.

**How do you set up load balancing with Ingress?**
- Two main steps: deploying an Ingress controller and defining Ingress resources
- Step 1 is to Deploy the Ingress Controller. Ingress controller is a pod or set of pods running in your cluster that takes care of routing traffic based on the Ingress rules.
- Once the Ingress controller is deployed, the next step is to define Ingress resources for your services. These resources specify how incoming traffic should be routed.

*The Ingress controller handles the external routing, while within each service, Kubernetes uses a service to load balance between multiple pods. So, if a service has multiple pods, Kubernetes automatically distributes traffic among them.
For example, if you have a service with three replicas (pods), Kubernetes will round-robin the traffic among those pods. The Ingress controller doesn’t have to worry about that part — it just ensures that the traffic gets directed to the correct service, and Kubernetes handles the internal load balancing within the service.*




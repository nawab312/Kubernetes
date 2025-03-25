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

**Annotations in Ingress**
- Annotations in an Ingress resource in Kubernetes are used to provide additional configuration or metadata that can control the behavior of the Ingress Controller 
- Unlike regular fields in an Ingress definition (such as `spec.rules` or `spec.backend`), annotations are typically used to specify configuration options that are specific to the Ingress Controller.
- `nginx.ingress.kubernetes.io/ssl-redirect: "true"` Controls whether HTTP traffic is redirected to HTTPS.
- `nginx.ingress.kubernetes.io/ingress.class: "nginx"` Specifies which Ingress Controller should handle this Ingress.
- `nginx.ingress.kubernetes.io/load-balancer-method: least_conn` Specifies the load balancing algorithm to use for requests.

**Let’s say we have an EKS cluster with multiple nodes. In this cluster, we have 20 services, each with 5 pods. Now, for each service, we want to implement a least-connection load balancing algorithm to distribute traffic more efficiently. How would you go about setting this up?**
By default, Kubernetes Services use round-robin load balancing for distributing traffic across pods. However, Kubernetes doesn't provide a built-in option for least-connection load balancing at the Service level.
- Using an External Load Balancer: We could start by using a load balancer outside the Kubernetes cluster (like AWS ALB (Application Load Balancer) or NGINX) that routes traffic to the Ingress controller.
- Choosing the Right Ingress Controller: For least-connection load balancing at the pod level, we should choose an Ingress controller that supports this algorithm. One such controller is NGINX
- First, we need to deploy the NGINX Ingress controller in the cluster. We can do this by using Helm or kubectl. `helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.ingressClass=nginx --set controller.replicaCount=2`
- Once the NGINX Ingress controller is deployed, we can configure it to use the least-connection load balancing algorithm. This can be done by editing the ConfigMap of the NGINX Ingress controller.
- `kubectl edit configmap nginx-ingress-controller -n ingress-nginx`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller
  namespace: nginx-ingress
data:
  enable-ssl-passthrough: "true"
  lb-method: least_conn

#https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/configmap-resource/
```

**Network Policy**
- A Network Policy is a Kubernetes object that controls traffic flow to/from Pods based on rules.
- By default, all traffic is allowed between Pods in a cluster. Network Policies restrict unwanted traffic.
- Define allow or deny rules for ingress and egress traffic.
https://github.com/nawab312/Kubernetes/blob/main/Ingress/NetworkPolcy/Notes.md

**Kubernetes does not allow overlapping Ingress rules with the same path (/app) and host (example.com) but different pathType values (Exact vs Prefix).**
```yaml
# Ingress Rule 1: Exact path matching
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-exact
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Exact
        backend:
          service:
            name: exact-service
            port:
              number: 80
```
```yaml
# Ingress Rule 2: Prefix path matching
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-prefix
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: prefix-service
            port:
              number: 80
```
Solution: https://github.com/nawab312/Kubernetes/blob/main/Ingress/Ingress_Resource/Ingress_Resource1.yaml

You have a **Kubernetes cluster** running multiple microservices, exposed using an **Nginx Ingress Controller**. One of your services (`order-service`) is suddenly **returning intermittent 404 errors**, but other services behind the same Ingress are working fine.

Debugging Clues: 
- The `order-service` pods are **running and healthy** (`kubectl get pods` shows them as `Running`).
- The `order-service` Ingress resource has not been modified recently.
- `kubectl get endpoints order-service` sometimes returns empty (None).
- `kubectl logs ingress-nginx-controller` shows some 404 errors for /order.
- A **recent HPA (Horizontal Pod Autoscaler)** change was made to order-service.

Question:
- Why is order-service returning intermittent 404 errors, while other services work fine?
- How would you troubleshoot and fix this issue?

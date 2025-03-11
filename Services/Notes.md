A Service in Kubernetes provides a way to expose your application running inside a set of Pods. We cannot scale a Kubernetes Service. **Endpoint** is an object that represents the IP addresses (Including Port) of the Pods that are dynamically assigned to a Service. It helps in routing traffic to the correct Pod(s) backing a Service.
`SERVICE ---> ENDPOINT (automaticlaly created) --> POD'S IP and PORT`

### Why Services are Needed ###
- Pod IPs Are Ephemeral: Pods are dynamic, and their IP addresses can change when they restart. A Service ensures a stable endpoint to communicate with the Pods.
- Load Balancing: Services distribute traffic across all Pods that match the selector, enabling better resource utilization and reliability

## Types of Services ##

### ClusterIP ###
- Exposes the Service only within the cluster. Pods within the same cluster can access it via its ClusterIP.
- Use Case: Backend services like databases that don’t need external exposure.
```yaml
apiVersion: v1 
kind: Service
metadata:
  name: my-clusterip-service
spec:
  selector:
    app: my-app 
  ports:
    - protocol: TCP
      port: 80 #PORT EXPOSED BY SERVICE
      targetPort: 8080 #PORT EXPOSED IN THE POD/CONTAINER
```
- How does network flow when a client hits clusterIP Service?
  - Client Request: The client sends a request to the ClusterIP of the Service. This is a virtual IP assigned by Kubernetes, which abstracts the underlying Pods.
  - *kube-proxy*: The kube-proxy running on each node intercepts the request. It uses `iptables` or `IPVS rules` to forward the request to one of the Pods that match the Service's label selector.
  - Pod Selection: kube-proxy performs load balancing and chooses a Pod based on round-robin (or another policy). The request is routed to the selected Pod's IP and port.
  - Pod Response: The Pod processes the request and sends the response back to the client. The response flows through the same network path, using the Pod's IP to communicate back.


### NodePort ###
- Exposes the Service on a static port `(30000-32767)` on each Node’s IP. External traffic can access the application via `NodeIP>:<NodePort>`
- Use Case: Debugging or exposing an application during development.

![NodePort](https://github.com/nawab312/Kubernetes/blob/main/Images/NodePort.png)

```yaml
apiVersion: v1 
kind: Service 
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30007
```

### LoadBalancer ###
- Automatically provisions an external load balancer (cloud-provider specific). Exposes the Service to the internet with a public IP. 
- Use Case: Production services that need external access, such as websites or APIs
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort:8080
```

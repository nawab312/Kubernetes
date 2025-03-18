### What is the Kubernetes Metrics Server? ###
The **Metrics Server** is a cluster-wide aggregator of resource usage data, such as CPU and memory, in a Kubernetes cluster. It collects real-time metrics from the **kubelet** on each node and makes them available via the **Metrics API**.

### How Does It Work? ###
- **Kubelet exposes metrics:** Each node’s kubelet collects CPU and memory stats using the *cAdvisor (Container Advisor)* and exposes them through the `metrics-server` API.
- **Metrics Server scrapes the data:** The Metrics Server queries each node’s kubelet on the `/metrics/resource` endpoint.
- **Aggregation and Storage:** It aggregates this data and exposes it through the *Metrics API* (`/apis/metrics.k8s.io`).
- **HPA and Other Components Use the Metrics:** The Kubernetes Horizontal Pod Autoscaler (HPA) and kubectl top command use this API to make scaling decisions.

### What is the Kubernetes Metrics Server? ###
The **Metrics Server** is a cluster-wide aggregator of resource usage data, such as CPU and memory, in a Kubernetes cluster. It collects real-time metrics from the **kubelet** on each node and makes them available via the **Metrics API**.

### How Does It Work? ###
- **Kubelet exposes metrics:** Each node’s kubelet collects CPU and memory stats using the *cAdvisor (Container Advisor)* and exposes them through the `metrics-server` API.
- **Metrics Server scrapes the data:** The Metrics Server queries each node’s kubelet on the `/metrics/resource` endpoint.
- **Aggregation and Storage:** It aggregates this data and exposes it through the *Metrics API* (`/apis/metrics.k8s.io`).
- **HPA and Other Components Use the Metrics:** The Kubernetes Horizontal Pod Autoscaler (HPA) and kubectl top command use this API to make scaling decisions.

### Architecture of Metrics Server ###
**Data Collection Layer (Kubelet & cAdvisor)**
- Each *Kubernetes node* runs a *kubelet*, which is responsible for managing containers.
- The kubelet uses *cAdvisor (Container Advisor)* to collect real-time *CPU & memory usage* of all pods and nodes.
- The kubelet exposes this data via the `/metrics/resource` endpoint on each node.

**Aggregation & Processing Layer (Metrics Server)**
- The *Metrics Server* is a *Deployment* running in the `kube-system` namespace.
- It queries *each node's kubelet* for CPU and memory usage using the `kubelet summary API`.
- It then *aggregates* the collected data from multiple nodes.
- The aggregated data is *temporarily stored* in memory (not a database).

**API Exposure Layer (Metrics API)**
- The Metrics Server exposes the aggregated metrics via the *Kubernetes Metrics API* (`/apis/metrics.k8s.io`).
- The API follows the *Custom Resource API pattern*.
- Kubernetes components like Horizontal Pod Autoscaler (HPA) and kubectl top use this API to fetch real-time metrics.

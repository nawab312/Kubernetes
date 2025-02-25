**Why is a Pod the Smallest Deployable Unit?**
- Encapsulation of Containers: A pod encapsulates one or more containers that need to work together. Kubernetes does not manage individual containers directly but rather manages Pods, making them the basic deployment unit
- Shared Resources: Containers in the same Pod share networking (localhost) and storage volumes, allowing them to communicate efficiently.
- Containers within a Pod run on the same node and are scheduled together, enabling them to work as a single logical unit.


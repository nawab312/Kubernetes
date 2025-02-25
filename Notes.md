## What is Kubernetes? ##
Kubernetes (often abbreviated as K8s) is an open-source platform for automating the deployment, scaling, and management of containerized applications. It helps you manage applications that are composed of many containers across multiple machines, ensuring that the system is highly available, scalable, and resilient.

### Why to use Kubernetes if we have Docker? ###
- - Docker is a powerful tool for creating and running containers, allowing developers to package applications with all their dependencies into lightweight, portable environments. However, Docker alone is not enough when managing multiple containers across different machines in a production environment.
  - Kubernetes is a container orchestration platform that automates the deployment, scaling, and management of containerized applications.
  - Unlike Docker, which requires manual intervention to start, stop, and scale containers, Kubernetes provides built-in features like automatic scaling, load balancing, self-healing, and rolling updates. It ensures high availability by restarting failed containers and distributing traffic efficiently across multiple instances
  - Additionally, Kubernetes makes it easy to manage multi-container applications across multiple servers, ensuring consistency and reliability. In short, while Docker helps in containerizing applications, Kubernetes is essential for managing and orchestrating them at scale in production environments.

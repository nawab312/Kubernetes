### Scenario ###
Your company is running a microservices-based application in Kubernetes. You have an NGINX-based backend service (`backend-svc`) exposed via a Kubernetes `LoadBalancer` Service type. You need to secure traffic to this service using an SSL certificate.

**Requirements:**
- The application should be accessible over HTTPS.
- The SSL certificate should be managed using Kubernetes secrets
- You should automate the renewal of the SSL certificate.
- The Load Balancer should handle SSL termination.

**How will you generate and store an SSL certificate in Kubernetes?**
- Generate a self-signed SSL certificate (if not using a third-party certificate authority like Let's Encrypt):
  ```bash
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=yourdomain.com/O=yourorganization"
  ```
  - `openssl req` Creates a certificate request.
  - `-x509` Generates a self-signed certificate.
  - `-nodes` Skips password protection (so the key can be used without a passphrase).
  - `-days 365` Sets the certificate validity for 1 year.
  - `-newkey rsa:2048` Creates a new 2048-bit RSA key.
  - `-keyout tls.key` Saves the private key as `tls.key`.
  - `-out tls.crt` Saves the certificate as `tls.crt`.
  - `-subj "/CN=yourdomain.com/O=yourorganization"` Defines the certificate deta
- Create a Kubernetes secret from the certificate and key:
  ```bash
  kubectl create secret tls backend-tls --cert=tls.crt --key=tls.key
  ```

**How will you configure the Load Balancer to use the SSL certificate for HTTPS termination?**

**Using NGINX Ingress Controller**
- If using an Ingress controller like NGINX, create an Ingress resource:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  namespace: default
spec:
  tls:
  - hosts:
    - backend.example.com
    secretName: backend-tls
  rules:
  - host: backend.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 443
```
- The `tls.secretName` field tells Kubernetes to use the stored SSL certificate.
- The Load Balancer terminates SSL and forwards unencrypted HTTP traffic to the backend.

**Using a Kubernetes LoadBalancer Service**
- If using a cloud provider’s LoadBalancer (e.g., AWS, GCP, Azure), you can add SSL termination at the cloud level.
- Example for AWS ALB (using annotations for SSL termination at the Load Balancer level):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:region:account:certificate/certificate-id"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 80
```
- This tells AWS ALB to use SSL termination at the Load Balancer, forwarding HTTP to the backend.

**If your certificate is self-signed, how will you distribute it to clients securely?**
When you use a **self-signed SSL certificate**, it is **not issued by a trusted Certificate Authority (CA)** like Let's Encrypt, DigiCert, or GlobalSign. Because of this, web browsers and clients won't automatically trust the certificate, and they may show a security warning like:
*"This site’s security certificate is not trusted!"*

Distributing a Self-Signed Certificate Securely 
- Since the certificate is not automatically trusted, you need to manually distribute it to clients so they recognize it as valid. Here’s how:
  - Install the Certificate on Client Machines (Trusted CA Store)
    - For Linux/macOS:
      ```bash
      sudo cp tls.crt /usr/local/share/ca-certificates/self-signed.crt
        sudo update-ca-certificates  # (For Debian-based systems)
    ```
    - For Browsers (Firefox, Chrome, Edge):
      - Open Settings → Security → Manage Certificates → Import `tls.crt`.
  - Configure Applications to Trust the Certificate
    - For Docker Containers: Mount the certificate inside the container and update CA certificates:
      ```bash
    - For Kubernetes Pods: Mount the certificate as a ConfigMap or Secret and inject it into pods.

- Never share the private key (tls.key)—only distribute the certificate (tls.crt).
    

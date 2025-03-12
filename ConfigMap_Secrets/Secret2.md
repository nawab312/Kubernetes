**How Encryption at Rest Works**
- A Pod or Deployment creates a Secret (e.g., a database password).
- Before storing it in etcd, Kubernetes encrypts the data using an encryption key.
- When a request is made to retrieve the Secret, Kubernetes decrypts it on the fly.

### Step-by-Step: Enabling Encryption at Rest ###

**Create an Encryption Configuration File**

- On the control plane node, create a file:
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0LWtleS1oZXJlCg==  # Replace with a real base64-encoded key
      - identity: {}
```
- `aescbc`: Uses AES-CBC encryption for Secrets.
- `identity`: Used for backwards compatibility (unencrypted Secrets)

**Restart the API Server with Encryption Config**
- Edit the API server manifest (usually in `/etc/kubernetes/manifests/kube-apiserver.yaml`) and add: `--encryption-provider-config=/etc/kubernetes/encryption-config.yaml`
- Then restart the API server: `systemctl restart kubelet`

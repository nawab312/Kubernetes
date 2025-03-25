**How Encryption at Rest Works**
Kubernetes protects sensitive data, such as Secrets, by encrypting them before storing them in etcd. When a request is made to access a Secret, Kubernetes decrypts it in real-time, ensuring security while maintaining accessibility.

- Create an Encryption Configuration File
  - On the control plane node, create an encryption configuration file at `/etc/kubernetes/encryption-config.yaml`
  - aescbc: Uses AES-CBC encryption to secure Secrets.
  - identity: Leaves Secrets unencrypted for backward compatibility.
  ```yaml
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

  - Configure the API Server to Use Encryption
    - Edit the API server manifest, typically found at: `/etc/kubernetes/manifests/kube-apiserver.yaml`
    - Add the following flag to enable encryption:
    ```yaml
    --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    ```

  - Restart the API Server
    ```bash
    systemctl restart kubelet
    ```

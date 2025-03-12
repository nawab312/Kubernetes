**How to Create a Secret?**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: dXNlcm5hbWU=  # Base64 encoded "username"
  password: cGFzc3dvcmQ=  # Base64 encoded "password"
```

### How to Use a Secret in a Pod? ###

**As an Environment Variable:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: myContainer
      image: nginx
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: username
  restartPolicy: Always
```

**As a Mounted Volume**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: myContainer
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: "/etc/secret-data"
          readOnly: true #readOnly: true → Prevents accidental modification of Secret files.
  volumes:
    - name: secret-volume
      secret:
        secretName: my-secret
  restartPolicy: Always
```

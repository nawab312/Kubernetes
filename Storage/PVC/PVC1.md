```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: production
  labels:
    app: my-app
    environment: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-storage
  volumeMode: Filesystem
  selector:
    matchLabels:
      storage-type: premium
  volumeName: my-pv-001
  dataSource:
    name: my-backup-volume
    kind: PersistentVolumeClaim
    apiGroup: ""
  fsType: ext4
```

- **metadata**
  - `name` Name of the PVC
  - `namespace` The Kubernetes namespace
  - `labels` Tags for the PVC, useful for identifying the PVC across clusters or environments.
- **spec**
  - `accessModes` Defines the access modes for the PVC. ReadWriteOnce means the volume can be mounted as read-write by a single node.
  - `resources.requests.storage`: Defines the requested storage amount. This is set to 50Gi in this case.
  - `storageClassName` Specifies which storage class to use for the PVC. In this example, the fast-storage class is used.
  - `volumeMode` Defines the mode of the volume. It could either be Filesystem (default) or Block
  - `selector` Used to match specific volumes with labels, typically to find specific Persistent Volumes (PVs) with certain criteria.
  - `volumeName` The specific Persistent Volume to bind to. If not specified, Kubernetes will look for a PV that satisfies the claim’s requirements.
  - `dataSource` Optionally, you can define a dataSource to create the PVC from an existing backup, snapshot, or another PVC.
  - `fsType` Specifies the filesystem type. ext4 is commonly used for Linux-based systems.

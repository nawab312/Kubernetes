Kubernetes Pods are ephemeral, meaning they can be terminated and recreated at any time. Without persistent storage, any data generated inside a Pod is lost when the Pod restarts. To address this, Kubernetes provides mechanisms to connect Pods to persistent storage systems.

### Persistent Volumes (PV) ###
- A Persistent Volume is a piece of storage in the cluster provisioned by an administrator or dynamically through a storage class. It’s like a disk that can be mounted to a Pod.
- Cluster-wide resource. Abstracts the underlying storage (NFS, AWS EBS, GCE PD, etc.). Defined and managed independently of Pods.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data"
```
- accessModes: ReadWriteOnce - Specifies that only one pod can access the PV at a time.
- persistentVolumeReclaimPolicy:
  - **Retain** - Determines what happens to the PV when the pod using it is deleted. In this case, the PV is retained and not automatically deleted.
  - **Delete** - Automatically deletes the underlying storage resource (e.g., an EBS volume) when the PV is released.
  - **Recycle** - Clears the PV's contents by running a basic rm -rf command on the volume, making it reusable for another PVC.
- hostPath: path: "/mnt/data" - A hostPath volume type allows you to mount a host directory onto a Pod. This means you're directly exposing a specific directory from the underlying host machine's filesystem into your Pod.

### Persistent Volume Claims (PVC) ###
- A Persistent Volume Claim is a request for storage by a user. Pods use PVCs to access a Persistent Volume.
- PVCs allow dynamic or manual binding to PVs. Ensures Pods get the right storage with the required access modes.
- `PVC binds to a PV`, it means that the PVC has successfully requested and matched a specific PV with the appropriate storage resources.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**storageClassName** field in a PVC specifies which StorageClass Kubernetes should use to dynamically provision a PV or match an existing one.
- Dynamic Provisioning: When a PVC includes storageClassName, Kubernetes will use the corresponding StorageClass to dynamically create a Persistent Volume that meets the PVC's requirements.
- Static Binding: If the cluster administrator has pre-created PVs, the storageClassName ensures that the PVC only binds to PVs with a matching storageClassName
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

- If `storageClassName: ""` This means do not use dynamic provisioning. Instead, the PVC will only bind to an existing PV.

### Kubernetes Storage Class ###
- Storage Classes enable dynamic provisioning of PVs. Different storage backends (like AWS EBS, GCP Persistent Disks, etc.) can have their own StorageClass configurations.
- `provisioner`: The plugin used to provision storage (e.g., kubernetes.io/aws-ebs).
- `parameters`: Backend-specific configurations like volume type or IOPS.
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
```

**What is the difference between static and dynamic provisioning of Persistent Volumes**

*Static provisioning* of Persistent Volumes (PVs) involves manually creating PVs ahead of time by the administrator, with the user requesting a Persistent Volume Claim (PVC) that matches the predefined PV. In *dynamic provisioning*, Kubernetes automatically creates PVs based on PVCs when a request is made, without requiring predefined PVs. Dynamic provisioning is more flexible and scalable, while static provisioning offers more control and predictability.

`PVC in one namespace cannot bind to a PV in another namespaceNo. PVs are cluster-wide resources, but PVCs are namespace-scoped. A PVC can only bind to a PV in the same namespace or one without any namespace restrictions```


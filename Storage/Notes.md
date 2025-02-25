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
- persistentVolumeReclaimPolicy: Retain - Determines what happens to the PV when the pod using it is deleted. In this case, the PV is retained and not automatically deleted.
- hostPath: path: "/mnt/data" - A hostPath volume type allows you to mount a host directory onto a Pod. This means you're directly exposing a specific directory from the underlying host machine's filesystem into your Pod.

### Persistent Volume Claims (PVC) ###
- A Persistent Volume Claim is a request for storage by a user. Pods use PVCs to access a Persistent Volume.
- PVCs allow dynamic or manual binding to PVs. Ensures Pods get the right storage with the required access modes.
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

### Kubernetes Storage Class ###
- Storage Classes enable dynamic provisioning of PVs. Different storage backends (like AWS EBS, GCP Persistent Disks, etc.) can have their own StorageClass configurations.
- `provisioner`: The plugin used to provision storage (e.g., kubernetes.io/aws-ebs).
- `parameters`: Backend-specific configurations like volume type or IOPS.

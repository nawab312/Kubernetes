Kubernetes Pods are ephemeral, meaning they can be terminated and recreated at any time. Without persistent storage, any data generated inside a Pod is lost when the Pod restarts. To address this, Kubernetes provides mechanisms to connect Pods to persistent storage systems.

### Persistent Volumes (PV) ###
- A Persistent Volume is a piece of storage in the cluster provisioned by an administrator or dynamically through a storage class. It’s like a disk that can be mounted to a Pod.
- *Cluster-Wide Resource*.
  - A Persistent Volume (PV) is a resource that exists at the cluster level, meaning it is available to any node or pod within the Kubernetes cluster. It’s not tied to a specific pod or node, but instead can be used across the entire cluster.
- Abstracts the underlying storage (NFS, AWS EBS, GCE PD, etc.) (Kubernetes hides the complexity and details of the underlying storage system from the user or application.) Example:
  - If you’re using an *AWS EBS volume* or a *Google Cloud Persistent Disk*, you don’t need to worry about the cloud-specific APIs or configurations directly when you work with storage in Kubernetes.
  - Kubernetes handles the interaction with those underlying systems for you, through the *Persistent Volume abstraction*, so you just define a PV in your configuration and Kubernetes takes care of the rest.
- Defined and managed independently of Pods.
  - *Persistent Volumes* are not tied directly to the lifecycle of any *pod*. Pods can request storage via *Persistent Volume Claims (PVCs)*, but the lifecycle of the storage itself is independent of the pod.
  - For example, a pod might use a volume from a PV to store data, but even if that pod is deleted or recreated, the data in the PV remains intact (as long as the reclaim policy allows it to persist). This is crucial for maintaining stateful data in a stateless environment like Kubernetes.

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
- accessModes:
    - **ReadWriteOnce** - The volume can be mounted as read-write by only one node at a time. Even if multiple Pods are on the same node, he volume can still be mounted and accessed by all those pods simultaneously on that single node.
    - **ReadOnlyMany** - The volume can be mounted as read-only by multiple nodes simultaneously.
    - **ReadWriteMany** - The volume can be mounted as read-write by multiple nodes simultaneously. You have an application where multiple Pods write logs to the same shared volume.
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
- Storage Classes enable *Dynamic Provisioning of PVs*. It means when you create a PVC, Kubernetes can automatically create the underlying storage for you — based on rules defined in a StorageClass. No manual PV creation.
- Different storage backends (like AWS EBS, GCP Persistent Disks, etc.) can have their own StorageClass configurations.
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

Example in Amazon EKS. If you define:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  storageClassName: gp3
  resources:
    requests:
      storage: 10Gi
```
  - Kubernetes calls the EBS CSI driver
  - A new EBS volume is created
  - A PV object is automatically generated
  - PVC binds to it
  - You didn’t create the PV manually.

---

**Dynamic provisioning can fail due to AZ mismatch or IAM permission**

*AZ Mismatch — What Does That Actually Mean?*
- EBS volumes are created in one specific Availability Zone.
- Node A → `ap-south-1a` , Node B → `ap-south-1b`
- If your PVC triggers dynamic provisioning:
  - The EBS volume gets created in one AZ only
  - That volume cannot attach to nodes in other AZs

- Scenario 1: Immediate binding mode (default in older setups)
  - PVC is created → Volume immediately created in some AZ (random/first available).
  - Later: Pod gets scheduled to a node in a different AZ.
  - Result: `AttachVolume.Attach failed`
  - The volume exists. The pod exists. But they are in different AZs.

*IAM Permission Failure — What Does That Mean?*
- Dynamic provisioning requires the CSI driver to call AWS APIs like:
  - `ec2:CreateVolume`
  - `ec2:AttachVolume`
  - `ec2:DeleteVolume`

- PVC stuck in `Pending`
  - StorageClass exists
  - EBS CSI driver is installed
  - Nodes are healthy
  - Event shows: `UnauthorizedOperation`
  - Which IAM role is actually failing — node role or service account role?
- Answer:
  - In modern Amazon EKS setups using the EBS CSI driver, the failing role is: *The IAM role attached to the ebs-csi-controller service account (via IRSA)*. Not the node role.
  - Dynamic provisioning (`CreateVolume`) happens in the controller pod, not on the worker node.
  - The node role is used for: `AttachVolume` , `Mounting`
  - But CreateVolume happens before attachment — and it is executed by the CSI controller.
  - Checking Precisely:
    - Step 1 — Check PVC events: `kubectl describe pvc <name>`. Look at Events, It often includes the exact AWS error.
    - Step 2 — Check CSI controller logs. It will show the exact AWS API call that failed.
      ```bash
      kubectl get pods -n kube-system | grep ebs
      kubectl logs -n kube-system <ebs-csi-controller-pod>
      ```
    - Step 3 — Verify which IAM role is attached
      ```bash
      kubectl get sa ebs-csi-controller -n kube-system -o yaml
      ```
    - Step 4 - Look for annotation:
      ```bash
      eks.amazonaws.com/role-arn
      ```
    - That ARN is the IAM role actually being used. Now inspect that role in AWS and verify it has:
      ```bash
      ec2:CreateVolume
      ec2:AttachVolume
      ec2:DeleteVolume
      ec2:Describe*
      ```

 
<img width="724" height="205" alt="image" src="https://github.com/user-attachments/assets/f1045800-df6f-4abb-9458-f8835a7c2546" />

---

**What is the difference between static and dynamic provisioning of Persistent Volumes**

*Static provisioning* of Persistent Volumes (PVs) involves manually creating PVs ahead of time by the administrator, with the user requesting a Persistent Volume Claim (PVC) that matches the predefined PV. In *dynamic provisioning*, Kubernetes automatically creates PVs based on PVCs when a request is made, without requiring predefined PVs. Dynamic provisioning is more flexible and scalable, while static provisioning offers more control and predictability.

**PVC in one namespace cannot bind to a PV in another namespace. PVs are cluster-wide resources, but PVCs are namespace-scoped**

*SCENARIO*
- You are working in a Kubernetes environment where you are tasked with setting up persistent storage for two different applications running in separate namespaces: App1 in the `namespace-app1` and App2 in the `namespace-app2`. You have a Persistent Volume (PV) that is intended to be shared between these applications.
  - Can a Persistent Volume Claim (PVC) from namespace-app1 bind to the PV if it's located in namespace-app2?
  - What would you need to do to ensure that App1 and App2 can both use the same PV?

*SOLUTION*
- No, a PVC from `namespace-app1` cannot bind to a PV that is located in `namespace-app2`. This is because PVCs are namespace-scoped, meaning a PVC can only bind to a PV that is either:
  - In the same namespace as the PVC.
  - A *cluster-wide PV* with no namespace restrictions (i.e., it has no specific namespace set).
- To allow both App1 in `namespace-app1` and App2 in `namespace-app2` to use the same Persistent Volume (PV), you would need to create a *cluster-wide PV (a PV with no namespace restrictions)* and make sure that each application has its own PVC within its respective namespace

**Handling Volume Expansion for Persistent Volumes**
- Volume Type: Not all storage types support volume expansion. For example, cloud-based storage solutions such as AWS Elastic Block Store (EBS), Google Persistent Disk (GCE PD), and Azure Disks support expansion. However, traditional storage backends like NFS may require additional steps for expansion.
- Storage Class: The PVC and PV must be associated with a StorageClass that allows volume expansion. The storage provisioner for the specific storage backend must also support resizing the volume. For instance, a StorageClass for AWS EBS must have `allowVolumeExpansion: true`.
- Dynamic Provisioning: The expansion typically works only for dynamically provisioned volumes. Pre-provisioned PVs will require manual intervention


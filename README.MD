
# The OpenShift Reattachment Pattern: A Practical Guide

This repository contains the practical steps and code examples for implementing the Reattachment Pattern for stateful applications on OpenShift, as discussed in the Red Hat Summit presentation "Disposable Clusters, Persistent Data: The Reattachment Pattern for Bulletproof OpenShift DR".

The goal of this pattern is to decouple the lifecycle of an OpenShift cluster from the lifecycle of its data. This enables faster disaster recovery, safer cluster upgrades, and a more resilient operational model by treating clusters as disposable "cattle" while the data remains persistent.

## The Scenario

A source OpenShift cluster has failed or is scheduled for an upgrade. The cluster is powered off, but the underlying storage volumes (e.g., VMDKs on a vSphere datastore) containing the application data are still intact. A new, clean OpenShift cluster has been provisioned with access to that same underlying storage.

Our goal is to reconnect our application in the new cluster to its existing data without performing a slow, traditional backup-and-restore.

## Prerequisites

1.  **Storage Accessibility:** The new OpenShift cluster's nodes must have access to the same underlying storage array/datastore where the original volumes reside.
2.  **Manifests Captured:** You have exported the original `PersistentVolume` (PV) and `PersistentVolumeClaim` (PVC) definitions as YAML or JSON.
3.  **Application Deployment:** You have a stable, repeatable process (e.g., GitOps, Helm, Operators) to deploy your application's components (Deployments, StatefulSets, etc.).

---

## The Reattachment Process: A Step-by-Step Guide

The core of the pattern is to "sanitize" the captured PV manifest, removing all metadata specific to the old cluster, and then use it to statically provision a volume in the new cluster.

### Step 1: Capture the Original PV and PVC Manifests

Before decommissioning the old cluster, you would save the definitions. Existing object will contain a lot of unnecessary metadata as described below. Make sure that you cleaned it up.

PV


---

### Sanitizing the `PersistentVolume` Manifest

To make the exported `PersistentVolume` object "cluster-agnostic," we must remove all fields that contain runtime state or unique identifiers from the original cluster. The goal is to be left with a purely declarative manifest that only describes the volume's identity and characteristics.

The following table details the specific fields to be removed and the reason for their removal.

| Field Path | Description | Why It Must Be Removed |
| :--- | :--- | :--- |
| **`.metadata.creationTimestamp`** | A timestamp added by the old cluster's API server when the object was created. | This is historical metadata. The new cluster will generate its own timestamp upon creation. |
| **`.metadata.uid`** | A unique ID assigned by the old cluster's API server to this specific object instance. | This ID is only unique within the old cluster's database (etcd). The new cluster must assign its own unique ID. |
| **`.metadata.resourceVersion`** | A value used by the old cluster's control plane for internal versioning and caching. | This is internal state from the old etcd and is meaningless to the new cluster. |
| **`.spec.claimRef.uid`** | The UID of the `PersistentVolumeClaim` that was bound to this PV in the old cluster. | This references a PVC that does not yet exist in the new cluster. This entire `claimRef` block should be removed. |
| **`.spec.claimRef.resourceVersion`** | The `resourceVersion` of the PVC object from the old cluster. | Like the PV's `resourceVersion`, this is internal state from the old cluster and must be removed. |
| **`.status`** | An entire block of data describing the PV's runtime status in the old cluster (e.g., `phase: Bound`). | The `status` of an object is managed by the control plane, not set by users. The new cluster will determine and report its own status for the PV. |

By surgically removing these fields, you are left with a clean definition that preserves the most critical piece of information: the `spec.csi.volumeHandle`, which is the physical address of your data on the storage array.

---

### Sanitizing the `PersistentVolumeClaim` Manifest

Similar to the `PersistentVolume`, the exported `PersistentVolumeClaim` (PVC) manifest contains runtime data from the old cluster that must be cleaned before it can be used in the new cluster. The goal is to create a clean "request for storage" that the new cluster can fulfill by binding it to the static PV we created.

The following table details the specific fields to be removed.

| Field Path | Description | Why It Should Be Removed |
| :--- | :--- | :--- |
| **`.metadata.annotations`** | Metadata added by controllers in the old cluster, often indicating the binding status (e.g., `pv.kubernetes.io/bind-completed`). | These annotations represent the runtime state from the old cluster and are not needed. The new cluster will add its own annotations as it processes the PVC. |
| **`.metadata.creationTimestamp`**| The timestamp when the PVC was created in the old cluster. | This is historical metadata. The new cluster will assign a new timestamp when the PVC is created. |
| **`.metadata.uid`** | The unique ID assigned by the old cluster's API server. | This ID is only unique within the old cluster's etcd. The new cluster must be allowed to generate its own UID. |
| **`.metadata.resourceVersion`** | The internal versioning value used by the old cluster's control plane. | This is internal state from the old etcd and is meaningless to the new cluster. |
| **`.status`** | The block of data describing the PVC's runtime status in the old cluster (e.g., `phase: Bound`, `capacity`). | The `status` is managed by the Kubernetes control plane. The new cluster will determine and report its own status for the PVC after binding. |

After removing these fields, you are left with a clean, declarative request for storage. When you apply this manifest, the OpenShift control plane will find the sanitized static `PersistentVolume` you already created, validate that they match, and bind them together.

### Step 2: Sanitize the Manifests for the New Cluster

This is the most critical step. We need to remove all fields that tied these objects to the old cluster's control plane.

#### **Sanitized `PersistentVolume` (Ready for Import)**

We remove cluster-specific metadata, finalizers, and references. The `volumeHandle` is the only thing that matters.

```json
{
    "apiVersion": "v1",
    "kind": "PersistentVolume",
    "metadata": {
        "finalizers": [
            "kubernetes.io/pv-protection",
            "external-attacher/csi-vsphere-vmware-com"
        ],
        "name": "pvc-<random-uid-assigned-by-OCP>"
    },
    "spec": {
        "accessModes": [
            "ReadWriteOnce"
        ],
        "capacity": {
            "storage": "50Gi"
        },
        "claimRef": {
            "apiVersion": "v1",
            "kind": "PersistentVolumeClaim",
            "name": "example-pvc-name-1",
            "namespace": "acme-namespace-test"
        },
        "csi": {
            "driver": "csi.vsphere.vmware.com",
            "fsType": "ext4",
            "volumeAttributes": {
                "storage.kubernetes.io/csiProvisionerIdentity": "<identity-id>-csi.vsphere.vmware.com",
                "type": "vSphere CNS Block Volume"
            },
            "volumeHandle": "<random-volume-id-assigned-by-vSphere-CSI-driver>"
        },
        "persistentVolumeReclaimPolicy": "Delete",
        "storageClassName": "example-vsphere-csi-storage-class",
        "volumeMode": "Filesystem"
    }
}
```

#### **Sanitized `PersistentVolumeClaim` (Ready for Import)**

For the PVC, we must remove the `volumeName` to allow the new cluster to perform the binding correctly.

```json
{
    "apiVersion": "v1",
    "kind": "PersistentVolumeClaim",
    "metadata": {
        "finalizers": [
            "kubernetes.io/pvc-protection"
        ],
        "labels": {
            "app.kubernetes.io/instance": "example-instance-label",
            "app.kubernetes.io/name": "example-name"
        },
        "name": "example-pvc-name-1",
        "namespace": "acme-namespace-test"
    },
    "spec": {
        "accessModes": [
            "ReadWriteOnce"
        ],
        "resources": {
            "requests": {
                "storage": "50Gi"
            }
        },
        "storageClassName": "example-vsphere-csi-storage-class",
        "volumeMode": "Filesystem",
        "volumeName": "pvc-<random-uid-assigned-by-OCP-as-seen-on-pv-object>"
    }
}
```

### Step 3: Apply Manifests and Deploy the Application

In the new cluster, apply the manifests in a specific order to ensure the static provisioning process works correctly.

```bash
# 1. Create the namespace first
oc apply -f namespace.yaml

# 2. Apply the sanitized PersistentVolume manifest. This registers the existing disk with the cluster.
oc apply -f sanitized-pv.yaml

# 3. Apply the sanitized PersistentVolumeClaim. OpenShift will now bind this PVC to the PV we just created.
oc apply -f sanitized-pvc.yaml

# 4. Check that the PV and PVC are successfully "Bound"
oc get pv,pvc -n acme-namespace-test

# 5. Deploy your application (via Helm, GitOps, etc.). The pods will start and connect to the reattached volume.
```

---

## Challenges and Considerations

*   **`persistentVolumeReclaimPolicy`:**
    *   The reattachment *procedure* works regardless of the policy because the old cluster is powered off.
    *   The choice of `Delete` vs. `Retain` is a **data governance decision**.
    *   `Delete`: Ensures no orphaned volumes are left on the storage array if an application owner deletes a reattached PVC.
    *   `Retain`: Provides maximum protection against accidental data deletion, but requires a separate administrative process to clean up unused volumes.

*   **CSI Driver Versions:** Ensure the CSI driver version in the new cluster is compatible with the volume format created by the old cluster's driver. This is generally not an issue but is a factor to consider.

*   **Other Cloud Providers:** This pattern is not specific to vSphere. The key is to identify and preserve the `volumeHandle`.
    *   **AWS:** The `volumeHandle` is the EBS Volume ID (e.g., `vol-0123456789abcdef0`).
    *   **Azure:** It is the full resource ID of the Azure Disk.
    *   **GCP:** It is the name of the GCE Persistent Disk.

## Conclusion

By treating cluster infrastructure and application data as separate, independent components, the Reattachment Pattern offers a powerful, fast, and reliable method for managing the lifecycle of stateful applications on OpenShift.

---

Jacek Bartyzel https://www.linkedin.com/in/jacek-bartyzel-14b06943/ 
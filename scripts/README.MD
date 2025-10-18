

## Scripts for Exporting and Sanitizing Kubernetes Storage Objects

This directory contains helper scripts designed to automate the "Capture" phase of the Reattachment Pattern. They read a list of namespaces from `namespace.csv` and export the `PersistentVolume` (PV) and `PersistentVolumeClaim` (PVC) manifests, stripping them of cluster-specific runtime metadata.

The output of these scripts are "sanitized" JSON files that can be used as a starting point for creating the static manifests for the new cluster.

## Prerequisites

Before running these scripts, ensure you have the following:

1.  **`oc` CLI:** You must be logged into the **source** OpenShift cluster with the `oc` command-line tool.
2.  **`jq`:** The powerful command-line JSON processor `jq` must be installed on your system.
3.  **Permissions:** The user you are logged in as must have permissions to `get` `pv` and `pvc` resources across the target namespaces.

## File Descriptions

*   `namespace.csv`: A simple CSV file listing the target namespaces to process. The first line is a header and is ignored.
*   `pv-export.sh`: Exports and sanitizes all `PersistentVolume` objects associated with PVCs in the target namespaces.
*   `pvc-export.sh`: Exports and sanitizes all `PersistentVolumeClaim` objects from the target namespaces.

## How to Use

1.  **Log into the source OpenShift cluster** where the original volumes are running.

2.  **Edit `namespace.csv`** to include the namespaces you want to export resources from. For example:
    ```csv
    namespace
    acme-example-test
    acme-example-other-namespace
    ```

3.  **Make the scripts executable**:
    ```bash
    chmod +x pv-export.sh pvc-export.sh
    ```

4.  **Run the export scripts**:
    ```bash
    # Export and sanitize the PV objects
    ./pv-export.sh

    # Export and sanitize the PVC objects
    ./pvc-export.sh
    ```

5.  **Check the output**. The scripts will generate one JSON file per namespace, named `pv-<namespace>.json` and `pvc-<namespace>.json`, in the current directory.

## What the Scripts Do: The Sanitization Logic

These scripts are non-destructive; they only perform `oc get` operations. The `jq` commands are used to surgically remove fields that are specific to the source cluster's runtime environment.

### `pv-export.sh`

This script removes the following fields from each `PersistentVolume` object:

| Field Path Removed | Reason for Removal |
| :--- | :--- |
| **`.metadata.creationTimestamp`** | Historical metadata specific to the old cluster. |
| **`.metadata.uid`** | A unique ID from the old cluster's database (etcd). |
| **`.metadata.resourceVersion`** | Internal versioning data from the old cluster. |
| **`.spec.claimRef.uid`** | A reference to the PVC's unique ID in the old cluster. |
| **`.spec.claimRef.resourceVersion`** | The PVC's version from the old cluster. |
| **`.status`** | The entire runtime status block (e.g., `phase: Bound`). |

### `pvc-export.sh`

This script removes the following fields from each `PersistentVolumeClaim` object:

| Field Path Removed | Reason for Removal |
| :--- | :--- |
| **`.metadata.annotations`** | Runtime state added by controllers (e.g., binding status). |
| **`.metadata.creationTimestamp`** | Historical metadata specific to the old cluster. |
| **`.metadata.uid`** | A unique ID from the old cluster's database (etcd). |
| **`.metadata.resourceVersion`** | Internal versioning data from the old cluster. |
| **`.status`** | The entire runtime status block (e.g., `phase: Bound`). |


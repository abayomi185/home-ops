# Storage Class

`csi-driver-nfs` creates a StorageClass that dynamically provisions a PersistentVolumeClaim (PVC) for a NFS volume. The StorageClass is named `nfs-default` and is created in the same namespace as the driver.

This means that a PVC can be created without specifying a PV, and the driver will automatically create a PV for the PVC.

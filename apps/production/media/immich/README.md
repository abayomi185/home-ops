# Immich production overlay


## Important invariants

- Use `${NFS_STORAGE_HOST_FAST}` for Immich storage paths.
- Keep `nfs-csi-immich` using `subDir: immich/$${pvc.metadata.name}` unless you are deliberately migrating data. This gives stable, human-readable NFS directories instead of `pvc-<uid>` paths.

## Current storage layout

- Library uploads: `immich-data-pvc` via `nfs-csi-immich`
- Machine-learning cache: ephemeral `emptyDir` managed by the Helm chart
- Database: CNPG storage on `nfs-csi-immich`
- External library mounts:
  - `${NFS_STORAGE_GLOBAL_DATA_PATH}/NAS-YOM/Picture-Edits`
  - `${NFS_STORAGE_GLOBAL_DATA_PATH}/NAS-YOM/FTP`

## When editing this overlay

- If you change `storageClassName`, `subDir`, or the external PV targets, plan the on-disk data migration first.

## Related notes

- `docs/operations/2026-06-12-immich-storage-recovery.md`

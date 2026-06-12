# 2026-06-12 Immich storage and NFS recovery

## Why this note exists

This recovery crossed app manifests, node networking, and NFS layout. The overlay README should stay focused on steady-state intent, so the dated recovery history lives here.

## Symptoms

- Pods mounting NFS through the cluster storage host timed out.
- Flux stopped reconciling cleanly after the node network issue.
- Immich library, machine-learning, and database storage needed stable, human-readable directories instead of CSI-generated `pvc-<uid>` paths.
- The CNPG database became unrecoverable after PVC recreation until the original PVC metadata relationship was restored.

## Root cause

The main failure was node-level, not an Immich chart bug: `knode2` and `knode3` were not bringing up the `eth1` storage-network address at boot. Workloads that correctly used `${NFS_STORAGE_HOST_FAST}` therefore lost reachability to the NFS server on `10.0.7.202`.

A second, smaller problem was operability: Immich dynamic NFS provisioning used opaque per-PV UID directories, which made storage migrations and PVC recovery harder to reason about.

## Home-ops changes

### Dedicated StorageClass for pinned Immich paths

`apps/production/media/immich/sc.yaml`

- creates `StorageClass/nfs-csi-immich`
- uses `${NFS_STORAGE_HOST_FAST}` and `${NFS_STORAGE_MAIN_DATA_PATH}`
- pins dynamically provisioned subdirectories to `immich/$${pvc.metadata.name}`
- keeps `reclaimPolicy: Retain`

### Library and database storage moved onto the dedicated class

- `apps/production/media/immich/pvc.yaml`
  - `immich-data-pvc` uses `nfs-csi-immich`
- `apps/production/media/immich/patch_helm-release.yaml`
  - server and ML image tags are pinned in the overlay
  - the machine-learning cache was briefly pinned during recovery and later reverted to the upstream ephemeral default
- `apps/base/media/immich/database.yaml`
  - CNPG storage uses `nfs-csi-immich`

### External library mounts kept manual but moved to the fast host

`apps/production/media/immich/pv.yaml`

- `immich-external-data-main-pv` points at `${NFS_STORAGE_HOST_FAST}:${NFS_STORAGE_GLOBAL_DATA_PATH}/NAS-YOM/Picture-Edits`
- `immich-external-data-ftp-pv` points at `${NFS_STORAGE_HOST_FAST}:${NFS_STORAGE_GLOBAL_DATA_PATH}/NAS-YOM/FTP`

## Paired infra changes outside this repo

These were required to make the manifest changes actually work:

- `nix-dotfiles/hosts/knode/configuration.nix`
  - `networking.useDHCP = false`
  - `eth0` keeps DHCP
  - `eth1` gets an explicit `10.0.7.x/24` static address
- `nix-dotfiles/hosts/lxc/network-share/configuration.nix`
  - NFS export generation corrected so the share layout matched the intended mounts

## Pinned data directories

The recovery intentionally moved Immich away from random CSI directory names to stable paths:

- library uploads: `immich/immich-data-pvc`
- database: `immich/immich-database-1`

The external library paths remain explicit manual mounts rather than dynamic CSI subdirectories.

## CNPG recovery note

During database PVC recovery, the CNPG cluster temporarily entered an unrecoverable state after the original PVC/PV objects were recreated. Recovery required restoring the original data directory, recreating the expected PVC, and re-establishing the metadata CNPG uses to associate the PVC with the primary instance.

Do not treat that sequence as a normal rolling maintenance operation; plan database-storage moves as a data migration with explicit recovery steps.

## Invariants after recovery

- Use `${NFS_STORAGE_HOST_FAST}` for Immich and other fast-NFS workloads. Do not roll back to `${NFS_STORAGE_HOST}` to mask reachability issues.
- Keep `nfs-csi-immich` with `subDir: immich/$${pvc.metadata.name}` unless you also migrate the on-disk data.
- Keep `Retain` on Immich storage.
- The machine-learning cache is intentionally ephemeral now. It is disposable, but pod restarts will re-download roughly 1.6 GiB of models and Hugging Face cache.

## Verification performed during the recovery

At the time of the change, the following checks were used to confirm the fix:

- restored `knode2` and `knode3` reachability to the storage-network NFS host
- reconciled Flux after controller restart
- confirmed `immich-deployment-server`, `immich-deployment-machine-learning`, and `immich-database-1` returned healthy
- confirmed Immich PV/PVC targets resolved to the cluster FQDN and pinned subdirectories

# Calibre-Web

Flux deploys this app independently through `calibre-web-media-production` in the
`media` namespace. The UI is at `https://calibre.local.${DOMAIN}` through Traefik.
Point that hostname at the LAN ingress address, following the existing local DNS
setup. The route uses the existing Let's Encrypt resolver.

## Before merging

On `${NFS_STORAGE_HOST_FAST}`, prepare these directories with ownership `1001:100`
and permissions `0770`:

- `${NFS_STORAGE_CONFIG_DATA_PATH}/calibre-web/config`
- `/mnt/mopower/swarm-data/calibre-web/library`

Copy an existing Calibre library, including `metadata.db`, into the library
directory, or create one with the Calibre desktop app first. A directory of loose
ebooks is not a Calibre library. This deployment uses a dedicated library so it
does not change the retired Readarr book directory.

The NFS export must permit writes by UID 1001 and GID 100. The pod mounts config at
`/config` and the library at `/books`. A single replica and the `Recreate` strategy
avoid overlapping writers during upgrades.

## First login

Open the UI, use the upstream initial account `admin` / `admin123`, and change the
password immediately. Set the library location to `/books`. Configure user access
before making the route available beyond the trusted LAN. A local hostname alone
is not an access control rule.

Ebook conversion is not enabled. Enable LinuxServer's optional Calibre mod only
if conversion is needed.

## Verify after deployment

```sh
kubectl -n media rollout status deployment/calibre-web-deployment-production
kubectl -n media logs deployment/calibre-web-deployment-production
```

Check that books appear and that the login and library settings survive a pod
restart. The PR validates manifests; first login and library access require the
cluster and prepared NFS directories.

Upstream: [LinuxServer Calibre-Web](https://docs.linuxserver.io/images/docker-calibre-web/).

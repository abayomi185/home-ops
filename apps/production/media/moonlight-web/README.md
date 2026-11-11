# Moonlight Web

Deploys [MrCreativ3001/moonlight-web-stream v2.10.0](https://github.com/MrCreativ3001/moonlight-web-stream/tree/v2.10.0)
as an independent Flux app, `moonlight-web-media-production`, in `media`.
The UI is at `https://moonlight.local.${DOMAIN}` through Traefik. Point that hostname
at the LAN ingress address using the existing local DNS setup.

## Before merging

On `${NFS_STORAGE_HOST_FAST}`, create
`${NFS_STORAGE_CONFIG_DATA_PATH}/moonlight-web/server` with ownership `1001:100`
and permissions `0770`. The NFS export must permit writes by that user and group.
It mounts at `/moonlight-web/server` and persists accounts, pairing data and config.
The image runs with Kubernetes UID 1001 and GID 100 because it does not implement
LinuxServer's PUID, PGID or UMASK variables.

The pod uses the node network, listens on TCP 8087 and allocates WebRTC UDP ports
40000 through 40010. Reserve those ports on eligible homelab nodes. It advertises
`status.hostIP` to browsers and uses `ClusterFirstWithHostNet` for cluster DNS.
Browsers must be able to reach that node IP and UDP range. The HTTPS route carries
the web UI and WebSocket traffic; it does not relay the WebRTC UDP stream.

A single replica with `Recreate` avoids concurrent writes to pairing data and port
conflicts during upgrades. The workload does not tolerate the dedicated VPS taint.
Allow traffic from the selected node to the gaming PC's Sunshine service.

## First login and pairing

1. Open the HTTPS URL and create the first account, which becomes the administrator.
   Do this before exposing the route to other users. A local hostname alone does
   not restrict access, and host networking also exposes TCP 8087 on the node.
2. Add the gaming PC using its LAN IP or resolvable hostname. Do not use `localhost`,
   which refers to the Kubernetes node here.
3. Enter the displayed pairing PIN in Sunshine on the gaming PC, then launch a game.

Sunshine runs on the gaming PC and is not installed by this deployment. HTTPS is
needed for browser controller and keyboard APIs. For clients that cannot reach the
node UDP ports, select **Web Sockets** under **Data Transport**. Remote WebRTC
streaming needs a reachable VPN route, suitable port forwarding, or a separately
configured TURN server. This PR does not configure internet port forwarding.

## Verify after deployment

```sh
kubectl -n media rollout status deployment/moonlight-web-deployment-production
kubectl -n media logs deployment/moonlight-web-deployment-production
```

Verify login, pairing, video, audio and controller input from a LAN browser, then
check that accounts and pairing survive a pod restart. The PR validates manifests;
end-to-end streaming requires the cluster and a Sunshine host.

Upstream: [Docker setup](https://github.com/MrCreativ3001/moonlight-web-stream/blob/v2.10.0/docker/README.md)
and [networking and HTTPS](https://github.com/MrCreativ3001/moonlight-web-stream/blob/v2.10.0/README.md).

# CrowdSec

Blocks abusive IPs on public routes.

- **Detection:** agents (DaemonSet, every node) parse Traefik access logs and Jellyfin logs, then report to the LAPI in `network-system`
- **Blocking:** Traefik bouncer plugin, applied per route via the `crowdsec` middleware. It fails open if the LAPI is down
- **Config:** `apps/production/network-system/crowdsec/configmap.yaml` (log sources, collections)

## Protect an app

1. Add the middleware to its public route:
   ```yaml
   middlewares:
     - name: crowdsec
       namespace: network-system
   ```
2. Optional, for app-specific detection (e.g. login brute force): add a log `acquisition` and a hub collection in the configmap

## Behind Cloudflare

Public DNS is Cloudflare-proxied, so requests arrive from Cloudflare edge IPs. Without handling, a ban would hit an edge and block every user behind it.

- Traefik `public` entrypoint trusts Cloudflare ranges for `X-Forwarded-For`
- Access logs are JSON, so CrowdSec uses the last XFF IP (the one Cloudflare added, so clients can't spoof it)
- Bouncer `forwardedHeadersTrustedIPs` skips Cloudflare ranges
- `crowdsecurity/cdn-whitelist` stops Cloudflare IPs from ever being banned (updates itself)
- Jellyfin **Known proxies** (`/config/network.xml`, not in git) include Cloudflare ranges so failed logins log the real IP

### Updating Cloudflare ranges

These ranges rarely change: <https://www.cloudflare.com/ips>. A missed range is not harmful: those clients go undetected, but `cdn-whitelist` still prevents bans.

Hand-copied in:
- `apps/production/network-system/traefik/patch_deployment.yaml` (`trustedIPs`)
- `apps/base/network-system/crowdsec/middleware.yaml`
- Jellyfin `network.xml` (Dashboard → Networking → Known proxies)

## Useful commands

```bash
kubectl -n network-system exec deploy/crowdsec-lapi -- cscli decisions list
kubectl -n network-system exec deploy/crowdsec-lapi -- cscli decisions delete --ip <ip>
kubectl -n network-system exec deploy/crowdsec-lapi -- cscli alerts list
```

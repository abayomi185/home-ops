# Home Assistant local access

`https://home.local.yomitosh.media` routes through the Kubernetes Traefik ingress
to `http://home.internal.yomitosh.media:8123`. The dedicated middleware limits
access to the trusted LAN and WireGuard networks.

Home Assistant runs outside this repository. Its HTTP settings must enable
`use_x_forwarded_for` and trust the Kubernetes nodes' egress addresses:

- `10.1.5.71/32` (knode1)
- `10.1.5.72/32` (knode2)
- `10.1.5.73/32` (knode3)

These entries were added to the existing trusted proxies on 2026-09-21. Preserve
other existing entries when editing this list. Home Assistant sees the node's
address after Kubernetes SNAT, rather than the Traefik pod address.

Home Assistant 2026.8 and later manage these settings under Settings → System →
Network → HTTP server. Saving restarts Home Assistant; confirm the settings
within five minutes after verifying access, or they automatically revert.

See the [Home Assistant HTTP documentation](https://www.home-assistant.io/integrations/http/#reverse-proxies).

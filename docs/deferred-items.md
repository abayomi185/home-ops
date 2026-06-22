# Deferred Follow-ups

Living list of work that is out of scope for the current effort but
must not be forgotten. Cross off as resolved; do not silently delete.

## Completed

- [x] **In-cluster Traefik + MetalLB cutover.** MetalLB L2 mode on
      `10.1.5.96` (VLAN 5), in-cluster Traefik v2 with Cloudflare
      DNS-01 ACME. All 15 k8s app routes served via IngressRoutes on
      `*.local.yomitosh.media`. DNS repointed in Unbound
      (`local.yomitosh.media` → `10.1.5.96`). External kloadbalancer
      LXC at `10.1.5.40` is bypassed for all k8s traffic.
      (PRs #45, #47, #48, #49)
- [x] **Convert all NodePort services to ClusterIP.** 19 service files
      converted. All external access now flows through Traefik
      IngressRoutes. Zero manually-specified NodePorts remain. Only
      Traefik's `type: LoadBalancer` service has auto-allocated
      NodePorts (30430, 31097) — expected, not removable.
      (PR #49)
- [x] **Drop `*.internal.yomitosh.media` from IngressRoutes.**
      `internal.yomitosh.media` is a Dnsmasq zone for infrastructure
      devices (knodes, firewall, network-share via DHCP reservations),
      not for k8s app routing. App hostnames like
      `bazarr.internal.yomitosh.media` never resolved. All
      IngressRoutes now match `*.local.yomitosh.media` only.
      (PR #48)
- [x] **Fix restreamer duplicate nodePort bug.** `service.yaml`
      assigned `nodePort: 30686` to five different ports. Converted to
      ClusterIP with unique port names. App is not deployed (not in
      any production kustomization) but the base manifest is now
      correct.

## LB migration — long tail

These land after the in-cluster Traefik + MetalLB cutover is stable.

- [ ] **Retire `kloadbalancer` LXC (10.1.5.40).** Goal per the migration
      plan. The LXC is now bypassed for all k8s traffic. It still
      serves non-k8s backends (see below). Blocked on migrating those
      or repointing their routes.
- [ ] **Move non-k8s backends into the cluster** (or replace with
      in-cluster equivalents). Currently routed by the external LXC
      Traefik via direct IP / internal DNS:
      - AdGuard Home — `10.0.7.53:80`
      - Authelia — `10.0.7.205:8008` (forward-auth dependency; see below)
      - Send2eReader — `10.0.7.205:3031`
      - Uptime-Kuma — `10.0.7.205:3001`
      - InfluxDB — `10.0.7.205:8086`
      - Guacamole — `10.0.7.205:8060`
      - MinIO / S3 — `10.0.7.201:9000`
      - Proxmox VE — `10.0.7.1:8006` (likely stays external; PVE is the
        hypervisor hosting the cluster — folding into the cluster it
        runs on is circular)
      - Home Assistant — `home.internal.yomitosh.media:8123`
      - LLM server — `machine-learning.internal.yomitosh.media:9000`
      - ComfyUI — `machine-learning.internal.yomitosh.media:8188`
      - 3D printer video — `k1c.internal.yomitosh.media:4409`
- [ ] **Expose Blocky web UI.** Blocky ad-blocker runs on the firewall
      (`blocky.nix`, port 4000) with no public route today. Decide
      whether it gets an IngressRoute or stays LAN-only.

## Authelia

- [ ] **Decide Authelia's future home.** In-cluster Traefik uses
      forward-auth pointing at the existing external Authelia
      (`10.0.7.205:8008`). Long-term options:
      (a) bring Authelia into the cluster as a k8s deployment;
      (b) keep it on the LXC/10.0.7 segment and treat it as a
          non-k8s backend via `ExternalName` Service + IngressRoute;
      (c) replace with a different identity provider.

## Grafana duplication

- [ ] **Consolidate the two Grafana instances.** One lives on the
      firewall (`prometheus.nix:79`, `10.1.10.1`), the other at
      `10.0.7.205:3000`. The external LB's `dynamic_config.nix` still
      routes `grafana.*` → `10.0.7.205:3000`, i.e. the route is
      **already stale** relative to the firewall instance. Pick one,
      repoint DNS / IngressRoute, decommission the other.

## Pre-existing bugs to clean up (not blocking the migration)

- [ ] **External LB `static_config.nix`** has
      `proxyProtocol.trustedIPs = ["10.13.13.1/24" "10.0.1.0/24"]` —
      `/24` mask on a single host (`10.13.13.1` is the VPS endpoint,
      not a subnet) and `10.0.1.0/24` is the leftover OPNsense LAN.
      Don't fix in place — the LXC is on borrowed time. The fix is
      "decommission the LXC".

## In-cluster Traefik hardening

- [ ] **Tighten `allowCrossNamespace=true`.** Convenient for the
      initial migration (IngressRoutes live alongside apps in any
      namespace). Restrict to `network-system` + explicit per-namespace
      grants once the migration is stable, if multi-tenant concerns
      ever arise.
- [ ] **Consider cert-manager vs Traefik-native ACME.** Initial
      cutover mirrors the external LB: Traefik-native ACME with
      Cloudflare DNS-01. cert-manager would give cluster-wide cert
      inventory, rotation history, and decouple cert issuance from
      the ingress controller. Revisit if cert surface grows.
- [ ] **Scale Traefik replicas.** Currently 2 replicas (manual
      scale-up from initial 1). No PodDisruptionBudget or
      autoscaling configured. Add HPA or fixed 3-replica + PDB for
      rolling update safety.

## Cluster bridge

- [ ] **10.0.7.0/24 is Proxmox-internal, not routed by the firewall.**
      Any non-k8s backend on that segment (Authelia, AdGuard, MinIO,
      etc.) is only reachable from the cluster via the knodes' eth1.
      When those services move into the cluster, the 10.0.7 segment's
      role shrinks; revisit whether the bridge is still needed.

## VPS as k3s node

- [ ] **Consider adding the VPS as a k3s agent node.** The VPS
      currently runs its own Traefik v3, Authelia, and Uptime-Kuma
      independently. If the VPS joins the cluster as an agent:
      - Public Traefik can be an in-cluster Deployment with a
        LoadBalancer service on the VPS's public IP (MetalLB L2
        announcement via WireGuard, or a dedicated pool for the
        VPS's public subnet).
      - Authelia, Uptime-Kuma, and other VPS-hosted services become
        k8s Deployments, managed by Flux like everything else.
      - The VPS's Traefik static config, file providers, and NixOS
        service definitions are replaced by k8s manifests.
      - **Workload isolation:** use taints + tolerations (like knode4
        with `dedicated=gpu:NoSchedule`) to restrict which pods run on
        the VPS — e.g. only the public Traefik + Authelia, not media
        workloads.
      - **Challenge:** the VPS is behind WireGuard to the LAN, not on
        the same L2 as the knodes. k3s agent traffic would tunnel
        through WireGuard to knode1's API server. Latency and
        reliability of the WG tunnel become cluster-critical.
      - **Challenge:** the VPS runs NixOS — k3s agent config would
        need to be added to `nix-dotfiles/hosts/vps/configuration.nix`.
      - This is a significant architecture change. Plan thoroughly
        before executing.

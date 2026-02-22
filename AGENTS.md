# HOME-OPS KNOWLEDGE BASE

**Generated:** 2026-02-22 00:24 UTC
**Commit:** f9c1dd7
**Branch:** main

## OVERVIEW

Flux-managed Kubernetes homelab running in Proxmox. GitOps workflow using Kustomize overlays with SOPS-encrypted secrets.

## STRUCTURE

```
home-ops/
├── apps/                    # Application manifests
│   ├── base/                # Shared base configs (no env-specific values)
│   └── production/          # Production overlays
├── clusters/homelab/        # Cluster-specific configs
│   ├── flux/flux-system/    # Flux bootstrap (DO NOT EDIT)
│   ├── configs/             # Cluster secrets & settings
│   ├── alerts/              # Alerting (Discord)
│   └── ingress/             # Ingress controllers
└── docs/                    # Documentation
```

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Add new app | `apps/base/<category>/<app>/` | Create deployment.yaml + service.yaml + kustomization.yaml |
| Override for production | `apps/production/<category>/<app>/` | Reference `../../../base/<category>/<app>` |
| Modify Flux config | `clusters/homelab/flux/flux-system/` | Re-run `flux bootstrap` after changes |
| Add cluster secret | `clusters/homelab/configs/` | Must be `.sops.yaml` encrypted |
| Configure alerts | `clusters/homelab/alerts/` | Discord webhook via SOPS |
| Ingress/routes | `clusters/homelab/ingress/` | Traefik configs |

## CONVENTIONS

### Kustomize Pattern
- Base configs in `apps/base/<category>/<app>/`
- Production overlays reference base: `../../../base/<category>/<app>`
- Use `nameSuffix: -production` in production kustomization.yaml

### Deployment Standard
- PUID: 1001, PGID: 100, UMASK: "007", TZ: Europe/London
- NFS volumes for config (`${NFS_STORAGE_IP}`, `${NFS_STORAGE_CONFIG_DATA_PATH}`)
- NFS volumes for media (`/mnt/mopower/swarm-data`)

### Secret Management
- Secrets MUST be SOPS-encrypted: `sops -e -i secret.sops.yaml`
- Pattern: `*.sops.yaml` or `*.enc.yaml`
- Pre-commit hook forbids unencrypted secrets

## ANTI-PATTERNS (THIS PROJECT)

- **DO NOT** edit `gotk-components.yaml` or `gotk-sync.yaml` (Flux-generated)
- **DO NOT** commit unencrypted secrets (blocked by pre-commit)
- **CONSIDER** changing `imagePullPolicy: Always` to `IfNotPresent` (18 deployments)

## COMMANDS

```bash
# Encrypt secret
export SOPS_AGE_KEY=$(kubectl get secret sops-age -n flux-system -o=jsonpath='{.data}' | jq -r '."age.agekey"' | base64 --decode | tail -n 1)
sops -e -i secret.sops.yaml

# Decrypt secret
sops -d secret.sops.yaml

# Re-bootstrap Flux
flux bootstrap github --owner=abayomi185 --repository=home-ops --branch=main --path=./clusters/homelab/flux --personal --token-auth

# Add SOPS age key to cluster
kubectl create secret generic sops-age --namespace=flux-system --from-file=age.agekey=/dev/stdin < age.agekey
```

## NOTES

- 3 AGE keys configured: mbp16, mbp14, cluster
- No CI/CD pipelines (GitOps only)
- Pre-commit validates secret encryption
- Apps organized by category: media, productivity, network-system, kube-system, cert-manager, storage, monitoring

# CLUSTERS/HOMELAB KNOWLEDGE BASE

**Generated:** 2026-02-22 00:24 UTC

## OVERVIEW

Cluster-specific configuration for the homelab Kubernetes cluster. Contains Flux system, cluster secrets, ingress controllers, and alerting.

## STRUCTURE

```
clusters/homelab/
├── flux/flux-system/        # Flux bootstrap (DO NOT EDIT manually)
│   ├── gotk-components.yaml # Flux controllers - auto-generated
│   ├── gotk-sync.yaml       # GitRepository + Kustomization - auto-generated
│   └── kustomization.yaml   # References above two files
├── configs/                 # Cluster settings & secrets
│   ├── kustomization.yaml
│   └── cluster-secrets.sops.yaml
├── alerts/                  # Alerting configuration
│   └── discord/
│       ├── kustomization.yaml
│       └── secret.sops.yaml
└── ingress/                 # Ingress controllers
    └── traefik/
        └── kustomization.yaml
```

## WHERE TO LOOK

| Task | Location |
|------|----------|
| Modify Flux sync | Re-run `flux bootstrap` command |
| Add cluster secret | `configs/` with `.sops.yaml` suffix |
| Configure Discord alerts | `alerts/discord/` |
| Modify Traefik ingress | `ingress/traefik/` |

## CONVENTIONS

### Flux Bootstrap Path
```bash
flux bootstrap github \
  --owner=abayomi185 \
  --repository=home-ops \
  --branch=main \
  --path=./clusters/homelab/flux \
  --personal \
  --token-auth
```

### Secret Pattern
- All cluster secrets in `configs/` as `*.sops.yaml`
- Must encrypt before commit: `sops -e -i secret.sops.yaml`

## ANTI-PATTERNS (THIS PROJECT)

- **DO NOT** edit `gotk-components.yaml` or `gotk-sync.yaml`
- These are Flux-generated; changes will be overwritten
- Use `flux bootstrap` to regenerate after cluster changes

## NOTES

- Flux deprecated APIs present (v1beta1) - upgrade planned
- SOPS age key stored in cluster as `sops-age` secret in `flux-system` namespace

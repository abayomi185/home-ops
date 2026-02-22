# APPS KNOWLEDGE BASE

**Generated:** 2026-02-22 00:24 UTC

## OVERVIEW

Application manifests using Kustomize base/overlay pattern. Base holds shared configs; production holds environment-specific overlays.

## STRUCTURE

```
apps/
├── kustomization.yaml       # Entry point → references production/
├── base/                    # Shared configs (no env values)
│   ├── media/               # Sonarr, Radarr, Jellyfin, etc.
│   ├── productivity/        # Nextcloud, Open-WebUI
│   ├── network-system/      # Traefik, MetalLB
│   ├── kube-system/         # CSI-driver, Intel plugin, NFD
│   ├── cert-manager/        # Certificate management
│   └── storage/             # Storage configs
└── production/              # Environment overlays
    ├── kustomization.yaml   # nameSuffix: -production, labels: env=production
    └── <category>/<app>/    # References ../../../base/<category>/<app>
```

## WHERE TO LOOK

| Task | Location |
|------|----------|
| Add media app | `base/media/<new-app>/` then `production/media/<new-app>/` |
| Add productivity app | `base/productivity/<new-app>/` then `production/productivity/<new-app>/` |
| Modify network config | `base/network-system/` and `production/network-system/` |
| Add secret | `production/<category>/<app>/secret.sops.yaml` |

## CONVENTIONS

### Adding New App
1. Create `base/<category>/<app>/kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - ./deployment.yaml
     - ./service.yaml
   ```
2. Create `deployment.yaml` with standard PUID/PGID/UMASK/TZ
3. Create `production/<category>/<app>/kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - ../../../base/<category>/<app>
   ```
4. Add to `production/<category>/kustomization.yaml` resources list

### Deployment Template
```yaml
spec:
  containers:
    - env:
        - name: UMASK
          value: "007"
        - name: TZ
          value: Europe/London
        - name: PUID
          value: "1001"
        - name: PGID
          value: "100"
      image: <image>:<tag>
      imagePullPolicy: IfNotPresent  # Prefer over Always
  volumes:
    - name: config-volume
      nfs:
        server: ${NFS_STORAGE_IP}
        path: ${NFS_STORAGE_CONFIG_DATA_PATH}
```

## ANTI-PATTERNS (THIS PROJECT)

- **DO NOT** hardcode NFS IPs in base/ (use env vars)
- **PREFER** `imagePullPolicy: IfNotPresent` over `Always`
- **DO NOT** put secrets in base/ (production-only)

# Wayom

Rust API (axum) for personal finance tracking. Syncs accounts from Lunch Flow
and GoCardless, stores transactions/holdings in PostgreSQL.

## Architecture

- **Image**: `ghcr.io/abayomi185/wayom` — built by GitHub Actions in
  [wayom-rs](https://github.com/abayomi185/wayom-rs) on push to `main` (`:latest`
  + `:sha-<short>`). Multi-arch (amd64 + arm64).
- **Database**: CNPG `Cluster` named `wayom-database` (1 instance, 4 Gi storage).
  Creates a `wayom-database-app` Secret with a `uri` key — used as
  `DATABASE_URL` by the server.
- **Migrations**: run automatically on server startup via `sqlx::migrate!`
  (embedded at compile time). No initContainer needed — the server panics
  if migrations fail, causing Kubernetes to restart the pod.
- **Sync scheduler**: in-server tokio task that calls `sync_accounts` for the
  seeded `system` user every hour (`WAYOM_SYNC_INTERVAL_SECS`). Builds a
  financial record without manual HTTP calls. Disable with
  `WAYOM_SYNC_ENABLED=false`.
- **Exposure**: ClusterIP + Traefik IngressRoute at `wayom.local.${DOMAIN}` (TLS via Let's Encrypt).

## Secrets

`secret.sops.yaml` must be SOPS-encrypted before committing:

```sh
export SOPS_AGE_KEY=$(kubectl get secret sops-age -n flux-system -o=jsonpath='{.data}' | jq -r '."age.agekey"' | base64 --decode | tail -n 1)
sops -e -i secret.sops.yaml
```

Required keys:

| Key | Notes |
|-----|-------|
| `jwt_secret` | **Must be < 32 characters** (the app panics if ≥ 32 — likely a bug to fix in wayom-rs, but respect it for now). |
| `github_oauth_client_id` | GitHub OAuth App client ID. |
| `github_oauth_client_secret` | GitHub OAuth App client secret. |
| `github_oauth_redirect_url` | e.g. `https://wayom.local.yomitosh.media/api/auth/github/callback` — must match the GitHub OAuth App callback URL. |
| `lunch_flow_api_key` | Lunch Flow personal API key. |

## Config

`configmap.yaml` mounts `wayom.toml` at `/config/wayom.toml`
(`WAYOM_CONFIG_PATH=/config/wayom.toml`). Update `base_url` to the externally
reachable URL (used for CORS origin matching and GoCardless callbacks).

## Image updates

The deployment pins `ghcr.io/abayomi185/wayom:latest`. After pushing a new
image to `main`, update the digest in `deployment.yaml` (Renovate will keep it
fresh once a `@sha256:` digest is pinned).

To pin a specific digest:
```sh
docker buildx imagetools inspect ghcr.io/abayomi185/wayom:latest --format '{{json .Manifest}}' | jq -r '.digest'
```
Then append `@sha256:<digest>` to the image reference.

## GHCR authentication

The wayom-rs repo is private, so the GHCR package requires authentication to
pull. The deployment references an `imagePullSecret` named `ghcr-pull-secret`,
backed by the SOPS-encrypted `ghcr-secret.sops.yaml`.

To generate the secret with your GitHub PAT (`read:packages` scope):

```sh
export SOPS_AGE_KEY_FILE=cluster.agekey

# 1. Generate the Secret manifest (replace <PAT> with your GitHub PAT)
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace=productivity \
  --docker-server=ghcr.io \
  --docker-username=abayomi185 \
  --docker-password=<PAT> \
  --dry-run=client -o yaml > apps/production/productivity/wayom/ghcr-secret.sops.yaml

# 2. SOPS-encrypt it in-place
sops --encrypt --in-place apps/production/productivity/wayom/ghcr-secret.sops.yaml
```

> **Note**: the committed file currently contains a placeholder — replace it
> with a real PAT before merging. Flux will fail to pull the image otherwise.

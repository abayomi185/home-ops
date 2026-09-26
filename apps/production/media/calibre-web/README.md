# Calibre-Web

UI: `https://calibre.local.${DOMAIN}`

## Storage

On `${NFS_STORAGE_HOST_FAST}`, group `100`, mode `2770`:

- `${NFS_STORAGE_CONFIG_DATA_PATH}/calibre-web/config` → `/config`
- `${NFS_STORAGE_MAIN_DATA_PATH}/calibre-web/library` → `/books` (Calibre library with `metadata.db`)

## First login

`admin` / `admin123`. Change the password, set library to `/books`.

## Kobo sync

Public at `https://calibre.${DOMAIN}/kobo/<token>` (only `/kobo/` exposed). Set as `api_endpoint` in `.kobo/Kobo/Kobo eReader.conf`.

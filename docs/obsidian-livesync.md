# Obsidian LiveSync

CouchDB backend for the [Self-hosted LiveSync](https://github.com/vrtmrz/obsidian-livesync) Obsidian plugin.

- **App:** `apps/{base,production}/productivity/obsidian-couchdb`
- **URL:** `https://obsidian-couchdb.${DOMAIN}` (public, CrowdSec), `https://obsidian-couchdb.local.${DOMAIN}` (LAN/VPN)
- **Data:** NFS `${NFS_STORAGE_CONFIG_DATA_PATH}/obsidian-couchdb` (`1000:100`, `2770`; pod runs as `1001:100`)
- **LiveSync server settings:** ConfigMap `livesync.ini`. Changes made at runtime (Fauxton, the plugin's "Fix" buttons) are lost on restart, so edit the ConfigMap

## Credentials

```bash
sops -d apps/production/productivity/obsidian-couchdb/secret.sops.yaml
```

`admin_user` / `admin_password` are what the plugin uses.

Check the server responds:

```bash
curl -u '<user>:<password>' https://obsidian-couchdb.<domain>/_up   # {"status":"ok",...}
```

## First device

1. Obsidian → Settings → Community plugins → install and enable **Self-hosted LiveSync**
2. Skip the setup wizard and go to the plugin settings → **Remote Configuration**:
   - Remote type: CouchDB
   - URI: `https://obsidian-couchdb.<domain>` (no trailing slash)
   - Username / password: from the secret
   - Database name: `obsidian` (lowercase, one database per vault)
3. **Test** the connection, then **Check database configuration**: every item should pass. Don't use "Fix"
4. Turn on **End-to-end encryption** and set a passphrase. Store it in your password manager: without it the data can't be read or restored
5. **Sync Settings** → preset **LiveSync** → Apply
6. The plugin asks to initialise the remote database: accept, so this vault is uploaded

## Other devices (desktop, mobile, work laptop)

1. On the first device: plugin settings → **Setup** → **Copy current settings as a new setup URI** and give it a passphrase. Send the URI to the other device yourself (password manager, notes)
2. On the new device: create an empty vault, install and enable Self-hosted LiveSync
3. **Use the copied setup URI**, paste it, enter the passphrase, then choose the option for a device joining an existing vault (not the one that initialises the remote)
4. Wait for the first full sync before editing anything

## Troubleshooting

- **Suddenly getting 403s from the public URL:** CrowdSec may have banned your IP:
  ```bash
  kubectl -n network-system exec deploy/crowdsec-lapi -- cscli decisions list
  kubectl -n network-system exec deploy/crowdsec-lapi -- cscli decisions delete --ip <ip>
  ```
- **Logs:** `kubectl -n productivity logs deploy/obsidian-couchdb-deployment`

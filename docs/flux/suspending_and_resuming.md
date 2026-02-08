To suspend a flux deployment, use the following command:

```sh
# Replace "jellyfin-media-production" with the name of your kustomization
flux suspend kustomization jellyfin-media-production -n flux-system
```

To resume the deployment, use:

```sh
flux resume kustomization jellyfin-media-production -n flux-system
```

For when the GitHub PAT used by Flux to access private repositories needs to be updated:

```sh
kubectl -n flux-system delete secret flux-system
```

Bootstrap Flux again with the new token: [[./bootstrap.md]]

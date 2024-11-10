<div align="center">

### Home Ops :octocat:

</div>

GitOps workflow for homelab.

Homelab:
[Kubernetes](https://kubernetes.io/) cluster running in [Proxmox](https://www.proxmox.com/) and using [Flux](https://github.com/fluxcd/flux2) with Kustomization.

Taking inspiration from https://github.com/onedr0p/home-ops

### To encrypt secrets:

```sh
# Get AGE_KEY from the cluster and set it as SOPS_AGE_KEY
# AGE_KEY=$(kubectl get secret sops-age -n flux-system -o=jsonpath='{.data}' | jq -r '."age.agekey"' | base64 --decode)
# export SOPS_AGE_KEY=$AGE_KEY
export SOPS_AGE_KEY=$(kubectl get secret sops-age -n flux-system -o=jsonpath='{.data}' | jq -r '."age.agekey"' | base64 --decode | tail -n 1)

# Encrypt using
sops -e -i secret.sops.yaml

# Decrypt using
sops -d secret.sops.yaml
```

### To update GitHub Token:

```sh
# Delete the secret
kubectl -n flux-system delete secret flux-system

export GITHUB_USER=abayomi185

# Bootstrap again
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=home-ops \
  --branch=main \
  --path=./clusters/homelab/flux \
  --personal \
  --token-auth
```

```sh
# Add the sops age secret to flux.
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
secret/sops-age created
```

```sh
# The secret may need to be regenerated.
# If so, all the secrets will have to be updated with the new age public key
age-keygen -o age.agekey
export SOPS_SECRET=<path-to-secret>/secret.sops.yaml
sops -d -i $SOPS_SECRET && sops -e -i $SOPS_SECRET # This can be paired with the find command
```

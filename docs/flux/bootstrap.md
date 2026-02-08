The initial bootstrap command used, useful for reference and various other tasks:

```sh
export GITHUB_USER="abayomi185"
export GITHUB_TOKEN="" # Remove from zsh history after use
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=home-ops \
  --branch=main \
  --path=./clusters/homelab/flux \
  --personal \
  --token-auth=false
```

If you don't have access to the cluster via kubectl, you will have to update the config/certs in the kubeconfig. Access the k3s cluster and grab the config using `cat /etc/rancher/k3s/k3s.yaml`. Be sure to update the `server`, `name`, and `current-context` fields as necessary.

If the cluster no longer has SOPS access, you can generate a new key for the cluster and add it to the cluster with:

```sh
# Generate new cluster key
age-keygen -o cluster.agekey
```

Then update `.sops.yaml` with the new public key. Re-encrypt all secrets with the new key:

```sh
# sops updatekeys --input-type age --age-keys cluster.agekey --encrypted-regex '^(data|stringData)$' --in-place $(find . -type f -name '*.yaml')
sops updatekeys **/*.enc.yaml
sops updatekeys **/*.sops.yaml
```

The last step is to add the private key to the cluster with:

```sh
cat cluster.agekey | kubectl create secret generic sops-age \
     --namespace=flux-system \
     --from-file=age.agekey=/dev/stdin
```

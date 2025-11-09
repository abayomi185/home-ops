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

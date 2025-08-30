# Init k8s

## Full Cilium stack
kube-proxy is disabled by skipping the addon during cluster init.
Existing cluster can delete kube-proxy and skip the addon during cluster upgrade.

```sh
kubectl -n kube-system delete daemonset kube-proxy
kubectl -n kube-system delete cm kube-proxy
```

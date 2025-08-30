# Join the control-plane

## Replace a node
If node still exsits you can do following to get it out.

```
kubectl drain node <NODE>
kubectl delete node <NODE>
# run on the node
kubeadm reset
```
Now the node was removed and all data were deleted on the server.

If you have a broken node which you can't recover then remove the node from k8s and also etcd. To join a new one without any issue.
```
kubectl drain node <NODE>
kubectl delete node <NODE>
kubectl exec -it etcd-<NODE> -n kube-system -- /bin/sh
# list all etcd nodes to get the id of the broken node
etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key member list
# Remove the etcd node from the cluster to get it back into stable state.
etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key member remove <NODE-ID>
```

Run the ansible script and the node can join as new control-plane node.
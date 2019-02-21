
### Use

```
$ kubectl apply -f rbac-kdd.yaml 
$ kubectl apply -f calico.yaml
```

```
$ kubectl get pods -n kube-system | grep calico
calico-node-bt2gq                        2/2     Running   0          15h
calico-node-gln4j                        2/2     Running   0          15h
calico-node-ttj6q                        2/2     Running   0          15h
calico-node-vzdv9                        2/2     Running   0          15h
calico-node-wsc7s                        2/2     Running   0          15h
```

#### 1. 升级CoreDNS
- KubeDNS为老版本的DNS，目前推荐使用CoreDNS
- 升级CoreDNS方式如下:
```
cd CoreDNS; tree
.
├── coredns.yaml.sed
└── deploy.sh

$ ./deploy.sh | kubectl apply -f -
```
#### 2. 移除掉老的kube-dns，查看并测试

```
$ kubectl delete --namespace=kube-system deployment kube-dns
$ kubectl get pods -n kube-system |grep coredns;kubectl cluster-info |grep CoreDNS
$ kubectl exec -ti busybox -- nslookup my-apache
```

- 部署参考文档：
https://jimmysong.io/kubernetes-handbook/practice/coredns.html
- 官方提供脚本地址：
https://github.com/coredns/deployment/blob/master/kubernetes/deploy.sh
https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed
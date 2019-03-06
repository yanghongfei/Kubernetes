#### 使用Promethues监控外接ETCD集群 

- Q：为什么要监控ETCD？
- A： 因为K8S数据都是存在ETCD里面，ETCD非常重要！

> 因为默认Prometheus-operator是没有收集ETCD里面的数据的，因为ETCD比较重要，我们ETCD使用外接二进制形式部署高可用，因此这里ETCD需要单独配置serviceMonitor和services

- 参考文档：https://www.qikqiak.com/post/prometheus-operator-monitor-etcd/

**首先我们将需要使用到的证书通过 secret 对象保存到集群中去：(在 etcd 运行的节点)**

我这里是外接ETCD，证书也是前面部署生成的，所以我知道证书位置，若你的etcd是Kubeadm默认部署的，你需要先查看信息`kubectl get pod <etcd_pod_name> -n kube-system -o yaml`详细请参考[Prometheus监控ETCD](https://www.qikqiak.com/post/prometheus-operator-monitor-etcd/)

```shell
$ kubectl -n monitoring create secret generic etcd-certs --from-file=/etc/etcd/ssl/etcd.pem --from-file=/etc/etcd/ssl/etcd-key.pem --from-file=/etc/etcd/ssl/ca.pem
```

然后将上面创建的 etcd-certs 对象配置到 prometheus 资源对象中，直接更新 prometheus 资源对象即可：

```yaml
#因为的promethues有2个节点，所以2台机器都进行操作修改
$ kubectl edit prometheus k8s -n monitoring
#添加如下secrets属性
nodeSelector:
  beta.kubernetes.io/os: linux
replicas: 2
secrets:
- etcd-certs
```

更新完成后，我们就可以在 Prometheus 的 Pod 中获取到上面创建的 etcd 证书文件了，具体的路径我们可以进入 Pod 中查看

```shell
$ kubectl exec -it prometheus-k8s-0 /bin/sh -n monitoring
Defaulting container name to prometheus.
Use 'kubectl describe pod/prometheus-k8s-0 -n monitoring' to see all of the containers in this pod.
/ $ ls /etc/prometheus/secrets/etcd-certs/
ca.pem      etcd.pem  etcd-key.pem
```

**创建ServiceMonitor**

现在 Prometheus 访问 etcd 集群的证书已经准备好了，接下来创建 ServiceMonitor 对象即可（prometheus-serviceMonitorEtcd.yaml）

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd-k8s
spec:
  jobLabel: k8s-app
  endpoints:
  - port: port
    interval: 30s
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.pem
      certFile: /etc/prometheus/secrets/etcd-certs/etcd.pem
      keyFile: /etc/prometheus/secrets/etcd-certs/etcd-key.pem
      insecureSkipVerify: true
  selector:
    matchLabels:
      k8s-app: etcd
  namespaceSelector:
    matchNames:
    - kube-system
```

上面我们在 monitoring 命名空间下面创建了名为 etcd-k8s 的 ServiceMonitor 对象，基本属性和前面章节中的一致，匹配 kube-system 这个命名空间下面的具有 k8s-app=etcd 这个 label 标签的 Service，jobLabel 表示用于检索 job 任务名称的标签，和前面不太一样的地方是 endpoints 属性的写法，配置上访问 etcd 的相关证书，endpoints 属性下面可以配置很多抓取的参数，比如 relabel、proxyUrl，tlsConfig 表示用于配置抓取监控数据端点的 tls 认证，由于证书 serverName 和 etcd 中签发的可能不匹配，所以加上了 insecureSkipVerify=true



**创建Service**

ServiceMonitor 创建完成了，但是现在还没有关联的对应的 Service 对象，所以需要我们去手动创建一个 Service 对象（prometheus-etcdService.yaml）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd-k8s
  namespace: kube-system
  labels:
    k8s-app: etcd
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: port
    port: 2379
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-k8s
  namespace: kube-system
  labels:
    k8s-app: etcd
subsets:
- addresses:
  - ip: 172.16.1.50
   # nodeName: K8S01-Master01   集群有多个IP都要写进去
  - ip: 172.16.1.51
   # nodeName: K8S01-Master02
  - ip: 172.16.1.52
   # nodeName: K8S01-Node01
  - ip: 172.16.1.53
    #nodeName: K8S01-Node02
  - ip: 172.16.1.54
    #nodeName: K8S01-Node03
  ports:
  - name: port
    port: 2379
    protocol: TCP
```

Endpoints 的 subsets 中填写 etcd 集群的地址即可，我们这里是单节点的，填写一个即可，直接创建该 Service 资源：

```shell
$ kubectl create -f prometheus-etcdService.yaml
```

创建完成后，隔一会儿去 Prometheus 的 Dashboard 中查看 targets，便会有 etcd 的监控项了：



**配置ETCD监控报警规则**

到了这一步集群里面已经能prometheus到ETCD了，并且可以拿到数据，接下来就根据数据配置prometheus-rules规则，更多规则请参考：https://github.com/yanghongfei/Kubernetes/tree/master/kube-prometheus/manifests/prometheus/prometheus_rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: prometheus-etcd-rules
  namespace: monitoring
spec:
  groups:
  - name: EtcdMonitoring
    rules:
    - alert: EtcdDown 
      annotations:
        detail:  "{{$labels.instance}}: etcd down (当前值: {{ $value }})"
        summary: "{{$labels.instance}}: etcd 出现异常，请管理员尽快排查"
      expr: |
        up{endpoint="port",job="etcd",namespace="kube-system",service="etcd-k8s"} == 0
      for: 1m
      labels:
        severity: 严重
```

**另外附上自己整理的监控K8S集群规则**

规则路径：https://github.com/yanghongfei/Kubernetes/tree/master/kube-prometheus/manifests/prometheus/prometheus_rules

```shell
.
├── prometheus-altermanager-rules.yaml    #监控Altermanger存活
├── prometheus-cpu-rules.yaml             #监控NodeCPU利用率和NodeCPULoad负载
├── prometheus-deployment-rules.yaml      #监控K8S集群中是否有部署报错
├── prometheus-disk-rules.yaml            #监控磁盘空间，磁盘IO后续更新
├── prometheus-etcd-rules.yaml            #监控ETCD集群存活
├── prometheus-grafana-rules.yaml         #监控Grafana是否存活
├── prometheus-k8s-pod-rules.yaml         #监控K8S集群中POD重启和异常POD信息
├── prometheus-k8s-rules.yaml             #监控K8SMaster核心组件，如:APIServer 调度器等
├── prometheus-memory-rules.yaml          #监控Node主机内存使用情况
├── prometheus-node_exporter-rules.yaml   #监控Node采集器存活情况
└── prometheus-prometheus-rules.yaml      #监控Promethues服务存活
```

### 06. 安装配置etcd

#### 6.1 安装etcd

```shell
//所有节点install
$ yum install etcd -y   
$ mkdir -p /var/lib/etcd
```

#### 6.2 配置etcd
> 注意： 每台机器的配置文件已在当前目录下

master01的`etcd.service`

```shell
cat <<EOF >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name k8s01 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.16.1.11:2380 \
  --listen-peer-urls https://172.16.1.11:2380 \
  --listen-client-urls https://172.16.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.1.11:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s01=https://172.16.1.11:2380,k8s02=https://172.16.1.12:2380,k8s03=https://172.16.1.13:2380,k8s04=https://172.16.1.14:2380,k8s05=https://172.16.1.15:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

master02的`etcd.service`

```shell
cat <<EOF >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name k8s02 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.16.1.12:2380 \
  --listen-peer-urls https://172.16.1.12:2380 \
  --listen-client-urls https://172.16.1.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.1.12:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s01=https://172.16.1.11:2380,k8s02=https://172.16.1.12:2380,k8s03=https://172.16.1.13:2380,k8s04=https://172.16.1.14:2380,k8s05=https://172.16.1.15:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

node01的`etcd.service`

```shell
cat <<EOF >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name k8s03 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.16.1.13:2380 \
  --listen-peer-urls https://172.16.1.13:2380 \
  --listen-client-urls https://172.16.1.13:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.1.13:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s01=https://172.16.1.11:2380,k8s02=https://172.16.1.12:2380,k8s03=https://172.16.1.13:2380,k8s04=https://172.16.1.14:2380,k8s05=https://172.16.1.15:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

node02的`etcd.service`

```shell
cat <<EOF >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name k8s04 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.16.1.14:2380 \
  --listen-peer-urls https://172.16.1.14:2380 \
  --listen-client-urls https://172.16.1.14:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.1.14:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s01=https://172.16.1.11:2380,k8s02=https://172.16.1.12:2380,k8s03=https://172.16.1.13:2380,k8s04=https://172.16.1.14:2380,k8s05=https://172.16.1.15:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

node03的`etcd.service`

```shell
cat <<EOF >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name k8s05 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.16.1.15:2380 \
  --listen-peer-urls https://172.16.1.15:2380 \
  --listen-client-urls https://172.16.1.15:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.1.15:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s01=https://172.16.1.11:2380,k8s02=https://172.16.1.12:2380,k8s03=https://172.16.1.13:2380,k8s04=https://172.16.1.14:2380,k8s05=https://172.16.1.15:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

#### 6.3  启动etcd

```shell
 systemctl daemon-reload
 systemctl enable etcd
 systemctl start etcd
 systemctl status etcd
```

#### 6.4 集群状态检查(维护)

> 使用v3版本API

```shell
$ echo "export ETCDCTL_API=3" >>/etc/profile  && source /etc/profile
$ etcdctl version
etcdctl version: 3.2.18
API version: 3.2
```

> 查看集群健康状态

```shell
$ etcdctl --endpoints=https://172.16.1.11:2379,https://172.16.1.12:2379,https://172.16.1.13:2379,https://172.16.1.14:2379,https://172.16.1.15:2379 --cacert=/etc/etcd/ssl/ca.pem   --cert=/etc/etcd/ssl/etcd.pem   --key=/etc/etcd/ssl/etcd-key.pem   endpoint health
//输出信息如下：
https://172.16.1.13:2379 is healthy: successfully committed proposal: took = 1.190355ms
https://172.16.1.14:2379 is healthy: successfully committed proposal: took = 1.678526ms
https://172.16.1.12:2379 is healthy: successfully committed proposal: took = 1.614457ms
https://172.16.1.15:2379 is healthy: successfully committed proposal: took = 2.220135ms
https://172.16.1.11:2379 is healthy: successfully committed proposal: took = 18.170259ms
```

> 查询所有key

```shell
$ etcdctl --endpoints=https://172.16.1.11:2379,https://172.16.1.12:2379,https://172.16.1.13:2379,https://172.16.1.14:2379,https://172.16.1.15:2379 --cacert=/etc/etcd/ssl/ca.pem   --cert=/etc/etcd/ssl/etcd.pem   --key=/etc/etcd/ssl/etcd-key.pem    get / --prefix --keys-only

// kubeadm初始化之前是没有任何信息的，初始化完成后查询得到的信息如：
/registry/apiregistration.k8s.io/apiservices/v1.
/registry/apiregistration.k8s.io/apiservices/v1.apps
/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.autoscaling
/registry/apiregistration.k8s.io/apiservices/v1.batch
........................
```

> 清除`所有/指定`key(**生成环境慎用**)
>
> 线上环境如有k8s组件出现问题,需要针对特定问题key进行清除操作。

```shell
 etcdctl --endpoints=https://172.16.1.11:2379,https://172.16.1.12:2379,https://172.16.1.13:2379,https://172.16.1.14:2379,https://172.16.1.15:2379 --cacert=/etc/etcd/ssl/ca.pem   --cert=/etc/etcd/ssl/etcd.pem   --key=/etc/etcd/ssl/etcd-key.pem    del / --prefix 
```
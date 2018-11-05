###                                    基于Kubeadm部署Kubernetes1.10集群 



### 01. 部署目的

#### 1.1 Kubernetes的特性

- 分布式部署
- 服务发现
- 服务实例保护
- 滚动升级
- 节点无痕维护
- 负载平衡
- 动态扩缩容

从而能够贴合未来微服部署维护的需求

#### 1.2 贴微服务，开发环境快速部署

> 通过docker镜像，可以快速并且一致的为每个开发人员提供相同的linux开发环境，省去了每个员工自行部署开发环境带来的疑问和冗余，让开发人员能够能专注于code本身，节省大量时间



### 02. 环境说明

- 五台机器进行部署K8S v1.10+集群环境，一台内网Harbor。其中`etcd`为所有节点部署
- Kubernetes中所有数据都是存储在etcd中的，etcd必须高可用集群
- Master使用keepalived高可用，Master主要是分发用户操作指令等操作；
- Master官方给出是用keepalived进行集群，建议也可以使用自建LB/商业AWS的（ALB ELB ）

|            System             |   Roles   |  IP Address  |
| :---------------------------: | :-------: | :----------: |
| CentOS Linux release 7.4.1708 | Master01  | 172.16.1.11  |
| CentOS Linux release 7.4.1708 | Master02  | 172.16.1.12  |
| CentOS Linux release 7.4.1708 |  Node01   | 172.16.1.13  |
| CentOS Linux release 7.4.1708 |  Node02   | 172.16.1.14  |
| CentOS Linux release 7.4.1708 |  Node01   | 172.16.1.15  |
| CentOS Linux release 7.4.1708 | VM Harbor | 172.16.0.181 |

#### 2.1 集群说明

|    Software     | Version |
| :-------------: | :-----: |
|   Kubernetes    |  1.10   |
|    Docker-CE    |  17.03  |
|      Etcd       |  3.1.5  |
|     Flannel     |  0.7.1  |
|    Dashboard    |  1.8.3  |
|    Heapster     |  1.5.0  |
| Traefik Ingress |  1.6.0  |



### 03. K8S集群名词说明

#### 3.1 Kubernetes

>  Kubernetes 是 Google 团队发起并维护的基于Docker的开源容器集群管理系统，它不仅支持常见的云平台，而且支持内部数据中心。建于 Docker 之上的 Kubernetes 可以构建一个容器的调度服务，其目的是让用户透过Kubernetes集群来进行云端容器集群的管理，而无需用户进行复杂的设置工作。系统会自动选取合适的工作节点来执行具体的容器集群调度处理工作。其核心概念是Container Pod（容器仓）。一个Pod是有一组工作于同一物理工作节点的容器构成的。这些组容器拥有相同的网络命名空间/IP以及存储配额，可以根据实际情况对每一个Pod进行端口映射。此外，Kubernetes工作节点会由主系统进行管理，节点包含了能够运行Docker容器所用到的服务。

####  3.2 Docker

>  Docker是一个开源的引擎，可以轻松的为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括VMs（虚拟机）、bare metal、OpenStack 集群和其他的基础应用平台。

####  3.3 Etcd

> ETCD是用于共享配置和服务发现的分布式，一致性的KV存储系统。

#### 3.4 Calico

> 同Flannel,用于解决 docker 容器直接跨主机的通信问题，Calico的实现是非侵入式的，不封包解包，直接通过iptables转发，基本没有消耗，flannel需要封包解包，有cpu消耗，效率不如calico，calico基本和原机差不多了 



### 04. 开始部署Kubernetes集群

#### 4.1 安装前准备

截至2018年06月，Kubernetes目前文档版本：v1.10+  官方版本迭代很快，我们选择目前文档版本搭建

**K8S所有节点配置主机名**

```shell
$ yum update
$ hostnamectl set-hostname OPS-SZNW-K8S01-Master01
$ hostnamectl set-hostname OPS-SZNW-K8S01-Master02
$ hostnamectl set-hostname OPS-SZNW-K8S01-Node01
$ hostnamectl set-hostname OPS-SZNW-K8S01-Node02
$ hostnamectl set-hostname OPS-SZNW-K8S01-Node03
```

```shell
$ cat <<EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.1.11 master01 OPS-SZNW-K8S01-Master01
172.16.1.12 master02 OPS-SZNW-K8S01-Master02
172.16.1.13 node01   OPS-SZNW-K8S01-Node01
172.16.1.14 node02   OPS-SZNW-K8S01-Node02
172.16.1.15 node03   OPS-SZNW-K8S01-Node03
EOF
```

**配置免密钥登陆**

```shell
$ ssh-keygen  
$ ssh-copy-id   master01
$ ssh-copy-id   master02
$ ssh-copy-id   node01
$ ssh-copy-id   node02
```

#### 4.2 安装Docker-CE

```shell
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-17.03.2.ce-1.el7.centos.x86_64.rpm
yum install docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
yum install docker-ce-17.03.2.ce-1.el7.centos.x86_64.rpm 
```

#### 4.3 安装Go

````shell
$ wget https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz
$ tar -C /usr/local -xzf go1.10.1.linux-amd64.tar.gz
$ export PATH=$PATH:/usr/local/go/bin
$ go version
go version go1.10.1 linux/amd64
````

#### 4.4 优化系统和集群准备

```shell
$ systemctl stop firewalld
$ systemctl disable firewalld
// 为了安全起见， docker 在 1.13 版本之后，将系统iptables 中 FORWARD 链的默认策略设置为 DROP
$ iptables -P FORWARD ACCEPT

$ swapoff -a 
$ sed -i 's/.*swap.*/#&/' /etc/fstab
$ yum install ntp ntpdate -y
$ service ntpd start

$ setenforce  0 
$ sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
$ sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
$ sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
$ sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config  

$ modprobe br_netfilter
$ cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sysctl -p /etc/sysctl.d/k8s.conf
$ ls /proc/sys/net/bridge

$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

$ echo "* soft nofile 204800" >> /etc/security/limits.conf
$ echo "* hard nofile 204800" >> /etc/security/limits.conf
$ echo "* soft nproc 204800"  >> /etc/security/limits.conf
$ echo "* hard nproc 204800"  >> /etc/security/limits.conf
$ echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
$ echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
```

#### 4.5 所有节点配置Docker镜像加速 

阿里云容器镜像加速器配置地址<https://dev.aliyun.com/search.html>  登录管理中心获取个人专属加速器地址

```shell
$ sudo mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://3csy84rx.mirror.aliyuncs.com"]
}
EOF
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```





### 05. 生成TLS证书和秘钥

#### 5.1 Kubernetes 集群所需证书

> `ca`证书为集群admin证书。
>
> `etcd`证书为etcd集群使用。
>
> `shinezone`证书为Harbor使用。

|      CA&Key       | etcd | api-server | proxy | kebectl | Calico | harbor |
| :---------------: | :--: | :--------: | :---: | :-----: | :----: | :----: |
|      ca.csr       |  √   |     √      |   √   |    √    |   √    |        |
|      ca.pem       |  √   |     √      |   √   |    √    |   √    |        |
|    ca-key.pem     |  √   |     √      |   √   |    √    |   √    |        |
|      ca.pem       |  √   |            |       |         |        |        |
|     etcd.csr      |  √   |            |       |         |        |        |
|   etcd-key.pem    |  √   |            |       |         |        |        |
| shinezone.com.crt |      |            |       |         |        |   √    |
| shinezone.com.key |      |            |       |         |        |   √    |



#### 5.2 安装CFSSL

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
export PATH=/usr/local/bin:$PATH
```

#### 5.3 创建CA文件,生成etcd证书

```
mkdir /root/ssl
cd /root/ssl
cat >  ca-config.json <<EOF
{
"signing": {
"default": {
  "expiry": "8760h"
},
"profiles": {
  "kubernetes-Soulmate": {
    "usages": [
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
    ],
    "expiry": "8760h"
  }
}
}
}
EOF

cat >  ca-csr.json <<EOF
{
"CN": "kubernetes-Soulmate",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
  "C": "CN",
  "ST": "shanghai",
  "L": "shanghai",
  "O": "k8s",
  "OU": "System"
}
]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

#hosts项需要加入所有etcd集群节点，建议将所有node也加入，便于扩容etcd集群。
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.16.1.11",
    "172.16.1.12",
    "172.16.1.13",
    "172.16.1.14",
    "172.16.1.15"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "shanghai",
      "L": "shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes-Soulmate etcd-csr.json | cfssljson -bare etcd
```

字段说明

- 如果 hosts 字段不为空则需要指定授权使用该证书的 **IP 或域名列表**

- `ca-config.json`：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
- `signing`：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 `CA=TRUE`；
- `server auth`：表示client可以用该 CA 对server提供的证书进行验证；
- `client auth`：表示server可以用该CA对client提供的证书进行验证；
- "CN"：`Common Name`，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
- "O"：`Organization`，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；



#### 5.4 分发证书到所有节点

> 本集群所有所有节点安装etcd，因此需要证书分发所有节点。

```shell
$ mkdir -p /etc/etcd/ssl
$ cp etcd.pem etcd-key.pem ca.pem /etc/etcd/ssl/
$ scp -r /etc/etcd/ master02:/etc/
$ scp -r /etc/etcd/ node01:/etc/
$ scp -r /etc/etcd/ node02:/etc/
$ scp -r /etc/etcd/ node03:/etc/
```





### 06. 安装配置etcd

#### 6.1 安装etcd

```shell
//所有节点install
$ yum install etcd -y   
$ mkdir -p /var/lib/etcd
```

#### 6.2 配置etcd

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





### 07 安装配置Kubeadm

> 集群所有节点安装`kebelet` `kubeadm` `kebectl`

```shell
$ yum install -y kubelet kubeadm kubectl
$ systemctl enable kubelet   //暂不启动，未初始化前启动也会报错
```

#### 7.1 配置kubelet

> 所有节点修改，kubelet类似Agent，每台Node上必须要安装的组件

```shell
// 所有机器执行
$ sed -i s#systemd#cgroupfs#g /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
$ echo 'Environment="KUBELET_EXTRA_ARGS=--v=2 --fail-swap-on=false --pod-infra-container-image=harbor.shinezone.com/shinezonetest/pause-amd64:3.1"' >>  /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

#### 7.2 加载配置文件

```shell
$ systemctl daemon-reload
$ systemctl enable kubelet
```



### 08 Master节点高可用

> Master节点高可用使用keepalived，也可以使用商业ELB ALB SLB，或自建N'gin'x负载均衡。

#### 8.1 安装keepalived

```shell
$ yum install -y keepalived
$ systemctl enable keepalived
```

#### 8.2 配置Keepalived

> 注意修改`interface`网卡名，`priority`权重值，`unicast_peer`

 Master01 配置文件

```shell
$ cat <<EOF >/etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://172.16.1.10:6443"    #VIP Address
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens32       #Your Network Interface Name
    virtual_router_id 61
    priority 120          #权重，数字大的为主，数字一样则选择第一台为Master
    advert_int 1
    mcast_src_ip 172.16.1.11  #local IP
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        #172.16.1.11
        172.16.1.12    #另外一台masterIP
    }
    virtual_ipaddress {
        172.16.1.10/24    # VIP
    }
    track_script {
        CheckK8sMaster
    }

}
EOF
```

Master02配置文件

```shell
cat <<EOF >/etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://172.16.1.10:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens32
    virtual_router_id 61
    priority 110
    advert_int 1
    mcast_src_ip 172.16.1.12
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        172.16.1.11
        #172.16.1.12
    }
    virtual_ipaddress {
        172.16.1.10/24
    }
    track_script {
        CheckK8sMaster
    }

}
EOF
```

#### 8.3 启动Keepalived

```shell
$ sed s#'KEEPALIVED_OPTIONS="-D"'#'KEEPALIVED_OPTIONS="-D -d -S 0"'#g /etc/sysconfig/keepalived -i   //配置日志文件
$ echo "local0.*    /var/log/keepalived.log" >> /etc/rsyslog.conf
$ service rsyslog restart
$ systemctl start keepalived
$ systemctl status keepalived
```

#### 8.4 测试Keepalived可用性

> 测试：关闭一台Master机器，看IP是否漂移，API是否可用。

```shell
//确认VIP在Master01上
$ ip a | grep inet |grep "172.16"
    inet 172.16.1.11/21 brd 172.16.7.255 scope global ens32
    inet 172.16.1.10/24 scope global ens32   //VIP
    
// 关闭Master01机器，确认VIP是否飘逸
$ ip a |grep inet |grep "172.16"
    inet 172.16.1.12/21 brd 172.16.7.255 scope global ens32
    inet 172.16.1.10/24 scope global ens32  //可以看到瞬间就偏移到了Master02机器上
    
// 确认APi服务可用性,也可在下步初始化集群后测试，直接访问dashboard看效果。
$ curl https://your_dashboard_address/ -I
HTTP/1.1 200 OK
Server: nginx/1.10.0
Date: Wed, 06 Jun 2018 05:58:22 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 990
Connection: keep-alive
Accept-Ranges: bytes
Cache-Control: no-store
Last-Modified: Tue, 13 Feb 2018 11:17:03 GMT
```

### 09 初始化集群

> Master机器添加初始化配置文件

```shell
$ cat <<EOF > config.yaml 
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
etcd:
  endpoints:
  - https://172.16.1.11:2379
  - https://172.16.1.12:2379
  - https://172.16.1.13:2379
  - https://172.16.1.14:2379
  - https://172.16.1.15:2379
  caFile: /etc/etcd/ssl/ca.pem
  certFile: /etc/etcd/ssl/etcd.pem
  keyFile: /etc/etcd/ssl/etcd-key.pem
  dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16
kubernetesVersion: 1.10.0
api:
  advertiseAddress: "172.16.1.10"
token: "b99a00.a144ef80536d4344"
tokenTTL: "0s"
apiServerCertSANs:
- 172.16.1.10
- 172.16.1.11
- 172.16.1.12
apiServerExtraArgs:
  basic-auth-file: /etc/kubernetes/pki/basic_auth_file
featureGates:
  CoreDNS: true
imageRepository: "harbor.shinezone.com/shinezonetest"
EOF
```

字段说明：

`endpoints`： 指定etcd地址

`networking`：pod网段地址

`advertiseAddress`： Master API接口地址，这里是Keepalived VIP``

`apiServerCertSANs`：指定哪些机器可以管理集群，这里默认只让Master管理集群

`apiServerExtraArgs`：后续Dashboard使用账户密码认证，basic_auth_file需要提前创建

`imageRepository`： 镜像库的地址，指定集群组件images从哪里进行拉取，我这里是Harbor私有镜像库

```
kubeadmin init –help可以看出，service默认网段是10.96.0.0/12
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf默认dns地址cluster-dns=10.96.0.10
```

#### 9.1 初始化集群

```shell
$ echo "admin,admin,2" >> /etc/kubernetes/pki/basic_auth_file   //密码文件
$ kubeadm init --config config.yaml 
```

初始化成功后输出信息：

```shell
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.16.1.10:6443 --token b99a00.a144ef80536d4344 --discovery-token-ca-cert-hash sha256:8c
```

初始化完成后执行相关命令

```shell
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



#### 9.2 初始化失败解决办法

```shell
$ kubeadm reset
// 或者删除相关文件和images
$ rm -rf /etc/kubernetes/*.conf
$ rm -rf /etc/kubernetes/manifests/*.yaml
$ docker ps -a |awk '{print $1}' |xargs docker rm -f
$ systemctl  stop kubelet
```

再次初始化前需要执行清除etcd所有数据的操作。

```shell
 $ etcdctl --endpoints=https://172.16.1.11:2379,https://172.16.1.12:2379,https://172.16.1.13:2379,https://172.16.1.14:2379,https://172.16.1.15:2379 --cacert=/etc/etcd/ssl/ca.pem   --cert=/etc/etcd/ssl/etcd.pem   --key=/etc/etcd/ssl/etcd-key.pem    del / --prefix 
```

#### 9.3 分发kebeadm生成的证书文件和密码文件

> 每台Master机器的证书和密码文件都是相同的，有新的Master加入，直接分发初始化即可。

```shell
$ scp -r /etc/kubernetes/pki  master02:/etc/kubernetes/
//然后初始化，执行命令
$ kubeadm init --config config.yaml 
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 10. 部署网络组件

> Flanneld和Calico都是解决容器通信组件，任选一个即可，这里使用DaemonSet部署，只在Master01执行

#### 10.1 Calico组件部署

```shell
$ wget https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
$ wget  https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
#修改calico.yaml里image镜像路径部分,已推内网harbor
$ grep image calico.yaml 

        - image: harbor.shinezone.com/shinezonetest/calico-typha:1.0
          image: harbor.shinezone.com/shinezonetest/calico-node:1.0
          image: harbor.shinezone.com/shinezonetest/calico-cni:1.0
          
$ kubectl apply -f rbac-kdd.yaml
$ kubectl apply -f calico.yaml
```

#### 10.2 Flannel组件部署

```shell
$ mkdir -p /run/flannel/
$ cat >/run/flannel/subnet.env <<EOF
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
EOF
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  #版本信息：quay.io/coreos/flannel:v0.10.0-amd64
$ kubectl create -f  kube-flannel.yml
```

#### 10.3 查看集群状态

> 集群中组件通信都要基于Calico/Flanneld，需要等到网络组件启动后才可以确认

```shell
$ kubectl get nodes 
NAME                      STATUS    ROLES     AGE       VERSION
ops-sznw-k8s01-master01   Ready     master    12h       v1.10.3
ops-sznw-k8s01-master02   Ready     master    12h       v1.10.3
ops-sznw-k8s01-node01     Ready     <none>    12h       v1.10.3
ops-sznw-k8s01-node02     Ready     <none>    12h       v1.10.3
ops-sznw-k8s01-node03     Ready     <none>    12h       v1.10.3

$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                              READY     STATUS    RESTARTS   AGE
kube-system   calico-node-2j7jl                                 2/2       Running   0          7m
kube-system   calico-node-qwkwj                                 2/2       Running   0          7m
kube-system   calico-node-tgwsh                                 2/2       Running   0          7m
kube-system   calico-node-z6z6c                                 2/2       Running   0          7m
kube-system   calico-node-zntn8                                 2/2       Running   0          7m
kube-system   coredns-7997f8864c-wqngj                          1/1       Running   0          9m
kube-system   coredns-7997f8864c-zz6l2                          1/1       Running   0          9m
kube-system   kube-apiserver-ops-sznw-k8s01-master01            1/1       Running   0          8m
kube-system   kube-apiserver-ops-sznw-k8s01-master02            1/1       Running   0          8m
kube-system   kube-controller-manager-ops-sznw-k8s01-master01   1/1       Running   0          8m
kube-system   kube-controller-manager-ops-sznw-k8s01-master02   1/1       Running   0          8m
kube-system   kube-proxy-b2cj2                                  1/1       Running   0          8m
kube-system   kube-proxy-bqf46                                  1/1       Running   0          9m
kube-system   kube-proxy-fk4ch                                  1/1       Running   0          7m
kube-system   kube-proxy-r7bsb                                  1/1       Running   0          7m
kube-system   kube-proxy-x4h5d                                  1/1       Running   0          7m
kube-system   kube-scheduler-ops-sznw-k8s01-master01            1/1       Running   0          8m
kube-system   kube-scheduler-ops-sznw-k8s01-master02            1/1       Running   0          8m
kube-system   kubernetes-dashboard-76666c8d7-bbqf4              1/1       Running   0          4m
kube-system   kubernetes-dashboard-76666c8d7-x55x9              1/1       Running   0          4m
```



### 11. 部署Dashboard

```yaml
$ cat <<EOF > kubernetes-dashboard.yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Configuration to deploy release version of the Dashboard UI compatible with
# Kubernetes 1.8.
#
# Example usage: kubectl create -f <this_file>

# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
# ------------------- Dashboard Service Account ------------------- #

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Role & Role Binding ------------------- #

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Deployment ------------------- #

kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: harbor.shinezone.com/shinezonetest/kubernetes-dashboard-amd64:v1.8.3
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
          #- --authentication-mode=basic
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule

---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
#    kubernetes.io/cluster-service: "true"
#    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30000
  selector:
    k8s-app: kubernetes-dashboard

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
```

```shell
$ kubectl create -f kubernetes-dashboard.yaml
$ kubectl create clusterrolebinding  login-on-dashboard-with-cluster-admin --clusterrole=cluster-admin --user=admin
```

 #### 11.1 登陆访问

`http://172.16.1.10:30000/#!/login `

获取token,通过令牌登陆

```shell
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```



### 12 部署Traefik Ingress

> Traefik 使用DS方式进行部署

```yaml
$ cat <<EOF > traefik-rbac.yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: default
EOF
```

```yaml
$ cat <<EOF > traefik-ui.yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: default
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: k8s-traefik.shinezone.com
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
EOF
```

```yaml
$ cat <<EOF > traefik-ds.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: default
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: default
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      restartPolicy: Always
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: traefik
        name: traefik-ingress-lb
        volumeMounts:
        - mountPath: "/etc/kubernetes/ssl"  #证书所在目录
          name: "ssl"
        - mountPath: "/root/K8S-Online/Traefik"        #traefik.toml文件所在目录
          name: "config"
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
        - name: https
          containerPort: 443
          hostPort: 443
        securityContext:
          privileged: true
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --configfile=/root/K8S-Online/Traefik/traefik.toml
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: default
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: http
    - protocol: TCP
      port: 443
      name: https
    - protocol: TCP
      port: 8080
      name: admin
  type: NodePort
EOF
```

#### 12.1 配置Tracefik HTTPS

```yaml
$ cat <<EOF > traefik.toml
defaultEntryPoints = ["http", "https"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      certFile = "/root/ssl/shinezone.com.crt"
      keyFile = "/root/ssl/shinezone.com.key"
EOF
```

```shell
$ kubectl create configmap traefik-conf --from-file=traefik.toml    //生成配置字典
$ kubectl create secret generic traefik-cert --from-file=/etc/kubernetes/ssl/shinezone.com.key --from-file=/etc/kubernetes/ssl/shinezone.com.crt                    //生成保密字典
```

#### 12.2 部署Traefik

```shell
$ kubectl create -f traefik-rbac.yaml 
$ kubectl create -f traefik-ds.yaml
$ kubectl create -f traefik-ui.yaml
```

#### 12.3 访问Traefik

> 默认端口Node IP+ NodePort 8080 ,也可以使用Traefik代理出来 使用域名访问

http://172.16.1.10:8080/dashboard/





















### 维护相关

#### 01. 强制删除Pod

```shell
$ kubectl delete  pods traefik-ingress-controller-2spgr   --grace-period=0 --force
```

#### 02. 重新获取集群Token

> 集群初始化时如未设置tokenTTL: "0s" 那么默认生成的token的有效期为24小时，当过期之后，该token就不可用了。另如果丢失kubeadm join加入集群命令，也同样可以通过以下方法解决

```shell
#重新生成token
$ kubeadm token create

#查看
$ kubeadm token list

#获取ca证书sha256编码hash值
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

#使用以上命令生成的token和hash值，操作节点加入集群
$ kubeadm join --token qfk0zc.uobckpxnh54v5at3 --discovery-token-ca-cert-hash sha256:68efb4e280d110f3004a4e16ed09c5ded6c2421cb24a6077cf6171d5167b04d2  10.0.0.15:6443 --skip-preflight-checks
```

#### 03. Master节点也允许Pod容器

> 默认集群安装完成后，Master节点是不进行Pod分配的，资源不足/实验环境，让Master 节点也参与分配pod

```shell
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

#### 04. Dashboard配置用户密码认证方式

> kube-api，dashboard配置使用basic认证方式,在执行kubeadm init前操作
>
> 创建/etc/kubernetes/pki/basic_auth_file 用于存放`密码，用户名,userid.`

```shell
$ mkdir -p /etc/kubernetes/pki/
$ echo 'passwd,user,uuid' > /etc/kubernetes/pki/basic_auth_file 
$ echo 'admin,admin,2' >> /etc/kubernetes/pki/basic_auth_file 
```

修改初始化config.yaml文件

```yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
etcd:
  endpoints:
  - https://172.16.1.11:2379
  - https://172.16.1.12:2379
  - https://172.16.1.13:2379
  - https://172.16.1.14:2379
  - https://172.16.1.15:2379
  caFile: /etc/etcd/ssl/ca.pem
  certFile: /etc/etcd/ssl/etcd.pem
  keyFile: /etc/etcd/ssl/etcd-key.pem
  dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16
kubernetesVersion: 1.10.0
api:
  advertiseAddress: "172.16.1.10"
token: "b99a00.a144ef80536d4344"
tokenTTL: "0s"
apiServerCertSANs:
- 172.16.1.10
- 172.16.1.11
- 172.16.1.12
apiServerExtraArgs:
  basic-auth-file: /etc/kubernetes/pki/basic_auth_file
featureGates:
  CoreDNS: true
imageRepository: "harbor.shinezone.com/shinezonetest"
```

> 打开kubernetes-dashboard.yaml中的  “- --authentication-mode=basic ”注释
>
> 授权 k8s1.6后版本都采用RBAC授权模型 给admin授权 默认cluster-admin是拥有全部权限的，将admin和cluster-admin bind这样admin就有cluster-admin的权限。 那我们将admin和cluster-admin bind在一起这样admin也拥用cluster-admin的权限了 

```shell
$ kubectl create clusterrolebinding  login-on-dashboard-with-cluster-admin --clusterrole=cluster-admin --user=admin
$ kubectl get clusterrolebinding/login-on-dashboard-with-cluster-admin -o yaml
```


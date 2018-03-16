## kubeadm-highavailiability - 基于kubeadm的kubernetes高可用集群部署，支持v1.9.x和v1.7.x版本以及v1.6.x版本



![k8s logo](images/v1.6-v1.7/Kubernetes.png)

- [中文文档(for v1.9.x版本)](README_CN.md)
- [English document(for v1.9.x version)](README.md)
- [中文文档(for v1.7.x版本)](v1.6-v1.7/README_CN.md)
- [English document(for v1.7.x version)](v1.6-v1.7/README.md)
- [中文文档(for v1.6.x版本)](v1.6-v1.7/README_v1.6.x_CN.md)
- [English document(for v1.6.x version)](v1.6-v1.7/README_v1.6.x.md)

---

- [GitHub项目地址](https://github.com/cookeem/kubeadm-ha/)
- [OSChina项目地址](https://git.oschina.net/cookeem/kubeadm-ha/)

---

- 该指引适用于v1.9.x版本的kubernetes集群

> v1.9.0以前的版本kubeadm还不支持高可用部署，因此不推荐作为生产环境的部署方式。从v1.9.x版本开始，kubeadm官方正式支持高可用集群的部署，安装kubeadm务必保证版本至少为1.9.0。

### 目录

1. [部署架构](#部署架构)
    1. [概要部署架构](#概要部署架构)
    1. [详细部署架构](#详细部署架构)
    1. [主机节点清单](#主机节点清单)
1. [安装前准备](#安装前准备)
    1. [版本信息](#版本信息)
    1. [所需docker镜像](#所需docker镜像)
    1. [系统设置](#系统设置)
1. [kubernetes安装](#kubernetes安装)
    1. [firewalld和iptables相关端口设置](#firewalld和iptables相关端口设置)
    1. [kubernetes相关服务安装](#kubernetes相关服务安装)
1. [配置文件初始化](#配置文件初始化)
    1. [初始化脚本配置](#初始化脚本配置) 
    1. [独立etcd集群部署](#独立etcd集群部署)
1. [第一台master初始化](#第一台master初始化)
    1. [kubeadm初始化](#kubeadm初始化)
    1. [安装基础组件](#安装基础组件)
1. [master集群高可用设置](#master集群高可用设置)
    1. [复制配置](#复制配置)
    1. [其余master节点初始化](#其余master节点初始化)
    1. [keepalived安装配置](#keepalived安装配置)
    1. [nginx负载均衡配置](#nginx负载均衡配置)
    1. [kube-proxy配置](#kube-proxy配置)
1. [node节点加入高可用集群设置](#node节点加入高可用集群设置)
    1. [kubeadm加入高可用集群](#kubeadm加入高可用集群)
    1. [验证集群高可用设置](#验证集群高可用设置)


### 部署架构

#### 概要部署架构

![ha logo](images/v1.6-v1.7/ha.png)

* kubernetes高可用的核心架构是master的高可用，kubectl、客户端以及nodes访问load balancer实现高可用。

---
[返回目录](#目录)

#### 详细部署架构

![k8s ha](images/v1.6-v1.7/k8s-ha.png)

* kubernetes组件说明

> kube-apiserver：集群核心，集群API接口、集群各个组件通信的中枢；集群安全控制；

> etcd：集群的数据中心，用于存放集群的配置以及状态信息，非常重要，如果数据丢失那么集群将无法恢复；因此高可用集群部署首先就是etcd是高可用集群；

> kube-scheduler：集群Pod的调度中心；默认kubeadm安装情况下--leader-elect参数已经设置为true，保证master集群中只有一个kube-scheduler处于活跃状态；

> kube-controller-manager：集群状态管理器，当集群状态与期望不同时，kcm会努力让集群恢复期望状态，比如：当一个pod死掉，kcm会努力新建一个pod来恢复对应replicas set期望的状态；默认kubeadm安装情况下--leader-elect参数已经设置为true，保证master集群中只有一个kube-controller-manager处于活跃状态；

> kubelet: kubernetes node agent，负责与node上的docker engine打交道；

> kube-proxy: 每个node上一个，负责service vip到endpoint pod的流量转发，当前主要通过设置iptables规则实现。

* 负载均衡

> keepalived集群设置一个虚拟ip地址，虚拟ip地址指向devops-master01、devops-master02、devops-master03。

> nginx用于devops-master01、devops-master02、devops-master03的apiserver的负载均衡。外部kubectl以及nodes访问apiserver的时候就可以用过keepalived的虚拟ip(192.168.20.10)以及nginx端口(16443)访问master集群的apiserver。

---

[返回目录](#目录)

#### 主机节点清单

主机名 | IP地址 | 说明 | 组件 
:--- | :--- | :--- | :---
devops-master01 ~ 03 | 192.168.66.100 ~ 102 | master节点 * 3 | keepalived、nginx、etcd、kubelet、kube-apiserver、kube-scheduler、kube-proxy、kube-dashboard、heapster、calico
无 | 192.168.66.10 | keepalived虚拟IP | 无
devops-node01 ~ 02 | 192.168.66.103 ~ 104 | node节点 * 2 | kubelet、kube-proxy

---

[返回目录](#目录)

### 安装前准备

#### 版本信息

* Linux版本：CentOS 7.4.1708
* 内核版本: 4.6.4-1.el7.elrepo.x86_64


```
$ cat /etc/redhat-release 
CentOS Linux release 7.4.1708 (Core) 

$ uname -r
4.6.4-1.el7.elrepo.x86_64
```

* docker版本：17.12.0-ce-rc2

```
$ docker version
Client:
 Version:   17.12.0-ce-rc2
 API version:   1.35
 Go version:    go1.9.2
 Git commit:    f9cde63
 Built: Tue Dec 12 06:42:20 2017
 OS/Arch:   linux/amd64

Server:
 Engine:
  Version:  17.12.0-ce-rc2
  API version:  1.35 (minimum version 1.12)
  Go version:   go1.9.2
  Git commit:   f9cde63
  Built:    Tue Dec 12 06:44:50 2017
  OS/Arch:  linux/amd64
  Experimental: false
```

* kubeadm版本：v1.9.3

```
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.3", GitCommit:"d2835416544f298c919e2ead3be3d0864b52323b", GitTreeState:"clean", BuildDate:"2018-02-07T11:55:20Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
```

* kubelet版本：v1.9.3

```
$ kubelet --version
Kubernetes v1.9.3
```

* 网络组件

> canal (flannel + calico)

---

[返回目录](#目录)

#### 所需docker镜像

* 相关docker镜像以及版本

```
# kuberentes basic components
docker pull gcr.io/google_containers/kube-apiserver-amd64:v1.9.3
docker pull gcr.io/google_containers/kube-proxy-amd64:v1.9.3
docker pull gcr.io/google_containers/kube-scheduler-amd64:v1.9.3
docker pull gcr.io/google_containers/kube-controller-manager-amd64:v1.9.3
docker pull gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7
docker pull gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
docker pull gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
docker pull gcr.io/google_containers/etcd-amd64:3.1.10
docker pull gcr.io/google_containers/pause-amd64:3.0

# kubernetes networks add ons
docker pull quay.io/coreos/flannel:v0.9.1-amd64
docker pull quay.io/calico/node:v3.0.3
docker pull quay.io/calico/kube-controllers:v2.0.1
docker pull quay.io/calico/cni:v2.0.1

# kubernetes dashboard
docker pull gcr.io/google_containers/kubernetes-dashboard-amd64:v1.8.3

# kubernetes heapster
docker pull gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3
docker pull gcr.io/google_containers/heapster-grafana-amd64:v4.4.3
docker pull gcr.io/google_containers/heapster-amd64:v1.4.2

# kubernetes apiserver load balancer
docker pull nginx:latest
```

---

[返回目录](#目录)

#### 系统设置

主机名称设置

```shell
hostnamectl --static set-hostname master01
hostnamectl --static set-hostname master02
hostnamectl --static set-hostname master03
hostnamectl --static set-hostname node01
hostnamectl --static set-hostname node02

cat >> /etc/hosts <<EOF
192.168.66.100 master01 etcd-host1
192.168.66.101 master02 etcd-host2
192.168.66.102 master03 etcd-host3
192.168.66.104 node01
192.168.66.105 node02
192.168.66.106 node03
EOF
```

* 在所有kubernetes节点上增加kubernetes仓库 

```
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

* 在所有kubernetes节点上进行系统更新

```shell
$ yum update -y
```

* 在所有kubernetes节点上设置SELINUX为permissive模式

```shell
# 禁用SELinux, 目的是让容器可以读取主机的文件系统.
setenforce 0
sed -i "s/^SELINUX=enforcing/SELINUX=permissive/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=permissive/g" /etc/selinux/config 
```

* 在所有kubernetes节点上设置iptables参数，否则kubeadm init会提示错误

```shell
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

* 在所有kubernetes节点上禁用swap

```shell
#关闭swap(貌似只有1.8以上需要操作,待确认)
swapoff -a
#再把/etc/fstab文件中带有swap的行删了,没有就无视
sed -i '/swap/'d /etc/fstab

# 确认swap已经被禁用
cat /proc/swaps
Filename                Type        Size    Used    Priority
```

* 在所有kubernetes节点上重启主机

```
reboot
```

---

[返回目录](#目录)

### kubernetes安装

#### firewalld和iptables相关端口设置

#####相关端口（master）

协议 | 方向 | 端口 | 说明
:--- | :--- | :--- | :---
TCP | Inbound | 16443*    | Load balancer Kubernetes API server port
TCP | Inbound | 6443     | Kubernetes API server
TCP | Inbound | 4001      | etcd listen client port
TCP | Inbound | 2379-2380 | etcd server client API
TCP | Inbound | 10250     | Kubelet API
TCP | Inbound | 10251     | kube-scheduler
TCP | Inbound | 10252     | kube-controller-manager
TCP | Inbound | 10255     | Read-only Kubelet API
TCP | Inbound | 30000-32767 | NodePort Services

#####在所有master节点上开放相关firewalld端口

在所有master节点上开放相关firewalld端口（因为以上服务基于docker部署，如果docker版本为17.x，可以不进行以下设置，因为docker会自动修改iptables添加相关端口）

```shell
systemctl status firewalld

firewall-cmd --zone=public --add-port=16443/tcp --permanent
firewall-cmd --zone=public --add-port=6443/tcp --permanent
firewall-cmd --zone=public --add-port=4001/tcp --permanent
firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=10251/tcp --permanent
firewall-cmd --zone=public --add-port=10252/tcp --permanent
firewall-cmd --zone=public --add-port=10255/tcp --permanent
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent

firewall-cmd --reload

firewall-cmd --list-all --zone=public

```

```properties
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens2f1 ens1f0 nm-bond
  sources: 
  services: ssh dhcpv6-client
  ports: 4001/tcp 6443/tcp 2379-2380/tcp 10250/tcp 10251/tcp 10252/tcp 10255/tcp 30000-32767/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```



#####相关端口（worker）

协议 | 方向 | 端口 | 说明
:--- | :--- | :--- | :---
TCP | Inbound | 10250       | Kubelet API
TCP | Inbound | 10255       | Read-only Kubelet API
TCP | Inbound | 30000-32767 | NodePort Services

#####所有worker节点上开放相关firewalld端口

在所有worker节点上开放相关firewalld端口（因为以上服务基于docker部署，如果docker版本为17.x，可以不进行以下设置，因为docker会自动修改iptables添加相关端口）

```
systemctl status firewalld

firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=10255/tcp --permanent
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent

firewall-cmd --reload

firewall-cmd --list-all --zone=public
```

```properties
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens2f1 ens1f0 nm-bond
  sources: 
  services: ssh dhcpv6-client
  ports: 10250/tcp 10255/tcp 30000-32767/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```



#####在所有kubernetes节点上允许kube-proxy的forward

```shell
firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 1 -i docker0 -j ACCEPT -m comment --comment "kube-proxy redirects"
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 1 -o docker0 -j ACCEPT -m comment --comment "docker subnet"
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 1 -i flannel.1 -j ACCEPT -m comment --comment "flannel subnet"
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 1 -o flannel.1 -j ACCEPT -m comment --comment "flannel subnet"
firewall-cmd --reload


firewall-cmd --direct --get-all-rules

ipv4 filter INPUT 1 -i docker0 -j ACCEPT -m comment --comment 'kube-proxy redirects'
ipv4 filter FORWARD 1 -o docker0 -j ACCEPT -m comment --comment 'docker subnet'
ipv4 filter FORWARD 1 -i flannel.1 -j ACCEPT -m comment --comment 'flannel subnet'
ipv4 filter FORWARD 1 -o flannel.1 -j ACCEPT -m comment --comment 'flannel subnet'
```

- 在所有kubernetes节点上，删除iptables的设置，解决kube-proxy无法启用nodePort。（注意：每次重启firewalld必须执行以下命令）

```shell
iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
```

---

[返回目录](#目录)

#### kubernetes相关服务安装

* 在所有kubernetes节点上验证SELINUX模式，必须保证SELINUX为permissive模式，否则kubernetes启动会出现各种异常

```
$ getenforce
Permissive
```

#####安装docker,docker-compose,k8s,

在所有kubernetes节点上安装并启动kubernetes 

```shell
 yum -y install docker-ce-17.12.1.ce-1.el7.centos
curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version

systemctl enable docker && systemctl start docker && systemctl status docker

cd kubeadm-rpm && rpm -ivh *.rpm
systemctl enable kubelet && systemctl start kubelet

```

#####配置kubectl

```properties
# 配置kubectl
#kubelet 修改 配置以使用本地自定义pause镜像
export KUBE_REPO_PREFIX=registry.cn-beijing.aliyuncs.com/k8s_len
export KUBE_PAUSE_IMAGE=${KUBE_REPO_PREFIX}/pause-amd64:3.0
cat > /etc/systemd/system/kubelet.service.d/20-pod-infra-image.conf <<EOF
[Service]
Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=${KUBE_PAUSE_IMAGE}"
EOF
systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet && systemctl status kubelet
```

#####在所有kubernetes节点上设置kubelet使用cgroupfs

在所有kubernetes节点上设置kubelet使用cgroupfs，与dockerd保持一致，否则kubelet会启动报错

```shell
# 默认kubelet使用的cgroup-driver=systemd，改为cgroup-driver=cgroupfs
# vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
#Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
#Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
sed  -i 's/--cgroup-driver=systemd/--cgroup-driver=cgroupfs/' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# 重设kubelet服务，并重启kubelet服务
$ systemctl daemon-reload && systemctl restart kubelet && systemctl status kubelet
```

#####在所有master节点上安装并启动keepalived

```shell
yum install -y keepalived
systemctl enable keepalived && systemctl restart keepalived && systemctl status keepalived
```

---

[返回目录](#目录)

### 配置文件初始化

#### 初始化脚本配置

* 在所有master节点上获取代码，并进入代码目录

```shell
#git clone https://github.com/cookeem/kubeadm-ha
git clone https://github.com/len-2017/kubeadm-ha.git
cd kubeadm-ha
git checkout len_v1.9.3 
```

* 在所有master节点上设置初始化脚本配置，每一项配置参见脚本中的配置说明，请务必正确配置。该脚本用于生成相关重要的配置文件

```shell
$ vi create-config.sh

#!/bin/bash

# local machine ip address
export K8SHA_IPLOCAL=192.168.66.100

# local machine etcd name, options: etcd1, etcd2, etcd3
export K8SHA_ETCDNAME=etcd1

# local machine keepalived state config, options: MASTER, BACKUP. One keepalived cluster only one MASTER, other's are BACKUP
export K8SHA_KA_STATE=MASTER

# local machine keepalived priority config, options: 102, 101, 100. MASTER must 102
export K8SHA_KA_PRIO=102

# local machine keepalived network interface name config, for example: eth0
export K8SHA_KA_INTF=ens34

#######################################
# all masters settings below must be same
#######################################

# master keepalived virtual ip address
export K8SHA_IPVIRTUAL=192.168.66.10

# master01 ip address
export K8SHA_IP1=192.168.66.100

# master02 ip address
export K8SHA_IP2=192.168.66.101

# master03 ip address
export K8SHA_IP3=192.168.66.102

# master01 hostname
export K8SHA_HOSTNAME1=master01

# master02 hostname
export K8SHA_HOSTNAME2=master02

# master03 hostname
export K8SHA_HOSTNAME3=master03

# keepalived auth_pass config, all masters must be same
export K8SHA_KA_AUTH=4cdf7dc3b4c90194d1600c483e10ad1d

# kubernetes cluster token, you can use 'kubeadm token generate' to get a new one
export K8SHA_TOKEN=7f276c.0741d82a5337f526

# kubernetes CIDR pod subnet, if CIDR pod subnet is "10.244.0.0/16" please set to "10.244.0.0\\/16"
export K8SHA_CIDR=10.244.0.0\\/16

# kubernetes CIDR service subnet, if CIDR service subnet is "10.96.0.0/12" please set to "10.96.0.0\\/12"
export K8SHA_SVC_CIDR=10.96.0.0\\/12

# calico network settings, set a reachable ip address for the cluster network interface, for example you can use the gateway ip address
export K8SHA_CALICO_REACHABLE_IP=192.168.66.1

##############################
# please do not modify anything below
##############################

# set etcd cluster docker-compose.yaml file
sed \
-e "s/K8SHA_ETCDNAME/$K8SHA_ETCDNAME/g" \
-e "s/K8SHA_IPLOCAL/$K8SHA_IPLOCAL/g" \
-e "s/K8SHA_IP1/$K8SHA_IP1/g" \
-e "s/K8SHA_IP2/$K8SHA_IP2/g" \
-e "s/K8SHA_IP3/$K8SHA_IP3/g" \
etcd/docker-compose.yaml.tpl > etcd/docker-compose.yaml

echo 'set etcd cluster docker-compose.yaml file success: etcd/docker-compose.yaml'

# set keepalived config file
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak

cp keepalived/check_apiserver.sh /etc/keepalived/

sed \
-e "s/K8SHA_KA_STATE/$K8SHA_KA_STATE/g" \
-e "s/K8SHA_KA_INTF/$K8SHA_KA_INTF/g" \
-e "s/K8SHA_IPLOCAL/$K8SHA_IPLOCAL/g" \
-e "s/K8SHA_KA_PRIO/$K8SHA_KA_PRIO/g" \
-e "s/K8SHA_IPVIRTUAL/$K8SHA_IPVIRTUAL/g" \
-e "s/K8SHA_KA_AUTH/$K8SHA_KA_AUTH/g" \
keepalived/keepalived.conf.tpl > /etc/keepalived/keepalived.conf

echo 'set keepalived config file success: /etc/keepalived/keepalived.conf'

# set nginx load balancer config file
sed \
-e "s/K8SHA_IP1/$K8SHA_IP1/g" \
-e "s/K8SHA_IP2/$K8SHA_IP2/g" \
-e "s/K8SHA_IP3/$K8SHA_IP3/g" \
nginx-lb/nginx-lb.conf.tpl > nginx-lb/nginx-lb.conf

echo 'set nginx load balancer config file success: nginx-lb/nginx-lb.conf'

# set kubeadm init config file
sed \
-e "s/K8SHA_HOSTNAME1/$K8SHA_HOSTNAME1/g" \
-e "s/K8SHA_HOSTNAME2/$K8SHA_HOSTNAME2/g" \
-e "s/K8SHA_HOSTNAME3/$K8SHA_HOSTNAME3/g" \
-e "s/K8SHA_IP1/$K8SHA_IP1/g" \
-e "s/K8SHA_IP2/$K8SHA_IP2/g" \
-e "s/K8SHA_IP3/$K8SHA_IP3/g" \
-e "s/K8SHA_IPVIRTUAL/$K8SHA_IPVIRTUAL/g" \
-e "s/K8SHA_TOKEN/$K8SHA_TOKEN/g" \
-e "s/K8SHA_CIDR/$K8SHA_CIDR/g" \
-e "s/K8SHA_SVC_CIDR/$K8SHA_SVC_CIDR/g" \
kubeadm-init.yaml.tpl > kubeadm-init.yaml

echo 'set kubeadm init config file success: kubeadm-init.yaml'

# set canal deployment config file

sed \
-e "s/K8SHA_CIDR/$K8SHA_CIDR/g" \
-e "s/K8SHA_CALICO_REACHABLE_IP/$K8SHA_CALICO_REACHABLE_IP/g" \
kube-canal/canal.yaml.tpl > kube-canal/canal.yaml

echo 'set canal deployment config file success: kube-canal/canal.yaml'

```

* 在所有master节点上运行配置脚本，创建对应的配置文件，配置文件包括:

- etcd集群docker-compose.yaml文件
- keepalived配置文件
- nginx负载均衡集群docker-compose.yaml文件
- kubeadm init 配置文件
- canal配置文件

```shell
./create-config.sh
set etcd cluster docker-compose.yaml file success: etcd/docker-compose.yaml
set keepalived config file success: /etc/keepalived/keepalived.conf
set nginx load balancer config file success: nginx-lb/nginx-lb.conf
set kubeadm init config file success: kubeadm-init.yaml
set canal deployment config file success: kube-canal/canal.yaml
```

---

[返回目录](#目录)

#### 独立etcd集群部署

* 在所有master节点上重置并启动etcd集群（非TLS模式）

```shell
# 重置kubernetes集群
$ kubeadm reset

# 清空etcd集群数据
$ rm -rf /var/lib/etcd-cluster

# 重置并启动etcd集群
docker-compose --file etcd/docker-compose.yaml stop
docker-compose --file etcd/docker-compose.yaml rm -f
docker-compose --file etcd/docker-compose.yaml up -d

# 验证etcd集群状态是否正常

docker exec -ti etcd etcdctl cluster-health
[root@master01 kubeadm-ha]# docker exec -ti etcd etcdctl cluster-health
member 472a2e811e55b95c is healthy: got healthy result from http://192.168.66.102:2379
member a3cef8b35b895bf6 is healthy: got healthy result from http://192.168.66.100:2379
member f29752f4343f3b79 is healthy: got healthy result from http://192.168.66.101:2379
cluster is healthy

docker exec -ti etcd etcdctl member list
472a2e811e55b95c: name=etcd3 peerURLs=http://192.168.66.102:2380 clientURLs=http://192.168.66.102:2379,http://192.168.66.102:4001 isLeader=true
a3cef8b35b895bf6: name=etcd1 peerURLs=http://192.168.66.100:2380 clientURLs=http://192.168.66.100:2379,http://192.168.66.100:4001 isLeader=false
f29752f4343f3b79: name=etcd2 peerURLs=http://192.168.66.101:2380 clientURLs=http://192.168.66.101:2379,http://192.168.66.101:4001 isLeader=false
```

---

[返回目录](#目录)

### 第一台master初始化

#### kubeadm初始化

* 在所有master节点上重置网络

```shell
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/

删除遗留的网络接口
ip a | grep -E 'docker|flannel|cni'
ip link del docker0
ip link del flannel.1
ip link del cni0

systemctl restart docker && systemctl status docker
systemctl restart kubelet && systemctl status kubelet
ip a | grep -E 'docker|flannel|cni'
```

* 在master01上进行初始化，注意，务必把输出的kubeadm join --token XXX --discovery-token-ca-cert-hash YYY 信息记录下来，后续操作需要用到

```properties
kubeadm init --config=kubeadm-init.yaml
...
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 7f276c.0741d82a5337f526 192.168.66.100:6443 --discovery-token-ca-cert-hash sha256:731e73e7a91afaac61b7f183cba840680b204ede71f30cce56c63bb12a112086
```

* 在所有master节点上设置kubectl客户端连接

```shell
cat << EOF >> ~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF
source ~/.bashrc

或者执行:
  #mkdir -p $HOME/.kube
  #sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  #sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 安装基础组件

#####在master01上安装flannel网络组件

```shell
# 没有网络组件的情况下，节点状态是不正常的
kubectl get node
NAME              STATUS    ROLES     AGE       VERSION
devops-master01   NotReady  master    14s       v1.9.1

# 安装canal网络组件
kubectl apply -f kube-canal/
configmap "canal-config" created
daemonset "canal" created
customresourcedefinition "felixconfigurations.crd.projectcalico.org" created
customresourcedefinition "bgpconfigurations.crd.projectcalico.org" created
customresourcedefinition "ippools.crd.projectcalico.org" created
customresourcedefinition "clusterinformations.crd.projectcalico.org" created
customresourcedefinition "globalnetworkpolicies.crd.projectcalico.org" created
customresourcedefinition "networkpolicies.crd.projectcalico.org" created
serviceaccount "canal" created
clusterrole "calico" created
clusterrole "flannel" created
clusterrolebinding "canal-flannel" created
clusterrolebinding "canal-calico" created

# 等待所有pods正常
kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                      READY     STATUS    RESTARTS   AGE       IP              NODE
kube-system   canal-hpn82                               3/3       Running   0          1m        192.168.20.27   devops-master01
kube-system   kube-apiserver-devops-master01            1/1       Running   0          1m        192.168.20.27   devops-master01
kube-system   kube-controller-manager-devops-master01   1/1       Running   0          50s       192.168.20.27   devops-master01
kube-system   kube-dns-6f4fd4bdf-vwbk8                  3/3       Running   0          1m        10.244.0.2      devops-master01
kube-system   kube-proxy-mr6l8                          1/1       Running   0          1m        192.168.20.27   devops-master01
kube-system   kube-scheduler-devops-master01            1/1       Running   0          57s       192.168.20.27   devops-master01
```

#####在devops-master01上安装dashboard

```shell
# 设置master节点为schedulable
kubectl taint nodes --all node-role.kubernetes.io/master-

kubectl apply -f kube-dashboard/
serviceaccount "admin-user" created
clusterrolebinding "admin-user" created
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

* 通过浏览器访问dashboard地址

> https://192.168.66.100:30000/#!/login

* dashboard登录页面效果如下图

![dashboard-login](images/dashboard-login.png)

* 获取token，把token粘贴到login页面的token中，即可进入dashboard

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

![dashboard](images/dashboard.png)

#####在devops-master01上安装heapster

```shell
#
kubectl apply -f kube-heapster/influxdb/
service "monitoring-grafana" created
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created

#
kubectl apply -f kube-heapster/rbac/
clusterrolebinding "heapster" created

#
kubectl get pods --all-namespaces 
NAMESPACE     NAME                                      READY     STATUS    RESTARTS   AGE
kube-system   canal-hpn82                               3/3       Running   0          6m
kube-system   heapster-65c5499476-gg2tk                 1/1       Running   0          2m
kube-system   kube-apiserver-devops-master01            1/1       Running   0          6m
kube-system   kube-controller-manager-devops-master01   1/1       Running   0          5m
kube-system   kube-dns-6f4fd4bdf-vwbk8                  3/3       Running   0          6m
kube-system   kube-proxy-mr6l8                          1/1       Running   0          6m
kube-system   kube-scheduler-devops-master01            1/1       Running   0          6m
kube-system   kubernetes-dashboard-7c7bfdd855-2slp2     1/1       Running   0          4m
kube-system   monitoring-grafana-6774f65b56-mwdjv       1/1       Running   0          2m
kube-system   monitoring-influxdb-59d57d4d58-xmrxk      1/1       Running   0          2m


# 等待5分钟
$ kubectl top nodes
NAME              CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%   
devops-master01   242m         0%        1690Mi          0%        
```

* 访问dashboard地址，等10分钟，就会显示性能数据

> https://192.168.66.100:30000/#!/login

![heapster-dashboard](images/heapster-dashboard.png)

![heapster](images/heapster.png)

* 至此，第一台master成功安装，并已经完成canal、dashboard、heapster的部署

---

[返回目录](#目录)

### master集群高可用设置

#### 复制配置

* 在master01上复制目录/etc/kubernetes/pki到devops-master02、devops-master03，从v1.9.x开始，kubeadm会检测pki目录是否有证书，如果已经存在证书则跳过证书生成的步骤

```shell
#
scp -r /etc/kubernetes/pki master02:/etc/kubernetes/
#
scp -r /etc/kubernetes/pki master03:/etc/kubernetes/

```

---
[返回目录](#目录)

#### 其余master节点初始化

#####master02 初始化

在master02进行初始化，等待所有pods正常启动后再进行下一个master初始化，**特别要保证kube-apiserver-{current-node-name}处于running状态**

```shell
# 输出的token和discovery-token-ca-cert-hash应该与devops-master01上的完全一致
$ kubeadm init --config=kubeadm-init.yaml
...
  kubeadm join --token 7f276c.0741d82a5337f526 192.168.66.101:6443 --discovery-token-ca-cert-hash sha256:731e73e7a91afaac61b7f183cba840680b204ede71f30cce56c63bb12a112086
```

#####master03初始化

在devops-master03进行初始化，等待所有pods正常启动后再进行下一个master初始化，特别要保证kube-apiserver-{current-node-name}处于running状态

```shell
# 输出的token和discovery-token-ca-cert-hash应该与devops-master01上的完全一致
$ kubeadm init --config=kubeadm-init.yaml
...
  kubeadm join --token 7f276c.0741d82a5337f526 192.168.66.102:6443 --discovery-token-ca-cert-hash sha256:731e73e7a91afaac61b7f183cba840680b204ede71f30cce56c63bb12a112086
```

#####master01上检查

在dmaster01上检查nodes加入情况

```shell
$ kubectl get nodes
NAME              STATUS    ROLES     AGE       VERSION
devops-master01   Ready     master    19m       v1.9.3
devops-master02   Ready     master    4m        v1.9.3
devops-master03   Ready     master    4m        v1.9.3
```

#####在devops-master01上检查高可用状态

```shell
$ kubectl get pods --all-namespaces -o wide 
NAMESPACE     NAME                                      READY     STATUS    RESTARTS   AGE       IP              NODE
kube-system   canal-cw8tw                               3/3       Running   4          3m        192.168.20.29   devops-master03
kube-system   canal-d54hs                               3/3       Running   3          5m        192.168.20.28   devops-master02
kube-system   canal-hpn82                               3/3       Running   5          17m       192.168.20.27   devops-master01
kube-system   heapster-65c5499476-zwgnh                 1/1       Running   1          8m        10.244.0.7      devops-master01
kube-system   kube-apiserver-devops-master01            1/1       Running   1          2m        192.168.20.27   devops-master01
kube-system   kube-apiserver-devops-master02            1/1       Running   0          11s       192.168.20.28   devops-master02
kube-system   kube-apiserver-devops-master03            1/1       Running   0          12s       192.168.20.29   devops-master03
kube-system   kube-controller-manager-devops-master01   1/1       Running   1          16m       192.168.20.27   devops-master01
kube-system   kube-controller-manager-devops-master02   1/1       Running   1          3m        192.168.20.28   devops-master02
kube-system   kube-controller-manager-devops-master03   1/1       Running   1          2m        192.168.20.29   devops-master03
kube-system   kube-dns-6f4fd4bdf-vwbk8                  3/3       Running   3          17m       10.244.0.2      devops-master01
kube-system   kube-proxy-59pwn                          1/1       Running   1          5m        192.168.20.28   devops-master02
kube-system   kube-proxy-jxt5s                          1/1       Running   1          3m        192.168.20.29   devops-master03
kube-system   kube-proxy-mr6l8                          1/1       Running   1          17m       192.168.20.27   devops-master01
kube-system   kube-scheduler-devops-master01            1/1       Running   1          16m       192.168.20.27   devops-master01
kube-system   kube-scheduler-devops-master02            1/1       Running   1          3m        192.168.20.28   devops-master02
kube-system   kube-scheduler-devops-master03            1/1       Running   1          2m        192.168.20.29   devops-master03
kube-system   kubernetes-dashboard-7c7bfdd855-2slp2     1/1       Running   1          15m       10.244.0.3      devops-master01
kube-system   monitoring-grafana-6774f65b56-mwdjv       1/1       Running   1          13m       10.244.0.4      devops-master01
kube-system   monitoring-influxdb-59d57d4d58-xmrxk      1/1       Running   1          13m       10.244.0.6      devops-master01
```

#####设置所有master的scheduable

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
node "devops-master02" untainted
node "devops-master03" untainted
```

#####对基础组件进行多节点scale

```shell
kubectl get deploy -n kube-system
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
heapster               1         1         1            1           3d
kube-dns               2         2         2            2           4d
kubernetes-dashboard   1         1         1            1           3d
monitoring-grafana     1         1         1            1           3d
monitoring-influxdb    1         1         1            1           3d

# dns支持多节点
kubectl scale --replicas=2 -n kube-system deployment/kube-dns
kubectl get pods --all-namespaces -o wide| grep kube-dns

```

---

[返回目录](#目录)

#### keepalived安装配置

#####在master上重启keepalived

```shell
systemctl restart keepalived && systemctl status keepalived
$ ping 192.168.66.10
```

---

[返回目录](#目录)

#### nginx负载均衡配置

* 在master上安装并启动nginx作为负载均衡

```shell
docker-compose -f nginx-lb/docker-compose.yaml up -d
```

* 在master上验证负载均衡和keepalived是否成功

```shell
curl -k https://192.168.66.10:16443
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}
```

---

[返回目录](#目录)

#### kube-proxy配置

- master01上设置proxy高可用，设置server指向高可用虚拟IP以及负载均衡的16443端口
```shell
$ kubectl edit -n kube-system configmap/kube-proxy
###修改 server: https://192.168.66.10:16443
```

- 在master上重启proxy

```shell
kubectl get pods --all-namespaces -o wide | grep proxy
kubectl delete pod -n kube-system kube-proxy-XXX

[root@master01 kubeadm-ha]# kubectl get pods --all-namespaces -o wide | grep proxy
kube-system   kube-proxy-n5hjz                        1/1       Running            0          4m        192.168.66.101   master02
kube-system   kube-proxy-pf2j9                        1/1       Running            0          4m        192.168.66.100   master01
kube-system   kube-proxy-r2xhb                        1/1       Running            0          14s       192.168.66.102   master03
```

---

[返回目录](#目录)

### node节点加入高可用集群设置

#### kubeadm加入高可用集群

- 在所有worker节点上进行加入kubernetes集群操作，这里统一使用devops-master01的apiserver地址来加入集群 

```shell
kubeadm join --token 7f276c.0741d82a5337f526 192.168.66.100:6443 --discovery-token-ca-cert-hash sha256:731e73e7a91afaac61b7f183cba840680b204ede71f30cce56c63bb12a112086
```

- 在所有worker节点上修改kubernetes集群设置，更改server为高可用虚拟IP以及负载均衡的16443端口

```shell
sed -i "s/192.168.66.100:6443/192.168.66.10:16443/g" /etc/kubernetes/bootstrap-kubelet.conf
sed -i "s/192.168.66.101:6443/192.168.66.10:16443/g" /etc/kubernetes/bootstrap-kubelet.conf
sed -i "s/192.168.66.102:6443/192.168.66.10:16443/g" /etc/kubernetes/bootstrap-kubelet.conf

sed -i "s/192.168.66.100:6443/192.168.66.10:16443/g" /etc/kubernetes/kubelet.conf
sed -i "s/192.168.66.101:6443/192.168.66.10:16443/g" /etc/kubernetes/kubelet.conf
sed -i "s/192.168.66.102:6443/192.168.66.10:16443/g" /etc/kubernetes/kubelet.conf

grep 192.168.66.10 /etc/kubernetes/*.conf 
/etc/kubernetes/bootstrap-kubelet.conf:    server: https://192.168.66.10:16443
/etc/kubernetes/kubelet.conf:    server: https://192.168.66.10:16443

$ systemctl restart docker kubelet
```


```shell
kubectl get nodes
NAME              STATUS    ROLES     AGE       VERSION
devops-master01   Ready     master    46m       v1.9.3
devops-master02   Ready     master    44m       v1.9.3
devops-master03   Ready     master    44m       v1.9.3
devops-node01     Ready     <none>    50s       v1.9.3
devops-node02     Ready     <none>    26s       v1.9.3
devops-node03     Ready     <none>    22s       v1.9.3
devops-node04     Ready     <none>    17s       v1.9.3
```

- 设置workers的节点标签

```
kubectl label nodes devops-node01 role=worker
kubectl label nodes devops-node02 role=worker
kubectl label nodes devops-node03 role=worker
kubectl label nodes devops-node04 role=worker
```

#### 验证集群高可用设置

- NodePort测试

```
# 创建一个replicas=3的nginx deployment
$ kubectl run nginx --image=nginx --replicas=3 --port=80
deployment "nginx" created

# 检查nginx pod的创建情况
$ kubectl get pods -l=run=nginx -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP              NODE
nginx-6c7c8978f5-558kd   1/1       Running   0          9m        10.244.77.217   devops-node03
nginx-6c7c8978f5-ft2z5   1/1       Running   0          9m        10.244.172.67   devops-master01
nginx-6c7c8978f5-jr29b   1/1       Running   0          9m        10.244.85.165   devops-node04

# 创建nginx的NodePort service
$ kubectl expose deployment nginx --type=NodePort --port=80
service "nginx" exposed

# 检查nginx service的创建情况
$ kubectl get svc -l=run=nginx -o wide
NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE       SELECTOR
nginx     NodePort   10.101.144.192   <none>        80:30847/TCP   10m       run=nginx

# 检查nginx NodePort service是否正常提供服务
$ curl devops-master01:30847
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

$ kubectl delete deploy,svc nginx
```

- pod之间互访测试

```
$ kubectl run nginx-server --image=nginx --port=80
$ kubectl expose deployment nginx-server --port=80
$ kubectl get pods -o wide -l=run=nginx-server
NAME                            READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-server-6d64689779-lfcxc   1/1       Running   0          2m        10.244.5.7   devops-node03

$ kubectl run nginx-client -ti --rm --image=alpine -- ash
/ # wget nginx-server
Connecting to nginx-server (10.102.101.78:80)
index.html           100% |*****************************************|   612   0:00:00 ETA
/ # cat index.html 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


$ kubectl delete deploy,svc nginx-server
```

- 至此kubernetes高可用集群完成部署😃


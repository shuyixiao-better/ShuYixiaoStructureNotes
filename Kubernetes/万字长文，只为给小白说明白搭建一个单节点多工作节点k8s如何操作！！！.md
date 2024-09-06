@[TOC](这里写目录标题)

> 写在前面，这里就是部署的是单个master多个worker节点的K8s。要是你本地内存够大的话，可以采用VMware多开多个虚拟机来进行操作练手。本次我的搭建环境是4台4C10G虚拟机进行演示。需要说明的是在开始之前我已经关闭了防火墙。话不多说，直接开车。

## 1. Kubernetes 基础概念

> Kubernetes（希腊语）：舵手、飞行员
> 源自于谷歌Borg
> 使用golang语言开发
> 简称为k8s

### 1.1 什么是 Kubernetes？
Kubernetes 是一个开源的容器编排平台，能够自动化应用程序的部署、扩展和管理。它主要用于管理容器化应用程序在集群环境中的运行。


### 1.2 核心组件
- **Pod**: Kubernetes 中的最小部署单位，一个 Pod 可以包含一个或多个容器，这些容器共享相同的网络命名空间和存储。
- **Node**: 是 Kubernetes 集群中的一台物理机或虚拟机，负责运行 Pod。Node 上运行着 Kubelet 和 kube-proxy。
- **Cluster**: 一组 Node 组成的集群。
- **Namespace**: 用于隔离集群中的资源和对象，便于不同环境或团队之间的资源管理。
- **Deployment**: 提供声明式更新的机制，管理 Pod 和 ReplicaSet。
- **Service**: 抽象出一组 Pod 的访问方式，提供负载均衡和服务发现功能。

## 本次搭建的机器演示
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/be4b07e3b3a7405aa9430f789d5a3a6e.png)
## 具体的安装部署步骤
### 修改主机的机器名称
下面这个操作是直接在上述四台服务器上分别执行。
```cpp
hostnamectl set-hostname master01 && bash
hostnamectl set-hostname worker01 && bash
hostnamectl set-hostname worker02 && bash
hostnamectl set-hostname worker03 && bash
```


### 依次修改主机的host文件

```cpp
vim /etc/hosts
# 执行下面命令
192.168.65.10	master01
192.168.65.20	worker01
192.168.65.21	worker02
192.168.65.22	worker03
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ee9d21a675594786964e5baad20b3d2f.png)

### 配置免密登录

> 配置 SSH 密钥在 Kubernetes 安装过程中有利于提高集群节点间的通信效率，增强安全性，并且支持无密码自动化操作，从而大大简化了集群的管理和维护流程。

执行下面命令一路敲击回车就行
```cpp
ssh-keygen
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b52e709f0ac842549a84a266bbbb45bc.png)
### 当前用户的 SSH 公钥复制到远程主机
执行下面的命令将上面master节点配置的公钥传输到服务器中，下面的命令在master节点的机器上运行的命令
```cpp
ssh-copy-id master01
ssh-copy-id worker01
ssh-copy-id worker02
ssh-copy-id worker03
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/35750f72ea384c3e8b9549ac2f1bf77c.png)
### 关闭交换区分区
这里采用永久关闭交换区分区的形式，可以直接编辑这个文件，或者使用finalshell工具双击打开之后注释掉下面截图中的信息之后重启机器。**凡是涉及到的机器都需要关闭一下交换区的分区。**
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/199ca247085c44c082a42284ce87e88f.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6099d50a6014449287356eeb6a3cee96.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/42a962dec5ff46aeb8675bb43ea95080.png)
重启之后交换区就会为0
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/774bf1e1cf6f413fb8ac75417fab5307.png)
### 加载 br_netfilter 内核模块
需要在四台机器上执行这个命令
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d4fa5e5960884808af1bf9e318d32fee.png)

#### 为什么安装 Kubernetes 需要执行这个命令？
在 Kubernetes 安装过程中，特别是当 Kubernetes 使用 kube-proxy 或其他网络插件（如 flannel, calico 等）来管理节点和 Pod 之间的通信时，需要在每个节点上进行网络包的转发和过滤。为确保跨网络桥接的数据包能够被正确处理，br_netfilter 模块必须被加载。

具体原因包括：

包过滤和转发：在 Kubernetes 中，集群内部的 Pod 之间需要通过网络进行通信。br_netfilter 模块确保这些网络包能够被 Linux 内核中的 iptables 规则所管理和过滤，以实现正确的网络隔离和安全性。
网络插件依赖：许多 Kubernetes 的网络插件（如 flannel, calico）依赖于内核的网络包过滤机制来管理和控制 Pod 的网络流量。加载 br_netfilter 确保这些插件能够正常工作，保证 Pod 的网络通信稳定和安全。
```cpp
modprobe br_netfilter
```
### 启动相关内核参数修改
四台机器都需要执行

```cpp
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF

sysctl -p /etc/sysctl.d/k8s.conf

```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f24258a37a5e4b1588acba4499303347.png)
### 修改 Linux 系统中 SELinux（Security-Enhanced Linux）的配置，将其状态从 enforcing 改为 disabled。
#### 为什么需要操作这一步
与容器和网络插件兼容性问题：Kubernetes 使用容器来运行应用程序，而一些容器引擎（如 Docker）以及 Kubernetes 网络插件可能在 SELinux 开启的情况下遇到权限问题。这是因为 SELinux 的强制访问控制机制会限制一些容器的行为，导致它们无法正常工作。

简化配置：对于一些 Kubernetes 安装和管理工具（如 kubeadm），禁用 SELinux 可以避免因为权限问题导致的复杂配置和兼容性问题，特别是在网络通信和文件系统操作方面。

避免权限冲突：SELinux 施加的严格权限管理可能与 Kubernetes 的访问控制策略冲突，特别是涉及到 Pod 之间的通信和共享资源时。因此，禁用 SELinux 可以避免不必要的权限冲突问题。
```cpp
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```
上述四台机器都要执行，执行完上述命令需要重启机器然后查看一下是否关闭了
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/31823b704b3149a1ae9f147ac006801e.png)

### 配置镜像下载的加速

```cpp
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
要是执行这个命令出现问题，可以安装一下，下面这个工具，说不定就可以配置上了。四台机器都需要配置加速！

```cpp
sudo yum install -y yum-utils
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6014dadad9d04348a56d89c8cce39c8a.png)
### 配置K8s镜像源加速
如果这个步骤不配置，需要在centos中开启科学上网，算了还是配置一下吧。

```cpp
vim /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
```
执行完上述操作之后复制给其他的三台worker节点机器

```cpp

scp /etc/yum.repos.d/kubernetes.repo worker01:/etc/yum.repos.d/
scp /etc/yum.repos.d/kubernetes.repo worker02:/etc/yum.repos.d/
scp /etc/yum.repos.d/kubernetes.repo worker03:/etc/yum.repos.d/
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6440db4d22164e218ccfd5de300330ab.png)
### 配置时间同步
下面这个操作四台机器都需要执行，这里有一个小插曲就是出现这个，上面说的命令执行时候最好是一段段分开执行，不然会出现下面的报错，这展示一下执行完的效果。

```cpp
yum -y install ntpdate # 安装软件
ntpdate cn.pool.ntp.org # 同步网络时间

crontab -e
* * * * * /usr/sbin/ntpdate cn.pool.ntp.org		# 定时任务同步时间，每分钟执行一次

重启定时任务
service crond restart
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/00501e6a30e147848cf57d5410d9b44d.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2f04c2f07cc647d8b520bb7f6fb50f88.png)
### 安装Docker-CE和Containerd服务
下面四台机器都需要执行，最好是分开执行。
```cpp
yum install -y containerd.io


# 生成containerd配置文件
containerd config default > /etc/containerd/config.toml

vim /etc/containerd/config.toml
# 修改/etc/containerd/config.toml，只需改动一下内容
SystemdCgroup = true
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.7"
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/039ec03b3d024f40bc1b209370c058a1.png)
#### 创建/etc/crictl.yaml文件

```cpp
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/010517668671486fbe781b34357ec60c.png)
#### 配置Containerd镜像加速器

```cpp
vim /etc/containerd/config.toml

config_path = "/etc/containerd/certs.d" #指定containerd的证书存放目录

```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4c096637a76c40fe9ccf292f819fc141.png)
#### 创建证书等目录文件

```cpp
# 创建目录
mkdir /etc/containerd/certs.d/docker.io/ -p

vim /etc/containerd/certs.d/docker.io/hosts.toml		# 创建文件

# 内容如下
[host."https://6yqx5sih.mirror.aliyuncs.com",host."https://registry.docker-cn.com"]
capabilities = ["pull","push"]
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b0734056c6864a9a921a484bc2597014.png)
#### 重启Containerd来使配置生效

```cpp
systemctl restart containerd
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dde66f6515e241459f310d050f5ca2cf.png)
#### 安装Docker，配置Docker镜像加速

```cpp
yum install -y docker-ce
systemctl enable docker

vim /etc/docker/daemon.json

{
    "registry-mirrors":[
        "https://6yqx5sih.mirror.aliyuncs.com",
        "https://registry.docker-cn.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://hub-mirror.c.163.com"
    ]
}

systemctl restart docker

```
### 安装部署Kubernetes集群
#### 安装Kubernetes集群需要的包
四台机器都执行
```cpp
yum install -y kubelet-1.27.0 kubeadm-1.27.0 kubectl-1.27.0
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/52af6774f9164e10a4dafc14072359d5.png)
#### 配置开机自启动

```cpp
#开机自启
systemctl enable kubelet
```
#### Kubeadm初始化Kubernetes集群
这里需要说明一下就是执行下面这个命令之后。会在你执行命令的路径下出现kubeadm.ymal这个文件，修改即可
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d63d4a4ed4564b53a4f892858ef7bf3f.png)

```cpp
kubeadm config print init-defaults > kubeadm.ymal
```
修改配置信息

```cpp
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.153.180	# ip地址
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: xiaolumaster1	# 主机
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers	# 阿里云镜像仓库
kind: ClusterConfiguration
kubernetesVersion: 1.27.0
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16	# pod网段
  serviceSubnet: 10.96.0.0/12	# service网段
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd

```
#### kubeadm初始化Kubenetes集群

```cpp
kubeadm init --config=kubeadm.ymal --ignore-preflight-errors=SystemVerification
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/742b9b2f782741ec94dcaf79665df81a.png)

### 配置Kubectl配置文件config

```cpp
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config # 将config拷过来更名

sudo chown $(id -u):$(id -g) $HOME/.kube/config # 设置权限

```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ffede4dcec2c4fe3b5852a38d6d4b86e.png)

#### 查看k8s集群

```cpp
kubectl get nodes
```
#### 扩容Kubernets集群
在master节点上执行
```cpp
kubeadm token create --print-join-command
```
执行完上述的命令之后，会出现下面的join命令，然后将下面的命令在worker节点执行并进行加入
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a2de713ed3db445eb22b1a3da5be354f.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4daf3775ab1c403b834a43741c595c8c.png)
加入完成之后，色现在都是None，通常来讲我们是将另外三个none角色的节点称之为工作节点，在controller中将它们两个设置为worker即可

```cpp
kubectl label node worker01 node-role.kubernetes.io/worker=worker
kubectl label node worker02 node-role.kubernetes.io/worker=worker
kubectl label node worker03 node-role.kubernetes.io/worker=worker
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/943e0c350b8343cfa3fcfafad9311cbc.png)
### 安装网络插件Calico
#### 下载文件信息

```cpp
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```
#### 执行文件加载命令

```cpp
kubectl apply -f calico.yaml
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dc5fb83121b5415db771c95c93051bac.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/336b7bed07cd4841a8d6574a93add9a7.png)

### 安装可视化界面kuboard

执行docker安装命令

```cpp
sudo docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 80:80/tcp \
  -p 10081:10081/tcp \
  -e KUBOARD_ENDPOINT="http://192.168.65.10:80" \
  -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
  -v /root/kuboard-data:/data \
  eipwork/kuboard:v3
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/37205c67128847d69ae1785dd311855d.png)
访问登录默认用户名密码
admin   Kuboard123
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1a12a7cd1e2e4bf4a283293ebb8adf0a.png)
#### 配置集群信息
按照下图所示进行
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c0be3ee982ee48ec9367a5a4c02baef7.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/256938745d3c4534bb8bbeb38a0060d2.png)
配置完成之后可以在自己的集群下建立个性化的节点信息
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/82626176c9894606b7d60e0c3b917989.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d770fe270cff4d58b771405fec7f9496.png)
## 致谢
最后感谢大佬的干货文章分享，然后要是上述有不清楚的地方可以结合下面文章一起查看就能轻易上手搭建出K8s服务。
https://blog.csdn.net/weixin_45248492/article/details/139200457?spm=1001.2014.3001.5506
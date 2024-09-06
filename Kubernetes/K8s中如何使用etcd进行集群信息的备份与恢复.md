@[TOC](这里写目录标题)
## ETCD是什么？
etcd 是 一致且高可用的键值存储，用作 Kubernetes 所有集群数据的后台数据库。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cb77d28a71114efc809f6f976e90b377.png)

> 这里需要说明一下的是，安装好K8s集群之后，在Master节点的机器上是会自带一个etcd服务。

但是在正式的介绍开始之前需要阐述一下**etcd和etcdctl 工具之间的关系是什么？**
### 1. **`etcd`（服务）**
- `etcd` 是一个**分布式键值存储系统**，用于存储数据并确保高可用性、数据一致性和分布式协调。它常用于 Kubernetes 集群中的配置管理和服务发现。
- Kubernetes 使用 `etcd` 来存储集群状态数据（例如 Pod 状态、服务信息、配置等），因此 `etcd` 是 Kubernetes 集群不可或缺的一部分。
- `etcd` 是一个服务，它运行在后台，提供 API 给外部客户端和系统进行数据操作。

### 2. **`etcdctl`（客户端工具）**
- `etcdctl` 是 `etcd` 的**命令行客户端工具**，用于与 `etcd` 服务交互。它允许用户通过命令行与 `etcd` 进行操作，比如存取数据、检查健康状态、创建快照、恢复数据等。
- `etcdctl` 可以帮助系统管理员直接与 etcd 进行交互，执行如备份、恢复、查询键值、检查集群状态等操作。
## 如何安装etcdctl（客户端工具）
### 查看目前K8s自带etcd中的版本信息

```cpp
kubectl exec -n kube-system -it etcd-master01 -- /bin/sh
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1debce033674423ca8fd48ebe2a1faaf.png)
### 安装对应版本的etcdutl工具

根据你当前的 `etcd` 版本（3.5.7），你应该下载与此版本匹配的 `etcdutl` 工具。这里需要说明一下的是etcdctl以及在近期的版本中弃用了，所以这边我们采用etcdutl工具进行备份的操作。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a222abaa805246168f6a0b397f6dbe57.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/81352f9c5f2b4466b1ca0369de4b5231.png)

### 下载 `etcdutl` 3.5.7 版本
从 etcd 的官方 GitHub 发行页面下载与 `etcd` 版本相匹配的 `etcdctl` 工具。下面是下载和安装步骤：

1. 访问 etcd releases 页面：[etcd releases](https://github.com/etcd-io/etcd/releases)

2. 找到 **v3.5.7** 版本，并下载适用于你系统的二进制文件。例如，对于 Linux 操作系统，下载链接为：
   ```bash
   wget https://github.com/etcd-io/etcd/releases/download/v3.5.7/etcd-v3.5.7-linux-amd64.tar.gz
   ```

3. 解压下载的文件：
   ```bash
   tar -xvf etcd-v3.5.7-linux-amd64.tar.gz
   ```

4. 将解压后的 `etcdctl` 文件复制到 `/usr/local/bin/` 或其他系统路径中，以便全局使用：
   ```bash
   sudo mv etcd-v3.5.7-linux-amd64/etcdutl /usr/local/bin/
   ```

5. 验证 `etcdctl` 版本：
   ```bash
   etcdutl  version
   ```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/687853656be843458353dc040c2804eb.png)
### 配置环境变量
这块需要确保在 Kubernetes 集群的主节点上运行这些命令，并且 /etc/kubernetes/pki/etcd/ 目录中包含所需的证书和密钥文件。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/14fff7396e964aad92eb934407dad4e7.png)
直接在master主机中执行下面的命令

```cpp
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS="https://127.0.0.1:2379" 
export ETCDCTL_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
export ETCDCTL_CERT="/etc/kubernetes/pki/etcd/server.crt"
export ETCDCTL_KEY="/etc/kubernetes/pki/etcd/server.key"
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1a502bc01038484b98d20fe29e2f3c93.png)
## 创建备份文件
这里本次是在opt/k8s路径之下，可以执行下面的命令
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/412e2de00fb44ec090aba079f7c9da79.png)

```cpp
etcdctl snapshot save /opt/k8s/etcd-snapshot.db
```
命令执行完成之后会得到一个.db的备份文件。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/36de1ac375464697ba8fb686eef3784d.png)
### 验证一下备份的快照文件
在status 后面跟上备份文件的信息即可，本地备份文件名称是etcd-snapshot.db，所以命令如下，执行这个命令需要在文件所在的目录下
```cpp
etcdutl --write-out=table snapshot status etcd-snapshot.db
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e121656aed0e445d9210f8931f67bdea.png)
或者要是不是在当前目录下就需要在命令中跟上你的文件所在目录是什么，示例如下

```cpp
etcdutl --write-out=table snapshot status /path/to/etcd-snapshot.db
```
## 备份文件恢复的效果演示
### 查看一下当前etcd的容器服务信息

```cpp
 sudo crictl ps -a | grep etcd
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/276f557c9704474d9d6880c0cc93c5d3.png)
### 查看当前etcd服务缓存的文件信息位置
文件所在的位置是/etc/kubernetes/manifests/etcd.yaml，查找--data-dir这个配置的信息
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b6d7b1f95d324498994ec11921013fc3.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a8c1b751ec3344308ea8aa66547370e7.png)
可以看到在上述路径下存在集群的配置信息![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ff57274990ad48e9b8812a232a78ae89.png)
通过kuboard可视化页面查看可以得到具体的信息详情
![**加粗样式**](https://i-blog.csdnimg.cn/direct/8131c6e7ab534cd19732244d91fb88ab.png)
### 删除配置文件信息模拟配置信息丢失
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cd0f8f4a743542bc988f75fd3dd9c032.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6f9e3986e4bd49cd8c519539baa1feed.png)

> 这里需要注意的一点的是，由于 Kubernetes 管理 `etcd`，当你手动停止 `etcd` 容器时，Kubernetes会**自动重新启动**它。这Kubernetes 的一种自我修复机制，确保关键组件（如`etcd`）始终保持运行状态，以保证集群的稳定性。

- **Kubernetes 控制器的自愈机制**：Kubernetes 会通过其控制器（如 `kubelet` 和 `Pod` 控制器）监控集群中的所有组件，包括 `etcd`。如果它检测到 `etcd` 容器停止或崩溃，Kubernetes 会自动尝试重新创建并启动一个新的 `etcd` 容器实例，以保持集群的正常运行。
- **Static Pod 定义**：通常 `etcd` 在 Kubernetes 集群中是以 **Static Pod** 的形式运行的。Static Pods 是由 `kubelet` 直接管理的，而不是通过 `kube-apiserver`。这些 Pod 定义文件通常位于 `/etc/kubernetes/manifests` 目录中。如果该目录中定义了 `etcd` Pod，`kubelet` 会不断尝试确保该 Pod 运行。

### 解决方案：
如果不希望 `etcd` 容器自动重新启动，可以按照以下步骤操作：

#### 1. **找到 `etcd` Static Pod 配置文件**
Kubernetes 的 `etcd` 通常是通过 **Static Pod** 配置文件进行管理，配置文件路径为 `/etc/kubernetes/manifests/etcd.yaml`。

使用以下命令检查该目录下是否存在 `etcd` 的配置文件：

```bash
ls /etc/kubernetes/manifests/
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/76b6df2bf9234659830f1b637a50a5d5.png)
#### 2. **停止 `etcd` 容器的自动重启**
要防止 Kubernetes 自动重新启动 `etcd` 容器，可以暂时**删除或移动**该配置文件。例如：

```bash
sudo mv /etc/kubernetes/manifests/etcd.yaml /root/
```

这样，`kubelet` 不再监控该 Pod，`etcd` 容器停止后将不会自动重新创建。

#### 3. **手动停止 `etcd` 容器**
一旦你移动了 `etcd` 的配置文件，`kubelet` 将不会再管理该 `etcd` 容器。此时你可以手动停止 `etcd` 容器，且不会再次自动启动：

```bash
sudo crictl stop <etcd-container-id>
```

#### 4. **恢复 `etcd` 容器**
完成需要的操作后，如果你希望 `etcd` 容器再次启动，只需将 `etcd.yaml` 文件放回原位置：

```bash
sudo mv /root/etcd.yaml /etc/kubernetes/manifests/
```

Kubernetes 会自动检测到该配置文件并重新创建 `etcd` 容器。

### 查看一下etcd服务的情况
刚刚由于手动删除配置文件之后，检测到之后自动重启，查看页面具体的集群信息
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d2f9212872754793a6f3c96477ca8d26.png)
由于集群信息文件丢失导致集群信息全部丢失
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/3ff09fecb23b48448d45b65307413e79.png)
### 使用备份文件恢复
恢复之前需要手动删除一下上面由于etcd检测到配置文件损坏之后导致的重启之后生成的节点信息，不然会出报错
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d7c2dfd8ed7c40fb92b3f69a7e79caf3.png)
删除文件之后执行

```cpp
etcdutl snapshot restore etcd-snapshot.db --data-dir=/var/lib/etcd
```
执行完之后查看文件已经恢复，查看一下etcd服务信息
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/03c45280780848d7a6aca63dde49dc63.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2b3d5a5c73984319a62398683799190a.png)
这里需要注意一下就是，经过多次尝试之后，默认会存在两个容器的信息，如果后续涉及更新需要将原先旧的信息进行停止和删除，不然容器不能自己重启因为没有位置了。重启完成之后查看集群信息已经恢复
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c7934843eab24377ac86cb08a44335b4.png)
参考文章：
https://blog.csdn.net/u011197085/article/details/139251415
https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/configure-upgrade-etcd/



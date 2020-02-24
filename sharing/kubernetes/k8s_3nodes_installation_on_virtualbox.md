# Kubernetes 三节点部署

Author: WingWJ

Date: 24th, Feb, 2020

<br/>

## 前言

工欲善其事，必先利其器。

实践 Kubernetes，面对的第一件事，就是环境搭建。然而，Kubernetes 几乎所有的安装组件和 部分Docker 镜像都放在 Google gcr 上。墙内环境安装，要么资源无法正常访问；要么即使能连上，速度也很慢。

解决的办法呢：要么准备科学工具；要么自己想办法手工替换国内源。相对来说，后者应该是更通用的使用方式。

所以，这里我整理了这份 在 VirtualBox 下 典型 3节点环境 的安装方法。熟练的话，在Host OS 准备好后，半小时左右 就能完成整套 Kubernetes 环境搭建，无需科学工具。

P.S. 附上[官网文档](https://kubernetes.io/docs/setup/independent/install-kubeadm/)以供参考。

<br/>

## 一. 环境准备

首先在 VirtualBox 中，准备三个节点。其中，ubuntu01 作为 Master 节点，其他两个作为 Node 节点。示例如下：

<img src="https://s2.ax1x.com/2020/02/24/33TWes.png" alt="k8s_3nodes_installation_on_virtualbox_001.png" style="zoom:50%;" />

其他需要说明的是：

- 单个VM/节点 推荐配置为 2U4G20G（注：之前试过 1U2G10G，感觉不太够用，尤其是硬盘）；
- VM 网络配置，建议采用 Bridge 单网卡方式（*案例参考：之前双网卡碰到的 [Service 问题](https://github.com/wingwj/wingwj.github.io/blob/master/sharing/kubernetes/log_a_k8s_svc_issue.md)*）。

- 所有节点操作系统均为 Ubuntu 16.04 LTS；
- Kubernetes 版本选用 v1.17.2；
- Kubernetes 网络模式采用 flannel vxlan 模式。

 <br/>

## 二. 预配置

这一阶段主要完成 Kubernetes 环境 正式安装配置前的各项准备。请按照章节顺序，依次在各个节点上完成以下步骤。

<br/>

### 1. 替换 apt源

替换 apt 更新源为阿里云的源，能快不少。首先，切换到 root 用户，修改对应文件：

```
$ sudo su
  mv /etc/apt/sources.list /etc/apt/sources.list.bak
  vi /etc/apt/sources.list
```

以下是对应 Ubuntu 16.04 的阿里云 更新源，请直接复制到 刚 `vi` 打开的文件 `/etc/apt/sources.list` 中：

```
deb-src http://archive.ubuntu.com/ubuntu xenial main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb http://archive.canonical.com/ubuntu xenial partner
deb-src http://archive.canonical.com/ubuntu xenial partner
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
```

修改后，重新更新：

```
$ apt-get update
  apt-get upgrade -y
```

<br/>

### 2. 禁用虚拟内存

继续使用 root 用户，执行如下命令来关闭：

```
$ swapoff -a
  sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

修改后，执行 `free -m`，确认已关闭：

<img src="https://s2.ax1x.com/2020/02/24/33ToWT.png" alt="k8s_3nodes_installation_on_virtualbox_002.png" style="zoom:50%;" />

<br/>

### 3. 关闭防火墙

```
$ systemctl stop firewalld & systemctl disable firewalld
```

<br/>

### 4. 安装 Docker

```
$ apt-get update && apt-get install docker.io -y
```

<br/>

### 5. 配置 Docker 镜像源

直接下载Docker 镜像太慢了，这里更改下默认地址。

目前，国内常用的有以下几个地址：

> - Docker 官方中国区：https://registry.docker-cn.com
> - 网易：http://hub-mirror.c.163.com
> - 中国科技大学：https://docker.mirrors.ustc.edu.cn
> - 阿里云：https://y0qd3iq.mirror.aliyuncs.com

使用 root 用户，先新增 Docker 的镜像源配置文件 `/etc/docker/daemon.json`。如果之前没有配置过，该文件默认是不存的。在其中，添加如下内容：

```
{
"registry-mirrors": ["https://y0qd3iq.mirror.aliyuncs.com"]
}
```

其中的 URL 就是指定的镜像源，可以将其设置为上面说的四个镜像源中的任何一个。这里依然配置为阿里云。

之后重启Docker 服务：

```
$ service docker restart
```

通过以下命令，查看配置是否已生效：

```
$ docker info|grep Mirrors -A 1
```

<img src="https://s2.ax1x.com/2020/02/24/38xeiQ.png" alt="k8s_3nodes_installation_on_virtualbox_003.png" style="zoom:50%;" />

<br/>

### 6. 安装 kubeadm，kubectl，kubelet

在所有节点上，安装以下三个Kubernetes 必要组件：

- kubeadm：用于初始化 Cluster；
- kubectl：Kubernetes 命令行工具；
- kubelet：运行在 Cluster 所有节点上，负责启动 Pod 和容器。

由于从官方源下载这些组件太慢，还是改用阿里云的。依次执行如下命令（注意：每一行尾不要带中文字符）：

```
$ apt-get update && apt-get install -y apt-transport-https curl
  curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

  cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
  deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
  EOF
```

之后，更新后开始安装：

```
$ apt-get update
  apt-get install -y kubelet kubeadm kubectl
```

<br/>

### 7. 准备 Kubernetes 相关镜像

在预配置的最后阶段，我们还需要预先下载 Kubernetes 需要的相关镜像。

首先得确定，安装中涉及哪些镜像。在Master 节点，执行 `kubeadm config images list`，就能够查看必备组件的列表及版本号：

<img src="https://s2.ax1x.com/2020/02/24/33TxFx.png" alt="k8s_3nodes_installation_on_virtualbox_004.png" style="zoom:50%;" />

由于 gcr 在墙内没法正常访问。所以，还是改用 阿里云 的源吧。

计划先把对应各组件下载下来，再通过docker 命令的方式来修改 tag。这样在后续 kubeadm 安装时，就不会再重复下载了。

这里有一点要 **特别说明**：以下命令，是按照当前 Kubernetes v1.17.2 版本来准备的。若有变化，直接更新组件版本号就好：

```
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.2 k8s.gcr.io/kube-apiserver:v1.17.2
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.2 k8s.gcr.io/kube-controller-manager:v1.17.2
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.2 k8s.gcr.io/kube-scheduler:v1.17.2
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2 k8s.gcr.io/kube-proxy:v1.17.2
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5 k8s.gcr.io/coredns:1.6.5
```

以上命令，可以直接保存成脚本，方便在各个节点执行。（*注：从原理上讲，确实没必要每个节点都下载所有镜像。但为了方便这里就一起进行了。源改到墙内后，下载很快的。。*）

<br/>

## 三. 创建 Kubernetes 集群

经过预配置，环境的基本准备已经完成。后续开始配置 Kubernetes 集群。

这里同样附上[官方文档](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)，供需要时查阅。

<br/>

### 1. 初始化 Master 节点

下载完毕后，在Master 节点上，就可以执行集群初始化命令了。如下：

```
$ kubeadm init --apiserver-advertise-address 192.168.56.5 --pod-network-cidr=10.244.0.0/16
```

- `--apiserver-advertise-address` 指明用 Master 节点的哪个 interface 与 Cluster 的其他节点通信。如果 Master 有多个interface，建议明确指定，如果不指定，kubeadm 会自动选择有默认网关的 interface；
- `--pod-network-cidr` 指定 Pod 网络的范围。Kubernetes 支持多种网络方案，而且不同网络方案对 `--pod-network-cidr` 有自己的要求，flannel 下默认要求该CIDR 段（10.244.0.0/16）。

<img src="https://s2.ax1x.com/2020/02/24/337A0A.png" alt="k8s_3nodes_installation_on_virtualbox_005.png" style="zoom:50%;" />

初始化过程如下：

① 初始化前检查；

② 生成 token 和证书；

③ 生成 KubeConfig 文件，kubelet 需要这个文件与 Master 通信；

④ 安装 Master 组件，默认会从 Google 的 Registry 下载组件的 Docker 镜像。我们刚已经下载好了；

⑤ 安装附加组件 kube-proxy 和 kube-dns；

⑥ Kubernetes Master 初始化成功；

⑦ 提示如何配置 kubectl；

⑧ 提示如何安装 Pod 网络；

⑨ 提示如何注册其他节点到 Cluster。

<br/>

### 2. 配置 kubectl

上一步，我们已经完成了 kubernetes 控制面的初始化。根据提示，我们还需要接着完成第 ⑦ ～ ⑨ 步的动作。首先是来配置 kubectl。

kubectl 是管理 Kubernetes Cluster 的命令行工具（CLI），前面我们已经在所有的节点安装了 kubectl。Master 初始化完成后，还需要做一些配置工作，之后就能使用 kubectl 了。

依照 `kubeadm init` 输出的第 ⑦ 步提示，推荐用 Linux 普通用户来执行 kubectl。本例中使用安装Ubuntu OS 时创建的 `wj` 用户。

```
$ su - wj
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

同时开启命令的自动补全功能：

```
$ echo "source <(kubectl completion bash)" >> ~/.bashrc
```

<br/>

### 3. 安装 flannel 网络

要让 Kubernetes Cluster 能够工作，必须安装相关网络组件，否则 Pod 之间无法通信。

Kubernetes 支持多种[网络方案](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)，本方案中使用 flannel。

执行如下命令部署 flannel 相关组件。

```
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

注意，在Master 节点执行时，同样需要使用普通用户`wj` ，否则会报错：

<img src="https://s2.ax1x.com/2020/02/24/337P6e.png" alt="k8s_3nodes_installation_on_virtualbox_006.png" style="zoom:50%;" />

注：若下载yaml 文件报错，这是因为 Github DNS 被污染导致的。请参考[之前文章](https://github.com/wingwj/wingwj.github.io/blob/master/sharing/tips/about_displayed_images.md)，通过修改 hosts 文件来恢复。

<br/>

### 4. 添加 Node 节点

在 ubuntu02 和 ubuntu03 上，分别执行在之前Master 初始化集群时第 ⑨ 步提示的命令，如下，将这两个节点逐个注册到 Cluster 中：

```
kubeadm join 192.168.56.5:6443 --token g7573r.siouz0edkfbdr74q \
  --discovery-token-ca-cert-hash sha256.5ed2c438819e4c97a3aea8e178de9ccd468ec7c46435712bcf9cf4ffa382ef2f
```

- 这里的 `--token` 来自前面 `kubeadm init` 输出的第 ⑨ 步提示，如果当时没有记录下来，可以通过 `kubeadm token list` 查看：

  <img src="https://s2.ax1x.com/2020/02/24/337iOH.png" alt="k8s_3nodes_installation_on_virtualbox_007.png" style="zoom:50%;" />

- `--discovery-token-ca-cert-hash` 指定的hash 值，需要在Master 节点上查看，命令为: `kubeadm token create --print-join-command`。（注：其实第⑨步整条语句，都可以从这个命令输出获取）

  <img src="https://s2.ax1x.com/2020/02/24/337Zkt.png" alt="k8s_3nodes_installation_on_virtualbox_008.png" style="zoom:50%;" />

- 注：其实也可以 `--discovery-token-unsafe-skip-ca-verification`，不认证 Master 的ca 证书，缺点是存在 Master 被冒充的风险，不过对我们测试环境不影响。这里只是提一下。

`kubeadm join` 执行结果如下：

<img src="https://s2.ax1x.com/2020/02/24/3GXk7D.png" alt="k8s_3nodes_installation_on_virtualbox_009.png" style="zoom:50%;" />

根据提示，可以通过 `kubectl get nodes` 查看节点的状态。

<img src="https://s2.ax1x.com/2020/02/24/33Tfwn.png" alt="k8s_3nodes_installation_on_virtualbox_010.png" style="zoom:60%;" />



通过观察，能够看到节点状态是逐渐从 NotReady 变为 Ready 的，这是因为每个节点都需要启动若干组件，这些组件都是在 Pod 中运行，也需要下载镜像（一部分已经提前下好了，因此启动比较快）。

有时可能会短时间起不来，需要多等待一会儿（可通过`kubectl describe pod` 确认）。这是因为 flannel pod 需要单独下载 `quay.io/coreos/flannel:v0.11.0-amd64` 等镜像。（注：未来这里可以同样优化到 Kubernetes 组件镜像的准备脚本中，提前下好来加速）

最后，通过 `kubectl get nodes` 查看节点的状态，节点状态都已变为 Ready。

<img src="https://s2.ax1x.com/2020/02/24/33Thoq.png" alt="k8s_3nodes_installation_on_virtualbox_011.png" style="zoom:50%;" />



可以通过如下命令查看 Pod 的状态，发现所有Pod 也已都处于 Running 状态了：

```
$ kubectl get pod --all-namespaces
```

<img src="https://s2.ax1x.com/2020/02/24/33THlF.png" alt="k8s_3nodes_installation_on_virtualbox_012.png" style="zoom:50%;" />

注：其他状态如 Pending、ContainerCreating、ImagePullBackOff，这些中间状态都表明 Pod 没有就绪，Running 才是就绪状态。

可以通过 `kubectl describe pod $pod_name` 查看 Pod 具体情况，比如：

```
$ kubectl describe pod kube-flannel-ds-amd64-2grd5 -n kube-system
```

<img src="https://s2.ax1x.com/2020/02/24/337SfK.png" alt="k8s_3nodes_installation_on_virtualbox_013.png" style="zoom:50%;" />

*注：`-n` 等同于 `--namespace`。*

这里只截取了`Event` 部分，这里展示了执行过程中的一些重要事件。比如因为网络质量不好而导致在下载 image 时失败，可以耐心等待，因为 Kubernetes 会重试。当然，先手工 `docker pull` 下载好也是一个办法。



至此，Kubernetes Cluster 创建成功，一切准备就绪。

<br/>

### 5. 完整组件图

我们把所有组件 都添加到环境各节点里，就能得到如下的组件架构图：

<img src="https://s2.ax1x.com/2020/02/24/3379SO.png" alt="k8s_3nodes_installation_on_virtualbox_014.png" style="zoom:40%;" />

**注意**：Master 节点上也有 kubelet 和 kube-proxy 服务。这是因为 Master 节点上也可以运行应用，即 Master 同时也是一个 Node。只是出于安全考虑，默认配置下 Kubernetes 不会将 Pod 调度到 Master 节点。（注：可通过命令解除限制，参见下文示例）

几乎所有的 Kubernetes 组件本身也运行在 Pod 里。这些系统组件都被放置在 `kube-system` namespace 中（**补充**：其他应用不指定时，会创建在`default` namespace里）。额外提一下 kube-dns，它为 Cluster 提供 DNS 服务，是在执行 `kubeadm init` 时（第 ⑤ 步）作为附加组件安装的。

而 kubelet 是**唯一没有以容器形式运行的 Kubernetes 组件**，它在 Ubuntu 中通过 Systemd 运行：

<img src="https://s2.ax1x.com/2020/02/24/337kmd.png" alt="k8s_3nodes_installation_on_virtualbox_016.png" style="zoom:50%;" />

<br/>

## 小结

环境搭建，想了想，好像没啥想总结的。。也就第一部分若能全部自动化，就更好了。。：）

Enjoy～

<br/>

## 篇外

### 1. 现象概述

在我自己的某次 三节点环境搭建后的环境检查中，偶然发现过一个小现象：

​		*——只有 ubuntu02 有cni，并且添加了对应路由；而ubuntu01 和 ubuntu03 却没有。*

三个节点的网络信息 与 路由表 如下。先来看 ubuntu02 的：

<img src="https://s2.ax1x.com/2020/02/24/337ETI.png" alt="k8s_3nodes_installation_on_virtualbox_017.png" style="zoom:50%;" />

 接着是 ubuntu01 和 ubuntu03 的。缺少 cni0 网桥，也缺条指向本机flannel 网段的路由。

<img src="https://s2.ax1x.com/2020/02/24/337up8.png" alt="k8s_3nodes_installation_on_virtualbox_018.png" style="zoom:50%;" />

<img src="https://s2.ax1x.com/2020/02/24/33TIYV.png" alt="k8s_3nodes_installation_on_virtualbox_019.png" style="zoom:50%;" />

<br/>

### 2. 现象解析

查看ubuntu01 的启动项，已经指定了网络插件使用cni：

<img src="https://s2.ax1x.com/2020/02/24/33T5F0.png" alt="k8s_3nodes_installation_on_virtualbox_020.png" style="zoom:50%;" />

查看对应节点的pod 的event 记录，也能看到成功安装了 cni：

<img src="https://s2.ax1x.com/2020/02/24/33TqOJ.png" alt="k8s_3nodes_installation_on_virtualbox_021.png" style="zoom:50%;" />

<br/>

其实，这是因为这两个节点上，还未创建过多个 pod，暂时不需要 cni 网桥。当在上面创建多个 pod 后，就会看到它了。：）

比如 部署一个 6 副本的 deployment，ubuntu03 上就有了：

<img src="https://s2.ax1x.com/2020/02/24/33TXwR.png" alt="k8s_3nodes_installation_on_virtualbox_022.png" style="zoom:50%;" />

ubuntu01 同理。若非要构造该场景，只需要把作为 Master 节点的它，同时设置为 Node 节点，再部署 Pod 就会出现了。

默认配置下，Master 是不会被用作 Node 节点的。如果希望将 ubuntu01也当作 Node 使用，可以执行如下命令（注：- 代表删除）：

```
kubectl taint node ubuntu01 node-role.kubernetes.io/master-
```

如果要恢复 Master Only 状态，执行如下命令：

```
kubectl taint node ubuntu01 node-role.kubernetes.io/master="":NoSchedule
```

<img src="https://s2.ax1x.com/2020/02/24/3GXF0O.png" alt="k8s_3nodes_installation_on_virtualbox_023.png.png" style="zoom:50%;" />


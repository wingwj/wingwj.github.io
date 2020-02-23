# Kubernetes Service 问题排查

Author: WingWJ

Date: 23rd, Feb, 2020 

<br/>

## 前言

本篇内容，来源于我参考其他博客，自学实践Kubernetes 时碰到的一个问题。通过来回的排查，加深了我对 Service 的理解。所以整理了下，一为记录，二为分享。

<br/>

## Kubernetes Service 是什么？

Service 是Kubernetes 核心资源对象之一，个人甚至认为，**Service 可能是Kubernetes 最核心的概念**。

我们都知道，Pod 作为Kubernetes 实际承载容器的最基本概念，很可能因为各种原因发生故障而死掉。Deployment 等 controller 会通过动态创建和销毁 Pod 来保证应用整体的健壮性。换句话说，**Pod 是脆弱的，但应用是健壮的**。

但每个 Pod 都有自己的 IP 地址（准确来说，是 Endpoint）。当各类controller 用新 Pod 替代发生故障的 Pod 时，新 Pod 会分配到新的 IP 地址——这就会影响到其对外提供的服务（比如 HTTP）。这时，Kubernetes Service 出现了：

- <u>对内</u>：Service 定义了一个服务的访问入口地址，其他Pod 应用通过这个入口地址，来访问其背后的一组由Pod副本组成的集群实例。 Service 一旦被创建，Kubernetes 就会自动为它分配一个全局唯一的虚拟IP地址，在Service 的整个生命周期内，这个地址都不会改变。这样，每个服务就变成了具备唯一IP地址的"通信节点"，而服务调用，就变成了最基础的TCP/IP 网络通信问题。

- <u>对外</u>：Service 提供了一个稳定的服务访问地址，外界不用关心Kubernetes 内部细节。而在Service 之上，再通过kube-proxy 这个软件负载均衡器（LB），来把对Service 的请求转发到后端的某个Pod 实例上，实现服务的负载均衡与会话保持。

可见，在Kubernetes 系统中，**Service 发挥了承上启下的关键作用**。

在微服务建设中，Service 同样是关键点。对一个完整的系统来说，若能将传统的大功能模块，按照不同业务能力，分析、识别、拆分、建模为彼此独立的Service 来组成，就自然能够实现分布式、弹性扩展和容错。

<br/>

## 本文目标

实践Service 的业务能力。能够通过Service Cluster IP 来访问后端Pod 服务。

通过实践来理解Service 的实现原理。

<br/>

## 实验环境

基于VirtualBox 搭建的三节点环境，OS 选择了 ubuntu 16.04 LTS，Kubernetes 版本为 v1.17.2，网络方案采用flannel vxlan 方式。

每个节点采用双网卡配置：

- *Host-Only*：用于节点互访；

- *NAT*：用于连接外网。

多解释下，这里为何没选择 Bridge 模式。这是因为最初考虑Kubernetes 实践过程中，需要频繁连外网，因为你懂的原因，而Bridge 没法直接应用本地科学工具。

整套物理环境构成，如下：

<img src="https://s2.ax1x.com/2020/02/23/314Ild.png" alt="k8s-svc-000.png" style="zoom:50%;" />

<br/>

## 过程记录

### 一. 部署应用

在Kubernetes 中，对象构建时 很多采用的是松耦合的方式。用纽带的方式来连接各实体。比如，Service 确实代表了一组Pod，但并没强行要求Service 创建时必须创建附属的Pod。你完全可以，先创建好 Pod，调试完毕后，再创建Service。当然了，若你需要一起创建时，也能够同时创建。Kubernetes 负责建立和维护好 Service 与 Pod 的映射关系就够了。这一点，非常灵活，喜欢～

#### 1. 问题显现

先创建下面的 Deployment：

<img src="https://s2.ax1x.com/2020/02/23/3lYGOx.png" alt="k8s-svc-001.png" style="zoom:50%;" />

一共会启动三个 Pod，运行httpd 镜像，label 是`run: httpd`，后续Service 将会用这个 label 来挑选 Pod。此外，Pod 还会分配各自的 IP，这些 IP 只能被 Kubernetes Cluster 中的容器和节点访问。这里让我们在Master 节点上试着访问一下各个pod：

<img src="https://s2.ax1x.com/2020/02/23/3lYakD.png" alt="k8s-svc-002.png" style="zoom:50%;" />

咦，怎么不通。。登陆到Node节点（ubuntu02）上再试试：

<img src="https://s2.ax1x.com/2020/02/23/3lY861.png" alt="k8s-svc-003.png" style="zoom:50%;" />

发现 **只能访问各自节点上的Pod，跨节点就不通**。

出师不利啊。。



#### 2. 环境确认

先查看下两个节点的网卡信息：

<img src="https://s2.ax1x.com/2020/02/23/3lYYm6.png" alt="k8s-svc-004.png" style="zoom:40%;" />

<img src="https://s2.ax1x.com/2020/02/23/3lYNTO.png" alt="k8s-svc-005.png" style="zoom:40%;" />

贴一张网上看到的 flannel 模式示意图，理理思路：

<img src="https://s2.ax1x.com/2020/02/23/3lYBpd.png" alt="k8s-svc-006.png" style="zoom:50%;" />

Flannel 由部署在各节点上的flanneld，通过vxlan 协议，为各个Pod 打通了一个跨节点的扁平网络。而cni 就是将各个容器挂载到该网络的执行者，保证了各容器间实现通信。

对照上图，我们按照正常流程中，从Master 节点访问Node 节点的容器时，`curl 10.244.1.63` 的全过程来逐步检查下：

1. 从Master 节点（ubuntu01），首先查看本节点路由表，找到对应10.244.1.0/24 网段，明确需要从flannel.1 （10.244.0.0）发出；

   <img src="https://s2.ax1x.com/2020/02/23/3lYD1A.png" alt="k8s-svc-007.png" style="zoom:50%;" />

2. flanneld 将数据包封装成vxlan，通过节点间的 flanneld 组成的扁平网络进行通信。查看flanneld 服务正常：

   <img src="https://s2.ax1x.com/2020/02/23/3lYr6I.png" alt="k8s-svc-008.png" style="zoom:50%;" />

3. 这时在Node 节点（ubuntu02）上查看下对应的路由表。flannel.1 在解包后，得到目的地址为 10.244.1.63，因此通过网桥cni0，转发给对应pod 以响应：

   <img src="https://s2.ax1x.com/2020/02/23/3lYc0f.png" alt="k8s-svc-009.png" style="zoom:50%;" />


4. 看上去各节点都正常。那具体问题出在哪儿了呢？



#### 3. 物理网络排查

想想最开始的问题表现，即Node 节点能够访问本机的Pod，但是无法访问其他节点的Pod。根据上面Node 节点（ubuntu02）上的路由表配置，证明本机网络应该是正常的。**那会不会是跨主机通信这里，出了问题？**再进一步来排查下：

1. 再检查下节点防火墙。确实已关闭：

   <img src="https://s2.ax1x.com/2020/02/23/3lYg78.png" alt="k8s-svc-010.png" style="zoom:50%;" />

2. ip转发呢？查一下也没问题：

   <img src="https://s2.ax1x.com/2020/02/23/3lYWtg.png" alt="k8s-svc-011.png" style="zoom:55%;" />

3. 检查下iptables，没发现啥不对的。安全起见，再重新放行一下：

   ```
   iptables -P INPUT ACCEPT
   iptables -P FORWARD ACCEPT
   iptables -F
   iptables -L -n
   ```

   <img src="https://s2.ax1x.com/2020/02/23/3lYfhQ.png" alt="k8s-svc-012.png" style="zoom:50%;" />

4. 重新试一下，还是不通。。



#### 4. 软件功能分析

这么看，物理网络这边，应该没什么问题了。**那问题是出在软件层面？**<u>难道是负责节点通信的flanneld，没有将 节点A 的请求消息，成功传递给节点B？</u>

1. 检查下各节点状态，确实是连通的，说明管理网是正常的：

<img src="https://s2.ax1x.com/2020/02/23/3lY4pj.png" alt="k8s-svc-013.png" style="zoom:50%;" />

2. 难道是 flanneld，用了其他网络么？想了想本环境，由于每个节点间，既要能相互访问，又要能够从外访问，所以使用的是 Host-Only + NAT 的双网卡模式——莫非，flanneld 用了那块NAT 的网卡，最后导致无法通信了？

3. 去网上搜了下，flanneld 选择网卡的方式（http://www.sel.zju.edu.cn/?p=690，https://blog.csdn.net/qingyafan/article/details/93519196），确定其遵循的是以下规则：

   ```
   --iface="": interface to use (IP or name) for inter-host communication. Defaults to the interface for the default route on the machine. This can be specified multiple times to check each option in order. Returns the first match found.
   
   --iface-regex="": regex expression to match the first interface to use (IP or name) for inter-host communication. If unspecified, will default to the interface for the default route on the machine. This can be specified multiple times to check each regex in order. Returns the first match found. This option is superseded by the iface option and will only be used if nothing matches any option specified in the iface options.
   ```

   翻译一下：

   - 即如果指定了`--iface` 参数，则按指定的来；

   - 否则判断是否用正则方式指定了网卡；

   - **如果都没指定，则选择默认路由对应的网卡**。

4. 回头看下刚才的路由信息，当前环境下，<u>确实默认路由指向的是 `enp0s8`</u>，即NAT 网络对应的网卡。**问题确认！**

   <img src="https://s2.ax1x.com/2020/02/23/3lYIcn.png" alt="k8s-svc-015.png" style="zoom:50%;" />

 

OK，到这里，问题定位结束。



#### 5. 解决问题

找到原因了，就可以考虑解决方法了。

按上面的描述，**只需要让 flanneld 能选择到正确的网卡就行**。无非两种方案：

1. 修改默认路由，将其对应到网卡 `enp0s3` 上；
2. 修改 flanneld 配置，指定选用网卡 `enp0s3`。

这里<u>选用第二种方式</u>，毕竟是在学容器么，还是尽量用容器的方式来解决。：）



在Master 节点上，通过 `kubectl get pod --all-namespaces` 能看到，当前环境中提供flannel 功能的 kube-flannel 是采用 DaemonSet 方式来部署的：

<img src="https://s2.ax1x.com/2020/02/23/3lYoXq.png" alt="k8s-svc-015.png" style="zoom:60%;" />

找到当时部署时采用的 yaml文件。官方来源为：https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml，`wget` 下载后，在yaml 以下位置，用 `--iface` 来指定使用网卡 `enp03s`。（注：yaml 中有好几处需修改）

<img src="https://s2.ax1x.com/2020/02/23/3lY7n0.png" alt="k8s-svc-016.png" style="zoom:50%;" />

*P.S. 由于DNS污染的原因，网址 `raw.githubusercontent.com` 无法访问，还同时还波及了Github 的图片服务。请参见我之前[文章](https://github.com/wingwj/wingwj.github.io/blob/master/sharing/tips/about_displayed_images.md)，可以在电脑端解决该问题。*

执行 `kubectl replace --force -f kube-flannel.yml`，重置所有 kube-flannel：

<img src="https://s2.ax1x.com/2020/02/23/3lYHBV.png" alt="k8s-svc-017.png" style="zoom:50%;" />

查看 pod 状态，确定每个pod 都已正常启动。

<img src="https://s2.ax1x.com/2020/02/23/3lYb7T.png" alt="k8s-svc-018.png" style="zoom:60%;" />



补充：若遇到问题，可采用 `kubectl logs $pod_name -n kube-system` 和 `kubectl describe pod $pod_name -n kube-system` ，来进一步定位。



最后，重新测试下各Pod 的连通状况。

<img src="https://s2.ax1x.com/2020/02/23/3lYXh4.png" alt="k8s-svc-019.png" style="zoom:60%;" />



问题解决！ ：）

<br/>

### 二. 部署 Service

说好的Service 实践，没想到之前的排错写了这么多。。但其实换个思路想，这不更证明之前提到的各对象解耦的好处。我们不正是先部署Deployment/Pod，调试完毕，再来构建Service 么？：）

赶紧来创建个 Service 试试。其配置文件如下：

<img src="https://s2.ax1x.com/2020/02/23/3ldowt.png" alt="k8s-svc-020.png" style="zoom:60%;" />

- 指明资源类型为 Service，名称为 httpd-svc;
- `selector` 指明挑选 label 为 `run: httpd` 的 Pod 作为 Service 的后端；

- 使用TCP 协议，将 Service 的 8080 端口映射到 Pod 的 80 端口。

执行 `kubectl apply` 创建 Service httpd-svc。

<img src="https://s2.ax1x.com/2020/02/23/3ldHFf.png" alt="k8s-svc-021.png" style="zoom:50%;" />

httpd-svc 分配到一个 CLUSTER-IP 10.103.102.170。可以通过该 IP 访问后端的 httpd Pod，根据前面的端口映射，这里要使用 8080 端口：

<img src="https://s2.ax1x.com/2020/02/23/3ldbY8.png" alt="k8s-svc-022.png" style="zoom:60%;" />



通过 `kubectl describe service` 可以查看 httpd-svc 与后端 Pod 的对应关系。

<img src="https://s2.ax1x.com/2020/02/23/3ldIeI.png" alt="k8s-svc-023.png" style="zoom:50%;" />

Endpoints 罗列了三个 Pod 的 IP 和端口。

<br/>

## 三. Service 原理解析

前面也写了那么多了，最后，也对Service 实现原理做个总结吧。

文章开头提到了，Kubernetes 维护了Service 和 各Pod 间的映射关系。Pod 的 IP 是在容器中配置的，那么 Service 的 Cluster IP 又是配置在哪里的呢？又是如何映射到 Pod IP 的呢？其实，在之前的问题排查中已经透露了他的身份，还是我们的老朋友，iptables。

Service Cluster IP 是一个虚拟 IP，Kubernetes 通过节点上的 iptables 规则，来进行管理。

通过 `iptables-save` 命令能够查询出当前节点的 iptables 规则，这里只截取与 httpd-svc Cluster IP 10.103.102.170 相关的信息：

<img src="https://s2.ax1x.com/2020/02/23/3ldXlQ.png" alt="k8s-svc-024.png" style="zoom:50%;" />

这两条规则的含义是：

1. 如果 Cluster 内的 Pod（源地址来自 10.244.0.0/16）要访问 httpd-svc，则允许；
2. 其他源地址访问 httpd-svc，跳转到规则 `KUBE-SVC-RL3JAE4GN7V0GDGP`。

`KUBE-SVC-RL3JAE4GN7VOGDGP` 规则如下：

<img src="https://s2.ax1x.com/2020/02/23/3ldjyj.png" alt="k8s-svc-025.png" style="zoom:50%;" />

1. 1/3 的概率跳转到规则 `KUBE-SEP-AJ0R7AIR7SS3FOXH`；
2. 1/3 的概率（剩下 2/3 的一半）跳转到规则 `KUBE-SEP-L2ZGWAEDHRQK0PHS`；
3. 1/3 的概率跳转到规则 `KUBE-SEP-EP24YNMCSB7HEA72`。

上面三个跳转的规则分别如下：

<img src="https://s2.ax1x.com/2020/02/23/3ld4OA.png" alt="k8s-svc-026.png" style="zoom:50%;" />

<img src="https://s2.ax1x.com/2020/02/23/3ldqfS.png" alt="k8s-svc-027.png" style="zoom:50%;" />

<img src="https://s2.ax1x.com/2020/02/23/3ldOSg.png" alt="k8s-svc-028.png" style="zoom:50%;" />

即将请求分别转发到后端的三个 Pod。

更新到部署图中，Service 和Pod 分布如下：

<img src="https://s2.ax1x.com/2020/02/23/314Ild.png" alt="k8s-svc-029.png" style="zoom:50%;" />

通过上面的分析，我们能得到如下**<u>结论</u>**：

- **iptables 将访问 Service 的流量转发到后端 Pod，而且使用了类似轮询的负载均衡策略**。

- Cluster 中的每个节点都配置了相同的 iptables 规则，确保了整个 Cluster 都能够通过 Service 的 Cluster IP 来访问 Service。

<br/>

## 题外

除了直接通过 Cluster IP 访问 Service，DNS 是更加便捷的方式。

此外，文中Service 使用的 Cluster IP 方式，只允许集群内对象来访问。而对于Service 需要对外暴露服务时，一般会采用 NodePort 方式。

但以上这些非本文重点，就不在此赘述了。

<br/>

## 小结

其实本文提到的Service，在整个容器/Kubernetes 领域中，其实是个很容易一目带过的知识点。因为它的粒度不大，原理掰开说也并不复杂，往常看可能就直接翻过去了。但正如前文提到的，Service 其实很重要。因为基本，所以在日常使用中，有些细节我们并未真的关注过，只是单纯用。而真的碰到问题时，可能才会着手来看。这次的问题也算是给我自己的提醒：**关注细节**。

此外，对于问题定位来说，和之前从0开始一步步的看Python 学习OpenStack 一样。遇到问题，要在理解原理的基础上，从源头逐项排查。抽丝剥茧，再大的问题也是有其表征的。看的多了自然经验就多，积累多了能力也就提升了，人类不就是这样越练越强的么。：）
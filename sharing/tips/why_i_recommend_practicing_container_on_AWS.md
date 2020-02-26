# 为什么推荐在 AWS 上实践容器

Author: WingWJ

Date: 26th, Feb, 2020

<br/>

## 简述

近期实践了容器领域的几大项目，包括 Docker、Swarm、Kubernetes。最开始我都是在 AWS 上进行的，后续才转移到 VirtualBox。发现在使用中，还是有不少差异。

所以，这里想谈谈，自己在这两个平台上的使用感受。

<br/>

## About AWS

### 优势

最开始，在 AWS 下实践各项目，大体感觉概括起来就是：爽。

尤其安装配置，按照官方教程等，直接执行。非常的流畅。镜像下载、yaml 下载、各类安装包安装源，大多和本地环境一样，非常快。所以，个人完全可以更多的关注流程 or 产品本身，不用过多考虑其他环境问题。个人感觉，是更纯粹的实践空间。

功能方面，我只在实践 Docker Macvlan 网络模式的时候，遇到过不兼容。问题出在 sub-interface 及 网卡混杂（promisc）模式配置失败。去 AWS 官网 找了下，发现是 AWS 出于安全考虑，而在 VPC 内做的限制。可以参考[这里](https://amazonaws-china.com/cn/answers/networking/vpc-security-capabilities/)，以下为节选：

> *It is not possible for a virtual instance running in promiscuous mode to receive or sniff traffic that is intended for a different virtual instance. While customers can elect to place their interfaces into promiscuous mode, the hypervisor will not deliver any traffic to an instance that is not addressed to it.*

*P.S. 我还按照老外一博客提供的方式试了下，发现还是无效，就没再找其他规避方案了。*

所以，目前为止，我也就碰到了这一处，由于 AWS 限制而带来的问题。除此之外，我在实践中没再遇到其他影响。包括多机管理、存储、以及多主机场景下的 Overlay、Weave、Flannel、Calico 等网络方案，都顺利跑通了。

<br/>

### 劣势

使用 AWS 的**最大障碍，源自网络的不稳定**。

AWS 不同 Region，各地各时段的网络访问状况都不一样，有的地区甚至无法连接。以一般最常用的东京 Region 来说，本地亲测，裸连一般都会存在10% 左右的丢包率。到了晚间忙时，会飙升到将近 20%。体现在使用上，会出现频繁的登陆中断（默认 ssh 方式），基本很难使用了。亚洲其他机房，包括 香港、首尔、新加坡，都有类似的问题，大体表现还不如东京 Region。

我不知道这是否和我用的[白嫖套餐](https://aws.amazon.com/cn/free)，无法确保 SLA 有关（注：话说三节点用起来，你咋都得花点钱不是。。）。我试过更换网段、绑定浮动 IP，但问题都没根本解决。不知是否有更好的办法。或者，改用某种科学工具能缓解该问题。欢迎推荐～

此外，使用公有云，还会有个**隐藏劣势：使用付费**。AWS 提供了多种服务，但在容器实践时，其实用不上几个。除了 EC2 外，VPC、IAM、EBS 基本就够了，这些在白嫖套餐里，基本都包住了。你唯一需要支付的费用，就是 三节点 EC2 VM 的费用：

- 如果你用的白嫖套餐里的 VM 规格（题外话：EC2 各个 Region 默认规格还不一样），那在 VM 总时长超过750h 后，需要支付额外时长的费用；
- 如果你 VM 使用的更高规格，那会直接按时长来收费（注意，三节点，就是三倍速哦。。）。

<br/>

### 特别之处

在 AWS 下实践一些流程时，可能会和一般的 本地 VirtualBox 方案的实现不一致。比如我之前写过的一篇——[《在AWS 下实践 Docker Machine》](https://github.com/wingwj/wingwj.github.io/blob/master/sharing/docker/run_docker_machine_on_AWS.md)的文章，就是在自学过程中，逐步调试完成的，吹一个“学以致用”吧。。我觉得对我理解 Docker Machine 相关概念和流程，是有帮助的。

这种 **对内容的再理解和再实践** 上的时间与精力的花费，我个人认为是值得的。当然，仁者见仁智者见智。也可能有些人认为，与教程不一致会不方便，还会影响实践进度。所以我把这一点单独成节，供大家比较和考虑。

以上，是我个人对 AWS 使用的理解。下面来说 VirtualBox。

<br/>

## About VirtualBox

### 优势

VirtualBox 下用起来就图个方便。本地启起来就能跑，没有太多的配置环境。总结下就是：便捷。

Oracle VirtualBox 是免费软件，可直接从官网下载，各 OS 镜像也能在对应官网找到。不像在 AWS 上，还需要支付使用费。只是对于使用 VirtualBox 的个人电脑，有一定的性能要求。毕竟 Swarm、Kubernetes 典型部署都要三个节点，最少你也得有个两节点来玩吧。

但我在使用 VirtualBox 实践容器各软件栈时， 其实用起来倒并不太愉悦。最大的问题，就在于和外网各资源站的连通性。

<br/>

### 劣势

不同于之前 OpenStack，只要把 软件包源 / Github 下载速度 的问题搞定，剩下就都是 OpenStack 环境自身配置的事了。等搭建完毕，基本就算全到位了。

而在容器实践中，由于不少组件是散布在各大网站源的，又因为你懂的原因，往往不是 404 就是慢的没法用。

且各组件配置，默认写的也都是官方地址。所以，你要么得改组件 yaml 本身（不推荐），要么得自己想办法，把这些镜像都搞下来。而且，由于组件解耦，安装一个完整组件，可能需要安装诸多镜像包。所以，你还得事先了解该组件涉及哪些镜像包，才能提前准备。。非常复杂。。

这一点，我在 VirtualBox 中搭建 ELK 和 Prometheus 的时候尤其明显。一众镜像，得来回手动下载。而且由于调度的不确定性，还得在两个 Node 节点上都准备上。。真的折腾。。相对应的是，我最初在 AWS 下实践时，根本就没有太多阻碍就装完直接用了。内部有些什么组件包，我甚至都没关注过（也许是缺点？）。

这里可能会有反对意见：比如，你产品出去商用，肯定得准备自己的相应配置和镜像仓库，不能还得依赖网络下载啊？——这点没问题，但考虑对于初学者而言，我可能只是想搭套环境，大致看看容器是什么。而这些额外的技术门槛，可能导致本来一小时内就能完成的搭建，资料查来查去，可能就两天后了。。

这种由于环境限制产生的属于学习内容以外的解决成本，个人觉得是不值得花太多时间/精力的，事倍功半。

<br/>

## 篇外

点名：**不推荐使用 [Docker Desktop](https://www.docker.com/products/docker-desktop) 作为主力环境**。

它其实是在你本地创建了一个 VM，然后把 Docker 跑在 Host OS 上。我最初使用时觉得也还算方便，但后续使用中，愈发觉得不顺手。毕竟中间多了一层 VM，有时你还得专门对付这台虚拟机。尤其是在那些要和本机打交道的场景，比如 操作 Voume、网络调试、日志收集等。刚上手的时候，你可能连 VM 对应目录都找不到。。后续使用时还有一些不兼容的小问题（路径啊、认证啊、网不通啊），调试起来也麻烦。

而使用 VirtualBox，和使用物理服务器几乎相同，不需要额外准备，实践经验也是共通的。

此外，还有一点要提的是，使用 Docker Desktop 甚至还会 [影响 VirtualBox](https://docs.docker.com/docker-for-windows/install/#system-requirements)。你确定要为了这棵树，放弃整片森林么？能完全控制的 VirtualBox 不香么。。

<br/>

## 小结

综上，在实践容器技术栈时，如果符合以下两个条件，还是优先推荐在 公有云 上来实践的：

1. 具备持续稳定的网络连接；
2. 接受可能产生的使用费用。

或许，<u>阿里云 等国内公有云厂商的 公有云香港 Region or VPS，是最优选择？</u>（*未实践过*）

对于不具备条件的同学，可能还得依赖本地环境来实践了。比如 Kubernetes，可以参考我之前写的 [《Kubernetes 三节点安装指导》](https://github.com/wingwj/wingwj.github.io/blob/master/sharing/kubernetes/k8s_3nodes_installation_on_virtualbox.md)，不需要科学工具，就能顺利完成环境搭建。

可见，说到底，这篇文章是个广告贴。。：）

<br/>
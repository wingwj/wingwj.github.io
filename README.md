# Summary
从2011年在云计算领域从业，至今已近10年。

从AWS，Eucalyptus，VMware，到从Diablo 版本起我从业最久的OpenStack，见证了一个个项目从起步，到成熟，到逐渐退潮。整个过程中遇到和解决了各种问题，但之前却从未记录分享出来。

近些年，随着Docker，Swarm，尤其是近些年的Kuberenetes，相关技术领域一直在进行着不断的变更，从未停歇。作为技术人，需要逐步更新自己的知识储备。写技术博客，也是个精炼的过程。

亡羊补牢，从此开始。

<br />

## 说明：关于近期文章图片无法显示

Author: WingWJ

Date: 19th, Feb, 2020

------

近期Github 图片在墙内全部无法显示，导致各文章中的图片都没法查看了。。

之前只是在自学容器时，发现 `raw.githbusercontent.com` 访问不了，得手动处理 yaml。

去网上搜了下，说似乎是由于Github 托管在 新加坡AWS上 被波及了。

我文章的图片是和文章一起，直接上传到Github 的。开始我以为只是把Github 本身的图片服务ban了，还专门去找了个第三方图床，想通过外链的方式来显示图片。整理完传上去试了下，发现图片还是挂（*注：可见柏林峰会那篇*）。。直接点图片链接进去，发现外链图片是通过 `camo.githubusercontent.com`  来中转的，所以问题依旧。。

想了下，文章的话我还是会继续更新着。

后续我再观察下，实在不行，可能得再找个地方来同步更新了。希望早日恢复吧～

<br />

------


## Legacy
* [Keystone 多级租户研究-Mitaka版本](sharing/keystone_hierarchical_projects/FAR_for_keystone_hierarchical_projects.md)
* [Keystone 多级配额研究](sharing/keystone_hierarchical_quota/keystone_hierarchical_quota.md)
* [DevStack 安装配置遇到的坑](sharing/tips/DevStack_installing.md)

<br />

## OpenStack

* [Cyborg 项目解析](sharing/cyborg/OpenStack%20Cyborg.md)
* [补遗：柏林峰会 观感总结](sharing/berlin_summit/OpenStack_Berlin_Summit.md)
* 边缘计算相关：**StarlingX** 技术详解
  * [stx-nfv project](sharing/starlingx/stx_nfv.md)
  * [stx-fault project](sharing/starlingx/stx_fault.md)
  * [stx-ha project](sharing/starlingx/stx_ha.md)
  * *to be continued..*

<br />

## Container

- Docker
  - [AWS上 部署Docker Machine](sharing/docker/run_docker_machine_on_AWS.md)
- Kubernetes
  - *to be continued..*
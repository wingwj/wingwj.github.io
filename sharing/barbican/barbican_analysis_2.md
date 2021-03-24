# Barbican 详解（二）

Author: WingWJ

Date: 24th, Mar, 2021

<br/>

## 1. 关于证书

上一篇，我们了解了Barbican中最基本的用法。这一篇，我们重点关注下Barbican中证书的相关用法。

证书（Certificate）在Barbican使用中，比简单的密码（Password）用法更复杂。上一节中，我们展示了如何创建证书。这一节，我们结合证书的常见使用，来考察下Barbican提供的能力。

<br/>

### 1.1 支持设置证书过期时间么？

支持。在创建和更新时，通过`--expiration`参数来指定，格式遵循ISO 8601格式。比如，当前时间为2021年3月22日16点12分46秒，即可使用如下格式：

- 形如`20210322T161246+08`，注意时区；
- 当然，也可直接使用换算后的 UTC时间，即`20210322T081246`）；
- 注意：API文档中指定的格式为：`YYYY-MM-DDTHH:MM:SSZ`。

在环境上实验下，如果指定错了会怎么样：

<img src="https://z3.ax1x.com/2021/03/24/6HXZu9.png" alt="barbican_201.png" />

可见，当设置的超时时间早于当前时间，Secret 会直接创建失败。

<br/>

### 1.2 证书过期后，会有什么表现？

我们直接在环境上操作，比较直观。这里先创建一个 Secret，然后将过期时间设置的很近。如下：

<img src="https://z3.ax1x.com/2021/03/24/6HXuAx.png" alt="barbican_202.png" />

如上图，Secret 已经创建完毕，且还有不到两分钟过期。等它过期后，再重新执行下查询操作，看看会发生什么：

<img src="https://z3.ax1x.com/2021/03/24/6HXEjJ.png" alt="barbican_203.png" />

可见，证书过期后会，Secret 查询会直接提示找不到。

我们去DB中看下证书的具体状态，可见，证书仍为 Active状态——即不会显式更改为`Expired` 状态！

<img src="https://z3.ax1x.com/2021/03/24/6HXeBR.png" alt="barbican_204.png" />

P.S. 和其他OpenStack对象一致，即使被删除了对象状态仍为`Active`，只是`deleted`和`deleted_at`两个字段会有值。

<br/>

### 1.3 通过update接口，能更新证书有效期么？

不行。Barbican中的`update`接口，实际是用于 <u>先创建metadata再更新payload</u> 的 Secret 创建方式——类似Glance中 <u>先只创建Image再上传</u> 这种方式。

<br/>

### 1.4 证书过期怎么办？支持自动续期么？

其实，没办法，证书过期了就不能用了，Barbican 也并不提供自动更新的功能。想了想其实也合理，即使Barbican自动更新证书，但又不负责&没无法将证书同步更新到各个应用中，反而引入了不一致。所以，Barbican直接没管。

<u>课外问题：证书临期时，是否有告警？——没有。</u>

<br/>

### 1.5 如果证书正被使用，能够删除 Secret 么？

这里先给出结论——可以，我后续将结合组件使用流程来验证。

关于Barbican未保护使用中的 Secret，可能会有些异议——为何不锁住防止意外删除呢？

Barbican提供的是一种通用服务，我按照使用方式来理（xi）解（di）：

- 除了证书外，有些是一次性的密码存储（如`password`类型），不涉及使用期保护；
- 对于证书，Barbican更多提供的是存储功能（接口名都是`store`，而不是常用的的`create`），Barbican不会/也无法去确认证书后续的用法，比如是安装在环境中了、安装到VM里了；甚至只把Barbican来当一个证书存储器，证书被拿去它用了。

所以，Barbican直接不管理这些过程：

- 如果用户不指定有效期，就可一直使用；
- 如果要删除，则需用户自行确认，资源不保留。而这与云平台中其他资源是类似的，比如：
  - VM 密码
  - VM keypair密钥

因此，若考虑基于Barbican提供商用功能时，建议这里将影响明示客户，防止误操作。

<br/>

### 1.6 说到这儿了，证书能用在什么流程中呢？

用的比较多的是这两个：

- Neutron/Octavia：用于 TLS终结（TLS-Terminated） HTTPS LB；
- Magnum：管理 K8s 使用的证书。

后面计划以LB这个来演示下，到时具体看。

<br/>

## 2. 证书这里似乎和Keypair 很像，两者有何区别呢？

其实是一样的。在早期发展中，虚拟机需要通过`ssh`提供密钥访问能力，所以这部分工作由Nova来代为实现了，后续也就继续用了。在代码层面，Nova实际也是通过Python对应包（`cryptography`、`paramiko`）来创建密钥对的。

从keypair的创建接口可以看出，支持指定`public-key`和`private-key` 以导入；当两者都不指定时将自动创建公私钥；然后公钥入库，私钥通过接口返回。

此外，没有了。。Nova中的这部分能力就是仅用于虚拟机访问的，无法独立出来提供给其他服务使用（也不应该）。所以，这也算是Barbican诞生的一个原因吧。

<br/>
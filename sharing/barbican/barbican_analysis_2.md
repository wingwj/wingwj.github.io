# Barbican 详解（二）

Author: WingWJ

Date: 24th, Mar, 2021

Updated at: 16th, May, 2021

<br/>

## 引子

上一篇，我们了解了 Barbican 中最基本的用法。这一篇，我们通过考察其中证书的相关用法，来感受下 Barbican 提供的其他能力。

<br/>

## 1. Secret 有效期

说到证书，除了它本身提供的身份鉴别属性外，还有另一个属性也不容轻视，即有效期。不同于人类社会中的各类资质类证书，一旦获取终身有效。由于 IT 系统以及人员的变动性，为了系统安全，必需要设置可靠的定期更迭的验证方式。试想一下，如果有人持有永久登陆系统的有效凭证，这对整个系统来说就是巨大的风险。

在 Barbican 设计时，当然也考虑到了这一点。下面我们会用几个具体例子，来考察下 Barbican 中的有效期设置。

<br/>

### 1.1 如何设置过期时间？

在创建和更新 Secret 时，通过`--expiration` 参数（简写为 `-x`）来指定，格式遵循 ISO 8601格式。比如，当前时间为：2021年5月16日20点59分34秒，即可使用如下格式：

- 形如 `20210516T205934+08`，注意时区；
- 当然，也可直接使用换算后的 UTC时间，即 `20210516T205934`）；
- 注意：API文档中指定的格式为：`YYYY-MM-DDTHH:MM:SSZ`。

在环境上实验下，如果指定错了会怎么样：

```
[root@cdpm03 opadmin]# date
Sun May 16 20:59:34 CST 2021
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret store -n wj_expr -x 20210516T165900+08
4xx Client error: Bad Request: Provided object does not match schema 'Secret': 'expiration' is before current time. Invalid property: 'expiration'
Bad Request: Provided object does not match schema 'Secret': 'expiration' is before current time. Invalid property: 'expiration'
```

可见，当设置的超时时间早于当前时间，Secret 会直接创建失败。

<br/>

### 1.2 过期后，会有什么表现？

我们直接在环境上操作，比较直观。这里先创建一个 Secret，然后将过期时间设置的很近。如下：

```
[root@cdpm03 opadmin]# date
Sun May 16 21:00:28 CST 2021
[root@cdpm03 opadmin]# openstack secret store -n wj_expr -x 20210516T210200+08
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/3fa6b963-1bba-4c2b-8033-25446a2ffbfc |
| Name          | wj_expr                                                                   |
| Created       | None                                                                      |
| Status        | None                                                                      |
| Content types | None                                                                      |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | 2021-05-16T21:02:00+08:00                                                 |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/3fa6b963-1bba-4c2b-8033-25446a2ffbfc
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/3fa6b963-1bba-4c2b-8033-25446a2ffbfc |
| Name          | wj_expr                                                                   |
| Created       | 2021-05-16T13:00:56+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | None                                                                      |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | 2021-05-16T13:02:00+00:00                                                 |
+---------------+---------------------------------------------------------------------------+
```

如上图，Secret 已经创建完毕，且还有不到两分钟过期。等它过期后，再重新执行下查询操作，看看会发生什么：

```
[root@cdpm03 opadmin]# date
Sun May 16 21:02:14 CST 2021
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/3fa6b963-1bba-4c2b-8033-25446a2ffbfc
4xx Client error: Not Found: Not Found. Sorry but your secret is in another castle.
Not Found: Not Found. Sorry but your secret is in another castle.
```

可见，证书过期后会，Secret 查询会直接提示找不到。

我们去 DB 中看下证书的具体状态，可见，证书仍为 `Active` 状态——即不会显式更改为`Expired` 状态！

```
MariaDB [barbican]> select * from secrets where id='3fa6b963-1bba-4c2b-8033-25446a2ffbfc';
+--------------------------------------+---------------------+---------------------+------------+---------+--------+---------+---------------------+-----------+------------+------+-------------+----------------------------------+--------------------------------------+
| id                                   | created_at          | updated_at          | deleted_at | deleted | status | name    | expiration          | algorithm | bit_length | mode | secret_type | creator_id                       | project_id                           |
+--------------------------------------+---------------------+---------------------+------------+---------+--------+---------+---------------------+-----------+------------+------+-------------+----------------------------------+--------------------------------------+
| 3fa6b963-1bba-4c2b-8033-25446a2ffbfc | 2021-05-16 13:00:56 | 2021-05-16 13:00:56 | NULL       |       0 | ACTIVE | wj_expr | 2021-05-16 13:02:00 | aes       |        256 | cbc  | opaque      | d3da001bf87e475c81ee7c9b2396c87e | a98ff827-5b5d-47ba-830d-98de49179011 |
+--------------------------------------+---------------------+---------------------+------------+---------+--------+---------+---------------------+-----------+------------+------+-------------+----------------------------------+--------------------------------------+
1 row in set (0.780 sec)
```

P.S. 和其他 OpenStack 对象一致，即使被删除了对象状态仍为`Active`，只是`deleted`和`deleted_at`两个字段会有值。

<br/>

### 1.3 通过 update接口，能更新有效期么？

不行。Barbican 中的`update`接口，实际是用于 <u>先创建 metadata 再更新 payload</u> 的 Secret 创建方式——类似Glance 中 <u>先只创建 Image 再上传</u> 这种方式。

<br/>

### 1.4 支持自动续期么？

不支持。Secret 过期了就不能用了，Barbican 也并不提供自动更新的功能。想了想其实也合理，即使 Barbican 自动更新证书，但又不负责&无法将证书同步更新到各个应用中，反而引入了不一致。所以，Barbican 直接没管。

<u>课外问题：证书临期时，是否有告警？——没有。</u>

<br/>

### 1.5 如果 Secret 正被使用，能被删除么？

这里先给出结论——可以，我后续将结合组件使用流程来验证。

关于 Barbican 未保护使用中的 Secret，可能会有些异议——为何不锁住防止意外删除呢？

Barbican 提供的是一种通用服务，我按照使用方式来理（xi）解（di）：

- 除了证书外，有些是一次性的密码存储（如`password`类型），不涉及使用期保护
- 对于证书，Barbican 更多提供的是存储功能（接口名都是`store`，而不是常用的的`create`），Barbican 不会/也无法去确认证书后续的用法，比如是安装在环境中了、安装到VM里了；甚至只把 Barbican 来当一个证书存储器，证书被拿去它用了

所以，Barbican 直接不管理这些过程：

- 如果用户不指定有效期，就可一直使用；
- 如果要删除，则需用户自行确认，资源不保留。而这与云平台中其他资源是类似的，比如：
  - VM 密码
  - VM keypair 密钥对

因此，若考虑基于 Barbican 提供商用功能时，建议这里将影响明示客户，防止误操作。

<br/>

## 2. 说到 keypair 了，证书似乎和它很像，两者有何区别？

其实是一样的。在早期发展中，虚拟机需要通过`ssh`提供密钥访问能力，所以这部分工作由Nova来代为实现了，后续也就继续用了。在代码层面，Nova 实际也是通过 Python 对应包（`cryptography`、`paramiko`）来创建密钥对的。

从 keypair 的创建接口可以看出，支持指定`public-key`和`private-key` 以导入；当两者都不指定时将自动创建公私钥；然后公钥入库，私钥通过接口返回。

此外，没有了。。Nova 中的这部分能力就是仅用于虚拟机访问的，无法独立出来提供给其他服务使用（也不应该）。所以，这也算是 Barbican 诞生的一个原因吧。

<br/>

## 3. 看之前举例，密钥/证书似乎都是手动导入到 Barbican 的。它支持自动创建么？

好问题！

如果观察仔细的话，会发现我都是通过 Secret 的 `store` 接口，将之前已创建好的密钥/证书，手动导入到 Barbican 里的（*直接存密码这种，其实也算是直接导入的*）。留意该接口名，用的是“store”，而不是一般对象常用的“create”接口。那是否意味着，Barbican 的 Secret，只能导入用？

这时，就该 Barbican 中的另一个对象 `Order` 登场了。有了它，在部分不想/无法手动导入密钥和证书的场景时，你**可以用它来请求 Barbican 自主生成 Secret**，虽然支持自主创建的对象类型有限。而在版本发展变迁中，随着 CA（*Certificate Manager*）功能的废弃，Barbican 已经不再支持申请创建证书（*Certificate*），当前支持申请创建的是另外两种资源：对称密钥（*symmetric keys*） 和 非对称密钥（*asymmetric keys*）。

我们将在下一讲着重讲解 `order` 对象。

<br/>

## 4. 总结下，Barbican 与 OpenStack 其他组件的交互情况：

随着组件发展，Barbican 也逐步与其他 OpenStack 组件实现了功能融合。这里把上文提到的证书使用情况，也放在一起统计，主要体现在以下几部分：

- Nova：暂态卷（*Ephemeral disk*）加密。用的不多

- Cinder：加密卷。与 Nova 配合，通过卷类型来实现的前端加密。有用
- Glance：镜像签名。需要在镜像上传前准备，能有效防止被篡改；但无法防止上传后被篡改的场景

- Octavia：提供 TLS终结 HTTPS LB
- Swift：使用对称密钥来加密 Swift 容器

- Sahara：没用过。不乱解释。。

- Magnum：可为 K8s 提供TLS证书

官网[安全文档](https://docs.openstack.org/security-guide/secrets-management/secrets-management-use-cases.html)中提供了对以上各方式的详细使用指导，感兴趣可以直接查看。

<br/>
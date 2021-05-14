# Barbican 详解（三）

Author: WingWJ

Date: 11th, May, 2021

<br/>

## Secret 与 周边

经过前两篇的铺垫，我们对 Barbican 中 Secret 的基本使用，有了初步的认识，这一节，我们来了解一些与 Secret 相关的对象。

（*注：由于本文命令截图过多，还要间或添加解说；因此本篇记录过程，均采用操作记录方式提供，就不额外截图了。看起来可能比用图床方式还更快。。*）

<br/>

### 1. Secret Metadata

和 OpenStack 中其他资源一样，Secret Metadata 是用来给 Secret 添加一些辅助类的 `key/value` 对的，常用来记录一些相关信息（*如地址信息、描述等*）。

这里相对简单就不举例了，可以直接查看[官网示例](https://docs.openstack.org/api-guide/key-manager/secret_metadata.html)。

这里额外提醒两点：

- 创建和更新，均支持批量。只是调用参数有区别：

  - 单条更新时，结构体形如：

    ```
    { "metadata": {
        "description": "contains the AES key",
        "geolocation": "12.3456, -98.7654"
      }
    }
    ```

  - 多条更新时，结构体形如：`{ "key": "access-limit", "value": "11" }`

- 若只需更新某个`key` 的对应值时，则只需重新提交这一对 `key/value` 就ok。

<br/>

### 2. Secret Type

在之前的用法中，我们都是直接指定 *Password/Certificate* 等 Secret 自身相关的参数，然后将其存储到 Barbican 中的，在 DB 中我们也看到相关记录了。

如果仔细想想，这里可能隐藏了一个这样的问题：对于这么多的 Secret，能否区分类型呢？

在早先 Barbican 版本中，确实所有 Secret 都是和前面用法一样，各种类型的都往里存，比较混乱。而在当前版本中，已经提供了多种类型以区分 Secret：

- `symmetric` - 类似对称加密密钥的字节数组（*byte arrays*）
- `public` - 非对称加密的公钥
- `private` - 非对称加密的私钥
- `passphrase` - 普通文本密码
- `certificate` - 类似 X.509 的加密证书
- `opaque` - 用于后向兼容（*backwards compatibility*）设置的保留类型：
  - 因为之前 API 不具备指定 `Secret Type` 参数
  - 当前不指定类型时，也会被归类到此类型
  - 新版本下，更鼓励 明确 Secret 的对应类型

<br/>

### 3. Secret Container

证书容器，类似证书存储的一个独立逻辑分区，拿对象存储的概念来对比，比较好理解：Secret 相当于 Object，Secret Container 相当于一个 Bucket。

使用 Secret Container 的最大好处，在于给用户提供了分区的能力，用户可以将不同类型、不同功用的 Secret，记录在不同的逻辑分区内。当用户拥有大量 Secret 时，这一点尤为重要。

与 Bucket 不同的是，Container 本身还具有额外的类型属性。当前支持三种：

- `Generic` - 通用型。这种用的最多，啥都可以往里放。Barbican 不限制放入的 Secret 的类型和数量
- `Certificate` - 证书型。存放和证书相关的 Secret。支持指定如下参数：
  - `certificate` - PEM 格式的 x509 证书。必选
  - `private_key` - 私钥。可选
  - `private_key_passphrase` - 私钥密码。可选
  - `intermediates` - PEM 格式的 PKCS7 证书链（*certificate chain*）。可选
- `RSA` - RSA型。存放 RSA 公私钥相关信息。三个参数均为必选项：
  - `public_key` - 公钥
  - `private_key` - 私钥
  - `private_key_passphrase` - 私钥密码

此外，还有一点与 Bucket 不同的地方：对象存储的一般用法，是在存储 Object 时先建好 Bucket，后续再指定放入。而 Secret Container 的用法并不一致：**在创建 Container 时，默认需要提前备好 Secret**：

- 只有 `Generic` 类型，支持后续通过 `POST` 方法，向已有 Container 中添加 Secrets

  - 接口定义如下。下文会有具体例子：

    ```
    POST /v1/containers/{container_uuid}/secrets
    Headers:
        X-Project-Id: {project_id}
    
    Content:
    {
        "name": "private_key",
        "secret_ref": "https://{barbican_host}/v1/secrets/{secret_uuid}"
    }
    ```

  - `Generic` 类型，支持后续通过 `DELETE` 方法，删除 Container 中的 Secrets

  - 是只解关联，还是直接释放？——猜测：*这里只会解关联，不会释放 Secret 本身。参考下文释放 Container*

- 其他两种，均不支持创建后再加入

此外，明确一点，系统不存在默认的 Container；未指定 Container 的 Secret，也并未存储在所谓的“默认 Container 内”。可通过 `GET` 直接列举 `Secret Container` 命令即可确定，示例略。

#### 如上文，那能否在创建 Generic 类型的 Container 时先不指定 Secret，靠后续再添加呢？

不行。虽然 `Generic` 类型支持后续通过接口添加，但在创建时，同样需要指定至少一个 Secret，否则接口会直接报错——这一点也是 API 同一性的正常表现。如下：

```
[root@cdpm03 opadmin]# openstack secret container create --type generic --name wj_test
Must supply at least one secret.
```

#### 那一个 Secret，能否加入不同的 Container？

让我们来测试下。首先，先来创建一个简单的 Secret：

```
[root@cdpm03 opadmin]# openstack secret store --payload wjpass=123
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438 |
| Name          | None                                                                      |
| Created       | 2021-05-10T02:10:23+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'text/plain'}                                               |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
```

指定该 Secret，创建第一个 Container：

```
[root@cdpm03 opadmin]# openstack secret container create --secret "wj_pass=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438"
+----------------+-----------------------------------------------------------------------------------+
| Field          | Value                                                                             |
+----------------+-----------------------------------------------------------------------------------+
| Container href | https://10.127.3.110:9311/v1/containers/8b9e6930-b787-4f0e-8c75-674c0fa9f458      |
| Name           | None                                                                              |
| Created        | None                                                                              |
| Status         | ACTIVE                                                                            |
| Type           | generic                                                                           |
| Secrets        | wj_pass=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438 |
| Consumers      | None                                                                              |
+----------------+-----------------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret container get https://10.127.3.110:9311/v1/containers/8b9e6930-b787-4f0e-8c75-674c0fa9f458
+----------------+-----------------------------------------------------------------------------------+
| Field          | Value                                                                             |
+----------------+-----------------------------------------------------------------------------------+
| Container href | https://10.127.3.110:9311/v1/containers/8b9e6930-b787-4f0e-8c75-674c0fa9f458      |
| Name           | None                                                                              |
| Created        | 2021-05-10T02:13:43+00:00                                                         |
| Status         | ACTIVE                                                                            |
| Type           | generic                                                                           |
| Secrets        | wj_pass=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438 |
| Consumers      | None                                                                              |
+----------------+-----------------------------------------------------------------------------------+
```

再指定该 Secret，尝试创建第二个 Container：

```
[root@cdpm03 opadmin]# openstack secret container create --secret "wj_pass_2=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438"
+----------------+-------------------------------------------------------------------------------------+
| Field          | Value                                                                               |
+----------------+-------------------------------------------------------------------------------------+
| Container href | https://10.127.3.110:9311/v1/containers/78cc4128-8263-4790-a5a7-559f615ac0fb        |
| Name           | None                                                                                |
| Created        | None                                                                                |
| Status         | ACTIVE                                                                              |
| Type           | generic                                                                             |
| Secrets        | wj_pass_2=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438 |
| Consumers      | None                                                                                |
+----------------+-------------------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret container get https://10.127.3.110:9311/v1/containers/78cc4128-8263-4790-a5a7-559f615ac0fb
+----------------+-------------------------------------------------------------------------------------+
| Field          | Value                                                                               |
+----------------+-------------------------------------------------------------------------------------+
| Container href | https://10.127.3.110:9311/v1/containers/78cc4128-8263-4790-a5a7-559f615ac0fb        |
| Name           | None                                                                                |
| Created        | 2021-05-10T02:16:02+00:00                                                           |
| Status         | ACTIVE                                                                              |
| Type           | generic                                                                             |
| Secrets        | wj_pass_2=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438 |
| Consumers      | None                                                                                |
+----------------+-------------------------------------------------------------------------------------+
```

可见，同样创建成功。即，答案是：**允许将同一个 Secret 加入不同的 Container**。

#### 删除 Container 时，会同步释放其中的 Secret 么？

我们接着上例来执行。再执行前，我们先想一下，如果同一个 Secret 能够加入不同 Container。那再释放时如果要同步释放的话，岂不是会打架了？——实测结果也确实如此：**删除 Container 也只释放 Container 对象本身，不会删除 Secret**。

这里具体来看下：先将上一节中创建的第二个 Container 释放掉，只保留第一个 Container： 

```
[root@cdpm03 opadmin]# openstack secret container delete https://10.127.3.110:9311/v1/containers/78cc4128-8263-4790-a5a7-559f615ac0fb
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret container list
+-------------------------+------+-------------------------+--------+---------+-------------------------+-----------+
| Container href          | Name | Created                 | Status | Type    | Secrets                 | Consumers |
+-------------------------+------+-------------------------+--------+---------+-------------------------+-----------+
| https://10.127.3.110:93 | None | 2021-05-10T02:13:43+00: | ACTIVE | generic | wj_pass=https://10.127. | None      |
| 11/v1/containers/8b9e69 |      | 00                      |        |         | 3.110:9311/v1/secrets/4 |           |
| 30-b787-4f0e-           |      |                         |        |         | 2a93eeb-9157-4d38-a40d- |           |
| 8c75-674c0fa9f458       |      |                         |        |         | acd944656438            |           |
+-------------------------+------+-------------------------+--------+---------+-------------------------+-----------+
```

再查询下 Secret，确定初始 Secret 仍然存在：

```
[root@cdpm03 opadmin]# openstack secret list
+-------------+------+------------+--------+---------------+-----------+------------+-------------+------+------------+
| Secret href | Name | Created    | Status | Content types | Algorithm | Bit length | Secret type | Mode | Expiration |
+-------------+------+------------+--------+---------------+-----------+------------+-------------+------+------------+
| https://10. | None | 2021-05-10 | ACTIVE | {u'default':  | aes       |        256 | opaque      | cbc  | None       |
| 127.3.110:9 |      | T02:10:23+ |        | u'text/plain' |           |            |             |      |            |
| 311/v1/secr |      | 00:00      |        | }             |           |            |             |      |            |
| ets/42a93ee |      |            |        |               |           |            |             |      |            |
| b-9157-4d38 |      |            |        |               |           |            |             |      |            |
| -a40d-acd94 |      |            |        |               |           |            |             |      |            |
| 4656438     |      |            |        |               |           |            |             |      |            |
+-------------+------+------------+--------+---------------+-----------+------------+-------------+------+------------+
```

#### 上一步的结果，是否由于该 Secret 仍有 Container 关联，导致没直接释放呢？

好问题。接下来我们把之前创建的第一个 Container 也删除掉，来考察下表现：

```
[root@cdpm03 opadmin]# openstack secret container delete https://10.127.3.110:9311/v1/containers/8b9e6930-b787-4f0e-8c75-674c0fa9f458
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret container list

[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret list
+-------------+------+------------+--------+---------------+-----------+------------+-------------+------+------------+
| Secret href | Name | Created    | Status | Content types | Algorithm | Bit length | Secret type | Mode | Expiration |
+-------------+------+------------+--------+---------------+-----------+------------+-------------+------+------------+
| https://10. | None | 2021-05-10 | ACTIVE | {u'default':  | aes       |        256 | opaque      | cbc  | None       |
| 127.3.110:9 |      | T02:10:23+ |        | u'text/plain' |           |            |             |      |            |
| 311/v1/secr |      | 00:00      |        | }             |           |            |             |      |            |
| ets/42a93ee |      |            |        |               |           |            |             |      |            |
| b-9157-4d38 |      |            |        |               |           |            |             |      |            |
| -a40d-acd94 |      |            |        |               |           |            |             |      |            |
| 4656438     |      |            |        |               |           |            |             |      |            |
+-------------+------+------------+--------+---------------+-----------+------------+-------------+------+------------+
```

可见，当所有关联 Container 被释放后，原 Secret 仍然存在。证明了我们的结论，即 **删除 Container 只处理该逻辑概念本身，并不会直接清理其中的 Secret**。

#### 那能否往 Container 中重复添加同一 Secret？

我们来验证下，首先仍创建一个 Container：

```
[root@cdpm03 opadmin]# openstack secret container create --secret "wj_test=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438"
+----------------+-----------------------------------------------------------------------------------+
| Field          | Value                                                                             |
+----------------+-----------------------------------------------------------------------------------+
| Container href | https://10.127.3.110:9311/v1/containers/b762d06b-d85d-4113-973f-a636ac8ad221      |
| Name           | None                                                                              |
| Created        | None                                                                              |
| Status         | ACTIVE                                                                            |
| Type           | generic                                                                           |
| Secrets        | wj_test=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438 |
| Consumers      | None                                                                              |
+----------------+-----------------------------------------------------------------------------------+
```

获取 token 后，使用 `curl` 命令（*无 CLI*），使用之前创建 Container 时填入的 Secret 信息，尝试向该  `generic` 类型 Container 中在此再次重复添加 Secret。观察结果发现，Barbican 会提示 `name` 及 `ID` 重复：

```
[root@cdpm03 opadmin]# curl -g -i --cacert "/opt/cluster_config/ssl/ca.crt" -X POST -H "X-Auth-Token: gAAAAABgmMwquK7Jr0eVTryT8w7Gj76AxXHeCk4eP_9Vs4MQ3JD03lmDw_kW6f3VXVjcFvhjHTDz_PQBnVqdczZeEDbI3B_HBw9AaMlPTzJmWKQqn5VDLlTpls_SYOWam4a0G3ASCfcDV0YAYYS4jqbFIvFJZd3-3ujZQWGUqoITT0-jmyFi1Ik" -H 'content-type:application/json' -d '{"name": "wj_test", "secret_ref":"https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438"}' https://10.127.3.110:9311/v1/containers/b762d06b-d85d-4113-973f-a636ac8ad221/secrets
HTTP/1.1 409 Conflict
Date: Mon, 10 May 2021 06:11:37 GMT
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_wsgi/4.6.4 Python/2.7
Content-Length: 202
x-openstack-request-id: req-32d72751-ae28-44b2-a797-f6df4af1e76c
Content-Type: application/json

{"code": 409, "description": "Conflict. A secret with that name and ID is already stored in this container. The same secret can exist in a container as long as the name is unique.", "title": "Conflict"}
```

我们把 Secret 命名从 `wj_test` 修改为 `wj_test_2` 后，仍旧使用原 `secret ref`，在 Container 中尝试添加，结果加入成功：

```
[root@cdpm03 opadmin]# curl -g -i --cacert "/opt/cluster_config/ssl/ca.crt" -X POST -H "X-Auth-Token: gAAAAABgmMwquK7Jr0eVTryT8w7Gj76AxXHeCk4eP_9Vs4MQ3JD03lmDw_kW6f3VXVjcFvhjHTDz_PQBnVqdczZeEDbI3B_HBw9AaMlPTzJmWKQqn5VDLlTpls_SYOWam4a0G3ASCfcDV0YAYYS4jqbFIvFJZd3-3ujZQWGUqoITT0-jmyFi1Ik" -H 'content-type:application/json' -d '{"name": "wj_test_2", "secret_ref":"https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438"}' https://10.127.3.110:9311/v1/containers/b762d06b-d85d-4113-973f-a636ac8ad221/secrets
HTTP/1.1 201 Created
Date: Mon, 10 May 2021 06:35:49 GMT
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_wsgi/4.6.4 Python/2.7
Location: https://10.127.3.110:9311/v1/containers/b762d06b-d85d-4113-973f-a636ac8ad221
Content-Length: 97
x-openstack-request-id: req-1073dbbe-bf43-42f3-a9da-f5e2793b5e06
Content-Type: application/json

{"container_ref": "https://10.127.3.110:9311/v1/containers/b762d06b-d85d-4113-973f-a636ac8ad221"}[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret container get https://10.127.3.110:9311/v1/containers/b762d06b-d85d-4113-973f-a636ac8ad221
+----------------+-------------------------------------------------------------------------------------+
| Field          | Value                                                                               |
+----------------+-------------------------------------------------------------------------------------+
| Container href | https://10.127.3.110:9311/v1/containers/b762d06b-d85d-4113-973f-a636ac8ad221        |
| Name           | None                                                                                |
| Created        | 2021-05-10T03:20:28+00:00                                                           |
| Status         | ACTIVE                                                                              |
| Type           | generic                                                                             |
| Secrets        | wj_test_2=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438 |
|                | wj_test=https://10.127.3.110:9311/v1/secrets/42a93eeb-9157-4d38-a40d-acd944656438   |
| Consumers      | None                                                                                |
+----------------+-------------------------------------------------------------------------------------+
```

因此，由以上验证结果可知：**往 Container 中添加 Secret 时，不允许 `name` 及 `secret ref` 都重复；只 `name` 不一致时，仍旧能够重复添加**。

<br/>

### 4. Secret ACL

与多数 OpenStack 服务一样，Barbican 内的资源（Secrets、Containers）默认是按**租户粒度**来提供的（在Barbican中），用户的接入权限由 `policy` 来定义。而在某些场景下，可能需要更细粒度的访问控制，比如，能够限制特定 Secret、Container 的访问许可范围。

- ACL 支持将 Secret、Container 标记为 `private`，即只有创建者有访问权限
  - 通过关键字 `project-access` 关键字来指定
  - 默认为 `True`，即允许租户内访问

- 支持指定添加其他 用户id，来将其加入访问列表
- 当前，仅允许定义`read` 操作，能够向指定用户开放读取权限（如 *get_secret*、*get_secret_metadata*、*get_secret_payload*、*get_container* 等）

- 资源所有者、租户管理员 支持设置 ACL

接着我们来测试下。由下图可见，默认 ACL规则为租户内放开，例外列表也为空：

```
[root@cdpm03 opadmin]# openstack secret store --payload wj_pass=123
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/ae2cf6d9-154c-4ade-a3e4-089e79a81d79 |
| Name          | None                                                                      |
| Created       | 2021-05-10T06:46:28+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'text/plain'}                                               |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack acl get https://10.127.3.110:9311/v1/secrets/ae2cf6d9-154c-4ade-a3e4-089e79a81d79
+----------------+----------------+-------+---------+---------+----------------------+
| Operation Type | Project Access | Users | Created | Updated | Secret ACL Ref       |
+----------------+----------------+-------+---------+---------+----------------------+
| read           | True           | []    | None    | None    | https://10.127.3.110 |
|                |                |       |         |         | :9311/v1/secrets/ae2 |
|                |                |       |         |         | cf6d9-154c-4ade-a3e4 |
|                |                |       |         |         | -089e79a81d79/acl    |
+----------------+----------------+-------+---------+---------+----------------------+
```

<br/>

#### 更新 ACL

支持更新 Secret、Container ACL 规则。请留意，接口会**整体替换**原有 ACL 规则，而不是单条。消息体形如以下形式，分别设定用户名单及租户开关：

```
{
  "read":{
    "users":[
      "2d0ee7c681cc4549b6d76769c320d91f",
      "721e27b8505b499e8ab3b38154705b9e",
      "c1d20e4b7e7d4917aee6f0832152269b"
    ],
    "project-access":false
  }
}
```

这里要额外提醒的是，ACL规则设定接口中，支持采用 `PUT` 或 `PATCH` 两种方式：

- `PUT` - 全量更新。

- `PATCH` - 可以只更新部分值。如，只更新用户列表，或者只设定开关。

具体可直接查看[官网例子](https://docs.openstack.org/api-guide/key-manager/acls.html#how-to-update-acl)，这里不重复。

#### 删除 ACL

其实就是 “重置（*reset*）”资源的访问规则，将之前设定改回默认规则（*因此多次调用，接口并不会报错*）。

<br/>

### 5. Secret Quotas

老面孔了，用于限定单个租户资源使用量，支持 Secrets、Containers、Orders、Consumers、CAs，默认不限制。

```
[quotas]
quota_secrets = -1
quota_orders = -1
quota_containers = -1
quota_consumers = -1
quota_cas = -1
```

用户可以使用 `GET` 请求来查询租户的配额。

**注意**：默认只有服务管理员（对应角色为：`key-manager:service-admin` ）能够使用 API 设置租户配额，而不是 `admin` ，可查看 `policy.json` 来确定。

<br/>

### 6. Secret Order

密钥订单，允许用户向 Barbican 请求生成 Secret，支持异步，适用于生成密钥对的场景。支持两种常见类型：

- `symmetric keys` - 对称密钥
- `asymmetric keys` - 非对称密钥

看接口，其实会发现 Secret Order 创建参数与 Secret 很像。查了下，这里应该有些历史遗留问题。原本申请 Secret 请求应该都是需要走 Order 的，而后续随着发展，相关功能已经全部转到 Secret 本身来完成了。查看[接口文档](https://docs.openstack.org/barbican/wallaby/api/reference/orders.html)，会明确看到已经标注P版后废弃了。因此，建议后续直接使用 Secret 即可。

<br/>

### 7. Secret Store

类似 Cinder 的后端存储，只是 Barbican 对接的是各类证书存储后端。不同密钥存储后端，提供了不同的访问能力及保密级别，管理员可以根据具体应用需要，来择优配置具体后端。多后端功能，在以下场景中会非常有用：

- 可以在低密的软件实现的 Secret Store 内，存放不重要信息 or 调试资源；而在高保密的 HSM（*Hardware security module，硬件安全模块*） 中存放重要生产数据
- 某些场景需要支持高并发能力，此时 HSM 可能表现不佳。或需要花费较大代价以水平扩容后，才能提供高性能支撑
- HSM 都有性能容量限制，需时刻关注其存储大小
- 部分局点有合规要求，如部分业务必须存储在 HSM 中。而其他用户/信息，使用软件 Secret Store 即可满足功能需求

Barbican 支持多后端配置。且默认提供了两个级别：租户首选后端、全局默认后端：

- 租户关联员可以指定首选后端。指定后，该租户下所有用户的 **新创建** Secret 都会使用该后端
- 未指定时，则选用全局设置

这些功能可通过修改 `barbican.conf` 中配置项 `enable_multiple_secret_stores` 来开启。形如：

```
[secretstore]
# Set to True when multiple plugin backends support is needed
enable_multiple_secret_stores = True
stores_lookup_suffix = software, kmip, pkcs11, dogtag, vault

[secretstore:software]
secret_store_plugin = store_crypto
crypto_plugin = simple_crypto

[secretstore:kmip]
secret_store_plugin = kmip_plugin
global_default = True

[secretstore:dogtag]
secret_store_plugin = dogtag_plugin

[secretstore:pkcs11]
secret_store_plugin = store_crypto
crypto_plugin = p11_crypto

[secretstore:vault]
secret_store_plugin = vault_plugin
```

各插件的说明及配置方法，[官网](https://docs.openstack.org/security-guide/secrets-management/barbican.html#secret-store-plugins)有明确记录，可直接查阅。

注意：多后端功能开启后，原默认配置项 `enabled_secretstore_plugins ` 和 `enabled_crypto_plugins ` 不再生效。这里附上默认配置以对比理解：

```
[secretstore]
namespace = barbican.secretstore.plugin
enabled_secretstore_plugins = store_crypto

[crypto]
namespace = barbican.crypto.plugin
enabled_crypto_plugins = simple_crypto
```

<br/>

### 8. Secret Consumer

一句话概括：Consumer 是 Container 的关注者，在 Container 删除前会得到通知。即，所有 Consumer 是注册在某个 Container 上的，可以指定 Container ID 来查询其所有关注者，支持分页查询。

删除时，需要在消息体内，注明待解关联的 Consumer 的 `name`、`URL`，[官网](https://docs.openstack.org/api-guide/key-manager/consumers.html)有具体例子。

*注：简单测试了下，用户参数似乎并不校验。加上本身非重点，考虑篇幅问题，这里不再展开描述了。*

<br/>

### 9. CA

CA 用来签发、认证、管理证书。在 Barbican 中用的较简单，有些功能已废弃，这里简单提一下。

与其他功能一样，Barbican 也是通过后端插件的方式来提供证书管理。其中，Dogtag 用的较多，它是 Red hat 证书系统的开源上游版本。Barbican 通过对应插件与 Dogtag 进行集成/交互：

- Dogtag CA（*Certificate Manager*） 子系统：用于签发、续订、撤销不同类型证书。可用作 Barbican 的 CA 后端，并通过 Dogtag CA 插件来交互
- Dogtag KRA（*Key Recovery Authority*） 子系统：用于通过存储在 软件NSS DB/HSM 中的主加密密钥 来加密存储 Secrets。可用作 Barbican 的 Secret Store，并通过 Dogtag KRA 插件来交互

官方提供了具体的[配置指导](https://docs.openstack.org/api-guide/key-manager/dogtag_setup.html)，这里不展开。

<br/>

## 小结

好了，至此，Barbican 中的基本对象都已经阐述过了，把主要对象放在一起，就能得到各对象关系图了。用字符图绘制如下：

```
                      ┌────────────┐
                      │  Consumer  │
                      └─────▲──────┘
                            │ N                  
                            │                   ┌─────────────┐
                            │ 1                 │    Quotas   │
                    ┌───────┴────────┐          └─────────────┘
                    │    Container   │
                    └───────┬────────┘
                            │ 1
                            │
                            │ N
                1  ┌────────▼────────┐ 1
      ┌────────────┤      Secret     ├─────────────┐
      │            └─┬─────────────┬─┘             │
      │             1│             │ 1             │
      │              │             │               │
      │ 1           1│             │ N             │ 1
┌─────▼─────┐  ┌─────▼─────┐  ┌────▼──────┐  ┌─────▼─────┐
│   Type    │  │    ACL    │  │ Metadata  │  │   Store   │
└───────────┘  └───────────┘  └───────────┘  └───────────┘
```

由上图可见，Barbican 从 Secret 这一基本概念出发，逐步延展出其他周边对象。

<br/>


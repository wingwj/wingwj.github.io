# Barbican 详解（三）

Author: WingWJ

Date: 11th, May, 2021

Updated at: 16th, May, 2021

<br/>

## Secret 与 周边

经过前两篇的铺垫，我们对 Barbican 中 Secret 的基本使用，有了初步的认识，这一节，我们来了解一些与 Secret 相关的对象。

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

### 3. Secret Store

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

### 4. Secret Order

密钥订单，用来请求 Barbican 自主生成 Secret。

在[上一篇](barbican_analysis_2.md)末尾，我提了下：在之前 Secret 使用中，多数是用户已生成了密码等数据，后续需要加密存储到 Barbican 中；而在某些场合，我们可能需要 Barbican 来生成 Secret。这时就可以使用 Order 来申请，支持异步，十分适用于生成密钥的场景（*注：申请自主创建证书的功能已废弃*）。

支持两种常见类型（*最下面一种已废弃*）：

- `symmetric keys` - 对称密钥。主要用于加密卷、暂态卷加密、对象存储加密 等场景下
- `asymmetric keys` - 非对称密钥。主要用于镜像签名及验证 等场景下
- `certificate` - 证书。已废弃，~~请当没看见~~。。

比对接口后，其实会发现 Order `create` 与 Secret `Store` 接口很像，毕竟最终落地还是要由 Secret 来承载。创建时，需要明确 Secret 的 `algorithm`、 `bit_length`、`mode` 参数——注意：[第一篇](barbican_analysis_1.md)时提出的“参数无用论”的疑问，这里就是答案——即 申请密钥创建时，会根据传入参数执行。因此，请按照实际需求，正确填写。

#### 对称密钥

下面是一个对称密钥的创建示例：

```
[root@cdpm03 opadmin]# openstack secret order create -n wj_so -a aes -b 256 -t application/octet-stream key
+----------------+--------------------------------------------------------------------------+
| Field          | Value                                                                    |
+----------------+--------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/6858db51-7271-4523-841c-b3f8807526e1 |
| Type           | Key                                                                      |
| Container href | N/A                                                                      |
| Secret href    | None                                                                     |
| Created        | None                                                                     |
| Status         | None                                                                     |
| Error code     | None                                                                     |
| Error message  | None                                                                     |
+----------------+--------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret order get https://10.127.3.110:9311/v1/orders/6858db51-7271-4523-841c-b3f8807526e1
+----------------+---------------------------------------------------------------------------+
| Field          | Value                                                                     |
+----------------+---------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/6858db51-7271-4523-841c-b3f8807526e1  |
| Type           | Key                                                                       |
| Container href | N/A                                                                       |
| Secret href    | https://10.127.3.110:9311/v1/secrets/2cb96791-c374-4b37-b81a-7b1098452514 |
| Created        | 2021-05-15T05:11:50+00:00                                                 |
| Status         | ACTIVE                                                                    |
| Error code     | None                                                                      |
| Error message  | None                                                                      |
+----------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/2cb96791-c374-4b37-b81a-7b1098452514
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/2cb96791-c374-4b37-b81a-7b1098452514 |
| Name          | wj_so                                                                     |
| Created       | 2021-05-15T05:11:50+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | symmetric                                                                 |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
```

可见，Order 状态已更新，对应 Secret 也已创建完毕。代表刚提交的对称密钥的创建申请，已经被正式执行了。

我们接着使用 `-p` 参数，来尝试获取下密钥明文，会发现 `payload-content-type` 参数对不上：

```
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/2cb96791-c374-4b37-b81a-7b1098452514 -p
4xx Client error: Not Acceptable: Secret payload retrieval issue seen - Wrong payload content-type.
Not Acceptable: Secret payload retrieval issue seen - Wrong payload content-type.
```

这是由于 CLI 默认使用 `text/plain` 方式，明确类型后重新获取。

咦，还是报错？提示无法正确解码：

```
[root@cdpm03 opadmin]# openstack secret get -p -t application/octet-stream https://10.127.3.110:9311/v1/secrets/2cb96791-c374-4b37-b81a-7b1098452514
'utf8' codec can't decode byte 0xbb in position 0: invalid start byte
```

这是如果你改用 `curl` 命令查看时，会看到接口其实返回的是 `HTTP 200`，即返回ok。后面是在 `osc_lib` 中抛出了这个 `UnicodeDecodeError` 的错误。

这里先不深究。我们改用 `curl` 命令来执行，会发现结果是能正常返回的，只是看上去像是乱码：

```
[root@cdpm03 opadmin]# curl -g --cacert "/opt/cluster_config/ssl/ca.crt" -X GET https://10.127.3.110:9311/v1/secrets/2cb96791-c374-4b37-b81a-7b1098452514/payload -H "Accept: application/octet-stream" -H "X-Auth-Token: ${AUTH_TOKEN}"
��L�q,��&,-�v0��L�z���
```

可以在 `curl` 中指定 `-o` 参数，来将结果直接输出到一个具体文件中，再通过 `od -x` 命令来查看，会看到具体值。

```
[root@cdpm03 opadmin]# od -x out_aes_data
0000000      bfef    efbd    bdbf    ef4c    bdbf    2c71    bfef    efbd
0000020      bdbf    2c26    ef2d    bdbf    3076    bfef    efbd    bdbf
0000040      ef4c    bdbf    ef7a    bdbf    bfef    efbd    bdbf    000a
0000057
```

其实，这个看上去像乱码的东西，就是 Barbican 在自主创建对称密钥时所使用的密码。

可以查看代码确认。对应简单加密（*专有名词，后续阐述原理时会详细解释*）场景，相关代码位于 `./barbican/plugin/crypto/simple_crypto.py ` 中。可见，原始密码是由 `os.urandom()` 方法，根据传入的密钥长度来自动生成的 ：

```
def generate_symmetric(self, generate_dto, kek_meta_dto, project_id):
    byte_length = int(generate_dto.bit_length) // 8
    unencrypted = os.urandom(byte_length)

    return self.encrypt(c.EncryptDTO(unencrypted),
                        kek_meta_dto,
                        project_id)
                            
def encrypt(self, encrypt_dto, kek_meta_dto, project_id):
    kek = self._get_kek(kek_meta_dto)
    unencrypted = encrypt_dto.unencrypted
    if not isinstance(unencrypted, six.binary_type):
        raise ValueError(
            u._(
                'Unencrypted data must be a byte type, but was '
                '{unencrypted_type}'
            ).format(
                unencrypted_type=type(unencrypted)
            )
        )
    encryptor = fernet.Fernet(kek)
    cyphertext = encryptor.encrypt(unencrypted)
    return c.ResponseDTO(cyphertext, None)
```

总结下，即：**通过 Barbican 提供的 Order 方法，能让 Barbican 来自主生成对称密钥；密码会采用随机的二进制字节流**。

#### 非对称密钥

同上面对称密钥，这里同样用具体实例来演示。如下，先来申请一个非对称密钥，指定算法为 `rsa`，密钥长度 2048：

```
[root@cdpm03 opadmin]# openstack secret order create -n wj_rsa_key -a rsa -b 2048 -t application/octet-stream asymmetric
+----------------+--------------------------------------------------------------------------+
| Field          | Value                                                                    |
+----------------+--------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/69d0c5f0-ce37-46fa-b726-b7b18f740dfe |
| Type           | Asymmetric                                                               |
| Container href | None                                                                     |
| Secret href    | N/A                                                                      |
| Created        | None                                                                     |
| Status         | None                                                                     |
| Error code     | None                                                                     |
| Error message  | None                                                                     |
+----------------+--------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret order get https://10.127.3.110:9311/v1/orders/69d0c5f0-ce37-46fa-b726-b7b18f740dfe
+----------------+------------------------------------------------------------------------------+
| Field          | Value                                                                        |
+----------------+------------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/69d0c5f0-ce37-46fa-b726-b7b18f740dfe     |
| Type           | Asymmetric                                                                   |
| Container href | https://10.127.3.110:9311/v1/containers/a168b740-d729-4cf7-ab24-4fbd2396a012 |
| Secret href    | N/A                                                                          |
| Created        | 2021-05-16T03:21:00+00:00                                                    |
| Status         | ACTIVE                                                                       |
| Error code     | None                                                                         |
| Error message  | None                                                                         |
+----------------+------------------------------------------------------------------------------+
```

创建 Order 后，我们查看具体其信息。会发现：与对称密钥不同的是，**非对称密钥处理时，不是直接创建一个 Secret，而是直接创建了一个类型是 `rsa` 包括了 公钥、私钥、私钥密码 的 Container**：

*P.S. 有关 Container，紧接着就会阐述，这里可以先当一个逻辑组合来理解。*

```
[root@cdpm03 opadmin]# openstack secret container get https://10.127.3.110:9311/v1/containers/a168b740-d729-4cf7-ab24-4fbd2396a012
+----------------+------------------------------------------------------------------------------+
| Field          | Value                                                                        |
+----------------+------------------------------------------------------------------------------+
| Container href | https://10.127.3.110:9311/v1/containers/a168b740-d729-4cf7-ab24-4fbd2396a012 |
| Name           | wj_rsa_key                                                                   |
| Created        | 2021-05-16 03:21:00+00:00                                                    |
| Status         | ACTIVE                                                                       |
| Type           | rsa                                                                          |
| Public Key     | https://10.127.3.110:9311/v1/secrets/4e584384-dbe1-463f-8ad0-aedc64f09ff5    |
| Private Key    | https://10.127.3.110:9311/v1/secrets/08ac5e55-341e-429b-b6f7-8379d98839fb    |
| PK Passphrase  | None                                                                         |
| Consumers      | None                                                                         |
+----------------+------------------------------------------------------------------------------+
```

可以通过 Secret 的具体查询命令，获取每一项的具体内容。如下：

```
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/4e584384-dbe1-463f-8ad0-aedc64f09ff5
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/4e584384-dbe1-463f-8ad0-aedc64f09ff5 |
| Name          | wj_rsa_key                                                                |
| Created       | 2021-05-16T03:21:00+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | rsa                                                                       |
| Bit length    | 2048                                                                      |
| Secret type   | public                                                                    |
| Mode          | None                                                                      |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/4e584384-dbe1-463f-8ad0-aedc64f09ff5 -p
+---------+------------------------------------------------------------------+
| Field   | Value                                                            |
+---------+------------------------------------------------------------------+
| Payload | -----BEGIN PUBLIC KEY-----                                       |
|         | MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyDaiO6ieGPuuJ1V9uw82 |
|         | l3qXMftQmrDjQ1TR8QMS2SwPG+t26rc+K6iPuXwe48PelfDDGkh+qLr4EuOWMyqv |
|         | txYuPWyJ6rPKtkHXQ7d4i66MUPDtBUKk0fVEIWm3WsSNZjmf91VDMdf19hHPMiqq |
|         | oCBAwFGHOLsHBSEEiPAnNYYMlzu5zQ8evM/11fRCzbWe2wTgh56KJxMq2NiW0N4y |
|         | WoAWcmgrm1ZYK/9vlDzK77kbASz4D/V30SLnD3/WT95ShOeUmwrN1YvyZeywC9Wj |
|         | oboK1LYHLxK4xMziM7v6JcJPsJlshba8wq4BI/M/VGU+jqN91basZXXXGM9sGF4O |
|         | xQIDAQAB                                                         |
|         | -----END PUBLIC KEY-----                                         |
|         |                                                                  |
+---------+------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/08ac5e55-341e-429b-b6f7-8379d98839fb
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/08ac5e55-341e-429b-b6f7-8379d98839fb |
| Name          | wj_rsa_key                                                                |
| Created       | 2021-05-16T03:21:00+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | rsa                                                                       |
| Bit length    | 2048                                                                      |
| Secret type   | private                                                                   |
| Mode          | None                                                                      |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/08ac5e55-341e-429b-b6f7-8379d98839fb -p
+---------+------------------------------------------------------------------+
| Field   | Value                                                            |
+---------+------------------------------------------------------------------+
| Payload | -----BEGIN PRIVATE KEY-----                                      |
|         | MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDINqI7qJ4Y+64n |
|         | VX27DzaXepcx+1CasONDVNHxAxLZLA8b63bqtz4rqI+5fB7jw96V8MMaSH6ouvgS |
|         | 45YzKq+3Fi49bInqs8q2QddDt3iLroxQ8O0FQqTR9UQhabdaxI1mOZ/3VUMx1/X2 |
|         | Ec8yKqqgIEDAUYc4uwcFIQSI8Cc1hgyXO7nNDx68z/XV9ELNtZ7bBOCHnoonEyrY |
|         | 2JbQ3jJagBZyaCubVlgr/2+UPMrvuRsBLPgP9XfRIucPf9ZP3lKE55SbCs3Vi/Jl |
|         | 7LAL1aOhugrUtgcvErjEzOIzu/olwk+wmWyFtrzCrgEj8z9UZT6Oo33VtqxlddcY |
|         | z2wYXg7FAgMBAAECggEAYP7G3ew0m5nipz+tp+AY7I4Bjb9ZL3gewdHn28FHclr7 |
|         | /uS2OcQIpJIG/y94r5OG1FFN0//nDMt3v37ul19IvYRLZoqczk3IGUAQj8fk6Jbp |
|         | d5Ug3vmIbAdMuHtEzv6GGk40h1iRMyaTDGFYZc9x1h2KASH+RqelIQD793ORK0Yo |
|         | gpaj8l7FTITsZPpyRNjW4xrUXdObTpL8ErgbgfWz93FKE415SHdIteQWQ+zlCJC/ |
|         | ax2TuQSFdZko0inJQx92cQun7nwdwJvdXHRaaZMwAl41xx/aD0fABprspqZw/KbP |
|         | SVhQJN7wuzmWW2x64AsG2P4T52S8CstV+0esrbb6gQKBgQDpJgtpY3imAK0y20Ab |
|         | sNHfdMVfD8+XWinWExbaq03NQxZKUA318EQnheMbRNXsqf/hXM9vHAUEBB7ter1/ |
|         | FEMEiKO4l6a/r5hgBo3UJbVzZeOYlOK75rtQWsPrxFD27caE0+p1Q4l18ZEQmqdG |
|         | 8BB/QlpDEF5aDywH4j/kgVossQKBgQDb1jVwF4RRAZavGMWbb237UvKSq0yDKKZa |
|         | zlWXnu8/nx6S1O9mh8yITNzv1D0Dercc39iaCqin1qNDlrSbJe1wElc8J70JWl0u |
|         | W0sAOznMwZGWT/hwVLpMSaFX0s9NkRjuvbmMclH/quyUxXTAH8fAJwsviQt4QMgB |
|         | T8Syha64VQKBgDyw7p+MiUeNPYjTkiijKr7kgsxwLTXU/rb/WR+rICGiqRbHKBsx |
|         | ZEx1idz7WkS1LCraIhVmUdftyq8/GD0QZTG08AmJUJrtdtjoW9sxxb44c7qwZyVK |
|         | ttAAEKg6/miJFPhWwd2sqwfMzlpoJ8tLir/V4fE7PZRsBqY2uzMciQDBAoGAQp1n |
|         | df76Tl2v3oEgKBic+CJLdRxJRBlGR4/sqdQ0ZU//QLkbjjMqTEcWT+o9TteZszs1 |
|         | dIA0WR+WO33oXncguuwj2QulobbrM4fgc0J/IkepqSW0f7188m8BYA52WOfV6Uo+ |
|         | douRw2p05CPtW+aFbfmmzxG1Ewx2TsdwMDSIHD0CgYEAyCBflv3UFvsUVvy+k+fj |
|         | NndiA49TPSMrYYBYVk7NePP1nsqQP8BT3oa+Ur3JLHLLkD8lnLSykc7pp0nt29pZ |
|         | gmUNiJlHX+xn6P8I/kkSnwwtIiBlvf5e6O/MJ+VNAlUlDAvwhGJrnB1QVHpNBQCK |
|         | B/oF41EOKj04MvHKcH0glAY=                                         |
|         | -----END PRIVATE KEY-----                                        |
|         |                                                                  |
+---------+------------------------------------------------------------------+
```

是不是特别方便？就不需要我们手动创建后再倒入了。

总结下，即：**通过 Order 方法申请非对称密钥，Barbican 会直接创建一个 `RSA` 类型的 Container**。

<br/>

#### 证书

说到这里了，就把证书也提一句吧。

虽然 Barbican 不再支持通过 Order 来申请创建证书，但用户仍然可以用手动的方式，通过 Secret 手动导入证书后，再创建一个 `certificate` 类型的 Container 来使用。

<br/>

#### 其他相关问题

##### 删除 Order 时，之前创建的 Secret 会连带删除么？

不会。删除 Order，相当于是把之前的请求任务给清除了；之前已创建了的 Secret 仍会被保留。

下面是个具体例子。首先，我们申请创建一个对称密钥，参照执行结果，对应 Order 和 Secret 均已成功创建：

```
[root@cdpm03 opadmin]# openstack secret order create -n wj_aes_test -a aes -b 256 -t application/octet-stream key
+----------------+--------------------------------------------------------------------------+
| Field          | Value                                                                    |
+----------------+--------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/20e805ed-ea64-42a4-b678-b826536bbae1 |
| Type           | Key                                                                      |
| Container href | N/A                                                                      |
| Secret href    | None                                                                     |
| Created        | None                                                                     |
| Status         | None                                                                     |
| Error code     | None                                                                     |
| Error message  | None                                                                     |
+----------------+--------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret order get https://10.127.3.110:9311/v1/orders/20e805ed-ea64-42a4-b678-b826536bbae1
+----------------+---------------------------------------------------------------------------+
| Field          | Value                                                                     |
+----------------+---------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/20e805ed-ea64-42a4-b678-b826536bbae1  |
| Type           | Key                                                                       |
| Container href | N/A                                                                       |
| Secret href    | https://10.127.3.110:9311/v1/secrets/50f501c3-ee61-4250-bb7d-8a72a9cb3d3c |
| Created        | 2021-05-16T13:51:25+00:00                                                 |
| Status         | ACTIVE                                                                    |
| Error code     | None                                                                      |
| Error message  | None                                                                      |
+----------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/50f501c3-ee61-4250-bb7d-8a72a9cb3d3c
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/50f501c3-ee61-4250-bb7d-8a72a9cb3d3c |
| Name          | wj_aes_test                                                               |
| Created       | 2021-05-16T13:51:25+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | symmetric                                                                 |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
```

之后，尝试删除该 Order。最后，再确认 Secret 具体状态。可见，Order 的释放，并不会关联删除对应的 Secret。

```
[root@cdpm03 opadmin]# openstack secret order delete https://10.127.3.110:9311/v1/orders/20e805ed-ea64-42a4-b678-b826536bbae1
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret order get https://10.127.3.110:9311/v1/orders/20e805ed-ea64-42a4-b678-b826536bbae1
4xx Client error: Not Found: Not Found. Sorry but your order is in another castle.
Not Found: Not Found. Sorry but your order is in another castle.
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/50f501c3-ee61-4250-bb7d-8a72a9cb3d3c
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/50f501c3-ee61-4250-bb7d-8a72a9cb3d3c |
| Name          | wj_aes_test                                                               |
| Created       | 2021-05-16T13:51:25+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | symmetric                                                                 |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
```

##### 能否直接删除之前 Order 创建出来的 Secret？会影响到该 Order 么？

同样不会。理由同上，Order 执行完后，就已经达成使命了。后续 Secret 等对象的后续使用，完全由该对象呈现。因此，是否删除 Secret，都不影响到之前 Order；Order 中记录的 Secret 信息，也不会随 Secret 的删除而被释放。

附具体例子，同上例，就不过多解释了：

```
[root@cdpm03 opadmin]# openstack secret order create -n wj_aes_test_2 -a aes -b 256 -t application/octet-stream key
+----------------+--------------------------------------------------------------------------+
| Field          | Value                                                                    |
+----------------+--------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/388f3647-1c16-4df3-af0b-96224fb4151c |
| Type           | Key                                                                      |
| Container href | N/A                                                                      |
| Secret href    | None                                                                     |
| Created        | None                                                                     |
| Status         | None                                                                     |
| Error code     | None                                                                     |
| Error message  | None                                                                     |
+----------------+--------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret order get https://10.127.3.110:9311/v1/orders/388f3647-1c16-4df3-af0b-96224fb4151c

+----------------+---------------------------------------------------------------------------+
| Field          | Value                                                                     |
+----------------+---------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/388f3647-1c16-4df3-af0b-96224fb4151c  |
| Type           | Key                                                                       |
| Container href | N/A                                                                       |
| Secret href    | https://10.127.3.110:9311/v1/secrets/6cd12513-5919-46e3-b90c-3f8ced596032 |
| Created        | 2021-05-16T13:54:14+00:00                                                 |
| Status         | ACTIVE                                                                    |
| Error code     | None                                                                      |
| Error message  | None                                                                      |
+----------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/6cd12513-5919-46e3-b90c-3f8ced596032
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/6cd12513-5919-46e3-b90c-3f8ced596032 |
| Name          | wj_aes_test_2                                                             |
| Created       | 2021-05-16T13:54:14+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | symmetric                                                                 |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret delete https://10.127.3.110:9311/v1/secrets/6cd12513-5919-46e3-b90c-3f8ced596032
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/6cd12513-5919-46e3-b90c-3f8ced596032
4xx Client error: Not Found: Not Found. Sorry but your secret is in another castle.
Not Found: Not Found. Sorry but your secret is in another castle.
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret order get https://10.127.3.110:9311/v1/orders/388f3647-1c16-4df3-af0b-96224fb4151c
+----------------+---------------------------------------------------------------------------+
| Field          | Value                                                                     |
+----------------+---------------------------------------------------------------------------+
| Order href     | https://10.127.3.110:9311/v1/orders/388f3647-1c16-4df3-af0b-96224fb4151c  |
| Type           | Key                                                                       |
| Container href | N/A                                                                       |
| Secret href    | https://10.127.3.110:9311/v1/secrets/6cd12513-5919-46e3-b90c-3f8ced596032 |
| Created        | 2021-05-16T13:54:14+00:00                                                 |
| Status         | ACTIVE                                                                    |
| Error code     | None                                                                      |
| Error message  | None                                                                      |
+----------------+---------------------------------------------------------------------------+
```

<br/>

### 5. Container

在 Order 实例演示中，就已出现了的概念。

证书容器，类似证书存储的一个独立逻辑分区，拿对象存储的概念来对比，比较好理解：Secret 相当于 Object，Secret Container 相当于一个 Bucket（*同 Swift 中的 Container 概念*）。

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
  - `private_key_passphrase` - 私钥密码（*#TBD：此处存疑。因为使用Order方式创建出来的该值为 None*）

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

### 6. Consumer

一句话概括：Consumer 是 Container 的关注者，在 Container 删除前会得到通知。即，所有 Consumer 是注册在某个 Container 上的，可以指定 Container ID 来查询其所有关注者，支持分页查询。

删除时，需要在消息体内，注明待解关联的 Consumer 的 `name`、`URL`，[官网](https://docs.openstack.org/api-guide/key-manager/consumers.html)有具体例子。

*注：简单测试了下，用户参数似乎并不校验。加上本身非重点，考虑篇幅问题，这里不再展开描述了。*

<br/>

### 7. ACL

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

### 8. CA

CA（*Certificate Manager*）用来签发、认证、管理证书。在 Barbican 中用的较简单，有些功能已废弃，这里简单提一下。

与其他功能一样，Barbican 也是通过后端插件的方式来提供证书管理。其中，Dogtag 用的较多，它是 Red hat 证书系统的开源上游版本。Barbican 通过对应插件与 Dogtag 进行集成/交互：

- Dogtag CA 子系统：用于签发、续订、撤销不同类型证书。可用作 Barbican 的 CA 后端，并通过 Dogtag CA 插件来交互
- Dogtag KRA（*Key Recovery Authority*） 子系统：用于通过存储在 软件NSS DB/HSM 中的主加密密钥 来加密存储 Secrets。可用作 Barbican 的 Secret Store，并通过 Dogtag KRA 插件来交互

官方提供了具体的[配置指导](https://docs.openstack.org/api-guide/key-manager/dogtag_setup.html)，这里不展开。

<br/>

### 9. Quotas

老面孔了，每个 OpenStack 服务中都有他的身影，用于限定单个租户资源使用量，支持 Secrets、Containers、Orders、Consumers、CAs，默认不限制。

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

## 小结

好了，Barbican 中的基本对象都已经阐述过了，把主要对象放在一起，就能得到各对象关系图了。用字符图绘制如下：

```
                       +---------------+      +---------------+
                       |   Consumer    |      |     Quotas    |
                       +-------+-------+      +---------------+
                               ^ N
                               |
                               |
                               | 1
                       +-------+-------+
                     1 |               | 1
        +--------------+   Container   +<-------------+
        |              |               |              |
      1 v              +-------+-------+              | 1
+-------+-------+              | 1            +-------+-------+
|      ACL      |              |              |     Order     |
+-------+-------+              |              +-------+-------+
      1 ^                      v 1                    | 1
        |            1 +-------+-------+ 1            |
        +--------------+               +<-------------+
                       |     Secret    |
        +--------------+               +--------------+
        |            1 +-------+-------+ 1            |
        |                    1 |                      |
        |                      |                      |
        |                      |                      |
      N |                    1 |                      | 1
+-------v-------+      +-------v-------+      +-------v-------+
|    Metadata   |      |     Store     |      |      Type     |
+---------------+      +---------------+      +---------------+
```

由上图可见，Barbican 从 Secret 这一基本概念出发，逐步延展出其他周边对象。

至此，再逐步熟悉了各对象用法，理清了各对象关系的基础下，接下来，我会开始尝试剖析有关 Barbican 的实现原理，帮助大家更好的理解该组件。

<br/>
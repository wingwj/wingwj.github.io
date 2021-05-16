# Barbican 详解（一）
Author: WingWJ

Date: 10th, Mar, 2021

Updated at: 16th, May, 2021

<br/>

## 前言

最近工作需要，想评估下 Barbican 项目能力，但一直没找到很全面的介绍；官网文档也依旧是一言难尽。。所以还是自己写份笔记吧。

<br/>

## 1. Barbican 是干什么的？
简言之，用来管理 OpenStack 平台中各种 Secret 的。而 Secret，指的是我们不想被他人获取而需安全保存的内容，比如：
* **Private Key**：私钥
* **Certificate**：证书，如 X.509
* **Password**：密码，比如一段文本
* **SSH Keys**：SSH 密钥

这些，后续会对应到 Barbican 中具体的参数 `secret_type`，放后面说。

<br/>

## 2. 还是有点抽象，有例子么？
有，用CLI 先来感受下。

<br/>

### 2.1 如何创建一个最简单的 Secret？
比如，最简单的，保存一段密文`wj_pass=abc123`，方式如下：

```
[root@cdpm03 opadmin]# openstack secret store --payload wj_pass=123
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/484f4e88-6354-4da7-992d-180777bdc8aa |
| Name          | None                                                                      |
| Created       | 2021-05-15T13:32:26+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'text/plain'}                                               |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
```

可见，我们除了密文内容外，其余参数均未指定。由上面结果可以看出来，Barbican 默认识别密文内容 `wj_pass=abc123` 为文本格式 `text/plain`，再采用 AES-256 算法，以`opaque`格式将其保存到 Barbican 中，且永不过期。

P.S. `payload`字段其实并未限制你必须使用 `key=value`的形式，把它就当个字符串形式，想存啥写啥就好。比如，通过额外指定`-name`参数来给 Secret 添加名字来区分，只存`payload=abc123`也没问题——不过，有时我们可能连提示项都不想被人看到呢，这样直接都写在`payload`里是更合适的——这完全取决于自身的用法。

<br/>

### 2.2 那我如何查看我存入的原始内容呢？
同样的，如果你通过了用户鉴权，就能通过 API 来获取。
我们来查询下上文中创建的 Secret。直接通过调用GET方法，并且显式指定`--payload`参数（简写为`-p`），就能直接查询了：

```
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/484f4e88-6354-4da7-992d-180777bdc8aa -p
+---------+-------------+
| Field   | Value       |
+---------+-------------+
| Payload | wj_pass=123 |
+---------+-------------+
```

注意，如果不指定`--payload`，则查询出来的结果，会和创建时返回的结构体一致，默认隐去 `payload`字段（因为CLI 的封装）。如下：

```
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/484f4e88-6354-4da7-992d-180777bdc8aa
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/484f4e88-6354-4da7-992d-180777bdc8aa |
| Name          | None                                                                      |
| Created       | 2021-05-15T13:32:26+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'text/plain'}                                               |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
```

<br/>

### 2.3 Secret 能重名么？

当然可以，唯一键为UUID。如下：

```
[root@cdpm03 opadmin]# openstack secret store --name wj_sec
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/7f99592c-8477-4989-a549-5b14a16906ba |
| Name          | wj_sec                                                                    |
| Created       | None                                                                      |
| Status        | None                                                                      |
| Content types | None                                                                      |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret store --name wj_sec
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/00efe51e-cae0-431f-ba67-2df127688728 |
| Name          | wj_sec                                                                    |
| Created       | None                                                                      |
| Status        | None                                                                      |
| Content types | None                                                                      |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret list -n wj_sec
+------------------------+--------+------------------------+--------+---------------+-----------+------------+-------------+------+------------+
| Secret href            | Name   | Created                | Status | Content types | Algorithm | Bit length | Secret type | Mode | Expiration |
+------------------------+--------+------------------------+--------+---------------+-----------+------------+-------------+------+------------+
| https://10.127.3.110:9 | wj_sec | 2021-05-15T13:36:14+00 | ACTIVE | None          | aes       |        256 | opaque      | cbc  | None       |
| 311/v1/secrets         |        | :00                    |        |               |           |            |             |      |            |
| /00efe51e-cae0-431f-   |        |                        |        |               |           |            |             |      |            |
| ba67-2df127688728      |        |                        |        |               |           |            |             |      |            |
| https://10.127.3.110:9 | wj_sec | 2021-05-15T13:36:03+00 | ACTIVE | None          | aes       |        256 | opaque      | cbc  | None       |
| 311/v1/secrets/7f99592 |        | :00                    |        |               |           |            |             |      |            |
| c-8477-4989-a549-5b14a |        |                        |        |               |           |            |             |      |            |
| 16906ba                |        |                        |        |               |           |            |             |      |            |
+------------------------+--------+------------------------+--------+---------------+-----------+------------+-------------+------+------------+
[root@cdpm03 opadmin]#
```

<br/>

## 3. OK，存密码这种我明白了。那其他类型 Secret 怎么存呢？
那我们来看下，用的最多的密钥方式吧，我们计划将一公钥倒入到 Barbican 中。
先使用`openssl`命令，创建一对 `RSA` 公私钥，再使用 base64 进行编码，记录编码结果：

```
[root@cdpm03 opadmin]# openssl genrsa -out private.pem 2048
Generating RSA private key, 2048 bit long modulus
.................................+++
........................................................................................................................................................................................................................+++
e is 65537 (0x10001)
[root@cdpm03 opadmin]# openssl rsa -in private.pem -out public.pem -pubout
writing RSA key
[root@cdpm03 opadmin]# wj_pub_base64=$(base64 < public.pem)
[root@cdpm03 opadmin]# echo "$wj_pub_base64"
LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FR
OEFNSUlCQ2dLQ0FRRUF1c2t5d2xTSjNTdWVUZHpiWG9aSQprU2VadnQ1Rmh1Zms0bzJYbjBLQ2Vh
WHZLV29IRWErUThaMm90VVVQVzRPaEVGeHB0R04wcFNaL3c3S1UzYWhZCmVya3pGSzN5SFlIQTNh
dS9rT09zdDhPamNpMUhyL00zUisvTmJlSVpsUTJJUi8vcjJlUGQ4djl1R04rMXcwS20KOVR4MnpR
WmxMaGdTTU1FdWo3dEx2Z2F0cVloN29QNTMxbUVkL2hwRHpBNzd2YWpDcVVBN1Vha09PMy83T1Jt
MwppUTAzcW0rRllwY1VkM2xDMWpwSjEzZnVkRFh3R0NzN3BMa0FnTFUvalE4ZjJaRWp3M0k1NDY4
a3h5UEc1WEJsCmx0YnB5bHAzUVRjYmMvWHJUa0xZSW9rK3FQSExrcG1MR3Y0RFdpKzk4RzFBNGFt
ZFZKN1pCaFkwZm5sQlJuZUMKUndJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tCg==
```

这里我们同样使用之前的命令来将证书存储到 Barbican 中，这次，我们需要相应的指定多个参数，与创建公私钥时的命令保持一致：

```
[root@cdpm03 opadmin]# openstack secret store -n wj_public_key -s public -a rsa -b 2048 -p "$wj_pub_base64" -t application/octet-stream -e base64
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/c1c7af28-e502-46b4-ac9e-a7bcb835ba28 |
| Name          | wj_public_key                                                             |
| Created       | None                                                                      |
| Status        | None                                                                      |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | rsa                                                                       |
| Bit length    | 2048                                                                      |
| Secret type   | public                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/c1c7af28-e502-46b4-ac9e-a7bcb835ba28
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/c1c7af28-e502-46b4-ac9e-a7bcb835ba28 |
| Name          | wj_public_key                                                             |
| Created       | 2021-05-16T03:47:06+00:00                                                 |
| Status        | ACTIVE                                                                    |
| Content types | {u'default': u'application/octet-stream'}                                 |
| Algorithm     | rsa                                                                       |
| Bit length    | 2048                                                                      |
| Secret type   | public                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/c1c7af28-e502-46b4-ac9e-a7bcb835ba28 -p
+---------+------------------------------------------------------------------+
| Field   | Value                                                            |
+---------+------------------------------------------------------------------+
| Payload | -----BEGIN PUBLIC KEY-----                                       |
|         | MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuskywlSJ3SueTdzbXoZI |
|         | kSeZvt5Fhufk4o2Xn0KCeaXvKWoHEa+Q8Z2otUUPW4OhEFxptGN0pSZ/w7KU3ahY |
|         | erkzFK3yHYHA3au/kOOst8Ojci1Hr/M3R+/NbeIZlQ2IR//r2ePd8v9uGN+1w0Km |
|         | 9Tx2zQZlLhgSMMEuj7tLvgatqYh7oP531mEd/hpDzA77vajCqUA7UakOO3/7ORm3 |
|         | iQ03qm+FYpcUd3lC1jpJ13fudDXwGCs7pLkAgLU/jQ8f2ZEjw3I5468kxyPG5XBl |
|         | ltbpylp3QTcbc/XrTkLYIok+qPHLkpmLGv4DWi+98G1A4amdVJ7ZBhY0fnlBRneC |
|         | RwIDAQAB                                                         |
|         | -----END PUBLIC KEY-----                                         |
|         |                                                                  |
+---------+------------------------------------------------------------------+
```

可见，在创建完后使用再查询，能够明确看到Barbican返回结果，是具备PEM header/footer 的 X.509 格式公钥。
和原始 `public.pem` 进行比对，确实是一致的：

```
[root@cdpm03 opadmin]# cat public.pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuskywlSJ3SueTdzbXoZI
kSeZvt5Fhufk4o2Xn0KCeaXvKWoHEa+Q8Z2otUUPW4OhEFxptGN0pSZ/w7KU3ahY
erkzFK3yHYHA3au/kOOst8Ojci1Hr/M3R+/NbeIZlQ2IR//r2ePd8v9uGN+1w0Km
9Tx2zQZlLhgSMMEuj7tLvgatqYh7oP531mEd/hpDzA77vajCqUA7UakOO3/7ORm3
iQ03qm+FYpcUd3lC1jpJ13fudDXwGCs7pLkAgLU/jQ8f2ZEjw3I5468kxyPG5XBl
ltbpylp3QTcbc/XrTkLYIok+qPHLkpmLGv4DWi+98G1A4amdVJ7ZBhY0fnlBRneC
RwIDAQAB
-----END PUBLIC KEY-----
```

由上文可以看到，我们指定了该 Secret 的名称，并且对应证书生成时的命令，明确该 Secret 由 RSA 算法生成且密钥长度为2048，使用了base64 方式编码，以原始二进制方式为密文内容，存储到了 Barbican 中。

<br/>

### 3.1 上面例子中，如果不按证书生成时对应的参数来填写，而是随便指定呢？
我们还是用例子来说话。
还是上文生成的已经 base64 编码过的公钥，我按文本方式`text/plain` 在存入 Barbican 时将算法改为 AES-256，还不指定 base64，能存储成功么？——答案是，能。

```
[root@cdpm03 opadmin]# openstack secret store -n wj_public_key_2 -s public -a aes -b 256 -p "$wj_pub_base64" -t text/plain
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | https://10.127.3.110:9311/v1/secrets/9c73ccb3-e31b-4df9-8b8b-c30d4701b86d |
| Name          | wj_public_key_2                                                           |
| Created       | None                                                                      |
| Status        | None                                                                      |
| Content types | {u'default': u'text/plain'}                                               |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | public                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
[root@cdpm03 opadmin]#
[root@cdpm03 opadmin]# openstack secret get https://10.127.3.110:9311/v1/secrets/9c73ccb3-e31b-4df9-8b8b-c30d4701b86d -p
+---------+------------------------------------------------------------------------------+
| Field   | Value                                                                        |
+---------+------------------------------------------------------------------------------+
| Payload | LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FR |
|         | OEFNSUlCQ2dLQ0FRRUF1c2t5d2xTSjNTdWVUZHpiWG9aSQprU2VadnQ1Rmh1Zms0bzJYbjBLQ2Vh |
|         | WHZLV29IRWErUThaMm90VVVQVzRPaEVGeHB0R04wcFNaL3c3S1UzYWhZCmVya3pGSzN5SFlIQTNh |
|         | dS9rT09zdDhPamNpMUhyL00zUisvTmJlSVpsUTJJUi8vcjJlUGQ4djl1R04rMXcwS20KOVR4MnpR |
|         | WmxMaGdTTU1FdWo3dEx2Z2F0cVloN29QNTMxbUVkL2hwRHpBNzd2YWpDcVVBN1Vha09PMy83T1Jt |
|         | MwppUTAzcW0rRllwY1VkM2xDMWpwSjEzZnVkRFh3R0NzN3BMa0FnTFUvalE4ZjJaRWp3M0k1NDY4 |
|         | a3h5UEc1WEJsCmx0YnB5bHAzUVRjYmMvWHJUa0xZSW9rK3FQSExrcG1MR3Y0RFdpKzk4RzFBNGFt |
|         | ZFZKN1pCaFkwZm5sQlJuZUMKUndJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tCg==     |
+---------+------------------------------------------------------------------------------+
```

可见，查询出来的结果就是已经过 base64 编码后的原文，或者换句话说，是原始`$pub_base64`参数中存储的内容。即，这里直接将编码后内容作为文本，直接存储到了 Barbican 中。

<u>题外话</u>：即使将类型通过 `-s symmetric` 调整成对称密钥类型，导入仍能成功，结果和上例一致。即，Barbican 对于导入类的 Secret，并不会内容本身的格式。

<br/>

### 3.2 如果此时，手动再 base64 解码，能拿到原始公钥么？
当然可以。因为从 Barbican 中获取到的结果，即原来的编码后内容，那么使用同样的方法解码自然能够获取，可直接使用 `base64 -d` 命令来验证。

<br/>

### 3.3 对了，那个`payload-content-type`参数是干什么的？好像很复杂？

确实，该参数是用来指定传入的`payload`的类型的。在文章中一共使用过两个值，分别是：
* `text/plain`：普通文本；
* `application/octet-stream`：原始二进制数据。

看上去似乎很复杂，为何要这样定义呢？——这其实是和RFC标准中定义的 `MIME`（`Multipurpose Internet Mail Extensions`，多用途互联网邮件扩展类型）一一对应，用来描述消息内容类型的因特网标准。
比如，`application_json`、`image_jpeg`等，以及在Web场景下常见的`Content-Type: text/HTML`，其实就是一种具体的 MIME 类型。它定义了数据的类型，以便数据能被适当的处理。

在 Barbican 中使用时，推荐按 Secret 原始内容形式提交即可。

<br/>

### 3.4 看上去，似乎 `algorithm`、`bit-length` 参数没用啊？
其实，导入时确实没用。。包括 `mode`，这三个参数都是用来标识的，API文档中的字段描述中写的就是 `(optional) Metadata provided by a user or system for informational purposes`。所以，你看上例中，不管填 `RSA`，或者 `AES`，其实都没有影响 Secret 的记录形式。

但反过来考虑下，如果真是无用参数，社区会专门做个可选参数要求填入么？直接放在常用的 `Metadata`或者`Description` 中不还省事呢。其实，**这些字段是有实际意义的**，后续在学到 `Order` 对象时就会看到了。

<br/>

### 3.5 从上面的例子中，我们学到了什么？

Barbican 本身并不会校验 Secret 内容本身与填入算法是否匹配（如上例中的 `RSA` 改为 `AES` ），在入库时只是按照传入参数来存储，而在查询时按照之前记录的方式来响应（如编解码）。

考虑正确性与便利性，在存储 Secret 时应该指定正确的参数。

<br/>

## 4. 看明白了，存证书这里还是挺有用的。回头看，在Barbican中单独保存密码，又有多大用呢？
上文举例中，我们只简单存了个无意义的字符密文 `wj_pass=abc123`，看着好像用处不大。但如果，我们记录的是有意义的密文呢？
比如，还以上文RSA私钥来举例。之前我们生成时使用的命令中，由于并未指定加密方式，因此生成的私钥是明文的。

如果在使用`openssl` 创建密钥时，指定`-aes256`参数，就会将私钥进行加密，此时就会让你输入私钥密码，比如我们还用 `wjtest`，最后就会生成一个经过 AES-256 加密后的私钥：

```
[root@cdpm03 opadmin]# openssl genrsa -out private2.pem -aes256 2048
Generating RSA private key, 2048 bit long modulus
..................................................................................+++
......................................+++
e is 65537 (0x10001)
Enter pass phrase for private2.pem:
Verifying - Enter pass phrase for private2.pem:
[root@cdpm03 opadmin]#
```

重新查看私钥，可见明确标明了“已加密“（*ENCRYPTED*），形式也与之前明文创建的不一样了：

```
[root@cdpm03 opadmin]# cat private2.pem
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-256-CBC,B41E76678CF7EE2BE8B758F149AEEB37

YJ1u6Qky49YHZttGqliKlZcknZorKKIy9IEzmWlaGGIF0DGXOK+PJlmgws5Pkx8v
lU6/IPx2CIv92gJTLfLbcY4K76LDqNPSTATW5inUTIEbR6gA02U9Gi4dGSE25UuG
rzrcsw7+l3RPzkg38be/2csDt/l4hcZGx+HUd1nWvE5zFgCrbQcJieT95zv7e8/y
FiAkQFT1/mBeVn/YXdsdvMwd94GGMVBaz4foo0Jn4gj+3nHS+yXV3ALt8MfhDycQ
VYQ6m9SgdD2zk9s6qSSbK8Qvnhyb+x0qZIPzhxGM1JIWHA9abb6wwH3ViRGf9fKx
YQqfIGnt6AFdGg9CeaHJPv5E+T9TUGvcwZFoz9mdkotPzOytBPIv8Jrxi7LdqS8F
VFHlQiIJrzFmMXe1h2oI5TTIaalEOS21OJV425xDyA7qqnM6ySMg84OeL0Za4Yqc
C1n//YdK5khlcwzTaTiJwfWH+Ckkg37Ek8qoNeF/xQWcj176I+yxvT7Al+RROBEL
xX0Z4BzUKieU59qkF1cngtmKAPEM6DP3dMU5FlRPhAqBExWiUJaJXnxbs8An7c4u
Gk7WYou5rbo7df0SZr8ki1iLuNzYIdY5FMaU25qWMJzmtEvoTNWhOGGwPvgprMyD
MEZWSnsEyKUs+FA47/RLsk2Ie9uRaD35ngU0Ce2pNZkWUHfwAJBfLtN543N6PoE1
PVy0U5r0uocqeFxFIAA2cfgAc79+w0E8mtKFxVRBEmgOiEbYkeiSpNHJBti4RbDs
H0HOXvWmYnsnIcYFRB9DwST5cUvM5plZ7ZB1qh5ext6IS5esT7l8GBW0aaxW7qYo
s0TsvOxdJC+9TSaw82ir/H6U8RvLnG/0wM5FCFhjix3gkzBfodvZPvg4fpH1tvHN
q+/djm8yWgQirrHADHL2ub/jZ+TPfrfvuY/9w0nSspfxMSFcgdalQymU5brIyFxx
w1rWaSXfsVL6Z528Q2mf8b9Klt6qCcuDzPJZ3IuIp9LnOiJWoGtQZjPj8vW11tQM
iPfIL/cKvqAtMuBjbn8vypp+CwId9E4Gt5p3wwGYy0XI9t7l4CBupgtBhcjRBdcf
6uhdcxAnUjLzv397Oi5ky31eAwcEfMRNvKhwWA+GgiL6FleYITUFmB+uSIw0OCvy
RnAZNznaijl4hd+dka7CncXkgYP/fUUdL2Keoo4xmrdrEQ/PB5fRDQURiNmufi1C
fS0J7fHLb9u1bLGtnV9B/coXjaodXZjKyft3pOCrWIEvoIp3lhhbQXdxusy/e0js
X1rFycKgZKKwaSfJr8GH9wCE9PPXaOBGED1Ubegtq8fMW7VzBpcbNUOZi4tvKVBk
ggswmqVh2/WwH9LA6FlMWhsIOKQDX0LLCvHdgusdreRta0QzYkrtJYcAk+3m85M8
aQzCSKB8jREhbDp5NeOzYsAVG/lQxqQZTbJ01jbQ3m9Lqr4uoeS0ReOsHhUq1h9l
qxJ4nviOMBUXkWJw6f5BBd+RAH1Qi9a8Iu1qNsIAYSP21DYir/01DzuPuoGOGI+Z
WkWrKYHddeUyx0EaD1ijomWlT6l5IXO4WVEaXZd0gqFLMArNPQMoIOrlYSnygcyT
-----END RSA PRIVATE KEY-----
```

而刚生成时键入的密码 `wjtest`，我们就得自己记录下来。这时，就可以通过 Barbican 来记录。这时，相当于把一套非对称密钥的私钥、公钥、及私钥密码，全都存储到 Barbican 中了——这一点自然是允许的，Barbican 本并不限定用法。只是把鸡蛋都放到一个篮子里，是否安全，就是要考虑的另一个问题了。。

当然，你也可以通过 Barbican，来记录个人生日、爱好、偶尔的灵感、甚至写日记。。完全看你的用法了 ：）～

<br/>

## 5. 信息在 DB 中是密文存储的么？

那是必须的。

查看下 DB `encrypted_data` 表结构，就能很容易的理解 Barbican 的转换逻辑：

```
MariaDB [barbican]> desc encrypted_data;
+-------------------+--------------+------+-----+---------+-------+
| Field             | Type         | Null | Key | Default | Extra |
+-------------------+--------------+------+-----+---------+-------+
| id                | varchar(36)  | NO   | PRI | NULL    |       |
| created_at        | datetime     | NO   |     | NULL    |       |
| updated_at        | datetime     | NO   |     | NULL    |       |
| deleted_at        | datetime     | YES  |     | NULL    |       |
| deleted           | tinyint(1)   | NO   |     | NULL    |       |
| status            | varchar(20)  | NO   |     | NULL    |       |
| content_type      | varchar(255) | YES  |     | NULL    |       |
| secret_id         | varchar(36)  | NO   | MUL | NULL    |       |
| kek_id            | varchar(36)  | NO   | MUL | NULL    |       |
| cypher_text       | text         | YES  |     | NULL    |       |
| kek_meta_extended | text         | YES  |     | NULL    |       |
+-------------------+--------------+------+-----+---------+-------+
11 rows in set (0.001 sec)
```

其中最关键的是以下几个字段：

- `kek_id` - 主密钥 id：用来对各 Secret 加密入库的总密钥（*后面讲原理时会解释，暂略*）
- `secret_id` - Secret id：代表各 Secret，用于表关联
- `cypher_text` 密文：各 Secret 经过加密后存储在 DB 中的密文信息

各字段大致如下，贴出来也给大家一个直观的认识：

```
| id                                   | created_at          | updated_at          | deleted_at          | deleted | status | content_type             | secret_id                            | kek_id                               | cypher_text                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | kek_meta_extended |

	...

| cfd80a89-9370-41f6-9057-0aecb8127bf1 | 2021-05-16 02:17:13 | 2021-05-16 02:17:13 | NULL                |       0 | ACTIVE | application/octet-stream | 23d2d42f-66fd-4a95-9b57-7e9dc6e30a40 | 371cb2a5-b588-4060-a57b-5d77983a8670 | Z0FBQUFBQmdvSUNwb21jNWVXR1VKbG1zQ1JQQlB5WF9aU1FqeldCQmpjQU5HdWs3VmpFMW9jU1ZieHBaTkRQR2tnNFhnVlNweDIxMjdKelNFZU5xcG40ZlFrY2xEX01mZ1IyME92WVIzUzFqMVo4Z0tNYzdqNnpXZ2dXRWFhREo3YjFXU3k5VVgzYUI=                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | NULL              |
| e0a74ad9-c543-46b7-952a-cb67d8153079 | 2021-05-16 03:51:18 | 2021-05-16 03:51:18 | NULL                |       0 | ACTIVE | text/plain               | 9c73ccb3-e31b-4df9-8b8b-c30d4701b86d | 371cb2a5-b588-4060-a57b-5d77983a8670 | Z0FBQUFBQmdvSmEyekVQVFFaUWJsZXZPbThGMnEwcV9lOE9LQnBxX2JOOWJGZHJrNVlCcVF2Q1pDYnRBRUljNlJXMzc4bUc2VzY4d2ozRU5WVTVJbkhYVkZhbndQckNaZERzU1dwS0JHN1h4a1JWLTQzUHIwZUsyb0hya09zYXB2aEM2NWlfZUNKemtaQVFQSHVVbncwbm11ajZ0c3ZPc0hnQTJVdGlfOUlwM29EZlFiSHlUQXhUQlhKWEtmMHByODBzZFlOU3dMY0RUSVllYzBfUFIwVkJhYkNjNGdrdzFaQndqZmhnek5ndDVtV0hDb1QzbG8xMlZ4T3JmNGUycFEwaDZDS2JpQmJSUlZkb01PWHluVGxmN1NibUI4TWl5aFhzS3I0MXFKZnhKcHVqVG5JWGp4Q1JaaFZlbzBYekU0MWc2Vk9yOTBtT2VRNmNEVDhiOExJUXRaeHR4OTE3SjB0Mm01OHdpN3FmNnVaTTZmNEd4bUhfQ296bW5FMkpraWxBOGg4U3lGTHJadVMySzBJdHk4bUpQMnhLT2c3WXpwRXAxT1FVS2VJWDczVXlUb3dLTy1Id1ZtcVV6MTVhNFl5U3NldFBScDFIN19tMjlhSXNYQUFhSE5ONU4wNDFBUUprN2JicDRPUGZKampGR2NCUlkyRVYyQkMxeVdaSFR6MzU1MkVJUTBMeC0zeGpwZEg5dURuN0E2ZklVb1NndlBlUlJDRzI4X05ZcnBvMUxIZVR0dHJreEJDNVNjZGNaVGJoUkh3eHFSekpsVFdkYlQ4ck03cnJCSkJiSjZhcm5RTGdkcFYtSEJ1MzU2c1ptVUVCdXhuUWtmbUxiVmN4ekFPMFVac3A5NlBaVVdydnViWFhrLUdWXzFkNEktN21ldlp6elRMeWVrS3VUYU9LNDdpYlg5SE1Ncjk2MVRyMjhPSFE4S0g3bHhvTjV5UzNsaGx5SG1QWU1kbTZrWGp2MlA5WkFRVnhwdHkxRmJ5NFViZW9WNGI4T1FMU0xoa1JjWXJiSlZzV1hVT1Q2bzFiYy13VGxqT3pSc3cwZkJsdUJqc3BSRW1VYmZ4ZXFib2V2UENCcnFVT19HRC02bm5BLTh6LVVza2FvNTVSN083WFluSmhSVDVHdTBHeVg5ell3ak9zM2d3WmVmanRSZlRuakwzSU9vQ2Noa3B4WVVuYXB2NWVGM1J4U3ZzNTk= | NULL              |
```

<br/>

## 6. 用 Barbican 来记录，真能确保安全么？

准确来说，Barbican 基于 OpenStack Keystone 实现的 RBAC 来进行权限管控的，它自身确保信息记录的保密性。只有正确权限的用户，才有权查看 Barbican 中用户记录的内容。

所以，显而易见的是，如果用户泄漏了个人鉴别信息，非法用户使用窃取到的用户信息顺利通过了 Keystone 鉴权而获得了合法身份，那么 Barbican 处也无法保证记录的信息不被获取。即个人信息与 Keystone 的安全性，会影响到整个 OpenStack 平台本身的安全性。

<br/>

## 小结

以上，是对 Barbican 的初步解析，先有个印象它大概是怎么用的。后续我会逐步深入的讲解各对象及其实现原理。
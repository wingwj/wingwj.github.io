# Barbican 解析（一）
Author: WingWJ

Date: 10th, Mar, 2021

<br/>

## 前言

最近工作需要，想评估下Barbican项目能力，但一直没找到很全面的介绍；官网文档也依旧是一言难尽。。所以还是自己写份笔记吧。

<br/>

## 1. Barbican 是干什么的？
简言之，用来管理OpenStack平台中各种 Secret 的。而 Secret，指的是我们不想被他人获取而需安全保存的内容，比如：
* **Private Key**：私钥
* **Certificate**：证书，如 X.509
* **Password**：密码，比如一段文本
* **SSH Keys**：SSH 密钥

这些，后续会对应到Barbican中具体的参数 secret_type，放后面说。

<br/>

## 2. 还是有点抽象，有例子么？
有，用CLI 先来感受下。

<br/>

### 2.1 如何创建一个最简单的 Secret？
比如，最简单的，保存一段密文`wj_pass=abc123`，方式如下：

<img src="https://s3.ax1x.com/2021/03/11/6t19qH.png" alt="barbican_101.png" />

可见，我们除了密文内容外，其余参数均未指定。由上面结果可以看出来，Barbican 默认识别密文内容 `wj_pass=abc123` 为文本格式 `text/plain`，再采用 AES-256 算法，以`opaque`格式将其保存到Barbican中，且永不过期。

P.S. `payload`字段其实并未限制你必须使用 `key=value`的形式，把它就当个字符串形式，想存啥写啥就好。比如，通过额外指定`-name`参数来给 Secret 添加名字来区分，只存`payload=abc123`也没问题——不过，有时我们可能连提示项都不想被人看到呢，这样直接都写在`payload`里是更合适的——这完全取决于自身的用法。

<br/>

### 2.2 那我如何查看我存入的原始内容呢？
同样的，如果你通过了用户鉴权，就能通过API 来获取。
我们来查询下上文中创建的 Secret。直接通过调用GET方法，并且显式指定`--payload`参数（简写为`-p`），就能直接查询了：

<img src="https://s3.ax1x.com/2021/03/11/6t1SMD.png" alt="barbican_102.png" />

注意，如果不指定`--payload`，则查询出来的结果，会和创建时返回的结构体一致，默认隐去 `payload`字段（因为CLI 的封装）。如下：

<img src="https://s3.ax1x.com/2021/03/11/6t1FII.png" alt="barbican_103.png" />

<br/>

## 3. OK，存密码这种我明白了。那其他类型 Secret 怎么存呢？
那我们来看下，用的最多的密钥方式吧，我们计划将一公钥倒入到Barbican中。
先使用`openssl`命令，创建一对RSA公私钥，再使用base64 进行编码，记录编码结果：
<img src="https://s3.ax1x.com/2021/03/11/6t1Zz8.png" alt="barbican_104.png" />

这里我们同样使用之前的命令来将证书存储到Barbican中，这次，我们需要相应的指定多个参数，与创建公私钥时的命令保持一致：
<img src="https://s3.ax1x.com/2021/03/11/6t1pse.png" alt="barbican_105.png" />

可见，在创建完后使用再查询，能够明确看到Barbican返回结果，是具备PEM header/footer 的X.509格式公钥。
和原始 `public.pem` 进行比对，确实是一致的：
<img src="https://s3.ax1x.com/2021/03/11/6t1Ait.png" alt="barbican_106.png" />

由上文可以看到，我们指定了该 Secret 的名称，并且对应证书生成时的命令，明确该 Secret 由RSA算法生成且密钥长度为2048，使用了base64方式编码，以原始二进制方式为密文内容，存储到了Barbican中。

<br/>

### 3.1 上面例子中，如果不按证书生成时对应的参数来填写，而是随便指定呢？
我们还是用例子来说话。
还是上文生成的已经base64编码过的公钥，我按文本方式`text/plain` 在存入Barbican时将算法改为 AES-256，还不指定base64，能存储成功么？
答案是，能。
<img src="https://s3.ax1x.com/2021/03/11/6t1VRf.png" alt="barbican_107.png" />

可见，查询出来的结果就是已经过base64编码后的原文，或者换句话说，是原始`$pub_base64`参数中存储的内容。即，这里直接将编码后内容作为文本，直接存储到了Barbican中。

<br/>

### 3.2 如果此时，我自己手动在用base64解码，能拿到原始公钥么？
当然可以。因为从Barbican中获取到的结果，即原来的编码后内容，那么使用同样的方法解码，自然能够获取。
<img src="https://s3.ax1x.com/2021/03/11/6t1EJP.png" alt="barbican_108.png" />

<br/>

### 3.3 那假如指定使用了base64，但是加密算法随便写为AES-256，还能成功么？

答案是，不能，参数不匹配。
<img src="https://s3.ax1x.com/2021/03/11/6t1mQS.png" alt="barbican_109.png" />

<br/>

### 3.4 对了，那个`payload-content-type`参数是干什么的？好像很复杂？

确实，该参数是用来指定传入的`payload`的类型的。在文章中一共使用过两个值，分别是：
* `text/plain`：普通文本；
* `application/octet-stream`：原始二进制数据。

看上去似乎很复杂，为何要这样定义呢？——这其实是和RFC标准中定义的 `MIME`（`Multipurpose Internet Mail Extensions`，多用途互联网邮件扩展类型）一一对应，用来描述消息内容类型的因特网标准。
比如，`application_json`、`image_jpeg`等，以及在Web场景下常见的`Content-Type: text/HTML`，其实就是一种具体的MIME 类型。它定义了数据的类型，以便数据能被适当的处理。用的时候，按 Secret 原始内容形式提交即可。

<br/>

### 3.5 看上去，似乎 `algorithm`、`bit-length` 参数没用啊？
其实，确实没用。。包括 `mode`，这三个参数都是用来标识的，API文档中的字段描述中写的就是 `(optional) Metadata provided by a user or system for informational purposes`。所以，你看上例中，不管填RSA，或者AES，其实都没有影响 Secret 的记录形式。

<br/>

### 3.6 从上面的例子中，我们学到了什么？

Barbican本身并不会校验 Secret 内容本身与填入算法是否匹配（如上例中的RSA改为AES），在入库时只是按照传入参数来存储，而在查询时按照之前记录的方式来响应（如编解码）。

考虑正确性与便利性，在存储 Secret 时应该指定正确的参数。
<img src="https://s3.ax1x.com/2021/03/11/6t1nsg.png" alt="barbican_110.png" />

<br/>

## 4. 看明白了，存证书这里还是挺有用的。回头看，在Barbican中单独保存密码，又有多大用呢？
上文举例中，我们只简单存了个无意义的字符密文`wj_pass=abc123`，看着好像用处不大。但如果，我们记录的是有意义的密文呢？
比如，还以上文RSA私钥来举例。之前我们生成时使用的命令中，由于并未指定加密方式，因此生成的私钥是明文的：

如果在使用`openssl` 创建密钥时，指定`-aes256`参数，就会将私钥进行加密，此时就会让你输入私钥密码，比如我们还用 `abc123`，最后就会生成一个经过AES-256加密后的私钥：
<img src="https://s3.ax1x.com/2021/03/11/6t1uLQ.png" alt="barbican_111.png" />
而刚生成时键入的密码，我们就得自己记录下来。这时，就可以通过Barbican来记录。此外，把私钥密码和公钥，甚至私钥，全都存储到Barbican中是允许的。只是都放到一个篮子里，是否安全，就是要考虑的另一个问题了。。

当然，你也可以通过Barbican，来记录某人生日、爱好、偶尔的灵感、甚至写日记。。完全看你的用法了 ：）～

<br/>

## 5. 说到这了，Barbican中的记录真的安全么？

准确来说，Barbican是以OpenStack Keystone实现的RBAC 来进行权限管控的。它能够保证它自身记录的保密性，比如DB中记录，那自然是加密存储的。只有正确权限的用户，才有权查看Barbican中用户记录的内容。
<img src="https://s3.ax1x.com/2021/03/11/6t1MZj.png" alt="barbican_112.png" style="zoom:50%;" />
而显而易见的是，如果用户泄漏了个人鉴别信息，非法用户使用窃取到的用户信息顺利通过了 Keystone鉴权而获得了合法身份，那么Barbican处 也无法保证记录的信息不被获取。即个人信息与Keystone的安全性，会影响到整个OpenStack平台本身的安全性。

以上，是对Barbican的初步解析，先有个印象它大概是怎么用的，后续才好学习其实现原理部分。
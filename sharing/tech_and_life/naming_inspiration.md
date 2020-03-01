# 技术生活：取名的灵感 —— 记录我的第一个爬虫程序

Author: WingWJ

Date: 1st, Mar, 2020

Python Version: 3.7

<br/>

## 起因

宝宝，不久就要降生了。

而取名真是个难题。。不知道男女，还得各想一个。。

<br/>

不想去找人算什么生辰八字，掏钱就把孩子可能一生的名字给定了。为人父母，还是想要亲力亲为。

我对古典诗词还是有爱的，至今仍然觉得当年流传下的一些诗词好美。我起名的参考方向，自然就转移到了浩如烟海的文学作品里。。《诗经》，整本翻完了，发现好些名字都被金庸大师抢注了（比如婉清）；《楚辞》看了一半，感觉生僻字好多，可能不太适合。另外，个人姓氏，也导致有些名字不合适选（><）。。所以，又开始读古诗词了。。但诗词的存世量真的很大，感觉找起来，少点儿方向性。

于是，考虑能否指定某些关键字，帮我缩小下参考范围呢？于是我动手查找了一些诗词网站，比如[《古诗文网》](https://gushiwen.org/)，但发现俩不足：

1. 搜索手段有限，无法指定特定主题 or 分类来搜索，不方便；
2. 即使有，由于分页显示的问题，一页能够显示的条目有限。且无法指定页面显示的最大条目，只得手工逐个点击页面来查看，效率低。

所以，有了这篇内容——我的第一个**爬虫程序**。

<br/>

## 实现

### 一. 简介

爬虫，其实就是在网络上抓取信息的程序。

没接触过的人，可能会觉得陌生，但其实应用很常见。比如用x度搜索时，经常一搜素页里内容雷同的各站，不少是用爬虫技术从其他网站抓取信息贴在自己网站里，靠引流来卖广告维生的各无效网站。。而不少电商、资源类网站（如豆瓣、xx音乐 等）资讯类网站（xx 新闻、贴吧等），由于内容的价值，常常会成为爬虫的目标；其他应用通过抓取他们的数据之后再加工，而产生新的数据：比如 豆瓣电影 Top100 榜单、xx音乐xx统计 等。

而在此次的场景里，需求很明确，就是：代我在诗词网站上，按照我指定的_关键词_，去搜索特定的分类，最终将满足条件的结果，以“诗文+出处”的方式显示。再具体下，就以[《古诗文网》](https://gushiwen.org/)中的["名句"](https://so.gushiwen.org/mingju/Default.aspx)领域为例，来搜索包含指定关键词的古诗词。

<img src="https://s2.ax1x.com/2020/03/01/32Pcgf.png" alt="naming_inspiration_001.png" style="zoom:50%;" />

### 二. 步骤

下面开始实践。

先从逻辑上来考虑下需求：我们需要从一个网站上抓取数据，先要选取页面，再经过页面解析，最后输出处理结果。从步骤上看，得分四部来完成各自的任务，这四个我就随便起名了，分别为：分发器、下载器、解析器、分析器。下面就按这个思路走，Python 走起～

#### 1. 分发器

首先，要使用爬虫，需要先圈定个范围，即你要告诉爬虫：帮你寻找哪些页面的资源。在通用爬虫程序中，一般会有个 URL 调度器，来管理和分发需要遍历检索的 URL 列表。而在本需求中，就是[《古诗文网》](https://gushiwen.org/)中的["名句"](https://so.gushiwen.org/mingju/Default.aspx)领域。不需要那么复杂，网址基本固定。

让我们观察下该站的页面 URL 构成：

- 不同 '分页'，在原始 URL 后面，添加 `?p=` 来标示不同页码；

- 对于不同 '主题'，在页码 URL 后面，再添加 `&c=` 来区分。后续还能再加 `&t=` 参数来再细分。

将以上分析结果，用代码来实现：

```
def combine_url(start, index, end):
    return start + str(index) + end
```

#### 2. 下载器

其次，要上网获取数据，得有联网功能吧。比如 Python 自带的 `urllib`/`urllib2` 等，此外还有第三方库比如 `requests` 可选。而此次实现的内容相对简单，选哪个都行。这里我就选 `requests` 库了，简单易懂。

`pip` 直接安装：

```
$ pip install requests
```

代码示例：

```
import requests

test_url = 'https://so.gushiwen.org/mingju/Default.aspx'
raw_html = requests.get(test_url)
print(raw_html.text)
```

输出是一大堆内容，看起来有点怕。。没事，我们下一步就来处理他们。

```
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head><meta http-equiv="Cache-Control" content="no-siteapp" /><meta http-equiv="Cache-Control" content="no-transform " /><meta http-equiv="Content-Type" content="text/html; charset=UTF-8" /><title>
经典名句_古诗文名句_古诗文网
</title>
<script type="text/javascript">
    if ((navigator.userAgent.match(/(phone|pad|pod|iPhone|iPod|ios|iPad|Android|Mobile|BlackBerry|IEMobile|MQQBrowser|JUC|Fennec|wOSBrowser|BrowserNG|WebOS|Symbian|Windows Phone)/i))) {
        window.location.href = "https://m.gushiwen.org/mingju/Default.aspx?p=1&c=&t=";
    } else {

    }
</script>
...
```

#### 3. 解析器

当联网的问题搞定了，页面内容也能顺利获取到了。但面对全是 `html/xml` 格式的页面内容，其中众多的标签，必须得有方法来提取我们关心的数据。**这一部分是整个爬虫程序的重点**，下文会详细说明。

这里，我先去调查了下页面解析的方案，发现当前用的较多的有以下三种：

- `re`：正则表达式；
- `XPath`：速度快，但只能处理 `xml` 格式；
-  `Beautiful Soup`：支持 `xml/html` 格式。

看上去 `Beautiful Soap` 支持更广，用起来也简单些，网上示例也不少。这次我也试试。

_P.S. 看到 Soap，就想起了早年在公司做项目时，系统接口是 SOAP 的，前端得专门套一个解析层，简直了。。远没有当前的 REST 简洁。扯远了。。_

同样，先安装 [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/)，位于 `bs4` 库中。相关的 `lxml` 库，不管之前装没装过也一并更新下，比默认的 `html` 解析器更好用：

```
$ pip install bs4
  pip install -U lxml
```

在进入编码部分之前，我们先来确定下我们的处理目标。

从上文中一大堆输出中，找了下，我们想获取的内容其实就是下文中这些词句：

```
...
<a style=" float:left;" target="_blank" href="/mingju/juv_2d8bb03f1e19.aspx">山有木兮木有枝，心悦君兮君不知。</a><span style=" color:#65645F; float:left; margin-left:5px; margin-right:10px;">——</span><a style=" float:left;" target="_blank" href="/shiwenv_4a96c8287eb5.aspx">佚名《越人歌》</a>
</div>
<div class="cont" style=" margin-top:12px;border-bottom:1px dashed #DAD9D1; padding-bottom:7px;">
<a style=" float:left;" target="_blank" href="/mingju/juv_43f5bfba7229.aspx">人生若只如初见，何事秋风悲画扇。</a><span style=" color:#65645F; float:left; margin-left:5px; margin-right:10px;">——</span><a style=" float:left;" target="_blank" href="/shiwenv_85e93138ed65.aspx">纳兰性德《木兰词·拟古决绝词柬友》</a>
</div>
<div class="cont" style=" margin-top:12px;border-bottom:1px dashed #DAD9D1; padding-bottom:7px;">
<a style=" float:left;" target="_blank" href="/mingju/juv_e87e6b55ba7e.aspx">曾经沧海难为水，除却巫山不是云。</a><span style=" color:#65645F; float:left; margin-left:5px; margin-right:10px;">——</span><a style=" float:left;" target="_blank" href="/shiwenv_bd4224133394.aspx">元稹《离思五首·其四》</a>
</div>
<div class="cont" style=" margin-top:12px;border-bottom:1px dashed #DAD9D1; padding-bottom:7px;">
<a style=" float:left;" target="_blank" href="/mingju/juv_05cbbd4b3d22.aspx">人生如逆旅，我亦是行人。</a><span style=" color:#65645F; float:left; margin-left:5px; margin-right:10px;">——</span><a style=" float:left;" target="_blank" href="/shiwenv_330cfc1476ad.aspx">苏轼《临江仙·送钱穆父》</a>
</div>
...
```

注意，这是我们在输出里直接看的。你要让程序来识别，得告诉它具体的位置结构。之前解析过 SOAP 或者说 `xml` 的都懂，得先从树形结构入手。

这里有个便捷的办法：直接用 Chrome 浏览器自带的 '开发者工具'，能帮我们减少不少工作量。它能直接确定内容的具体位置，而不用手工解析整支树再逐级确定内容路径了。能省不少事。操作步骤如下：

1. 首先在 Chrome 浏览器中，打开我们需要获取的《古诗文网》中的 ["名句"](https://so.gushiwen.org/mingju/Default.aspx) 页面；

2. 选取诗词区域，比如选取第五句诗文，右键 '检查'，之后 '开发者工具' 会自动弹出；

3. 在右侧工作区，指定需要的区域，右键 'copy' -- 'copy selector'，复制指定路径：

      <img src="https://s2.ax1x.com/2020/03/01/32Pgv8.png" alt="naming_inspiration_002.png" style="zoom:50%;" />

4. 分别将 '诗文' 部分和 '出处' 部分。分别复制下来：

      ```
      body > div.main3 > div.left > div.sons > div:nth-child(5) > a:nth-child(1)
      body > div.main3 > div.left > div.sons > div:nth-child(5) > a:nth-child(3)
      ```

##### 第一版解析器

下面可以写测试程序了。在上一步骤中完成的**下载器**的基础上，先写个第一版解析器看看，路径我们先用刚复制下来的第一条：

```
import requests
from bs4 import BeautifulSoup

test_url = 'https://so.gushiwen.org/mingju/Default.aspx'
raw_html = requests.get(test_url)
#print(raw_html.text)

soup = BeautifulSoup(raw_html.text, 'lxml')
origin_data = soup.select('body > div.main3 > div.left > div.sons > div:nth-child(5) > a:nth-child(1)')
print(origin_data)
```

来看看输出：

```
[<a href="/mingju/juv_7c9202538459.aspx" style=" float:left;" target="_blank">十年生死两茫茫，不思量，自难忘。</a>]
```

嗯，有点意思了。：）

回头看看，刚才复制出来的结构：其实代表该页面中的第五条诗文。而诗文内容又能细分成三部分，分别是：

- 诗文：即 "十年生死两茫茫，不思量，自难忘。"；
- 分隔符 '——'，可以在 ''开发者工具' 中看到；
- 出处：即 "苏轼《江城子·乙卯正月二十日夜记梦》"。

##### 第二版解析器

有了上面的理解，我们替换下之前代码中的路径。在 `div` 中不指定具体序号：

```
body > div.main3 > div.left > div.sons > div > a:nth-child(1)
```

重新跑一下：

```
[<a href="/mingju/juv_2d8bb03f1e19.aspx" style=" float:left;" target="_blank">山有木兮木有枝，心悦君兮君不知。</a>, <a href="/mingju/juv_43f5bfba7229.aspx" style=" float:left;" target="_blank">人生若只如初见，何事秋风悲画扇。</a>, <a href="/mingju/juv_e87e6b55ba7e.aspx" style=" float:left;" target="_blank">曾经沧海难为水，除却巫山不是云。</a>, <a href="/mingju/juv_05cbbd4b3d22.aspx" style=" float:left;" target="_blank">人生如逆旅，我亦是行人。</a>, <a href="/mingju/juv_7c9202538459.aspx" style=" float:left;" target="_blank">十年生死两茫茫，不思量，自难忘。</a>, <a href="/mingju/juv_a1e34e2152e2.aspx" style=" float:left;" target="_blank">玲珑骰子安红豆，入骨相思知不知。</a>, <a href="/mingju/juv_f65c48d99ca7.aspx" style=" float:left;" target="_blank">雨打梨花深闭门，忘了青春，误了青春。</a>, ...]
```

可见，在不指定时，`Beautiful Soap` 会将页面所有诗文条目都遍历下来。

此外，如果想获取所有诗句的出处，在路径中将子元素从 `1` 改为 `3` 就能得到：

```
body > div.main3 > div.left > div.sons > div > a:nth-child(3)
```

结果示例如下：

```
[<a href="/shiwenv_4a96c8287eb5.aspx" style=" float:left;" target="_blank">佚名《越人歌》</a>, <a href="/shiwenv_85e93138ed65.aspx" style=" float:left;" target="_blank">纳兰性德《木兰词·拟古决绝词柬友》</a>, <a href="/shiwenv_bd4224133394.aspx" style=" float:left;" target="_blank">元稹《离思五首·其四》</a>, <a href="/shiwenv_330cfc1476ad.aspx" style=" float:left;" target="_blank">苏轼《临江仙·送钱穆父》</a>, <a href="/shiwenv_567fcf6ffefb.aspx" style=" float:left;" target="_blank">苏轼《江城子·乙卯正月二十日夜记梦》</a>, <a href="/shiwenv_a2b349dd32e8.aspx" style=" float:left;" target="_blank">温庭筠《南歌子词二首 / 新添声杨柳枝词》</a>, <a href="/shiwenv_826758a2d666.aspx" style=" float:left;" target="_blank">唐寅《一剪梅·雨打梨花深闭门》</a>, ...]
```

这里，基本信息其实就都拿到了。可以将 '诗文'、'出处' 两个结果，各自存成一个`list`，两者之间通过相同的索引来对应。程序算是基本可用了。

但以上程序还有俩问题：

1. 还有少量的 `xml` 结构干扰结果，不易读；
2. '诗文' 与 '出处' 两个列表其实并无关联，只是简单用索引对应；

所以，有了下一版的改进程序。

##### 第三版解析器

考虑上面两个问题，我们重新修改下页面路径：

```
body > div.main3 > div.left > div.sons > div > a
```

可见，输出变为了 '诗文' 与 '出处' 在一起的：

```
[<a href="/mingju/juv_2d8bb03f1e19.aspx" style=" float:left;" target="_blank">山有木兮木有枝，心悦君兮君不知。</a>, <a href="/shiwenv_4a96c8287eb5.aspx" style=" float:left;" target="_blank">佚名《越人歌》</a>, <a href="/mingju/juv_43f5bfba7229.aspx" style=" float:left;" target="_blank">人生若只如初见，何事秋风悲画扇。</a>, <a href="/shiwenv_85e93138ed65.aspx" style=" float:left;" target="_blank">纳兰性德《木兰词·拟古决绝词柬友》</a>, <a href="/mingju/juv_e87e6b55ba7e.aspx" style=" float:left;" target="_blank">曾经沧海难为水，除却巫山不是云。</a>, <a href="/shiwenv_bd4224133394.aspx" style=" float:left;" target="_blank">元稹《离思五首·其四》</a>, <a href="/mingju/juv_05cbbd4b3d22.aspx" style=" float:left;" target="_blank">人生如逆旅，我亦是行人。</a>, <a href="/shiwenv_330cfc1476ad.aspx" style=" float:left;" target="_blank">苏轼《临江仙·送钱穆父》</a>, ...]
```

改动下程序。对上文再稍加解析下：

```
import requests
from bs4 import BeautifulSoup

test_url = 'https://so.gushiwen.org/mingju/Default.aspx'
raw_html = requests.get(test_url)
#print(raw_html.text)

soup = BeautifulSoup(raw_html.text, 'lxml')
origin_data = soup.select('body > div.main3 > div.left > div.sons > div > a')
#print(origin_data)

def _parse_data(data):
    for item in data:
        cont = item.get_text()
        print(cont)
_parse_data(origin_data)
```

修改后的输出，变为了纯文本样式，且一行 '诗文'，一行 '出处'。一下豁然开朗了：

```
有木兮木有枝，心悦君兮君不知。
佚名《越人歌》
人生若只如初见，何事秋风悲画扇。
纳兰性德《木兰词·拟古决绝词柬友》
曾经沧海难为水，除却巫山不是云。
元稹《离思五首·其四》
人生如逆旅，我亦是行人。
苏轼《临江仙·送钱穆父》
十年生死两茫茫，不思量，自难忘。
苏轼《江城子·乙卯正月二十日夜记梦》
...
```

至此，页面信息，已经完全解析完毕了，再没有恼人的 `xml` 结构了。而刚提到的对应问题，在下一节一并处理。

#### 4. 处理器

在完成了网页数据的抓取与解析，最后剩的工作，就是按照要求来输出最终结果了。这里还有以下几件事要完成，分别是：

- 首先，为了解决 '诗文' 与 '出处' 的`list` 对应问题，可通过写个字典，将两者内容逐个作为 `key` 和 `value` 来做对应；

- 其次，第一步完成的 '分发器' 还没用上呢。把它加上，使得程序可以直接遍历每张页面。参考页面右下角翻页数为`100`，程序中也先设置为`100`；

- 最后，考虑下我们的需求：要求能够按照指定关键字进行内容过滤。所以，还需要实现个简单的过滤功能。

以上每一步，就不一一解释了。最终完成的代码，如下：

```
import requests
from bs4 import BeautifulSoup

url_start = 'https://so.gushiwen.org/mingju/Default.aspx?p='  # 对应网址
url_end = '&c=%e4%ba%ba%e7%94%9f'  # 对应'人生'主题，不指定则设为''

def combine_url(start, index, end):
    return start + str(index) + end

result = {}
for i in range(1, 100):  # 其实由于该网站限制，页面能够访问的最大翻页数仅为20
    url = combine_url(url_start, i, url_end)
    raw_html = requests.get(url)
    soup = BeautifulSoup(raw_html.text, 'lxml')
    origin_data = soup.select('body > div.main3 > div.left > div.sons > div > a')
    #print(origin_data)

    def _parse_data(data):
        num = 0
        temp_key = ''
        for item in data:
            cont = item.get_text()
            # 先将'诗句'设为key，'出处'设为value
            if (num % 2) == 0:
                # 先读取'诗句'作为key，'出处'暂时设为0，待下一步更新
                result[cont] = 0
            else:
                result[temp_key] = cont
            num += 1
            temp_key = cont
    _parse_data(origin_data)

def search_by_str(keywords):
    for k,v in result.items():
        if keywords in k:
            print(k + ' -- ' + v)

search_by_str(u"无")
```

让我们来看下，最终按照关键字 '无'，在'人生' 领域，过滤出的名句：

```
身无彩凤双飞翼，心有灵犀一点通。 -- 李商隐《无题·昨夜星辰昨夜风》
林花谢了春红，太匆匆。无奈朝来寒雨晚来风。 -- 李煜《相见欢·林花谢了春红》
菩提本无树，明镜亦非台。 -- 惠能《菩提偈》
当年不肯嫁春风，无端却被秋风误。 -- 贺铸《芳心苦·杨柳回塘》
锦瑟无端五十弦，一弦一柱思华年。 -- 李商隐《锦瑟》
夕阳无限好，只是近黄昏。 -- 李商隐《乐游原 / 登乐游原》
落红不是无情物，化作春泥更护花。 -- 龚自珍《己亥杂诗·其五》
...
```

怎么样，是不还挺像样的？：）

想按什么过滤，直接替换关键词，就能得到想要的名句与出处了。

<br/>

## 小结

在完成我的第一个爬虫程序之前，其实觉得该领域还是挺神秘的。但通过自己一步步动手实践，觉得还挺有意思。虽然这个功能并没多强大， 方案也并不通用，示例网站还只允许查看20页；方案最后也没添加诸如“用户登陆、多线程访问、数据存储”等进阶功能。但却是我根据个人实际需要，逐步实践完成的第一款爬虫。简单，却有效。：）

<br/>

## 篇外

空余时间写技术博客，其实要花不少心思。以这次爬虫程序为例，程序可能两个小时就完成了，但这篇博文要花费数倍的时间。

这次程序写完，该起的名字还是要起的，真的挺难想。。希望我近期就能想出来合适的名字。：）

最后，祝宝贝平安降生，健健康康。
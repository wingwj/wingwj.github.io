# 解决近期 Github 图片无法显示

`Author: WingWJ`
`Date: 20th, Feb, 2020`



## 背景

近期Github 图片在墙内全部无法显示，导致各文章中的图片都没法查看了。。

之前只是在自学容器时，发现 `raw.githbusercontent.com` 访问不了，得手动处理 yaml。

我文章的图片是和文档一起，直接上传到Github 的。开始我以为只是把Github 本身的图片服务ban了，还专门去找了个第三方图床，想通过外链的方式来显示图片。整理完传上去试了下，发现图片还是挂（*注：可见柏林峰会那篇*）。。直接点图片链接进去，发现外链图片是通过 `camo.githubusercontent.com`  来中转的，所以问题依旧。。

<br/>

## 解决

网上搜了下，说是近期DNS被污染了。

参考该[博客](https://reishin.me/github-dns/)，可在`/etc/hosts` 添加以下地址即可解决。

```
199.232.28.133 gist.githubusecontent.com
199.232.28.133 user-images.githubusercontent.com
199.232.28.133 raw.githubusercontent.com
199.232.28.133 camo.githubusercontent.com
199.232.28.133 cloud.githubusercontent.com
199.232.28.133 avatars0.githubusercontent.com
199.232.28.133 avatars1.githubusercontent.com
199.232.28.133 avatars2.githubusercontent.com
199.232.28.133 avatars3.githubusercontent.com
199.232.28.133 avatars4.githubusercontent.com
199.232.28.133 avatars5.githubusercontent.com
199.232.28.133 avatars6.githubusercontent.com
199.232.28.133 avatars7.githubusercontent.com
199.232.28.133 avatars8.githubusercontent.com
```

修改完毕，刷新Github 页面。搞定～ 

修改后，之前不能下载的`raw.githubusercontent.com` 网址上的各yaml 文件，也能下载了。

这下又有动力继续写文章了。：）

*P.S. 不过手机端，由于无法直接修改 hosts 文件，所以还是不能正常显示；我也再去找找其他办法。希望DNS早日恢复正常。*

<br/>


## 后话

服务修复后，我又重新观察了下两个页面的图片显示：

1. [柏林峰会观感](https://github.com/wingwj/wingwj.github.io/blob/master/sharing/berlin_summit/OpenStack_Berlin_Summit.md)：所有图片均链接至第三方图床；
2. [STX-HA 项目详解](https://github.com/wingwj/wingwj.github.io/blob/master/sharing/starlingx/stx_ha.md)：所有图片仍然使用之前上传至Github 的图片目录。

前者比后者快了不少。我再观察下，后续可能考虑将所有图片都转移至该图床，来加快显示。


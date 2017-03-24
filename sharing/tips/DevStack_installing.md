# DevStack 安装配置



## 背景

​	记下走过的坑。





## OS配置

1. 虚拟机安装时，强烈建议磁盘多分点儿，否则后续还得再挂盘；
2. 时区语言等全选英文。




## 安装

1. 提示包冲突时，比如如下的packaging包：

   ```pkg_resources.VersionConflict: (packaging 16.7/opt/stack/requirements/.venv/lib/python2.7/site-packages), Requirement.parse('packaging>=16.8'))```

   这时，建议不用浪费时间手动执行手动更新：

   ```pip install --upgrade packaging>=16.8```

   因为一般还是绕不过包依赖，再次安装还是会提示同样错误。建议直接手动修改包依赖版本，跳过此次检查。

   比如在此例中，直接修改`/opt/stack/requirements/upper-constraints.txt` 文件中packaging 对应版本，从

   `packaging===16.7`修改为`packaging>=16.8`, 保存退出，重新执行`./stack.sh`

   即可继续安装了。

   ​

2. 安装Nova中，出现 

   `rm -rf /opt/stack/data/nova/instances/` 失败，提示`device or resource busy`：

   1. 首先检查下文件夹权限，是否正确；
   2. 其次`lsof +D /opt/stack/data/nova/instances/`，检测是否有程序占用。结果也是没有；
   3. 最后，想到这个目录不就是VM目录么，当时空间不够，我好像挂了个其他目录过来。。`df -h`，果然发现有磁盘挂载。`umount /opt/stack/data/nova/instances/`，问题解决。（P.S. 现在知道为啥在开头我说磁盘尽量留够了吧。。）




3. 如果虚拟机的磁盘，之前分小了后面发现不够了怎么办：

   问题类似上文，假如我一开始搭建的空间不够了，后面大镜像的VM创不起来，该怎么办呢？这里提供两种方式：

   1. 第一种简单些：如果用的是VirtualBox 在本机装的devstack，那么直接新挂一个磁盘到VM上，比如/dev/sdb，然后再把新磁盘挂载到对应目录下就好了。过程简单就不写具体命令了；

   2. 第二种麻烦些，直接扩容LVM的逻辑卷组，但相当于直接把整机空间扩大，就不用针对单个目录来挂载了。方法如下：

      1. 同样先准备一个磁盘如/dev/sdb，然后划一个分区sdb1，最后指定为LVM格式：

         1. 如果需要修改磁盘类型，可使用`fdisk /dev/sdb`，输入`t`后指定`8e`（注：8e是LVM代码）；
         2. 之后`w`保存退出；

      2. 由于devstack 限制了lvm扫描位置，路径`/dev/sdb` 会被直接过滤掉，因此需要修改lvm设置：

         `vi /etc/lvm/lvm.conf`，将下面这句话注释掉，`:wq`保存退出：

         `` #global_filter = [ "a|loop0|", "a|loop1|", "a|sda5|", "r|.*|" ]  # from devstack``

      3. 之后，就可以添加物理卷（PV）到卷组（VG）中了。devstack 默认使用ubuntu-vg作为VG名， 
         所以需要先创建一个新的物理卷（PV），再把卷组（VG）扩充到该物理卷上：

         ```pvcreate /dev/sdb1```

         ```vgextend ubuntu-vg /dev/sdb1```

      4. 最后，扩容一下root分区，`lvextend –L +32G /dev/ubuntu-vg/root`，最后扩容一下对应的文件系统就大功告成了：`resize2fs /dev/dm-0`。

         1. 注：devstack默认dm-0对应ubuntu-vg/root，dm-1对应ubuntu-vg/swap，后者可不管。
         2. 执行`df -h`检查下，可以看到原/dev/dm-0 已经扩容完成。Enjoy！
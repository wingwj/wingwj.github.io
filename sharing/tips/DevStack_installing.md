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




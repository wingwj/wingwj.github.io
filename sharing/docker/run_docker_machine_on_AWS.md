# AWS上 部署Docker Machine

Author: WingWJ

Date: 18th, Feb, 2020



## 一. 引言

### 1.1 Docker Machine 是什么

在Docker 学习中，最初只需要一台实验机，所有的容器都跑在这一台Host 上就够了。但在真正的环境中必然会有多个 host，容器在这些 host 中启动、运行、停止和销毁，而相关容器会通过网络相互通信，无论它们是否位于相同的 host。对于这样一个 multi-host 环境，如何高效地进行管理呢？这里面临的第一个问题是：为所有的 host 安装和配置 Docker。

比如手工安装一个 docker host，步骤包括：

1. 安装 Host OS
2. 配置各类证书/密钥
3. 添加 Docker apt 源
4. 安装 Docker
5. 配置 Docker

而这么多的操作步骤，对于多主机环境手工方式效率低，且不容易保证一致性。对于该问题的解决方案，就是本篇的 Docker Machine。



### 1.2 Docker Machine 能干什么

一句话概述：**Docker Machine 可以批量安装和配置 docker host**。

在Docker Machine中，Machine 指的是运行 docker daemon 的主机（Host）。这个 host 可以是本地的虚拟机、物理机，也可以是公有云中的云主机。Docker Machine 支持在不同的环境下安装配置 docker host，包括：

- 常规 Linux 操作系统
- 虚拟化平台：VirtualBox、VMWare、Hyper-V
- OpenStack
- 公有云：AWS、Microsoft Azure、GCP等

Docker Machine 为这些类似后端的环境，起了一个统一的名字：provider。对于某个特定的 provider，Docker Machine 使用相应的 driver 来安装和配置 docker host，如下图所示：

![](images/docker-machine-001.png)



### 1.3 为何选择 AWS EC2

这里提的AWS，准确来讲是 AWS EC2，即亚马逊的弹性云服务。

选择AWS EC2，有以下几点理由：

1. 最主要原因：没墙，各种方便。。（注：稍后我会专门写篇文章来说明）

2. 便捷：只需提前准备一个Docker host
3. 便宜：AWS 有免费套餐。。（*这点好像才是关键。。*）



## 二. 安装流程

### 2.1 实验环境

不同于 VirtualBox等 本地安装方案 需提前配置三个主机来安装Docker Machine，在AWS EC2上只需要提前准备一台配置好Docker服务的主机。

如下，通过AWS EC2服务，提前创建了一个Docker host（172.31.44.89），文中称为Host 0。镜像选用的是AWS推荐的Amazon AMI（linux）。Docker配置流程略。

P.S. *题外话，还是**建议首选业界常用的 ubuntu 或 CentOS**。我这是已经上船了。。惨痛经历到时一起写。。*

整体环境准备如下：

<img src="images/docker-machine-002.png" style="zoom:50%;" />

稍后我们将在该主机（172.31.44.89）上安装 Docker Machine，然后通过 docker-machine 命令，在其他两个AWS VM上部署 docker。



### 2.2 安装 Docker Machine

**说明**：本文使用的Docker Machine 版本为：**v0.16.0**。考虑到开源软件版本更迭时可能会有较大变化，原有的一些内容或安装方式可能不再适用于新版。因此，如按下文安装遇到问题时，可以参考最新版本的[官方安装文档](https://docs.docker.com/machine/install-machine/)。

这里开始正式展示安装过程。先执行如下命令下载代码包：

 ```
$ base=https://github.com/docker/machine/releases/download/v0.16.0 &&
  curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/tmp/docker-machine &&
  sudo mv /tmp/docker-machine /usr/local/bin/docker-machine &&
  chmod +x /usr/local/bin/docker-machine
 ```

![](images/docker-machine-003.png)

下载的执行文件会被放到 `/usr/local/bin` 中。

注意：对于Amazon AMI 镜像，**`$PATH` 默认是不包含`/usr/local/bin` 目录的**，所以当执行后续 docker-machine 命令时会失败。因此需要额外把 `/usr/local/bin` 添加到`$PATH`中去，执行命令如下图：

![](images/docker-machine-004.png)

*P.S. 了解为什么之前推荐用 ubuntu 或 CentOS 了吧。。还有坑呢，到时一起吐槽。。*

执行 `docker-machine version` 验证命令是否可用：

![](images/docker-machine-005.png)

这里，Docker Machine 已经安装好了。

当然，为了得到更好的体验，可额外安装命令行补齐脚本（bash completion script），这样使用 bash 能够通过 tab 键补全 docker-machine 的子命令和参数。仍旧参照[官方文档](https://docs.docker.com/machine/install-machine/)执行。

先确认下补齐目录：

![](images/docker-machine-006.png)

 执行命令：

 ```
$ base=https://raw.githubusercontent.com/docker/machine/v0.16.0
  for i in docker-machine-prompt.bash docker-machine-wrapper.bash docker-machine.bash
  do
    sudo wget "$base/contrib/completion/bash/${i}" -P /etc/bash_completion.d
  done
 ```

![](images/docker-machine-007.png)

将其放置到 `/etc/bash_completion.d` 目录下。

另外，由于当前**官方教程有些问题**，需要修改下。需要再将如下代码添加到 `~/.bashrc`。这两段内容，<u>前者是为了持久化环境变量，后者source不加会报错</u>。以上代码都已在Amazon AMI镜像中验证通过：

 ```
PATH=$PATH:/usr/local/bin

export PATH



PS1='[\u@\h \W$(__docker_machine_ps1)]\$ '

source /etc/bash_completion.d/docker-machine-prompt.bash
 ```

![](images/docker-machine-008.png)

可以执行 `docker-machine ls` 查看一下当前的 machine：

![](images/docker-machine-010.png)

可见，当前环境中还没有安装 machine，下一节会创建第一个 machine。

至此。Docker Machine 已经就绪，当前环境如下：

<img src="images/docker-machine-009.png" style="zoom:50%;" />



### 2.3 **创建** **Docker Machine**

对于 Docker Machine 来说，术语 Machine 就是运行 docker daemon 的主机。“创建 Machine” 指的就是在 host 上安装和部署 docker。如文章开头提到的，Docker Machine 对于不同provider，提供了对应的[多种driver](https://docs.docker.com/machine/drivers/) 可选。本文需要选择AWS方式，可点击[本链接](https://docs.docker.com/machine/drivers/aws/)直达。

从这里开始，就是 Docker Machine 在不同provider下的差异部分了。

与默认 generic driver下通过ssh安装其他Linux发行版不同的是，这里创建的 **docker host 是在AWS上的VM里创建的**（镜像默认使用ubuntu 16.04 LTS）。对于docker machine 安装来说，主要需要配置好IAM鉴权相关的内容，再在安装时，修改对应provider/driver为 amazonec2。

#### 2.3.1 准备用户

作为预备条件：需要在创建Docker Machine前，在AWS IAM中创建对应的用户。操作可以参考[官方用户指导文档](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/getting-started_create-admin-group.html)及[命令行手册](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration)。

登录AWS 账号后，需要在AWS IAM中，创建一个具备AK、SK的可登录用户（不能直接使用根账户）：

![](images/docker-machine-aws-001.png)

之后，在docker-machine 主机上，执行 aws configure，配置AK与SK（注：SK部分已隐去）。

![](images/docker-machine-aws-002.png)

![](images/docker-machine-aws-003.png)

#### 2.3.2 安装 Machine

在配置好用户后，在节点上直接执行以下命令来创建名为aws01的主机，指定 driver类型为 amzonec2：

```
$ docker-machine create --driver amazonec2 aws001
```

![](images/docker-machine-aws-004.png)

请观察下命令回显，create操作中，其实一共完成了以下任务：

1. 创建VM；
2. 待服务可用时，通过 ssh 登录到远程主机。
3. 根据不同Hypervisor，安装 docker；
4. 拷贝证书；
5. 配置 docker daemon；
6. 启动 docker 服务。

 至此，可以看到对应docker-machine 已经在AWS指定用户下创建完成了：

```
$ docker-machine ls
```

![](images/docker-machine-aws-005.png)

 当然，通过ec2tools（即 AWS EC2 CLI） 提供的VM查询CLI，来指定AK 和 SK，也能查询到该VM：

```
$ ec2-describe-instances --aws-access-key AKIA3VUM6BBZDOQN6S5U --aws-secret-key ***** 
```

![](images/docker-machine-aws-006.png)

**注意**：对于使用非Amazon AMI 镜像的主机，可能需要安装对应OS版本的 ec2tools。

**提示**：如果不想在使用AWS CLI时，频繁输入AK、SK，可以先导入环境变量，一劳永逸：

![](images/docker-machine-aws-009.png)

#### 2.3.3 说明

这里有一点要**特别说明**的是：在刚才创建 docker-machine 时，我们其实仅指定了几个必选参数，部分其他参数都是使用的默认值。比如因未指定 `--amazonec2-region`，导致 Region 会默认使用 `us-east-1`。所以当你登陆AWS Dashboard 时，可能看不到新创建的VM，需要切换到对应Region 才能看到。

比如，如果想使用特定AMI 在指定Region内（比如东京）创建Docker Machine，命令示例如下：

```
$ docker-machine create --driver amazonec2 --amazonec2-region ap-northeast-1 --amazonec2-ami ami-746aba14 aws-test01
```

这些默认值，都已详细记录在[官方文档](https://docs.docker.com/machine/drivers/aws/)中。若想修改，可在创建Docker Machine 时直接指定。

![](images/docker-machine-aws-014.png)

最后，若需要登录 docker-machine时，可使用以下命令：

```
$ docker-machine ssh aws001
```

#### 2.3.4 安装小结

同理，按照以上方法，完成aws02 的安装。全部创建完毕后，查看当前环境如下：

![](images/docker-machine-011.png)

注：两台机器的IP地址不同，是因为EC2下有公有/私有两个IP转换。可暂不关心。

更新部署图：

<img src="images/docker-machine-012.png" style="zoom:75%;" />

 

## 三. 管理 **Docker Machine**

用 docker-machine 创建 machine 的过程很简洁，非常适合多主机环境。除此之外，Docker Machine 也提供了一些子命令方便对 machine 进行管理。其中最常用的就是无需登录到 machine 就能执行 docker 相关操作。

在最初Docker 学习中，要执行远程 docker 命令我们可用通过 `-H` 来指定目标主机的连接字符串，比如对于 aws01：

```
$ docker -H tcp://18.179.41.170:2376 ps
```

但是，在执行以上命令时可能会提示失败，提示“通信方式不对：http无法连接到https下”。这是由于在初始docker host（即本例中的Host 0）上执行时，新docker host 默认使用TLS保证通信安全，而原有主机Host 0 不一定配置了。

可以执行以下命令来查看 aws01的环境变量，确实是启用了TLS认证：

```
$ docker-machine env aws01
```

![](images/docker-machine-013.png)

而Docker Machine 则让这个过程更简单。根据提示，直接执行以下命令：

```
$ eval $(docker-machine env aws01)
```

![](images/docker-machine-014.png)

可以看到，命令行提示符已经变为了aws01，其原因是之前在 `$HOME/.bashrc` 中配置了 `PS1='[\u@\h \W$(__docker_machine_ps1)]\$ '`，用于显示当前 docker host。

 在此状态下执行的所有 docker 命令其效果都相当于在 aws01 上执行，例如启动一个 busybox 容器：

![](images/docker-machine-015.png)

同样可以切换到 aws02上：

```
$ eval $(docker-machine env host2)
```

![](images/docker-machine-016.png)

最后，再介绍几个有用的 docker-machine 子命令：

- `docker-machine upgrade`：更新machine 的 docker 到最新版本，可以批量执行：

![](images/docker-machine-017.png)

- `docker-machine config`：查看machine 的 docker daemon 配置：

  ![](images/docker-machine-018.png)

  - `--H tcp://0.0.0.0:2376` 使docker daemon 接受远程连接。

  - `--tls*` 对远程连接启用安全认证和加密。

- `stop/start/restart`：是**对 machine 的操作系统操作**，而不是 stop/start/restart docker daemon。

- `docker-machine scp`：可以在不同 machine 之间拷贝文件，比如：

  ```
  $ docker-machine scp aws01:/tmp/a aws02:/tmp/b
  ```



可见，在多主机环境下，使用 Docker Machine 可以大大提高操作与运维效率，操作也简单易用。



## 四. 登陆 Machine

按照以上流程，功能上确实已全部实现，整个流程其实也算结束了。

但在AWS下却还有个易用性问题，即在AWS上创建的VM，无法直接ssh登陆使用，只能通过上面的docker-machine等命令来中转，还是有些不方便——这是因为：**Keypair 是在创建过程中生成的，没法直接使用原有的keypair**。

我按官网提供的和key相关的各类参数组合，都没能试出来。。查了下，对于在amazonec2 driver下使用已有keypair 登陆的[问题](https://github.com/docker/machine/issues/1898)，Docker 社区似乎现在也没解决。

不过我找到了一种曲线救国的方式，来解决ssh 登陆AWS中docker machine的问题。

说来简单，当前的问题，在于无法直接通过在创建docker machine时通过amazonec2 driver 注入已有keypair。换个思路，那么就等它创建好后再注入就好了啊。

### 4.1 准备密钥

还是先按照之前方式，在指定Region里创建好docker machine。如下：

![](images/docker-machine-aws-010.png)

 这时去AWS上检查，会看到确实已经创建了与主机同名的keypair：

![](images/docker-machine-aws-011.png)

继续准备材料，这里直接从原来已有私钥来生成公钥（比如本例之前步骤中创建的 `wj.pem`）。说明可参见[AWS文档](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-key-pairs.html#retrieving-the-public-key)，具体命令如下，路径请自行替换：

```
$ ssh-keygen -y -f */path_to_key_pair/my-key-pair.pem*
  chmod 400 *my-key-pair.pem*
```

### 4.2 注入公钥

好了，下面到了关键的时候了：之前不是提到过，可以通过 `eval` 命令，或者 `docker-machine ssh` 方式，来登陆各个 machine么？所以完全可以通过登陆后，再把已有公钥添加到ssh目录的方式来实现登陆。

之前以展示过eval 命令的方式，下面换用 ssh 的方式来演示。直接登陆aws01，将上一步骤中转化后的公钥，直接拷贝到 authorized_keys 文件内，再重启ssh 服务。具体命令如下：

```
$ docker-machine ssh aws01
  
  (after login..)
  sudo vi /root/.ssh/authorized_keys
  sudo serice ssh restart
```

![](images/docker-machine-aws-012.png)

 重开个session，直接`-i` 指定原私钥进行ssh登陆即可。问题解决，同理可处理 aws02。

![](images/docker-machine-aws-013.png)

 

### 4.3 补充思考

在写这篇文章时，其实又想到了另一种方式：<u>在创建docker-machine 时，是不可以尝试直接通过userdata 把手里已有的公钥注入呢</u>（指定 `--amazonec2-userdata: Path to file with cloud-init user data`）？如果可行的话，能一步解决该问题，而不用登录到每个docker machine 里去修改了。

环境都清理了。这里偷懒，就不实验了。。 ：） 



## 五. 清理环境

环境用完了就要清除。使用后不再需要的 docker主机，毕竟是在AWS上的，建议还是提前释放，避免可能产生的额外费用（*注：真的，血泪史。。*）。可以直接调用ec2tools 命令行来释放：

```
$ ec2-terminate-instances i-0476229a7caa69ab5 --aws-access-key AKIA3VUM6BBZDOQN6S5U --aws-secret-key ***
```

删除时，顺便还能做个实验。看下图：**直接删除底层VM后，docker-machine状态也会同步刷新为Error**。

![](images/docker-machine-aws-007.png)

 最后，删除剩余 docker-machine，完成环境清理：

![](images/docker-machine-aws-008.png)


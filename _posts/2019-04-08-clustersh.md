---
layout: post
title: 利用clustersh在集群中执行shell脚本
date: 2019-04-08
categories: 运维
tags: clustersh 运维 我的开源库
cover: http://upload-images.jianshu.io/upload_images/10192684-410c2f7a7df41a25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
---

# 引言
---

本文将介绍一个叫做`clustersh`的命令行小工具。

如果你想要在许多刚刚装完linux系统的服务器（可能有上百台）
上统一执行某个shell脚本，那么`clustersh`就非常适合你。

“刚刚装完linux操作系统”仅仅是为强调`clustersh`不需要在集群上
安装任何东西，并不是`clustersh`运行的必要条件。

如果你的集群中包含很多不同种类的Linux发行版系统，`clustersh`还可以自动
识别操作系统类型，并为你选择执行相应的脚本。

# Github地址
---

[https://github.com/DQinYuan/clustersh/tree/master/zhdocs](https://github.com/DQinYuan/clustersh/tree/master/zhdocs)

如果觉得有用的，欢迎给个Star。Github上有更加完善的文档

# 使用介绍
---


`clustersh`的使用非常简单。只需任选一台与集群网络联通的linux机器
，在其上按照如下步骤操作.

假设我们的任务是给集群内所有机器统一安装一个nfs客户端，集群内有centos机器和
ubuntu机器

### 下载clustersh

去[下载地址](https://github.com/DQinYuan/clustersh/releases/download/v0.1.0/clustersh)下载clustersh的二进制文件，
然后将其移动到linux的PATH路径下：

```bash
mv clustersh /usr/local/bin
```

尝试运行一下命令：

```bash
clustersh --help
```

可以看到相关的帮助信息


### 准备一个文件夹

之后准备一个文件夹(假设是`~/clustershtest`)：

```bash
mkdir ~/clustershtest
cd ~/clustershtest
```


### 配置ips

在文件夹下创建一个名叫ips的文件:

```bash
touch ips
```

然后在里面配置上集群中所有机器的ip，
假设我的集群中有5台机器，分别是10.10.108.23,10.10.108.71,
10.10.108.72,10.10.108.73,10.10.108.90。
于是我们可以如下配置ips：

```bash
10.10.108.23
10.10.108.71-73
10.10.108.90
```

这里我们使用了`71-73`直接指定了一个范围的ip来简化配置，`clustersh`
目前只支持在ip地址的第四段使用范围指定。

默认情况下配置文件名叫做ips，如果你不想让它叫做ips的花，可以在后面
执行`clustersh`命令是使用`--ips`指定。

### 编写shell脚本

在文件夹下写如下两个脚本：

 - `nfs_centos.sh`,用于在centos机器上安装nfs-client

```bash
#!/bin/sh

yum install -y  nfs-utils
```

 - `nfs_ubuntu.sh`,用于在ubuntu上安装nfs-client
 
```bash
#!/bin/sh

apt install -y nfs-common
```

在开始下一步之前，你最好确保你写的
所有shell脚本在对应操作系统上都测试通过。

### 执行clustersh

最后在文件夹下执行如下命令即可：

```bash
clustersh nfs -U root -P xxxxxx
```

`clustersh`会寻找当前目录下的`ips`文件，将其中
的ip地址读出，依次使用命令行提供的用户名和密码
（这里的用户名为`root`，密码为`xxxxxx`）登陆
这些ip。（在实践中，集群大多有统一的用户名和密码，
所以这里就使用统一的用户名与密码登陆集群了）

`nfs`是shell脚本的**简称**，`clustersh`会根据服务器
的操作系统类型将其扩充为`nfs_操作系统类型.sh`，
如果`nfs_操作系统类型.sh`文件不存在的话则扩充为`nfs.sh`.

比如`clustersh`登陆到一台centos服务器后，发现
操作系统是centos，于是就会尝试寻找`nfs_centos.sh`，
如果有的话就执行它，没有的话则执行`nfs.sh`

如果在集群中还有更多的操作系统类型，请以如下格式命名脚本：

```bash
简称_操作系统类型.sh
```

`clustersh`当前支持识别的操作系统类型有：

|操作系统类型|
|----|
|centos|
|rhel|
|aliyun|
|fedora|
|debian|
|ubuntu|
|raspbian|

你也可以再提供一个`简称.sh`用于在**操作系统类型无法识别**或者
是**没有提供针对该种操作系统的脚本**时执行。

如果你写的脚本对所有操作系统都通用的话，你直接给一个`简称.sh`即可。

### 查看输出

虽然shell脚本在相应的操作系统上都测试通过了，
但是在集群中某些机器运行时还是有可能因为一些
难以预料的原因（比如磁盘空间不足，DNS配置错误等等）
失败，`clustersh`在运行时会打印每台机器运行的成败情况，
对于少数失败的机器，最好手动登陆上去完成操作。


![clustersh fail](http://upload-images.jianshu.io/upload_images/10192684-7f0ccebdbe9db328.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


比如从上面的输出中看到`10.10.108.41`因为某些原因没能成功
执行脚本，最好手工登陆上去操作，不过这种情况属于少数，
并不会花费太多的精力。


[案例源代码](https://github.com/DQinYuan/clustersh/tree/master/examples/nfs)


# 更多特性
---

`cluster`还有一个比较重要的特性就是，在你的shell脚本中，可以使用当前工作目录及其子目录的任意文件，因为这些目录与文件都会被`clustersh`自动拷贝到目标服务器上去。

对于集群中的每一台服务器，`clustersh example`的执行流程如下：

 - 将当前工作及其子目录中的所有文件拷贝到目标服务器上
 - 识别目标服务器类型（centos,ubuntu等）
 - 根据目标服务器类型，查看用户是否提供针对该操作系统的脚本(比如example_centos.sh)，如果提供，则执行；反之，直接执行shname.sh


![clustersh summary](http://upload-images.jianshu.io/upload_images/10192684-410c2f7a7df41a25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)








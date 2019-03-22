---
layout: post
title: Kubernetes:v1.13.4的Worker节点墙内部署文档(CentOS7)
date: 2019-03-22
categories: kubernetes
tags: kubernetes
cover: /assets/img/kubernetes/kubernete.jpg
---



# 引言


Kubernetes的Worker节点的部署要比Master简单不少，核心的是一条join命令，步骤如下。

操作系统是CentOS7，假设使用的用户是root（如果不是root的话，请自行添加"sudo"）



# 步骤一 调节Docker版本

Kubernetes和Docker的版本兼容性一直是令人头疼的问题，之前我的服务器上装的docker是1.13.1版本，安装时会报很多莫名其妙的错误，让人困惑很久，经过测试Docker的1.13.0版本是能够和Kubernetes的v1.13.4版本完美配合的。



**1.查看docker版本**

```python
docker -v
```

如果发现版本刚好是1.13.0的话就可以直接跳到步骤二了

**2.如果发现版本不是1.13.0的话，建议将Docker卸载并且重新安装合适的版本**

```python
# 删除已有的docker
yum erase -y docker*
# 删除已有的docker镜像(防止不同版本的docker无法兼容各自的镜像而产生服务无法启动的错误)
mv -f /var/lib/docker /var/lib/docker.old
# 添加可以下载老版本docker的yum源
cat>> /etc/yum.repos.d/docker-main.repo <<EOF
[docker-main]
name=docker-main
baseurl=http://mirrors.aliyun.com/docker-engine/yum/repo/main/centos/7/
gpgcheck=1
enabled=1
gpgkey=http://mirrors.aliyun.com/docker-engine/yum/gpg
EOF
# 安装docker 1.13.0
yum -y install docker-engine-1.13.0
```

**3. 开启docker服务**
```python
systemctl enable docker
systemctl start docker
```

# 步骤二 安装kubeadm

kubeadm是一个可以方便我们部署kubernetes节点的工具。

```python
# 添加安装kubeadm的yum源
cat>> /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
enable=1
EOF
# 安装kubeadm
yum install -y kubeadm
```

在安装kubeadm的同时，系统还自动帮你安装了kubectl，kubelet和kubernetes-cni

设置kubelet开机自启
```python
systemctl enable kubelet.service
```

注意此时不要尝试通过`systemctl start`来启动`kubelet`，即使尝试了也不会成功，`kubelet`最终必须由kubeadm来负责启动。



# 步骤三 拉取kubernetes的系统镜像

kubernetes在Worker节点上的基础系统，除了kubelet以外，其他都是通过容器部署的。本来这些镜像都可以在Worker启动时自动拉取的，但是因为国内墙的原因，我们事先在本地准备好（通过大佬在[github上建立的镜像](https://github.com/anjia0532/gcr.io_mirror)）。

在shell中执行如下代码：

```python
images=(kube-proxy:v1.13.4 pause:3.1)
for imageName in ${images[@]} ; do
docker pull gcr.azk8s.cn/google-containers/$imageName
docker tag gcr.azk8s.cn/google-containers/$imageName k8s.gcr.io/$imageName
docker rmi gcr.azk8s.cn/google-containers/$imageName
done
```

# 步骤四  禁用swap分区

```python
swapoff -a
```

# 步骤五 修改主机名

集群内的主机名最好能有比较统一的格式，我没有在集群中配置DNS，所以采用的方案就是使用统一的`/etc/hosts`，集群中的主机ip是`10.10.108.xx`，于是就约定其主机名是`hxx`，我生成了一份这样的`hosts`文件，并且用脚本拷贝到了每一台机器上，同时使用下面的命令更改主机名：

(假设ip是`10.10.10.108.73`)

```python
hostnamectl set-hostname "h73"
```


# 步骤六 加入集群

如果Master节点启动时产生的`join`命令还保存着的话，直接使用那条命令就可以了。

否则，可以通过`kubeadm token list`命令查看token。

![tokenlist](/assets/img/kubernetes/tokenlist.png)

其中有一列`EXPIRES`代表token过期的时间，找到一个还没过期的token然后执行以下命令：

```python
kubeadm join Master节点的IP:6443 --token 找到的token --discovery-token-ca-cert-hash sha256:主节点启动时生成的sha256值
```

如果所有token都过期了，可以使用如下命令创建一个：
```python
kubeadm token create
```

默认的token有效期是23h，也可以通过如下命令创建永久有效的token：

```python
kubeadm token create --ttl 0
```

这时就会生成一个永不过期的token

执行完join命令如果没有报错就成功加入集群了，可以在主节点上通过`kubectl get nodes`查看是否有该主机。


# 附录1：如何排错

有时候`join`命令打印出的信息并不详尽，可以使用如下的命令查看更加详尽的报错：

```python
journalctl -xe
```

# 附录2：部署脚本

脚本默认使用root用户执行。

需要根据自身集群的情况修改脚本部分内容

```python
#!/bin/bash

# 指定需要的docker版本
docker_version=1.13.0

# 如果k8s在该台机器上已启动
kube_line_num=`ps -ef | grep kubelet | wc -l`

if [[ $kube_line_num>1 ]]; then
    echo "该台机器已经启动过k8s了,不必重复启动,脚本退出"
    exit
fi


function docker_install() {
    cat>> /etc/yum.repos.d/docker-main.repo <<EOF
[docker-main]
name=docker-main
baseurl=http://mirrors.aliyun.com/docker-engine/yum/repo/main/centos/7/
gpgcheck=1
enabled=1
gpgkey=http://mirrors.aliyun.com/docker-engine/yum/gpg
EOF
    yum erase -y docker*
    rm -rf /var/lib/docker.old
    rm -f /etc/yum.repos.d/docker.repo
    mv -f /var/lib/docker /var/lib/docker.old
    yum -y install docker-engine-1.13.0
}

# check if docker install
function check_and_install()
{
    echo "检查Docker......"
    docker -v
    if [ $? -eq  0 ]; then
        echo "检查到Docker已安装!"
    else
        echo "安装docker环境..."
        docker_install
        echo "安装docker环境...安装完成!"
    fi

    match=`docker -v | grep $docker_version`

    if [[ -z $match ]];
    then
        echo "docker版本不对,需要的版本是 $docker_version ,正在重新安装"
        docker_install
    fi
}

check_and_install

echo "关闭swap"
swapoff -a

cat>> /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF

sysctl --system

echo "准备安装kubeadm"
cat>> /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
enable=1
EOF

yum install -y kubeadm

systemctl enable docker
systemctl enable kubelet.service
systemctl start docker

echo "拉取镜像"
images=(kube-proxy:v1.13.4 pause:3.1)
for imageName in ${images[@]} ; do
docker pull gcr.azk8s.cn/google-containers/$imageName
docker tag gcr.azk8s.cn/google-containers/$imageName k8s.gcr.io/$imageName
docker rmi gcr.azk8s.cn/google-containers/$imageName
done

echo "处理主机名和hosts"
\cp -f ./unihosts /etc/hosts
# 获得ip的最后一段
ip_seg=`ip a | grep -P "10.10.108.\d+" -o | awk -F '.' '{print $4}'`
hostnamectl set-hostname "h$ip_seg"

echo "加入集群"
kubeadm join 10.10.108.73:6443 --token gdvgh1.bnlcjnlcet9l78zo \
--discovery-token-ca-cert-hash sha256:c6516103a534da7d660895080e90f8a77a5f8d74a417a685c0bcdcd748f82365
```

**需要根据集群情况修改的地方有：**

1.提供一份集群统一的`hosts`配置

```python
echo "处理主机名和hosts"
\cp -f ./unihosts /etc/hosts
```

这里的`./unihosts`修改成你准备的统一的集群的主机名配置

2.集群ip特征

```python
# 获得ip的最后一段
ip_seg=`ip a | grep -P "10.10.108.\d+" -o | awk -F '.' '{print $4}'`
```

这里的 `10.10.108.\d+`要换成集群自己的ip模式

3.join命令

```python
kubeadm join 10.10.108.73:6443 --token gdvgh1.bnlcjnlcet9l78zo \
--discovery-token-ca-cert-hash sha256:c6516103a534da7d660895080e90f8a77a5f8d74a417a685c0bcdcd748f82365
```

最后一行的`join`要替换成自己集群的`join`命令
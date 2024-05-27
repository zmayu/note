# 			centos7搭建k8s集群（v1.19.3）

> 使用kubeadm搭建k8s本身是很容易的事情，但是因为国内网络环境原因，通常在搭建过程中有各种网络问题，经过多次实践和仿照前人经验，亲测按照以下方式可傻瓜式搭建k8s集群。本次只部署k8s集群，并搭建Dashboard。后续会持续更新如何部署应用，如何配置ingress，以及使用helm部署应用，请持续关注。

# 一、部署环境

**主机列表：**

| 主机名 | Centos版本 | ip              | docker version | flannel version | 主机配置 | k8s版本 |
| ------ | ---------- | --------------- | -------------- | --------------- | -------- | ------- |
| master | 7.8.2003   | 192.168.214.128 | 19.03.13       | v0.13.0-rc2     | 2C2G     | v1.19.2 |
| node01 | 7.8.2003   | 192.168.214.129 | 19.03.13       | /               | 2C2G     | v1.19.2 |
| node02 | 7.8.2003   | 192.168.214.130 | 19.03.13       | /               | 2C2G     | v1.19.2 |

共有3台服务器 1台master，2台node。

# 二、安装准备工作

### 1.0 关闭防火墙以及永久关闭selinux 

```shell
[root@centos7 ~] systemctl stop firewalld && systemctl disable firewalld 
# 永久关闭selinux 
[root@centos7 ~] sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config;cat /etc/selinux/config 
# 临时关闭selinux 
root@centos7 ~] setenforce 0
```

### 1.1 修改主机名

>三个机器对应分别执行

```shell
[root@centos7 ~] hostnamectl set-hostname master
[root@centos7 ~] hostnamectl set-hostname node01
[root@centos7 ~] hostnamectl set-hostname node02
```

退出重新登陆即可显示新设置的主机名master

### 1.2 修改hosts文件

```shell
[root@master ~] cat >> /etc/hosts << EOF
192.168.175.128    master
192.168.175.129    node01
192.168.175.130    node02
EOF
```

## 2. 验证mac地址uuid

```shell
[root@master ~] cat /sys/class/dmi/id/product_uuid
```

保证各节点mac和uuid唯一

## 3. 禁用swap

### 3.1 临时禁用

```shell
[root@master ~] swapoff -a
```

### 3.2 永久禁用

若需要重启后也生效，在禁用swap后还需修改配置文件/etc/fstab，注释swap

```shell
[root@master ~] sed -i.bak '/swap/s/^/#/' /etc/fstab
```

## 4. 内核参数修改

本文的k8s网络使用flannel，该网络需要设置内核参数bridge-nf-call-iptables=1

### 4.1 内核参数临时修改

```shell
[root@master ~] sysctl net.bridge.bridge-nf-call-iptables=1 
[root@master ~] sysctl net.bridge.bridge-nf-call-ip6tables=1
```

### 4.2 内核参数永久修改

```shell
[root@master ~] cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
[root@master ~] sysctl -p /etc/sysctl.d/k8s.conf
```

## 5. 设置kubernetes源

### 5.1 新增kubernetes源

```shell
[root@master ~] cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

> - [] 中括号中的是repository id，唯一，用来标识不同仓库
> - name 仓库名称，自定义
> - baseurl 仓库地址
> - enable 是否启用该仓库，默认为1表示启用
> - gpgcheck 是否验证从该仓库获得程序包的合法性，1为验证
> - repo_gpgcheck 是否验证元数据的合法性 元数据就是程序包列表，1为验证
> - gpgkey=URL 数字签名的公钥文件所在位置，如果gpgcheck值为1，此处就需要指定gpgkey文件的位置，如果gpgcheck值为0就不需要此项了

### 5.2 更新缓存

```shell
[root@master ~] yum clean all
[root@master ~] yum -y makecache
```

# 三、Docker安装（已安装可跳过）

master node节点都执行本部分操作。

## 1. 安装依赖包

```shell
[root@master ~] yum install -y yum-utils   device-mapper-persistent-data   lvm2
```

## 2. 设置Docker源

```shell
#3.设置镜像的仓库
[root@master ~] yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
#默认是从国外的，不推荐
#推荐使用国内的
[root@master ~] yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

```

## 3. 安装Docker CE

### 3.1 docker安装版本查看

```shell
[root@master ~] yum list docker-ce --showduplicates | sort -r
```

### 3.2 安装docker

```shell
[root@master ~] yum install docker-ce-19.03.13 docker-ce-cli-19.03.13 containerd.io -y
```

指定安装的docker版本为19.03.13

## 4. 启动Docker并开机自启

```shell
[root@master ~] systemctl start docker
[root@master ~] systemctl enable docker
```

## 5. 命令补全

### 5.1 安装bash-completion

```shell
[root@master ~] yum -y install bash-completion
```

### 5.2 加载bash-completion

```shell
[root@master ~] source /etc/profile.d/bash_completion.sh
```

## 6. 镜像加速

由于Docker Hub的服务器在国外，下载镜像会比较慢，可以配置镜像加速器。主要的加速器有：Docker官方提供的中国registry mirror、阿里云加速器、DaoCloud 加速器，本文以阿里加速器配置为例。

### 6.1 登陆阿里云容器模块

登陆地址为：[https://account.aliyun.com](https://cr.console.aliyun.com/) ,使用支付宝账户登录

### 6.2 配置镜像加速器

**配置daemon.json文件**

```shell
[root@master ~] mkdir -p /etc/docker
[root@master ~] tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://23h04een.mirror.aliyuncs.com"]
}
EOF
```

**重启服务**

```shell
[root@master ~] systemctl daemon-reload
[root@master ~] systemctl restart docker
```

加速器配置完成

## 7. 验证

```shell
[root@master ~] docker --version
[root@master ~]  
```

通过查询docker版本和运行容器hello-world来验证docker是否安装成功。

## 8. 修改Cgroup Driver

### 8.1 修改daemon.json

修改daemon.json，新增‘”exec-opts”: [“native.cgroupdriver=systemd”’

```shell
[root@master ~] vim /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://23h04een.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

### 8.2 重新加载docker

```shell
[root@master ~] systemctl daemon-reload
[root@master ~] systemctl restart docker
```

修改cgroupdriver是为了消除告警：
[WARNING IsDockerSystemdCheck]: detected “cgroupfs” as the Docker cgroup driver. The recommended driver is “systemd”. Please follow the guide at <https://kubernetes.io/docs/setup/cri/>

在1.20以上的版本中，kubelet使用的驱动是systemd，不再是cgroupfs，所以如果使用docker作为运行时，就需要将docker的驱动也改为systemd。
# 四、k8s安装

master node节点都执行本部分操作。

## 1. 版本查看

```shell
[root@master ~] yum list kubelet --showduplicates | sort -r
```

本文安装的kubelet版本是1.16.4，该版本支持的docker版本为1.13.1, 17.03, 17.06, 17.09, 18.06, 18.09。

## 2. 安装kubelet、kubeadm和kubectl

### 2.1 安装三个包

```shell
[root@master ~] yum install -y kubelet-1.19.2 kubeadm-1.19.2 kubectl-1.19.2
```

### 2.2 安装包说明

> - **kubelet** 运行在集群所有节点上，用于启动Pod和容器等对象的工具
> - **kubeadm** 用于初始化集群，启动集群的命令工具
> - **kubectl** 用于和集群通信的命令行，通过kubectl可以部署和管理应用，查看各种资源，创建、删除和更新各种组件

### 2.3 启动kubelet

启动kubelet并设置开机启动

```shell
[root@master ~] systemctl enable kubelet && systemctl start kubelet
```

### 2.4 kubectl命令补全

```shell
[root@master ~] echo "source <(kubectl completion bash)" >> ~/.bash_profile
[root@master ~] source .bash_profile 
```

## 3. 下载镜像

### 3.1 镜像下载的脚本

Kubernetes几乎所有的安装组件和Docker镜像都放在goolge自己的网站上,直接访问可能会有网络问题，这里的解决办法是从阿里云镜像仓库下载镜像，拉取到本地以后改回默认的镜像tag。本文通过运行image.sh脚本方式拉取镜像。

```shell
[root@master ~] vim image.sh 
#!/bin/bash
url=registry.aliyuncs.com/google_containers
version=v1.19.2
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]} ; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done
```

url为阿里云镜像仓库地址，version为安装的kubernetes版本。

### 3.2 下载镜像

运行脚本image.sh，下载指定版本的镜像

```shell
[root@master ~] chmod 775 image.sh
[root@master ~] ./image.sh
[root@master ~] docker images
```

# 五、初始化Master

master节点执行本部分操作。

## 1. master初始化

```shell
提前规划好service-cidr和pod-network-cidr网络
[root@master ~] kubeadm init --apiserver-advertise-address=192.168.175.128 --kubernetes-version v1.19.2 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16

说明：
service-cidr=10.96.0.0/12 指定service clusterip的取值范围
pod-network-cider=10.244.0.0/16 指定pod的ip取值范围

```
出现如下内容，即表明k8s的控制面已经安装完成
<img width="1467" alt="image" src="https://github.com/zmayu/note/assets/28054451/1c3401ea-ab79-41b5-919d-1bee4517a201">


记录kubeadm join的输出，后面需要这个命令将node节点和其他control plane节点加入集群中。



**初始化失败：**

如果初始化失败，可执行kubeadm reset后重新初始化

```shell
[root@master ~] kubeadm reset
[root@master ~] rm -rf $HOME/.kube/config
```

## 3. 加载环境变量，使kubelet能够读取并进行apiserver的连接

```shell
[root@master ~] echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
[root@master ~] source .bash_profile
```

本文所有操作都在root用户下执行，若为非root用户，则执行如下操作：

```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## 4. 安装flannel网络

### 4.1 配置host

> 因为国内网络无法解析raw.githubusercontent.com，因此先访问[https://tool.chinaz.com/dns/?type=1&host=raw.githubusercontent.com&ip=](https://tool.chinaz.com/dns/?type=1&host=raw.githubusercontent.com&ip=)查看**raw.githubusercontent.com**的真实IP，并对应修改host

```shell
cat >> /etc/hosts << EOF
151.101.108.133    raw.githubusercontent.com
EOF
```

### 4.2 在master上新建flannel网络

```shell
[root@master ~] wget  https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```
将flannel的配置文件download本地后，编辑文件，将容器网络地址更改为5.1中规划的pod-network-cidr


# 六、node节点加入集群

## 1. node加入集群

```shell
在master节点上执行如下命令，获取节点加入集群的命令
kubeadm token create --print-join-command

---获取内容一般如下
kubeadm join 192.168.175.128:6443 --token 4xpmwx.nw6psmvn9qi4d3cj \
    --discovery-token-ca-cert-hash sha256:c7cbe95a66092c58b4da3ad20874f0fe2b6d6842d28b2762ffc8d36227d7a0a7
 
执行上面的内容即可 
---执行过程中可能会报主机名无法访问到问题
	这个一般是配置主机名的问题，通过命令  ping hostname测试一下
```

## 2. 集群节点查看

```shell
[root@master ~] kubectl get nodes
```

# 七、Dashboard搭建

> Dashboard提供了可以实现集群管理、工作负载、服务发现和负载均衡、存储、字典配置、日志视图等功能。

## 1. 下载yaml

```shell
[root@client ~] wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

如果连接超时，可以多试几次。recommended.yaml已上传，也可以在文末下载。

## 2. 配置yaml

### 2.1 修改镜像地址

```shell
[root@client ~] sed -i 's/kubernetesui/registry.cn-hangzhou.aliyuncs.com\/loong576/g' recommended.yaml
```

由于默认的镜像仓库网络访问不通，故改成阿里镜像

### 2.2 外网访问

```shell
[root@client ~] sed -i '/targetPort: 8443/a\ \ \ \ \ \ nodePort: 30001\n\ \ type: NodePort' recommended.yaml
```

配置NodePort，外部通过https://NodeIp:NodePort 访问Dashboard，此时端口为30001

## 3. 部署访问

### 3.1 部署Dashboard

```shell
[root@client ~] kubectl apply -f recommended.yaml
```

### 3.2 状态查看

```shell
[root@client ~] kubectl get all -n kubernetes-dashboard 
```

### 3.3 令牌生成

> 创建service account并绑定默认cluster-admin管理员集群角色：

```shell
# 创建用户
$ kubectl create serviceaccount dashboard-admin -n kube-system

# 用户授权
$ kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
```

### 3.4 令牌查看

```shell
[root@client ~] kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

### 3.4 访问

**请使用火狐浏览器访问**：https://192.168.175.128:30001/
接受风险，输入令牌，即可图形化管理k8s集群

## 4. 推荐应用
### 4.1 dashboard https://kuboard.cn/

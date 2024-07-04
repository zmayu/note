
[toc]


## 							K8S中的网络

### 一：linux  network namespace

#### 1.1 Network namespace介绍

> network namespace是实现网络虚拟化的重要功能，它能创建多个隔离的网络空间，它们拥有独立的网络栈信息。Linux下的每一个进程都会属于一个特定的network namespace。对于每一个network namespace来说，他会有自己独立的网卡、路由表、ARP表、iptables等和网络相关的资源。新建的网络名称空间与主机默认网络名称空间之间是隔离的。我们平时默认操作的是主机的默认名称空间，默认名称空间是default。  ip命令提供了ip netns exec 子命令可以在对应的network namespace中执行命令。

#### 1.2 使用ip命令操作netnamespace

#####  	1.2.1 ip命令的安装

```shell
ip命令一般都会默认安装
centos安装：yum install iproute iproute-doc
ubuntu安装：apt-get install iproute iproute-doc
route命令安装
yum install net-tools
```

#####     1.2.2  ip netns命令的介绍

```
[root@k8s-master ~]# ip netns help  
Usage: ip netns list
       ip netns add NAME
       ip netns set NAME NETNSID
       ip [-all] netns delete [NAME]
       ip netns identify [PID]
       ip netns pids NAME
       ip [-all] netns exec [NAME] cmd ...
       ip netns monitor
       ip netns list-id
```
> 使用ip netns 创建的network namespace都会出现在/var/run/netns/目录下，如果需要管理其他不是由ip netns 创建的network namespace，只要在这个目录下创建一个指向对应network namespace文件的链接就行。


#####     1.2.3 创建network namespace

```
[root@k8s-master ~]# ip netns add net1
[root@k8s-master ~]# ip netns list
net1
[root@k8s-master ~]# cd /var/run/netns/
[root@k8s-master /var/run/netns]# ls
net1
```

##### 	1.2.4 查看net1 下的网络信息

```
[root@k8s-master /var/run/netns]# ip netns exec net1 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

#### 1.3 Network namespace之间的通信

```
有了不同的network namaspace之后，也就有了网络的隔离，如果他们之间没有办法通信，那么也就毫无意义。要把两个network namespace连接起来，linux提供了veth pair。
```

#####  	1.3.1 veth pair介绍 

```
VETH (Virtual Ethernet) veth-pair
    是Linux提供的的一种特殊的虚拟网络设备，称为虚拟网络接口。它总是成对出现.一端连着协议栈，一端彼此相连着。
    正因为这个特效，它常常充当着一个桥梁，连接各种虚拟网络设备。典型的例子如”两个namespace之间的连接”，“Bridge,OVS之间的连接”，“Docker容器之间的连接”等等，以此构建出非常复杂的虚拟网络结构。
    如果各个namespace之间需要通信，可以通过veth-pair来做桥梁。
根据连接的方式和规模，可以分为“直接相连“，”通过Bridge相连“，“通过OVS相连”
  *直接相连
    直接相连是最简单的方式，一对veth-pair直接将两个namespace连接在一起，那么这两个namespace就可以进行通信了。
  *通过Bridge相连
    Linux Bridge相当于一台交换机，可以中转两个namespace的流量。
  *通过OVS相连
```

##### 	1.3.2 两个network namespace之间的通信

```
前面已经建立一个net1的network namespace，现在再建一个net2
[root@k8s-master]# ip netns add net2
创建veth pari
[root@k8s-master]# ip link add type veth
[root@k8s-master]# ip link
101: veth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 02:ef:75:60:11:be brd ff:ff:ff:ff:ff:ff
102: veth2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 9a:18:28:03:a3:64 brd ff:ff:ff:ff:ff:ff
将veth pair分别放到已经建好的两个namespace里面。
[root@k8s-master]# ip link set veth0 netns net1
[root@k8s-master]# ip link set veth2 netns net2
[root@k8s-master /var/run/netns]# ip netns exec net1 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
101: veth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 02:ef:75:60:11:be brd ff:ff:ff:ff:ff:ff
[root@k8s-master /var/run/netns]# ip netns exec net2 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
12: veth2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 12:7f:29:40:b8:a2 brd ff:ff:ff:ff:ff:ff

```

为这对veth pair配置上ip地址，并启用它们。

```
[root@k8s-master /var/run/netns]# ip netns exec net1 ip link set veth0 up
[root@k8s-master /var/run/netns]# ip netns exec net1 ip addr add 192.168.99.1/24 dev veth0
[root@k8s-master /var/run/netns]# ip netns exec net2 ip link set veth1 up
[root@k8s-master /var/run/netns]# ip netns exec net2 ip addr add 192.168.99.2/24 dev veth2
[root@k8s-master /var/run/netns]# ip netns exec net2 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
102: veth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 9a:18:28:03:a3:64 brd ff:ff:ff:ff:ff:ff
    inet 192.168.99.2/24 scope global veth2
       valid_lft forever preferred_lft forever
    inet6 fe80::9818:28ff:fe03:a364/64 scope link 
       valid_lft forever preferred_lft forever
[root@k8s-master /var/run/netns]# ip netns exec net1 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
101: veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:ef:75:60:11:be brd ff:ff:ff:ff:ff:ff
    inet 192.168.99.1/24 scope global veth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ef:75ff:fe60:11be/64 scope link 
       valid_lft forever preferred_lft forever

测试
[root@k8s-master /var/run/netns]# ip netns exec net1  ping 192.168.99.2
PING 192.168.99.2 (192.168.99.2) 56(84) bytes of data.
64 bytes from 192.168.99.2: icmp_seq=1 ttl=64 time=0.038 ms
64 bytes from 192.168.99.2: icmp_seq=2 ttl=64 time=0.039 ms
64 bytes from 192.168.99.2: icmp_seq=3 ttl=64 time=0.042 ms
```

网络拓扑图

![image](https://github.com/zmayu/note/assets/28054451/0e61521b-5b1e-46af-94be-fb1540b0c965)


#### 1.3 linux  bridge网桥的介绍使用

##### 	1.3.1 介绍

```
Linux Bridge,即linux网桥设备，是Linux提供的一种虚拟网络设备。其工作方式类似于物理的网络交换机设备。Linux Bridge可以工作在二层，也可以工作在三层，默认工作在二层。工作在二层时，可以在同一网络的不同主机间转发以太网报文，一旦你给一个Linux Briage分配了IP地址，也就是开启了该Briage的三层工作模式。
拥有了ip地址便可以充当一个网关
```

##### 	1.3.2 安装

```

cnetos:yum install bridge-utils 
ubuntu:apt-get install bridge-utils
使用：brctl
brctl help
never heard of command [help]
Usage: brctl [commands]
commands:
        addbr           <bridge>                add bridge
        delbr           <bridge>                delete bridge
        addif           <bridge> <device>       add interface to bridge
        delif           <bridge> <device>       delete interface from bridge
        hairpin         <bridge> <port> {on|off}        turn hairpin on/off
        setageing       <bridge> <time>         set ageing time
        setbridgeprio   <bridge> <prio>         set bridge priority
        setfd           <bridge> <time>         set bridge forward delay
        sethello        <bridge> <time>         set hello time
        setmaxage       <bridge> <time>         set max message age
        setpathcost     <bridge> <port> <cost>  set path cost
        setportprio     <bridge> <port> <prio>  set port priority
        show            [ <bridge> ]            show a list of bridges
        showmacs        <bridge>                show a list of mac addrs
        showstp         <bridge>                show bridge stp info
        stp             <bridge> {on|off}       turn stp on/off
```

##### 	1.3.3 说明

```
通过ip命令建桥和brctl建桥的区别
```



#### 1.4 多个network namespace之间的通信（相同网段）

##### 	1.4.1 说明

```
虽然veth pair可以实现两个network namespace之间的通信，当多个network namespace之间通信时就需要借助路由器和交换机，因为这里只是同个网络，所以只用到交换机的功能。
这里我们使用交换机网桥链接多个network namespace。
```

##### 	1.4.2 新建一个bridge

```
ip link add br0 type bridge 或者用 brctl addbr br0
查看链接到bridge上的veth pair
bridge link
```

##### 	1.4.3 新建network namespace

```
ip netns add netns1
ip netns add netns2
```

##### 	1.4.4 新建veth pair

```
ip link add tap1 type veth peer name tap1_peer    # ip link add type veth
ip link add tap2 type veth peer name tap2_peer
```

##### 	1.4.5 将pair加入到对应的命名空间

```
ip link set dev tap1 netns netns1
ip link set dev tap2 netns netns2
```

##### 	1.4.6 将对端的pair加入到bridge

```
ip link set dev tap1_peer master br0
ip link set dev tap2_peer master br0
```

##### 	1.4.7 为pair配置地址

```
ip netns exec netns1 ip addr add 192.168.27.1/24 dev tap1
ip netns exec netns2 ip addr add 192.168.27.2/24 dev tap2
```

##### 	1.4.8 启动

```
ip link set dev br0 up
ip link set dev tap1_peer up
ip netns exec netns1 ip link set dev tap1 up
ip link set dev tap2_peer up
ip netns exec netns2 ip link set dev tap2 up

```

##### 	1.4.9 测试

```
ip netns exec netns1 ping 192.168.27.2
```

##### 	1.4.10 为bridge设置ip,此时br0就具有三层设备的功能

```
ip addr add 192.168.27.3/24 dev br0
```

##### 	1.4.11 bro所处的network namespace是主机默认的，此时直接在主机做ping测试。

```
ping 192.168.27.1
ping 192.168.27.2

route说明：目的地址的网络地址为192.168.27.0的流量发送到bro接口，由br0接口进行处理。
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.27.0    0.0.0.0         255.255.255.0   U     0      0        0 br0

```

##### 1.4.12 网络架构图

![image](https://github.com/zmayu/note/assets/28054451/b15e754c-498f-491f-8ecf-a07c2d518005)


#### 1.4 多个network namespace之间的通信（不同网段）

#####  	1.4.1 linux配置开启路由转发功能，自己可以充当一个路由器进行三层流量的转发

```shell
/proc/sys/net/ipv4/ip_forward为1
```

##### 	1.4.2 创建network namespace

```
ip netns add ns1
ip netns add ns2
```

##### 	1.4.3 创建veth pair 

```
ip link add tap1 type veth peer name tap1_peer
ip link add tap1 type veth peer name tap1_peer
```

##### 	1.4.4 将veth的tap端移动到network namespace

```
ip link set tap1 netns ns1
ip link set tap2 netns ns2
```

##### 	1.4.5 为veth的tap端配置ip

```
ip netns exec ns1 ip addr add local 10.10.10.10/24 dev tap1
ip netns exec ns2 ip addr add local 10.10.20.10/24 dev tap2
```

##### 	1.4.6 为veth的peer端配置ip

```
这一端实际上是在主机默认的network namespace下
ip addr add local 10.10.10.1/24 dev tap1_peer
ip addr add local 10.10.20.1/24 dev tap2_peer
```

##### 	1.4.7 配置静态路由（路由的优先级：静态路由高于动态路由）

```
ip netns exec ns1 route add -net 10.10.20.0 netmask 255.255.255.0 gw 10.10.10.1
ip netns exec ns2 route add -net 10.10.10.0 netmask 255.255.255.0 gw 10.10.20.1
```

##### 	1.4.8 测试，进入到ns1或ns2 ping对对方的ip地址

```
ip netns exec ns1 ping 10.10.20.10
```

##### 1.4.9 网络架构图

![image](https://github.com/zmayu/note/assets/28054451/005db7e8-e019-432a-84c4-7783d20a7471)


### 二： Docker中的网络

#### 	2.1 容器网络简介和基本原理

```
要实现网络通信，机器需要至少一个网络接口（物理接口或者虚拟接口）来收发数据包。同一子网内通信需要交换机，不同子网通信需要路由机制。
Docker中的网络接口默认都是虚拟的接口。虚拟接口的优势之一是转发效率较高。linux通过内核中进行数据复制来实现虚拟接口之间的数据转发，发送接口发送缓存中的数据包被直接复制到接受接口的接受缓存中。对于本地系统和容器内系统来看就像是一个正常的以太网卡，只是它不需要真正同外部网络设备通信，速度要快很多。
容器网络主要解决两大核心问题：一是容器的ip地址分配，二是容器之间的互相通信。
实现容器跨主机通信的最简单方式就是直接使用host网络，此时容器ip就是宿主机的ip,复用宿主机的网络协议栈以及underlay网络，原来的主机能够通信，容器自然也能够通信，这样最大的问题就是端口冲突的问题。
一般容器会配置与宿主机不一样的属于自己的ip地址。由于是容器自己配置的ip，underlay平面的底层网络设备如交换机、路由器等完全不感知这些ip的存在，也就导致容器的ip不能直接路由出去实现跨主机通信。
解决以上问题两个思路：
	思路一：修改底层网络设备配置，加入容器网络ip地址的管理，修改路由器网关等，该方式主要和SDN结合。
	思路二：完全不修改底层网络设备配置，复用原有的underlay平面网络，解决容器跨主机通信，主要有如下两种方式。
			1.Overlay隧道传输：把容器的数据包封装到原主机网络的三层或者四层数据包中，然后使用原来的网络使用ip或者tcp/udp传输到目标主机，目标主机再拆包转发给容器。Overlay隧道如Vxlan、ipip等。目前使用Overlay技术的主流容器网络如Falnnel、Weave等。
			2.修改主机路由。把容器网络驾到主机路由表中，把主机当作容器网关，通过路由规则转发到指定的主机，实现容器的三层互通。目前通过路由技术实现容器跨主机通信的网络如Flannel host-gw、Calico等。
```

#### 	2.2 iptables策略说明

```

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:8080

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:8080

路由规则：
	snat: 原网络地址转换，其作用是将ip数据包的源地址转换成另外一个地址。俗称ip地址欺骗或伪装（masquerade）
	dnat: 目的地址转换
```

#### 	2.3 容器网络类型

```
docker run的时候通过-net参数指定容器的网络配置

--net=bridge 这个是默认值，连接到默认的网桥。
--net=host 告诉 Docker 不要将容器网络放到隔离的命名空间中，即不要容器化容器内的网络。此时容器使用本地主机的网络，它拥有完全的本地主机接口访问权限。容器进程可以跟主机其它 root 进程一样可以打开低范围的端口，可以访问本地网络服务比如 D-bus，还可以让容器做一些影响整个主机系统的事情，比如重启主机。因此使用这个选项的时候要非常小心。如果进一步的使用 --privileged=true，容器会被允许直接配置主机的网络堆栈。
--net=container:NAME_or_ID 让 Docker 将新建容器的进程放到一个已存在容器的网络栈中，新容器进程有自己的文件系统、进程列表和资源限制，但会和已存在的容器共享 IP 地址和端口等网络资源，两者进程可以直接通过 lo 环回接口通信。
	这种方式在K8S中应用，每一个pod都会包含两个两个container，一个业务container，一个pause container，这两个container共享一个网络协议栈。
--net=none 让 Docker 将新容器放到隔离的网络栈中，但是不进行网络配置。之后，用户可以自己进行配置。

/var/run/docker/netns/
```

#### 	2.4 linux bridge 网桥自学习

```
linux网桥是软件实现，但是与硬件网桥类似，linux网桥维护了一个2层转发表（也称为mac学习表，转发数据库，或者仅仅称为FDB）,它跟踪了mac地址与端口的对应关系。当一个网桥在端口N收到一个包时（源MAC地址为X）,它在FDB中记录为MAC地址X可以从端口N到达。这样的话，以后当网桥需要转发一个包到地址X时，它就可以从FDB查询知道转发到哪里。构建一个FDB常常称之为“MAC学习”或仅仅称为“学习”过程。
```



#### 	2.5 实例应用中的网络设置

- K8S pod中的container

```
docker inspect containerid
业务pod
	pause
	"SandboxKey": "/var/run/docker/netns/74eb60c5dfd2",    //network namespace
    "Networks": {
         "none": {     																   //网络模型使用none
         		........
         }
  业务容器
  	   "SandboxKey": "",                                  //这里都是空
       "Networks": {}                                     //空
kube-apiserver
	pasuse
 		"SandboxKey": "/var/run/docker/netns/default",        //使用的是default netns
 		"Networks": {
      	"host": {                                         //网络模型使用host
          .........
      }
   kube-apiserver容器
     "SandboxKey": "",                                   //这里都是空
     "Networks": {}                                      //空
```

- 直接使用 docker创建container

```
"SandboxKey": "/var/run/docker/netns/c8c3b1a99c06",         //network namespace 
"Gateway": "172.17.0.1",
"IPAddress": "172.17.0.2",
"Networks": {
   "bridge": {                                               //网络模型默认使用bridge
   		........
   }
```



### 三：K8S中的网络

#### 

#### 3.1 channel

##### 3.1.1 简介

- Flannel是CoreOS团队针对Kubernetes设计的一个网络规划服务，简单来说，它的功能是让集群中的不同节点主机创建的Docker容器都具有全集群唯一的虚拟IP地址。

  在默认的Docker配置中，每个节点上的Docker服务会分别负责所在节点容器的IP分配。这样导致的一个问题是，不同节点上容器可能获得相同的内外IP地址。并使这些容器之间能够之间通过IP地址相互找到，也就是相互ping通。

  Flannel的设计目的就是为集群中的所有节点重新规划IP地址的使用规则，从而使得不同节点上的容器能够获得“同属一个内网”且”不重复的”IP地址，并让属于不同节点上的容器能够直接通过内网IP通信。

  Flannel实质上是一种“覆盖网络(overlaynetwork)”，也就是将数据包装在另一种网络包里面进行路由转发和通信，目前已经支持udp、vxlan、host-gw、aws-vpc、gce和alloc路由等数据转发方式，默认的节点间数据通信方式是UDP转发。	

#### 	

##### 	3.1.2特点

- 使集群中的不同Node主机创建的Docker容器都具有全集群唯一的虚拟IP地址。

- 建立一个覆盖网络（overlay network），通过这个覆盖网络，将数据包原封不动的传递到目标容器。覆盖网络是建立在另一个网络之上并由其基础设施支持的虚拟网络。覆盖网络通过将一个分组封装在另一个分组内来将网络服务与底层基础设施分离。在将封装的数据包转发到端点后，将其解封装。

- 创建一个新的虚拟网卡flannel0接收docker网桥的数据，通过维护路由表，对接收到的数据进行封包和转发（vxlan）。

- etcd保证了所有node上flanned所看到的配置是一致的。同时每个node上的flanned监听etcd上的数据变化，实时感知集群中node的变化。

- flannel默认使用8285端口作为UDP封装报文的端口，Vxlan使用8472端口。

##### 	3.1.3Flannel对网络要求提出的解决办法

- flannel利用Kubernetes API或者etcd用于存储整个集群的网络配置，根据配置记录集群使用的网段。

- flannel在每个主机中运行flanneld作为agent，它会为所在主机从集群的网络地址空间中，获取一个小的网段subnet，本主机内所有容器的IP地址都将从中分配。



##### 	3.1.4 实验

- 架构图

![image](https://github.com/zmayu/note/assets/28054451/2c390cfa-a807-4156-8e78-c8e747d82473)

  **说明**：Node1和Node2是k8s集群中的两个节点，现在Node2中启动pod-n(nginx容器，监听80端口)。在Node1中启动pod-m,并在pod-m访问pod-n。

  **pod-m和pod-n通信的过程**

  1. pod-m中产生的请求信息经过路由策略将数据包发送到cni0.
  2. cni0根据节点的路由策略将数据包发送到隧道设备flannel.1中。
  3. Flannel.d从flannel.1中读取数据包，从数据包中获取目的ip,再从etcd中读取虚拟网络信息，获取到目的ip对应的MAC地址和node2节点的ip,重新构建数据包，构建完成的数据包data由VXLAN+MAC(pod-n容器的mac地址)+IP(pod-n容器的IP)+tcp(nginx80端口)+http payload数据组成。
  4. flannel.d将构建完成的data数据包通过eth1物理网卡发送目的node2节点。
  5. 数据从node2节点的eth1进入，flannel.d监听8472端口，读取到了data数据，flannel.d将vxlan header数据处理后写入到flannel.1，flannel.1根据节点路由策略将数据包发送cni0,数据包最终经过cni0网桥到达pod-n

​	

- master节点分配

  ```
  [root@VM-144-206-centos /run/flannel]# cat subnet.env 
  FLANNEL_NETWORK=10.244.0.0/16
  FLANNEL_SUBNET=10.244.0.1/24
  FLANNEL_MTU=1450
  FLANNEL_IPMASQ=true
  ```

- node1节点IP分配

  ```
  [root@VM-144-172-centos//]# cat /run/flannel/subnet.env 
  FLANNEL_NETWORK=10.244.0.0/16
  FLANNEL_SUBNET=10.244.1.1/24
  FLANNEL_MTU=1450
  FLANNEL_IPMASQ=true
  ```

- node2节点IP分配

  ```
  [root@VM-144-168-centos ~]# cat /run/flannel/subnet.env 
  FLANNEL_NETWORK=10.244.0.0/16
  FLANNEL_SUBNET=10.244.2.1/24
  FLANNEL_MTU=1450
  FLANNEL_IPMASQ=true
  ```

  ![image](https://github.com/zmayu/note/assets/28054451/c452df62-fb28-40bb-a807-74572c354023)

- 组件解释

  **cni0**:网桥设备，每创建一个pod都会创建一对veth pair,其中一端放在pod,即pod中的eth0,另一段挂载到cni0上，从pod中发出的流量都会到达cni0网桥

  ```
  cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
      link/ether d6:2d:00:58:e3:08 brd ff:ff:ff:ff:ff:ff
      inet 10.244.1.1/24 brd 10.244.1.255 scope global cni0
      valid_lft forever preferred_lft forever
  cni0将获取到本子网中的第一个ip    
  ```

  **flannel.1**:overlay网络设备,用来进行 vxlan 报文的处理（封包和解包）。不同node之间的pod数据流量都从overlay设备以隧道的形式发送到对端。

  ```
  flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
      link/ether b2:c5:b5:fe:50:b8 brd ff:ff:ff:ff:ff:ff
      inet 10.244.1.0/32 scope global flannel.1
      valid_lft forever preferred_lft forever
  ```

  **flanneld**: 在每个主机中运行flanneld作为agent，它会为所在主机从集群的网络地址空间中，获取一个小的网段subnet，本主机内所有容器的IP地址都将从中分配。同时Flanneld监听K8s集群数据库，为flannel.1设备提供封装数据时必要的mac，ip等网络数据信息。当flanneld启动时，它会在主机上创建一个flannel.1接口，并配置它使用VXLAN。这个配置过程告诉内核，当有数据包发送到flannel.1接口时，应该使用VXLAN协议将其封装，同样，当收到一个VXLAN报文时，内核应该将其解封装
  	

  ​**疑惑**：这里对于flannel.1进行vxlan报文的封包和解包表示不敢认同，首先flannel.1即是一个tun/tap设备，这个设备负责是流量的传输，真正处理vxlan报文的应该是flanneld。

  **解答**：2024.05.17 这里确实是flannel.1进行vxlan报文的封包和解包，flannel.1是一个虚拟的网卡设备。一般物理网卡的数据的读写方是网线侧和内核侧，而现在flanneld充当了网线侧的功能。一般内核侧把数据发到网卡，网卡把数据沿着网线侧发走。而这里内核把数据发到虚拟网卡flannel.1，最后是flanneld读走了数据，这个数据就是网络包。
  **解答** 2024.07.3 flannel.1是一个虚拟网卡，现在从cni0网桥依据路由策略数据包路由到了flannel.1网卡中，flannel.1网卡创建的时候配置了VXLAN模块，那么从flannel.1网卡中出去的数据包在最外层需要VXLAN协议进行封装，经过VXLNA模块封装后，从flannel.1网卡出去的数据包就是【VXLAN|MAC|IP|TCP|HTTP】,正常来说网卡后面接的是网线，从网线中发出去的就是【VXLAN|MAC|IP|TCP|HTTP】数据包，但是现在flannel.1网卡后面接入的网线变成了flanneld，flanneld从flannel.1网卡中收到了数据包【VXLAN|MAC|IP|TCP|HTTP】，现在flanneld需要将数据包【VXLAN|MAC|IP|TCP|HTTP】作为一个业务数据包基于UDP协议通过node节点的网卡eth1发送到node2节点flaneld监听的8472端口中。node2的flanneld接收到业务数据包【VXLAN|MAC|IP|TCP|HTTP】，然后再将这个业务数据包发送给node2的flannel.1处理，flannel.1网卡在创建的时候配置了VXLAN模块，所以现会处理掉数据的外层VXLAN，然后再处理【MAC|IP|TCP|HTTP】一个正常的http网络数据包，flannel.1连接的是cni0网桥，按照路由策略将数据包【MAC|IP|TCP|HTTP】路由到node2的pod-n容器的eth0网卡中。	



- node节点的路由表信息

  **Node1中的路由表**

  ```
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
  10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
  10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
  
  cni0是虚拟网桥，目的IP的网络地址为10.244.1.0/24的请求通过cni0网桥发送到Node1中的容器。
  目的IP的网络地址为10.244.2.0/24和10.244.0.0/24的请求发送flannel.1 
  ```

  **Node2中的路由表**

  ```
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
  10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
  10.244.2.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
  cni0是虚拟网桥，目的IP的网络地址为10.22.2.0/24的请求通过cni0网桥发送到Node2中的容器。
  目的IP的网络地址为10.244.0.0/24和10.244.1.0/244的请求发送到flannel.1
  ```

  

- tcpdump抓包数据分析
  在node1节点9.135.91.57上抓包分析
  tcpdump -i eth1  port 8472 and  host 9.135.91.57 -w /tmp/tcpdump_save.cap



**最外层协议信息**

![image](https://github.com/zmayu/note/assets/28054451/471ad3b6-23b2-491a-b448-801b66259d3f)

```
MAC地址：
	源地址：   52 24 00 ad 41 39
	目的地址： Fe ee cf 97 ef 06
IP地址
	源IP:     09 87 5b 39 > 9.135.91.57
	目的IP:   09 87 90 a8  > 9.135.144.168 
UDP端口
	源端口：   9b ef  > 39919
  目的端口：  21 18 > 8472

	
```

**内层协议信息**

![image](https://github.com/zmayu/note/assets/28054451/8602c219-521a-4935-b895-189a5c76e514)

```
VXLAN header
	:08 00 00 00 00 00 01 00
VXLAN协议报文组成说明
	VXLAN Header由8个byte组成
	Flags:占 8bits，具体是 RRRRIRRR，其中 I 必须设置为 1，表示是是一个合法的 VxLAN ID，其它 7bits 的 R 作为保留字段并且设置为 0。
	VNI（VxLAN Network Identifie）：占 24bits，VxLAN 的 ID 标识，这个字段就对比于 12bits 的 VLAN ID，支持 ID 个数扩展为 2^24 = 16777216，约等于 16M 个。
	Reserved：有两个字段，分别占 24 bits 和 8 bits，作为保留字段并且设置为 0。


MAC地址：
	 源地址: 		 2a fe 82 35 16 30
   目的地址: 		b2 c5 b5 fe 50 b8
```

![image](https://github.com/zmayu/note/assets/28054451/e2688196-4ae7-42a6-b6ca-59e6d1538157)

```
IP地址：
	 源IP:     0a f4 01 aa  -->  10.244.1.170
	 目的IP:   0a f4 02 09   --> 10.244.2.9
```

![image](https://github.com/zmayu/note/assets/28054451/84b8a54c-f274-4ab2-aaf6-a0981cba6e12)

```
TCP端口号：
		源端口号：		00 50   >  80
		目的端口号：	8b 58  >  35672
```



##### 3.1.4不同后端封包策略

1.Hostgw

​	hostgw是最简单的backend，它的原理非常简单，直接添加路由，将目的主机当做网关，直接路由原始封包。

​	因为Machine A和Machine B处于同一个子网内，它们原本就能直接互相访问。因此最简单的方法是：在Machine A中的容器要访问Machine B的容器时，我们可以将Machine B看成是网关，当有封包的目的地址在subnet 10.1.16.0/24范围内时，就将其直接转发至B即可。

![image](https://github.com/zmayu/note/assets/28054451/b2889f27-c308-4929-8c8a-597591d5a381)

```
图中那条红色标记的路由就能完成：我们从etcd中监听到一个EventAdded事件subnet为10.1.15.0/24被分配给主机Machine A Public IP 192.168.0.100，hostgw要做的工作就是在本主机上添加一条目的地址为10.1.15.0/24，网关地址为192.168.0.100，输出设备为上文中选择的集群间交互的网卡即可。对于EventRemoved事件，只需删除对应的路由。

​	这种方式是最简单也是最有效率的为什么不使用这种方式呢？
```

2.UDP

```
		当backend为hostgw时，主机之间传输的就是原始的容器网络封包，封包中的源IP地址和目的IP地址都为容器所有。这种方法有一定的限制，就是要求所有的主机都在一个子网内，即二层可达，否则就无法将目的主机当做网关，直接路由。

而udp类型backend的基本思想是：既然主机之间是可以相互通信的（并不要求主机在一个子网中），那么我们为什么不能将容器的网络封包作为负载数据在集群的主机之间进行传输呢？这就是所谓的overlay。具体过程如图所示
```

![image](https://github.com/zmayu/note/assets/28054451/18aedaa9-70f4-45df-91d2-f85a2be383ec)

```
当容器10.1.15.2/24要和容器10.1.20.3/24通信时：

1）因为该封包的目的地不在本主机subnet内，因此封包会首先通过网桥转发到主机中。最终在主机上经过路由匹配，进入如图的网卡flannel0。需要注意的是flannel0是一个tun设备，它是一种工作在三层的虚拟网络设备，而flanneld是一个proxy，它会监听flannel0并转发流量。当封包进入flannel0时，flanneld就可以从flannel0中将封包读出，由于flannel0是三层设备，所以读出的封包仅仅包含IP层的报头及其负载。最后flanneld会将获取的封包作为负载数据，通过udp socket发往目的主机。同时，在目的主机的flanneld会监听Public IP所在的设备，从中读取udp封包的负载，并将其放入flannel0设备内。由此，容器网络封包到达目的主机，之后就可以通过网桥转发到目的容器了。

最后和hostgw不同的是，udp backend并不会将从etcd中监听到的事件里所包含的lease信息作为路由写入主机中。每当收到一个EventAdded事件，flanneld都会将其中的subnet和Public IP保存在一个数组中，用于转发封包时进行查询，找到目的主机的Public IP作为udp封包的目的地址。
```



3.VXLAN

```
首先，我们对vxlan的基本原理进行简单的叙述。从下图所示的封包结构来看，vxlan和上文提到的udp backend的封包结构是非常类似的，不同之处是多了一个vxlan header，以及原始报文中多了个二层的报头。
```

![image](https://github.com/zmayu/note/assets/28054451/9df21037-8d8c-438e-a8cf-24238ebaebf8)

```
如上图所示，当主机B加入flannel网络时，和其他所有backend一样，它会将自己的subnet 10.1.16.0/24和Public IP 192.168.0.101写入etcd中，和其他backend不一样的是，它还会将vtep设备flannel.1的mac地址也写入etcd中。

之后，主机A会得到EventAdded事件，并从中获取上文中B添加至etcd的各种信息。这个时候，它会在本机上添加三条信息：

1) 路由信息：所有通往目的地址10.1.16.0/24的封包都通过vtep设备flannel.1设备发出，发往的网关地址为10.1.16.0，即主机B中的flannel.1设备。

2) fdb信息：MAC地址为MAC B的封包，都将通过vxlan首先发往目的地址192.168.0.101，即主机B

3）arp信息：网关地址10.1.16.0的地址为MAC B

现在有一个容器网络封包要从A发往容器B，和其他backend中的场景一样，封包首先通过网桥转发到主机A中。此时通过，查找路由表，该封包应当通过设备flannel.1发往网关10.1.16.0。通过进一步查找arp表，我们知道目的地址10.1.16.0的mac地址为MAC B。到现在为止，vxlan负载部分的数据已经封装完成。由于flannel.1是vtep设备，会对通过它发出的数据进行vxlan封装（这一步是由内核完成的，相当于udp backend中的proxy），那么该vxlan封包外层的目的地址IP地址该如何获取呢？事实上，对于目的mac地址为MAC B的封包，通过查询fdb，我们就能知道目的主机的IP地址为192.168.0.101。

最后，封包到达主机B的eth0，通过内核的vxlan模块解包，容器数据封包将到达vxlan设备flannel.1，封包的目的以太网地址和flannel.1的以太网地址相等，三层封包最终将进入主机B并通过路由转发达到目的容器。

事实上，flannel只使用了vxlan的部分功能，由于VNI被固定为1，本质上工作方式和udp backend是类似的，区别无非是将udp的proxy换成了内核中的vxlan处理模块。而原始负载由三层扩展到了二层，但是这对三层网络方案flannel是没有意义的，这么做也仅仅只是为了适配vxlan的模型。
```

*参考资料*
   * <a href="https://support.huawei.com/enterprise/zh/doc/EDOC1100087027">什么是VXLAN</a>
   * <a href="https://info.support.huawei.com/info-finder/encyclopedia/zh/VLAN.html">什么是VLAN</a>
   * <a herf="https://cloud.tencent.com/developer/article/1412795">图文详解VLAN</a>
 
 

*比较*

  vxlan和UDP的区别是vxlan是**内核封包**，而UDP是flanneld用**户态程序封包**，所以UDP的方式性能会稍差；


##### 3.1.5存在的问题

1.不支持pod之间的网络隔离。Flannel设计思想是将所有的pod都放在一个大的二层网络中，所以pod之间没有隔离策略。

2.设备复杂，效率不高。Flannel模型下有三种设备，数量经过多种设备的封装、解析，势必会造成传输效率的下降。

```
对于flannel网络介绍的文章也很多，其中有一个点有明显的分歧，就是对于flanned的作用。分歧点在于：使用UDP作为后端网络时，flanned会将flanne.1设备的流量经过自己的处理发送给对端的flanned。但是在分析vxlan作为后端网络时明显不是这么做的，在vxlan中flanned作用是获取必要的mac地址，ip地址信息，没有直接处理数据流。
```



#### 3.2 calico

##### 	3.2.1 简介

```
Calico 是一种容器之间互通的网络方案。在虚拟化平台中，比如 OpenStack、Docker 等都需要实现 workloads 之间互连，但同时也需要对容器做隔离控制，就像在 Internet 中的服务仅开放80端口、公有云的多租户一样，提供隔离和管控机制。而在多数的虚拟化平台实现中，通常都使用二层隔离技术来实现容器的网络，这些二层的技术有一些弊端，比如需要依赖 VLAN、bridge 和隧道等技术，其中 bridge 带来了复杂性，vlan 隔离和 tunnel 隧道则消耗更多的资源并对物理环境有要求，随着网络规模的增大，整体会变得越加复杂。我们尝试把 Host 当作 Internet 中的路由器，同样使用 BGP 同步路由，并使用 iptables 来做安全访问策略，最终设计出了 Calico 方案。
```

- 使用场景：k8s环境中的pod之间需要隔离 ——暂时不太明白

- 设计思想：Calico 不使用隧道或 NAT 来实现转发，而是巧妙的把所有二三层流量转换成三层流量，并通过 host 上路由配置完成跨 Host 转发。

- 设计优势

  ```
  1.更优的资源利用
  二层网络通讯需要依赖广播消息机制，广播消息的开销与 host 的数量呈指数级增长，Calico 使用的三层路由方法，则完全抑制了二层广播，减少了资源开销。
  另外，二层网络使用 VLAN 隔离技术，天生有 4096 个规格限制，即便可以使用 vxlan 解决，但 vxlan 又带来了隧道开销的新问题。而 Calico 不使用 vlan 或 vxlan 技术，使资源利用率更高。
  2.可扩展性
  Calico 使用与 Internet 类似的方案，Internet 的网络比任何数据中心都大，Calico 同样天然具有可扩展性。
  3.简单而更容易 debug
  因为没有隧道，意味着 workloads 之间路径更短更简单，配置更少，在 host 上更容易进行 debug 调试。
  4.更少的依赖
  Calico 仅依赖三层路由可达。
  5.可适配性
  Calico 较少的依赖性使它能适配所有 VM、Container、白盒或者混合环境场景。
  ```

  ##### 3.2.2calico的架构









##### 	3.2.3calico网络Node之间的两种网络

- IPIP

  从字面来理解，就是把一个IP数据包又套在一个IP包里，即把 IP 层封装到 IP 层的一个 tunnel。它的作用其实基本上就相当于一个基于IP层的网桥！一般来说，普通的网桥是基于mac层的，根本不需 IP，而这个 ipip 则是通过两端的路由做一个 tunnel，把两个本来不通的网络通过点对点连接起来。

- BGP

  边界网关协议（Border Gateway Protocol, BGP）是互联网上一个核心的去中心化自治路由协议。它通过维护IP路由表或‘前缀’表来实现自治系统（AS）之间的可达性，属于矢量路由协议。BGP不使用传统的内部网关协议（IGP）的指标，而使用基于路径、网络策略或规则集来决定路由。因此，它更适合被称为矢量性协议，而不是路由协议。BGP，通俗的讲就是讲接入到机房的多条线路（如电信、联通、移动等）融合为一体，实现多线单IP，BGP 机房的优点：服务器只需要设置一个IP地址，最佳访问路由是由网络上的骨干路由器根据路由跳数与其它技术指标来确定的，不会占用服务器的任何系统。



##### 3.2.4 IPIP工作模式

- 测试	

- 容器信息

   Node1

  - 网卡信息

  ```
  [root@my-nginx-5967cb6fd8-m4tr9 /]# ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
      link/ether 4e:ad:b9:ae:ef:47 brd ff:ff:ff:ff:ff:ff
      inet 192.168.135.163/32 brd 192.168.135.163 scope global eth0
         valid_lft forever preferred_lft forever
  **
  	看到eth0的IP是/32主机地址，表示将容器A作为一个单点的局域网    
  yum install -y ethtool
  ethtool -S eth0  //可以看到veth pair的对端
  ```

  - 路由信息

  ```
  [root@my-nginx-5967cb6fd8-m4tr9 /]# route
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  default         gateway         0.0.0.0         UG    0      0        0 eth0
  gateway         0.0.0.0         255.255.255.255 UH    0      0        0 eth0
  [root@my-nginx-5967cb6fd8-m4tr9 /]# ping gateway
  PING gateway (169.254.1.1) 56(84) bytes of data.
  路由信息看到网关地址为169.254.1.1，这明显是一个公网IP，在我们的node节点以及容器中都找到不到一个网卡来对应这个ip,那这是怎么回事呢？
  当一个数据包的目的地址不是本机时，就会查询路由表，从路由表中查到网关后，它首先会通过 ARP 获得网关的 MAC 地址，然后在发出的网络数据包中将目标 MAC 改为网关的 MAC，而网关的 IP 地址不会出现在任何网络包头中。也就是说，没有人在乎这个 IP 地址究竟是什么，只要能找到对应的 MAC 地址，能响应 ARP 就行了。
  
  ```

  - ***知识点--代理ARP和ARP***

    因为容器网卡eth0的ip地址为192.168.135.163/32，表示这个容器是一个单点的局域网，没有网关。

  ```
  第一，代理ARP仅仅是正常ARP的一个拓展使用，是可选项而不是必要项；
  第二：代理ARP有特定的应用场景，与网关/路由的设置有直接关系：当电脑没有网关/路由功能时，并且需要跨网站通信时，则会触发代理ARP。换句话说，如果有网关/路由功能，则不需要代理ARP；
  第三：正常环境下，当用户接入网络时，都会通过DHCP协议或手工配置的方式得到IP和网关信息（所以不需要代理ARP）。
  arp和代理arp的使用场景
  ①电脑没有网关时，ARP直接询问目标IP对应的MAC地址（跨网段），采用代理ARP；
  ②电脑有网关时，ARP只需询问网关IP对应的MAC地址（同网段），采用正常ARP；
  ③无论是正常ARP还是代理ARP，电脑最终都拿到同一个目标MAC地址：网关MAC。
  
  ①当电脑没有网关（采用代理ARP）时："跨网段访问谁，就问谁的MAC"
  ②当电脑有网关（采用正常ARP）时："跨网段访问谁，都问网关的MAC"
  ③无论哪种ARP，跨网段通信时，发送方请求得到的目标MAC地址都是网关MAC。
  ```

  - 查看是否开启代理ARP

  ```
  cat /proc/sys/net/ipv4/conf/calicba2f87f6bb(网卡名称)/proxy_arp
  ```

  - 总结

  ```
  1.Calico 通过一个巧妙的方法将 workload 的所有流量引导到一个特殊的网关 169.254.1.1，从而引流到主机的 calixxx 网络设备上，最终将二三层流量全部转换成三层流量来转发。
  2.在主机上通过开启代理 ARP 功能来实现 ARP 应答，使得 ARP 广播被抑制在主机上，抑制了广播风暴，也不会有 ARP 表膨胀的问题。
  ```

  Node2

  -  网卡

  ```
  [root@my-nginx-5967cb6fd8-5hbr5 /]# ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  2: tunl0: <NOARP> mtu 0 qdisc noop state DOWN group default 
      link/ipip 0.0.0.0 brd 0.0.0.0
  4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
      link/ether 26:3b:65:28:ff:6d brd ff:ff:ff:ff:ff:ff
      inet 192.168.92.144/32 brd 192.168.92.144 scope global eth0
      valid_lft forever preferred_lft forever
  ```

  - 路由信息

  ```
  [root@my-nginx-5967cb6fd8-5hbr5 /]# route
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  default         gateway         0.0.0.0         UG    0      0        0 eth0
  gateway         0.0.0.0         255.255.255.255 UH    0      0        0 eth0
  [root@my-nginx-5967cb6fd8-5hbr5 /]# ping gateway
  PING gateway (169.254.1.1) 56(84) bytes of data.
  ```

- 主机信息

  Node1

  - 网卡信息

  ```
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
      link/ether 52:54:00:04:f5:fd brd ff:ff:ff:ff:ff:ff
      inet 10.20.0.5/24 brd 10.20.0.255 scope global eth0
         valid_lft forever preferred_lft forever
      inet6 fe80::5054:ff:fe04:f5fd/64 scope link 
         valid_lft forever preferred_lft forever
  5: cali5b020232aa3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
      link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff
      inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
         valid_lft forever preferred_lft forever
  6: tunl0: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default 
      link/ipip 0.0.0.0 brd 0.0.0.0
      inet 192.168.92.128/32 brd 192.168.92.128 scope global tunl0
         valid_lft forever preferred_lft forever
  ```

  - 路由信息

  ```
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  default         10.20.0.1       0.0.0.0         UG    0      0        0 eth0
  10.20.0.0       0.0.0.0         255.255.255.0   U     0      0        0 eth0
  link-local      0.0.0.0         255.255.0.0     U     1002   0        0 eth0
  172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
  192.168.92.128  0.0.0.0         255.255.255.192 U     0      0        0 *
  192.168.92.129  0.0.0.0         255.255.255.255 UH    0      0        0 cali5b020232aa3
  192.168.92.130  0.0.0.0         255.255.255.255 UH    0      0        0 cali76f0123054d
  192.168.92.144  0.0.0.0         255.255.255.255 UH    0      0        0 calib545d2f8c2d
  192.168.92.162  0.0.0.0         255.255.255.255 UH    0      0        0 calia3a4ea81108
  192.168.92.163  0.0.0.0         255.255.255.255 UH    0      0        0 calid3a3ea35e90
  192.168.92.164  0.0.0.0         255.255.255.255 UH    0      0        0 cali5e2174023e3
  192.168.92.165  0.0.0.0         255.255.255.255 UH    0      0        0 cali61704a52610
  192.168.135.128 10.20.0.3       255.255.255.192 UG    0      0        0 tunl0
  192.168.182.128 10.20.0.13      255.255.255.192 UG    0      0        0 tunl0
  ```

  Node2

  - 网卡信息

  ```
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
      link/ether 52:54:00:76:b6:6f brd ff:ff:ff:ff:ff:ff
      inet 10.20.0.3/24 brd 10.20.0.255 scope global eth0
         valid_lft forever preferred_lft forever
      inet6 fe80::5054:ff:fe76:b66f/64 scope link 
         valid_lft forever preferred_lft forever
  5: cali2580f7c45d2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
      link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff
      inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
         valid_lft forever preferred_lft forever
  6: tunl0: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default 
      link/ipip 0.0.0.0 brd 0.0.0.0
      inet 192.168.135.128/32 brd 192.168.135.128 scope global tunl0
         valid_lft forever preferred_lft forever
  ```

  - 路由信息

  ```
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  default         10.20.0.1       0.0.0.0         UG    0      0        0 eth0
  10.20.0.0       0.0.0.0         255.255.255.0   U     0      0        0 eth0
  link-local      0.0.0.0         255.255.0.0     U     1002   0        0 eth0
  172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
  192.168.92.128  10.20.0.5       255.255.255.192 UG    0      0        0 tunl0
  192.168.135.128 0.0.0.0         255.255.255.192 U     0      0        0 *
  192.168.135.129 0.0.0.0         255.255.255.255 UH    0      0        0 cali2580f7c45d2
  192.168.135.158 0.0.0.0         255.255.255.255 UH    0      0        0 cali081bbffda73
  192.168.135.162 0.0.0.0         255.255.255.255 UH    0      0        0 cali4bf8525bb2e
  192.168.135.163 0.0.0.0         255.255.255.255 UH    0      0        0 cali37d73dfc2ac
  192.168.135.181 0.0.0.0         255.255.255.255 UH    0      0        0 cali1344e31b3c9
  192.168.135.182 0.0.0.0         255.255.255.255 UH    0      0        0 cali930f9edf0b2
  192.168.182.128 10.20.0.13      255.255.255.192 UG    0      0        0 tunl0
  ```

- wireshark抓包请求

  Node_IP_1:10.20.0.3   容器_IP_1:192.168.135.163

  Node_IP_2:10.20.0.5   容器_IP_2:192.168.92.144

  容器2请求容器1，如下为抓包内容

![image](https://github.com/zmayu/note/assets/28054451/6cabba49-7605-4938-ab73-4df0bf7b817a)

  ***
  
  ​	封包的时候是从上往下进行封包。
  
  ​	解包的时候是从下往上开始解包。
  
  ​	





##### 3.2.5 BGP工作模式

- 说明：Calico会使用kernel来实现一个虚拟的路由器，来维护这些路由表，然后每个节点作为虚拟的路由器之后，再根据BGP来相互交换这些路由信息，导致能让它在整个集群节点这些数据之间的交换，而且calico来实现k8s网络策略提供ACL功能

  实际上，Calico项目提供的网络解决方案，与Flannel的host-gw模式几乎一样。也就是说，Calico也是基于路由表实现容器数据包转发，但不同于Flannel使用flanneld进程来维护路由信息的做法，而Calico项目使用BGP协议来自动维护整个集群的路由信息。
  BGP英文全称是Border Gateway Protocol，即边界网关协议，它是一种自治系统间的动态路由发现协议，与其他 BGP 系统交换网络可达信息。



##### 3.2.6 将IPIP模式更改为BGP模式

​	calico默认为IPIP模式

​	#kubectl edit ds calico-node -n kube-system

​	kubectl edit ippool	

```yaml
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  annotations:
    projectcalico.org/metadata: '{"uid":"0a86b475-fbde-4170-89eb-e800ffdf5a61","creationTimestamp":"2020-10-16T13:44:40Z"}'
  creationTimestamp: "2020-10-16T13:44:40Z"
  generation: 1
  name: default-ipv4-ippool
  resourceVersion: "9152"
  selfLink: /apis/crd.projectcalico.org/v1/ippools/default-ipv4-ippool
  uid: 409a4bd6-c19b-404b-9618-a704430e2963
spec:
  blockSize: 26
  cidr: 192.168.0.0/16
  ipipMode: Always     ##默认为ipip模式，将Always改为Nerver，则变为BGP模式
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```









#### 3.3 K8s NetworkPoclicy

#### 3.3.1 简介

```
K8s NetworkPoclicy 用于实现Pod间的网络隔离。在使用Network Policy前，必须先安装支持K8s NetworkPoclicy的网络插件，包括：Calico，Romana，Weave Net，Trireme，OpenContrail等。
```

```
etcd 
pod内容器之间的访问
各个pod之间的访问
node之间的访问

overlay Network（延展网络，覆盖网络）：它是指构建在另一种网络上的计算机网络，这是一种网络虚拟化的技术形式。基于底层网络互联互通的基础上加上隧道技术去构建一个虚拟的网络。overlay的核心其实就是打隧道（Tunnel）。基于隧道技术实现，overlay网络的流量要跑在underlay网络上。
underlay（承载网络）：属于底层网络，负责互联互通。

flannel:是centos团队针对k8s设计的一个网络规划服务，它的功能是让集群中的不同节点主机创建的docker容器具有全集群唯一的虚拟ip地址。
flannel的实质是一种overlay（覆盖网络），也就是将tcp数据包装在另一种网络包里面进行路由转发和通信，目前已经支持udp、vxlan、host-gw等数据转发方式，默认的节点间数据通信方式是UDP转发。
flannel的特点
1.使集群中的不同Node主机创建的Docker容器都具有全集群唯一的虚拟IP地址。
2.建立一个覆盖网络（overlay network），通过这个覆盖网络，将数据包原封不动的传递到目标容器。覆盖网络是建立在另一个网络之上并由其基础设施支持的虚拟网络。覆盖网络通过将一个分组封装在另一个分组内来将网络服务与底层基础设施分离。在将封装的数据包转发到端点后，将其解封装。
3.创建一个新的虚拟网卡flannel0接收docker网桥的数据，通过维护路由表，对接收到的数据进行封包和转发（vxlan）。
4.etcd保证了所有node上flanned所看到的配置是一致的。同时每个node上的flanned监听etcd上的数据变化，实时感知集群中node的变化。
查看flannel的网络配置：cat /run/flannel/subnet.env

ipvs linked-netnsid

CNI插件采用了Calico，而kube-proxy则是开启了IPVS模式。在容器网络的构建中，CNI和kube-proxy两者都有参与。
CNI 插件包括 Calico、flannel、Terway、Weave Net 以及 Contiv。
cat /etc/cni/net.d/10-flannel.conflist  查看cni的配置

calico:是一个虚拟网络解决方案，完全利用路由规则实现动态组网，通过BGP协议通告路由。calico的好处是endpoints组成的网络是单纯的三层网络，报文的流向完全通过路由规则控制，没有overlay等额外开销。


  pathPrefix: "/etc/cni/net.d"
  pathPrefix: "/etc/kube-flannel"
  pathPrefix: "/run/flannel"

利用 etcdctl 命令进行连接
本地连接, 默认使用 http://127.0.0.1:2379 作为默认 endpoint
假如需要执行远程连接, 可以通过定义 endpoint 实现
ex:  etcd --endpoint http://10.199.205.229:2379

cni（Container Network Interface）网络容器接口
Tunnel :隧道
VPN
	Peer-to-Peer VPN
	Overlay VPN
	
查看端口监听信息	
netstat -tunlp｜grep 端口号
-t:仅显示tcp相关选项
-u:仅显示udp相关选项
-l:仅列出listen（监听）的服务状态
-p:显示建立相关链接的程序名

探测udp端口
nc安装 yum install nmap-ncat
nc -vuz ip port

tcpdump抓包分析
tcpdump -i eth1  port 8472 and  host 9.135.91.57 -w /tmp/tcpdump_save.cap
侦听指定网卡、指定端口
	
default via 192.168.26.2 dev ens192 proto dhcp metric 100 
blackhole 100.89.161.128/26 proto bird 
100.89.161.133 dev cali91d58cc8707 scope link 
100.89.161.134 dev cali8cf2a3b651f scope link 
100.118.167.128/26 via 192.168.26.246 dev tunl0 proto bird onlink 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.26.0/24 dev ens192 proto kernel scope link src 192.168.26.247 metric 100	
	
抓包分析
tcpdump -i eth0 host 10.10.10.10 -w /tmp/tcpdump_save.cap
说明:
	-i:指定网卡出口
	host:ip地址，包括源地址和目的地址
	-w:将抓取的内容写入到文件中
	

VTEP:virtual tunnel endpoint 
VXLAN:Virtual eXtensible Local Area Network.目前最热门的网络虚拟化技术，网络虚拟化是指在一套物理设备上虚拟出多个二层网络。VXLAN协议将Ethernet帧封装在UDP内，再加上8个字节的VXLAN header,用来标示不同的网络。
	
检查linux网桥当前的转发表或Mac学习表	
bridge fdb
brctl showmacs <bridge-name>
OTV:overlay virtualization transport 

业界目前有两种跨数据中心二层互联技术，VXLAN EVPN和OTV(Overlay Transport Virtualization)
	
	
node network：承载kubernetes集群中各个“物理”Node(master和minion)通信的网络；

service network：由kubernetes集群中的Services所组成的“网络”；

flannel network： 即Pod网络，集群中承载各个Pod相互通信的网络。


	
pods
cat <<EOF >/etc/kubelet.d/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
EOF
	
	
 	
14:53:53.772618 IP 9.135.144.168.37826 > 9.135.91.57.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.2.9.36304 > 10.244.1.170.http: Flags [S], seq 1584751359, win 14100, options [mss 1410,sackOK,TS val 212680651 ecr 0,nop,wscale 7], length 0
14:53:53.772853 IP 9.135.91.57.37247 > 9.135.144.168.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.1.170.http > 10.244.2.9.36304: Flags [S.], seq 3563824638, ack 1584751360, win 13980, options [mss 1410,sackOK,TS val 923724485 ecr 212680651,nop,wscale 7], length 0
14:53:53.772935 IP 9.135.144.168.37826 > 9.135.91.57.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.2.9.36304 > 10.244.1.170.http: Flags [.], ack 1, win 111, options [nop,nop,TS val 212680651 ecr 923724485], length 0
14:53:53.773001 IP 9.135.144.168.37826 > 9.135.91.57.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.2.9.36304 > 10.244.1.170.http: Flags [P.], seq 1:77, ack 1, win 111, options [nop,nop,TS val 212680651 ecr 923724485], length 76
14:53:53.773127 IP 9.135.91.57.37247 > 9.135.144.168.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.1.170.http > 10.244.2.9.36304: Flags [.], ack 77, win 110, options [nop,nop,TS val 923724485 ecr 212680651], length 0
14:53:53.773177 IP 9.135.91.57.37247 > 9.135.144.168.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.1.170.http > 10.244.2.9.36304: Flags [P.], seq 1:239, ack 77, win 110, options [nop,nop,TS val 923724485 ecr 212680651], length 238
14:53:53.773198 IP 9.135.144.168.37826 > 9.135.91.57.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.2.9.36304 > 10.244.1.170.http: Flags [.], ack 239, win 119, options [nop,nop,TS val 212680651 ecr 923724485], length 0
14:53:53.773218 IP 9.135.91.57.37247 > 9.135.144.168.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.1.170.http > 10.244.2.9.36304: Flags [P.], seq 239:851, ack 77, win 110, options [nop,nop,TS val 923724485 ecr 212680651], length 612
14:53:53.773233 IP 9.135.144.168.37826 > 9.135.91.57.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.2.9.36304 > 10.244.1.170.http: Flags [.], ack 851, win 129, options [nop,nop,TS val 212680651 ecr 923724485], length 0
14:53:53.773297 IP 9.135.144.168.37826 > 9.135.91.57.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.2.9.36304 > 10.244.1.170.http: Flags [F.], seq 77, ack 851, win 129, options [nop,nop,TS val 212680651 ecr 923724485], length 0
14:53:53.773412 IP 9.135.91.57.37247 > 9.135.144.168.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.1.170.http > 10.244.2.9.36304: Flags [F.], seq 851, ack 78, win 110, options [nop,nop,TS val 923724485 ecr 212680651], length 0
14:53:53.773435 IP 9.135.144.168.37826 > 9.135.91.57.otv: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.244.2.9.36304 > 10.244.1.170.http: Flags [.], ack 852, win 129, options [nop,nop,TS val 212680651 ecr 923724485], length 0 	
 	
/bitnami/mongodb/data/db/WiredTiger.wt: handle-open: open: File exists"
/bitnami/mongodb/data/db/WiredTiger.wt to /bitnami/mongodb/data/db/WiredTiger.wt.84: file-rename: rename: Permission denied"}}
	
	
	
node network：承载kubernetes集群中各个“物理”Node(master和minion)通信的网络；
service network：由kubernetes集群中的Services所组成的“网络”；
flannel network： 即Pod网络，集群中承载各个Pod相互通信的网络。



/etc/cni/net.d

这里容器获取的是 /32 位主机地址，表示将容器 A 作为一个单点的局域网。
	
```



测试







 

![image](https://github.com/zmayu/note/assets/28054451/926a568b-7d77-4938-9655-83f7d2bb7a5f)

MAC

52 24 00 ad 41 39

Fe ee cf 97  ef   06

IP

09 87 5b 39 :9.135.91.57

09 87 90 a8 :9.135.144.168

UDP

9b ef :39919

21 18:8472

MAC

源Mac: 2a fe 82 35 16 30

目的Mac: b2 c5 b5 fe 50 b8

![image](https://github.com/zmayu/note/assets/28054451/1ff05235-3603-4bfd-9085-1db53acd4c17)

IP

源IP:0a f4 01 aa.    --> 10.244.1.170

目的IP:0a f4 02 09   -->. 10.244.2.9

![image](https://github.com/zmayu/note/assets/28054451/31b33d38-da1c-4140-9a6b-6aee55c566e1)



TCP

00 50:源端口号：80

8b 58:目的端口号: 35672

![image](https://github.com/zmayu/note/assets/28054451/9d57f039-1009-41f6-93ca-a5cd0b39e5e5)

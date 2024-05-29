
### 技巧：
#### kubelet的日志输出到指定目录
* 通过日志来详细了解kubelet的运行机制以及通过日志观察kubelet的运行情况
* kubelet是通过systemd启动的，默认是输出到了/var/log/messages文件中了，但是这个文件内容是包括了所有通过systemd启动的服务日志
  
解决方式

* 在systemd日志的输出中添加kubelet标识
  * 这一步可以不进行操作，目前在centos、k8s-1.23版本验证，这一步免操作。
```
# 查看kubelet.service 配置
systemctl cat kubelet.service
# kubelet.service 加入如下配置
[Service]
SyslogIdentifier=kubelet
StandardOutput=journal
StandardError=journal
```
* 配置日志过滤，讲对应标识的日志也到指定文件。
  /etc/rsyslog.d/kubelet.conf
```
if $programname == 'kubelet' then /var/log/kubelet/kubelet.log

```
* logrotate配置日志轮转.
  
  /etc/logrotate.d/kubelet.conf
```
/var/log/kubelet/kubelet.log
      {
          missingok
          notifempty
          create
          sharedscripts
          rotate 16
          maxsize 300M
          daily
          compress
          delaycompress
          dateext
          dateformat %Y-%m-%d.%s
          postrotate
      /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
          endscript
      }
```
* 重启日志服务和kublet
```
    systemctl restart rsyslog
    systemctl daemon-reload
    systemctl restart kubelet
```







#### 问题
* linux里面对于资源（cpu、内存、网络、IO）的限制是通过cgroup进行的，对于cgroup的driver有cgroupfs和systemd。docker可以作为k8s的容器运行时，那么docker通过cgroup对访问资源进行了限制，那么k8s里面为什么还要通过systemd进行资源限制呢？暂时无法立即。
``` 首先，我们需要理解 Docker 和 Kubernetes 在资源管理上的不同层次。Docker 是一个容器运行时，它负责运行和管理单个的容器。而 Kubernetes 是一个容器编排系统，它负责管理和调度一组容器（即 Pod）。
当我们在 Docker 中运行一个容器时，Docker 会使用 cgroups 来限制和隔离这个容器的资源使用。例如，我们可以告诉 Docker，这个容器最多可以使用 1GB 的内存和 1 个 CPU 核心。Docker 会创建一个 cgroup，并设置这个 cgroup 的内存限制为 1GB，CPU 限制为 1 核心。然后，Docker 会将这个容器的所有进程添加到这个 cgroup 中，这样这个容器就只能使用 1GB 的内存和 1 个 CPU 核心了。
当我们在 Kubernetes 中运行一个 Pod 时，情况就有些不同了。一个 Pod 可以包含多个容器，这些容器共享相同的网络和存储，可以看作是一个逻辑主机。Kubernetes 需要对整个 Pod 的资源进行管理，而不是单个的容器。因此，Kubernetes 也会使用 cgroups 来限制和隔离 Pod 的资源使用。例如，我们可以告诉 Kubernetes，这个 Pod 最多可以使用 2GB 的内存和 2 个 CPU 核心。Kubernetes 会创建一个 cgroup，并设置这个 cgroup 的内存限制为 2GB，CPU 限制为 2 核心。然后，Kubernetes 会将这个 Pod 中的所有容器的所有进程添加到这个 cgroup 中，这样这个 Pod 就只能使用 2GB 的内存和 2 个 CPU 核心了。
所以，即使 Docker 已经使用 cgroups 来管理容器的资源，Kubernetes 仍然需要使用 cgroups 来管理 Pod 的资源。这两者是在不同的层次上使用 cgroups，没有冲突
```
* MemoryQos具体是什么，如何应用到k8s的表现。



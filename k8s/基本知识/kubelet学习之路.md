
### 技巧：
#### kubelet的日志输出到指定目录
##### 背景
* 通过日志来详细了解kubelet的运行机制以及通过日志观察kubelet的运行情况
* kubelet是通过systemd启动的，默认是输出到了/var/log/messages文件中了，但是这个文件内容是包括了所有通过systemd启动的服务日志
  
##### 解决方式

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

* 调整日志级别
```
vim /var/lib/kubelet/config.yaml

logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
  #debug 级别
  verbosity: 5  
```
* 重启日志服务和kublet
```
    systemctl restart rsyslog
    systemctl daemon-reload
    systemctl restart kubelet
```



#### 分析docker-shim的代码，了解sandbox容器和业务容器是如何组合成一个pod


#### 理解pod.ID的概念，如何正确的理解。
  * 在pleg里面通过pod.ID关联了podStatus 
  * 对于每一个pod，kubelet都会启动一个协程managePodLoop()

  ```go
  # 其实这里也是很好理解，对于容器运行时，如何区分和管理容器，站在容器的角度，它是没有一个pod的概念，只能去通过名字进行区分。
   对于容器会有很多状态，那么这些状态要映射到pod的状态
  //k8s_POD_nginx-deployment-9456bbbf9-lqfrp_default_6a9c450a-d36a-4da0-8bdc-6e5d3e603a9d_0
  /*
  k8s_POD_nginx-deployment-9456bbbf9-lqfrp_default_6a9c450a-d36a-4da0-8bdc-6e5d3e603a9d_0
  k8s_nginx_nginx-deployment-9456bbbf9-lqfrp_default_6a9c450a-d36a-4da0-8bdc-6e5d3e603a9d_0

  kublet是如何进行区分容器是sandbox还是container,是通过名称进行区分。对名称进行下划线_分割。
  parts[0] = k8s            : 说明该容器是由k8s管理创建
  parts[1] = POD/nginx      : POD说明该容器是sandbox，nignx说明是业务容器，ningx为容器的名称
  parts[2] = pod name       : nginx-deployment-9456bbbf9-lqfrp
  parts[3] = pod namespace  : default
  parts[4] = pod uid        : 6a9c450a-d36a-4da0-8bdc-6e5d3e603a9d
  parts[4] = pod attempt    : 0 

  sandbox.id : 为容器的id
  sandbox.Metadat.Uid : 为pod的id,通过这个uid将sandbox和container绑定关联在一起。
  sandbox.Metadat.Uid 就是pod uid
  */

  // 
  return &runtimeapi.PodSandboxMetadata{
		Name:      parts[2],
		Namespace: parts[3],
		Uid:       parts[4],
		Attempt:   attempt,
	}
  ```

"SyncLoop DELETE" source="api" pods=[default/nginx-deployment-9456bbbf9-cvrr7]
"Pod is marked for graceful deletion, begin teardown" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Processing pod event" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 updateType=1
"Pod worker has observed request to terminate" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
 "syncTerminatingPod enter" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Generating pod status" pod="default/nginx-deployment-9456bbbf9-cvrr7"
"Got phase for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7" oldPhase=Running phase=Running
"Pod terminating with grace period" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 gracePeriod=30
"Killing container with a grace period override" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containerName="nginx" containerID="docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c" gracePeriod=30
"Killing container with a grace period" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containerName="nginx" containerID="docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c" gracePeriod=30
"Event occurred" object="default/nginx-deployment-9456bbbf9-cvrr7" kind="Pod" apiVersion="v1" type="Normal" reason="Killing" message="Stopping container nginx"
"Patch status for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7" patch="{\"metadata\":{\"uid\":\"4f3761db-ee39-4974-a2cc-9bf0a48e5c13\"}}"
"Status for pod is up-to-date" pod="default/nginx-deployment-9456bbbf9-cvrr7" statusVersion=2
"Pod is terminated, but some containers are still running" pod="default/nginx-deployment-9456bbbf9-cvrr7"
Destroyed container: "/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod4f3761db_ee39_4974_a2cc_9bf0a48e5c13.slice/docker-dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c.scope" (aliases: [k8s_nginx_nginx-deployment-9456bbbf9-cvrr7_default_4f3761db-ee39-4974-a2cc-9bf0a48e5c13_0 dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c], namespace: "docker")
"Container exited normally" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containerName="nginx" containerID="docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c"
"Calling network plugin to tear down the pod" pod="default/nginx-deployment-9456bbbf9-cvrr7" networkPluginName="cni"
"Deleting pod from network" pod="default/nginx-deployment-9456bbbf9-cvrr7" podSandboxID={Type:docker ID:df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a} podNetnsPath="/proc/28118/ns/net" networkType="loopback" networkName="cni-loopback"
 "Deleted pod from network" pod="default/nginx-deployment-9456bbbf9-cvrr7" podSandboxID={Type:docker ID:df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a} networkType="loopback" networkName="cni-loopback"
"Deleting pod from network" pod="default/nginx-deployment-9456bbbf9-cvrr7" podSandboxID={Type:docker ID:df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a} podNetnsPath="/proc/28118/ns/net" networkType="calico" networkName="k8s-pod-network"


 "SyncLoop DELETE" source="api" pods=[default/nginx-deployment-9456bbbf9-cvrr7]
"Deleted pod from network" pod="default/nginx-deployment-9456bbbf9-cvrr7" podSandboxID={Type:docker ID:df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a} networkType="calico" networkName="k8s-pod-network"
 Destroyed container: "/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod4f3761db_ee39_4974_a2cc_9bf0a48e5c13.slice/docker-df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a.scope" (aliases: [k8s_POD_nginx-deployment-9456bbbf9-cvrr7_default_4f3761db-ee39-4974-a2cc-9bf0a48e5c13_0 df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a], namespace: "docker")
"getSandboxIDByPodUID got sandbox IDs for pod" podSandboxID=[df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a] pod="default/nginx-deployment-9456bbbf9-cvrr7"
"Post-termination container state" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containers="(nginx state=exited exitCode=0 finishedAt=2024-06-07T06:54:46.591410983Z)"
 "Pod termination stopped all running containers" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"syncTerminatingPod exit" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Pod terminated all containers successfully" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Processing pod event done" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 updateType=1
"Processing pod event" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 updateType=2
 "Removing volume from desired state" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 volumeName="kube-api-access-cfqv5"
"getSandboxIDByPodUID got sandbox IDs for pod" podSandboxID=[df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a] pod="default/nginx-deployment-9456bbbf9-cvrr7"
"PLEG: Write status" pod="default/nginx-deployment-9456bbbf9-cvrr7"


 "SyncLoop (PLEG): event for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7" event=&{ID:4f3761db-ee39-4974-a2cc-9bf0a48e5c13 Type:ContainerDied Data:dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c}
"syncTerminatedPod enter" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Generating pod status" pod="default/nginx-deployment-9456bbbf9-cvrr7"


"SyncLoop (PLEG): event for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7" event=&{ID:4f3761db-ee39-4974-a2cc-9bf0a48e5c13 Type:ContainerDied Data:df89af7490d2456a7698b2773e44f08eb593c27319080676a6a27811e95d800a}
"Got phase for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7" oldPhase=Running phase=Running
"Waiting for volumes to unmount for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7"
"All volumes are unmounted for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7"
 "Pod termination unmounted volumes" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Pod termination removed cgroups" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Pod is terminated and will need no more status updates" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"syncTerminatedPod exit" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Pod is complete and the worker can now stop" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Processing pod event done" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 updateType=2


 "SyncLoop RECONCILE" source="api" pods=[default/nginx-deployment-9456bbbf9-cvrr7]
 "Patch status for pod" pod="default/nginx-deployment-9456bbbf9-cvrr7" patch=
{
    "metadata": {
        "uid": "4f3761db-ee39-4974-a2cc-9bf0a48e5c13"
    },
    "status": {
        "$setElementOrder/conditions": [
            {
                "type": "Initialized"
            },
            {
                "type": "Ready"
            },
            {
                "type": "ContainersReady"
            },
            {
                "type": "PodScheduled"
            }
        ],
        "conditions": [
            {
                "lastTransitionTime": "2024-06-07T06:54:47Z",
                "message": "containers with unready status: [nginx]",
                "reason": "ContainersNotReady",
                "status": "False",
                "type": "Ready"
            },
            {
                "lastTransitionTime": "2024-06-07T06:54:47Z",
                "message": "containers with unready status: [nginx]",
                "reason": "ContainersNotReady",
                "status": "False",
                "type": "ContainersReady"
            }
        ],
        "containerStatuses": [
            {
                "containerID": "docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c",
                "image": "nginx:1.14.2",
                "imageID": "docker-pullable://nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d",
                "lastState": {

                },
                "name": "nginx",
                "ready": false,
                "restartCount": 0,
                "started": false,
                "state": {
                    "terminated": {
                        "containerID": "docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c",
                        "exitCode": 0,
                        "finishedAt": "2024-06-07T06:54:46Z",
                        "reason": "Completed",
                        "startedAt": "2024-05-29T06:42:45Z"
                    }
                }
            }
        ]
    }
}





 "Status for pod updated successfully" pod="default/nginx-deployment-9456bbbf9-cvrr7" statusVersion=3 status={Phase:Running Conditions:[{Type:Initialized Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2024-05-29 14:42:44 +0800 CST Reason: Message:} {Type:Ready Status:False LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2024-06-07 14:54:47 +0800 CST Reason:ContainersNotReady Message:containers with unready status: [nginx]} {Type:ContainersReady Status:False LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2024-06-07 14:54:47 +0800 CST Reason:ContainersNotReady Message:containers with unready status: [nginx]} {Type:PodScheduled Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2024-05-29 14:42:44 +0800 CST Reason: Message:}] Message: Reason: NominatedNodeName: HostIP:9.135.91.57 PodIP:172.16.219.71 PodIPs:[{IP:172.16.219.71}] StartTime:2024-05-29 14:42:44 +0800 CST InitContainerStatuses:[] ContainerStatuses:[{Name:nginx State:{Waiting:nil Running:nil Terminated:&ContainerStateTerminated{ExitCode:0,Signal:0,Reason:Completed,Message:,StartedAt:2024-05-29 14:42:45 +0800 CST,FinishedAt:2024-06-07 14:54:46 +0800 CST,ContainerID:docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c,}} LastTerminationState:{Waiting:nil Running:nil Terminated:nil} Ready:false RestartCount:0 Image:nginx:1.14.2 ImageID:docker-pullable://nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d ContainerID:docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c Started:0xc0021010cc}] QOSClass:BestEffort EphemeralContainerStatuses:[]}

"Pod is terminated and all resources are reclaimed" pod="default/nginx-deployment-9456bbbf9-cvrr7"


"SyncLoop DELETE" source="api" pods=[default/nginx-deployment-9456bbbf9-cvrr7]
"Pod is finished processing, no further updates" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Pod fully terminated and removed from etcd" pod="default/nginx-deployment-9456bbbf9-cvrr7"


"SyncLoop REMOVE" source="api" pods=[default/nginx-deployment-9456bbbf9-cvrr7]
 "Pod has been deleted and must be killed" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Pod is finished processing, no further updates" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
 "Pod does not exist on the server" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 pod="default/nginx-deployment-9456bbbf9-cvrr7"
 "Pod is being synced for the first time" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
 "Pod is orphaned and must be torn down" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Processing pod event" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 updateType=1
"Pod worker has observed request to terminate" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"syncTerminatingPod enter" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
"Pod terminating with grace period" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 gracePeriod=1
"Killing container with a grace period override" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containerName="nginx" containerID="docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c" gracePeriod=1
 "Killing container with a grace period" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containerName="nginx" containerID="docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c" gracePeriod=1
"Event occurred" object="default/nginx-deployment-9456bbbf9-cvrr7" kind="Pod" apiVersion="v1" type="Normal" reason="Killing" message="Stopping container nginx"
"Container termination failed with gracePeriod" err="rpc error: code = Unknown desc = Error response from daemon: No such container: dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containerName="nginx" containerID="docker://dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c" gracePeriod=1
 "Kill container failed" err="rpc error: code = Unknown desc = Error response from daemon: No such container: dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13 containerName="nginx" containerID={Type:docker ID:dc18bc33e7914de8b6db57585ce56f2fbbb36c8deab970d5d0014f882ce1748c}
 "syncTerminatingPod exit" pod="default/nginx-deployment-9456bbbf9-cvrr7" podUID=4f3761db-ee39-4974-a2cc-9bf0a48e5c13
^C













#### kubelet目录功能介绍

├── OWNERS
├── active_deadline.go
├── active_deadline_test.go
├── apis
├── cadvisor
├── certificate
├── checkpointmanager
├── client
├── cloudresource
├── cm
├── config
├── configmap
├── container
├── cri
├── custommetrics
├── doc.go
├── dockershim
├── envvars
├── errors.go
├── events
├── eviction
├── images
├── kubelet.go
├── kubelet_dockerless_test.go
├── kubelet_dockershim.go
├── kubelet_dockershim_nodocker.go
├── kubelet_getters.go
├── kubelet_getters_test.go
├── kubelet_network.go
├── kubelet_network_linux.go
├── kubelet_network_others.go
├── kubelet_network_test.go
├── kubelet_node_status.go
├── kubelet_node_status_others.go
├── kubelet_node_status_test.go
├── kubelet_node_status_windows.go
├── kubelet_pods.go
├── kubelet_pods_linux_test.go
├── kubelet_pods_test.go
├── kubelet_pods_windows_test.go
├── kubelet_resources.go
├── kubelet_resources_test.go
├── kubelet_test.go
├── kubelet_volumes.go
├── kubelet_volumes_linux_test.go
├── kubelet_volumes_test.go
├── kubeletconfig
├── kuberuntime
├── leaky
├── legacy
├── lifecycle
├── logs
├── metrics
├── network
├── nodeshutdown
├── nodestatus
├── oom
├── pleg
├── pluginmanager
├── pod
├── pod_container_deletor.go
├── pod_container_deletor_test.go
├── pod_workers.go
├── pod_workers_test.go
├── preemption
├── prober
├── qos
├── reason_cache.go
├── reason_cache_test.go
├── runonce.go
├── runonce_test.go
├── runtime.go
├── runtimeclass
├── secret
├── server
├── stats
├── status
├── sysctl
├── time_cache.go
├── time_cache_test.go
├── token
├── types
├── util
├── volume_host.go
├── volumemanager
└── winstats


.
├── apis
│   ├── config
│   │   ├── fuzzer
│   │   ├── scheme
│   │   │   └── testdata
│   │   │       ├── KubeletConfiguration
│   │   │       │   ├── after
│   │   │       │   ├── before
│   │   │       │   └── roundtrip
│   │   │       │       └── default
│   │   │       └── SerializedNodeConfigSource
│   │   │           ├── after
│   │   │           ├── before
│   │   │           └── roundtrip
│   │   │               └── default
│   │   ├── v1alpha1
│   │   ├── v1beta1
│   │   └── validation
│   └── podresources
│       └── testing
├── cadvisor
│   └── testing
├── certificate
│   └── bootstrap
│       └── testdata
├── checkpointmanager
│   ├── checksum
│   ├── errors
│   └── testing
│       └── example_checkpoint_formats
│           └── v1
├── client
├── cloudresource
├── cm
│   ├── admission
│   ├── containermap
│   ├── cpumanager
│   │   ├── state
│   │   │   └── testing
│   │   └── topology
│   ├── cpuset
│   ├── devicemanager
│   │   └── checkpoint
│   ├── memorymanager
│   │   └── state
│   ├── topologymanager
│   │   └── bitmask
│   └── util
├── config
├── configmap
├── container
│   └── testing
├── cri
│   ├── remote
│   │   ├── fake
│   │   └── util
│   └── streaming
│       ├── portforward
│       └── remotecommand
├── custommetrics
├── dockershim
│   ├── cm
│   ├── libdocker
│   │   └── testing
│   ├── metrics
│   ├── network
│   │   ├── cni
│   │   │   └── testing
│   │   ├── hairpin
│   │   ├── hostport
│   │   ├── kubenet
│   │   ├── metrics
│   │   └── testing
│   └── remote
├── envvars
├── events
├── eviction
│   └── api
├── images
├── kubeletconfig
│   ├── checkpoint
│   │   └── store
│   ├── configfiles
│   ├── status
│   └── util
│       ├── codec
│       ├── equal
│       ├── files
│       ├── panic
│       └── test
├── kuberuntime
│   └── logs
├── leaky
├── legacy
├── lifecycle
├── logs
├── metrics
│   └── collectors
├── network
│   └── dns
├── nodeshutdown
│   └── systemd
├── nodestatus
├── oom
├── pleg
├── pluginmanager
│   ├── cache
│   ├── metrics
│   ├── operationexecutor
│   ├── pluginwatcher
│   │   └── example_plugin_apis
│   │       ├── v1beta1
│   │       └── v1beta2
│   └── reconciler
├── pod
│   └── testing
├── preemption
├── prober
│   ├── results
│   └── testing
├── qos
├── runtimeclass
│   └── testing
├── secret
├── server
│   ├── metrics
│   └── stats
│       └── testing
├── stats
│   └── pidlimit
├── status
│   └── testing
├── sysctl
├── token
├── types
├── util
│   ├── cache
│   ├── format
│   ├── ioutils
│   ├── manager
│   ├── queue
│   ├── sliceutils
│   └── store
├── volumemanager
│   ├── cache
│   ├── metrics
│   ├── populator
│   └── reconciler
└── winstats








#### 问题
* linux里面对于资源（cpu、内存、网络、IO）的限制是通过cgroup进行的，对于cgroup的driver有cgroupfs和systemd。docker可以作为k8s的容器运行时，那么docker通过cgroup对访问资源进行了限制，那么k8s里面为什么还要通过systemd进行资源限制呢？暂时无法立即。
``` 首先，我们需要理解 Docker 和 Kubernetes 在资源管理上的不同层次。Docker 是一个容器运行时，它负责运行和管理单个的容器。而 Kubernetes 是一个容器编排系统，它负责管理和调度一组容器（即 Pod）。
当我们在 Docker 中运行一个容器时，Docker 会使用 cgroups 来限制和隔离这个容器的资源使用。例如，我们可以告诉 Docker，这个容器最多可以使用 1GB 的内存和 1 个 CPU 核心。Docker 会创建一个 cgroup，并设置这个 cgroup 的内存限制为 1GB，CPU 限制为 1 核心。然后，Docker 会将这个容器的所有进程添加到这个 cgroup 中，这样这个容器就只能使用 1GB 的内存和 1 个 CPU 核心了。
当我们在 Kubernetes 中运行一个 Pod 时，情况就有些不同了。一个 Pod 可以包含多个容器，这些容器共享相同的网络和存储，可以看作是一个逻辑主机。Kubernetes 需要对整个 Pod 的资源进行管理，而不是单个的容器。因此，Kubernetes 也会使用 cgroups 来限制和隔离 Pod 的资源使用。例如，我们可以告诉 Kubernetes，这个 Pod 最多可以使用 2GB 的内存和 2 个 CPU 核心。Kubernetes 会创建一个 cgroup，并设置这个 cgroup 的内存限制为 2GB，CPU 限制为 2 核心。然后，Kubernetes 会将这个 Pod 中的所有容器的所有进程添加到这个 cgroup 中，这样这个 Pod 就只能使用 2GB 的内存和 2 个 CPU 核心了。
所以，即使 Docker 已经使用 cgroups 来管理容器的资源，Kubernetes 仍然需要使用 cgroups 来管理 Pod 的资源。这两者是在不同的层次上使用 cgroups，没有冲突
```
* MemoryQos具体是什么，如何应用到k8s的表现。



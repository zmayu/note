# 																															HELM

## 1.helm简介说明

### 1.1Helm简介

我们知道 Kubernetes 是一个分布式的容器集群管理系统，它把集群中的管理资源抽象化成一个个 API 对象，并且推荐使用声明式的方式创建，修改，删除这些对象，每个 API 对象都通过一个 `yaml` 格式或者 `json` 格式的文本来声明。这带来的一个问题就是这些 API 对象声明文本的管理成本，每当我需要创建一个应用，都需要去编写一堆这样的声明文件。

Helm 就是用来管理这些 API 对象的工具。它类似于 CentOS 的 YUM 包管理，Ubuntu 的 APT 包管理，你可以把它理解成 **Kubernetes 的包管理工具**。它能够把创建一个应用所需的所有 Kubernetes API 对象声明文件组合并打包在一起。并提供了仓库的机制便于分发共享，还支持模版变量替换,同时还有版本的概念，使之能够对一个应用进行版本的管理。

Helm是管理Kubernetes包的工具，Helm能提供以下能力：

- 创建新的charts
- 将charts打包成tgz文件
- 与chart仓库交互
- 安装和卸载Kubernetes的应用
- 管理使用Helm安装的charts的生命周期

### 	1.1 Helm2版本的介绍

​		![image-20200924115015674](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924115015674.png)	

Helm架构由Helm客户端，Till服务器端和chart仓库组成，Tiller部署在Kubernetes中，Helm客户端从Chart仓库中获取Chart安装包，并将其部署到Kubernetes中，

Helm 2采用客户端/服务器架构，有如下组件（概念）组成：

- **Chart:** 就是 Helm 的一个包（package），包含一个应用所有的 Kubernetes manifest 模版，类似于 YUM 的 RPM 或者 APT 的 dpkg 文件。
- **Helm CLI:** Helm 的客户端组件，它通过 gRPC aAPI 向 tiller 发送请求。
- **Tiller:** Helm 的服务器端组件，在 Kubernetes 群集上运行，负载解析客户端端发送过来的 Chart，并根据 Chart 中的定义在 Kubernetes 中创建出相应的资源，tiller 把 release 相关的信息存入 Kubernetes 的 ConfigMap 中。
- **Repository:** 用于发布和存储 Chart 的仓库。
- **Release:** 可以理解成 Chart 部署的一个实例。通过 Chart 在 Kubernetes 中部署的应用都会产生一个唯一的 Release，即使是同一个 Chart，部署多次就会产生多个 Release。

### 	1.2 Helm3版本的介绍			![FFC25686-4123-4D4A-80A1-500F88F54720](/var/folders/sy/sqd8vbyj0_125jq8vb2dhws00000gn/T/com.yinxiang.Mac/WebKitDnD.wxCLuV/FFC25686-4123-4D4A-80A1-500F88F54720.png)

在 Helm 3 中移除了 `Tiller`，其余内容与helm2基本相同。 版本相关的数据直接存储在了 Kubernetes 中。主要关注简单性、安全性和可用性。

- Helm3中移除Tiller的考量	

  ​		从kubernetes 1.6开始默认开启RBAC。这是Kubernetes安全性/企业可用的一个重要特性。但是在RBAC开启的情况下管理及配置Tiller变的非常复杂。为了简化helm的尝试成本我们给出了一个不需要关注安全规则的默认配置。但是，这会导致一些用户意外获得了他们并不需要的权限。

  ​		移除掉Tiller大大简化了hlem的安全模型实现方式。Helm3现在可以支持所有的kubernetes认证及鉴权等全部安全特性。Helm和本地的kubeconfig flie中的配置使用一致的权限。管理员可以按照自己认为合适的粒度来管理用户权限。



## 2.helm3的安装

- 从官网下载helm-v3.3.1-linux-amd64.tar.gz安装包.   

  ```shell
   wget https://get.helm.sh/helm-v3.3.1-linux-amd64.tar.gz
  ```

-  将tar包解压

  ```shell
  tar -zxvf helm-v3.3.1-linux-amd64.tar.gz
  ```

- 将linux-amd64文件中的helm文件拷贝到/usr/local/bin

  ```shell
  cp linux-amd64/helm  /usr/local/bin
  ```

- 测试helm,查看helm的安装版本

  ```shell
  helm version
  
  version.BuildInfo{Version:"v3.3.1", GitCommit:"249e5215cde0c3fa72e27eb7a30e8d55c9696144", GitTreeState:"clean", GoVersion:"go1.14.7"}
  ```





## 3.Helm3 chart文件说明

- 首先使用helm  create  mychart创建一个名为mychart的示例,可以使用tree mychart命令看一下chart的目录结构	 

```shell
mychart
├── Chart.yaml
├── charts 											# 该目录保存其他依赖的 chart（子 chart）
├── templates 									# chart 配置模板，用于渲染最终的 Kubernetes YAML 文件
│   ├── NOTES.txt 							# 用户运行 helm install 时候的提示信息
|   ├── hpa.yaml  							#pod水平自动扩缩。 horizontal pod autoscaler
│   ├── _helpers.tpl 						# 用于创建模板时的帮助类
│   ├── deployment.yaml 				# Kubernetes deployment 配置
│   ├── ingress.yaml 						# Kubernetes ingress 配置
│   ├── service.yaml 						# Kubernetes service 配置
│   ├── serviceaccount.yaml 		# Kubernetes serviceaccount 配置
│   └── tests
│       └── test-connection.yaml
└── values.yaml 								# 定义 chart 模板中的自定义配置的默认值，可以在执行 helm install 或 helm update 的时候覆盖
```



## 4.Helm3常用的命令

```shell
* helm create										 #在本地创建新的 chart；
* helm dependency   						 #管理 chart 依赖；
* helm intall										 #安装 chart；
* helm lint											 #检查 chart 配置是否有误；
* helm list											 #列出所有 release；
* helm package                   #打包本地 chart；
* helm repo											 #列出、增加、更新、删除 chart 仓库；
* helm rollback                  #回滚 release 到历史版本；
* helm pull                      #拉取远程 chart 到本地；
* helm search                    #使用关键词搜索 chart；
* helm uninstall                 #卸载 release；
* helm upgrade									 #升级 release；
* helm uninstall --keep-history  #卸载release，通过参数 --keep-history 可以保留这个release，以后可以进行重新恢复。通过helm list --uninstalled查看。helm history [name] 查看历史版本，通过helm rollback [release-name] [version]
* helm get manifest  RELEASE_NAME [flags]  #打印出所有上传到k8s的资源。每个文件都以 --- 开始作为yaml文档的开始，然后是一个自动生成的注释行，告诉我们模版文件生成的yaml文档。
```



## 5.安装chart

#### 5.1从公有仓库安装tomcat

​		***https://hub.helm.sh/charts  ：helm提供的Chart公有仓库，可以参考。

-  添加仓库

  ```shell
  helm repo add bitnami https://charts.bitnami.com/bitnami
  
  "bitnami" has been added to your repositories
  ```

- 将chart pull 到本地

  ```
  helm pull bitnami/tomcat --version 6.4.0
  tomcat-6.4.0.tgz
  ```

- 将tomcat-6.4.0.tgz解压，更改values.yaml文件，将type: LoadBalance改为type: NodePort

  ```
  tar zxvf tomcat-6.4.0.tgz
  ------------------------因为本地无法通过LoadBalance进行负载均衡访问
  service:
    ## Service type
    ##
    type: NodePort
  ------------------------没有指定pv存储卷
  persistence:
    ## If true, use a Persistent Volume Claim, If false, use emptyDir
    ##
    enabled: false
  ```

- 安装tomcat chart到kubernetes

  ```
  cd tomcat 
  helm install tomcat  -f values.yaml  .
  
  -----如果出现如下内容则安装成功
  NAME: tomcat
  LAST DEPLOYED: Thu Sep 24 16:06:44 2020
  NAMESPACE: default
  STATUS: deployed
  REVISION: 1
  TEST SUITE: None
  NOTES:
  ** Please be patient while the chart is being deployed **
  
  1. Get the Tomcat URL by running:
  
    export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services tomcat)
    export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT/
  
  2. Login with the following credentials
  
    echo Username: user
    echo Password: $(kubectl get secret --namespace default tomcat -o jsonpath="{.data.tomcat-password}" | base64 --decode)  
  ```

- 访问tomcat

  ```
  --获取nodeport访问端口
  kubectl get svc
  --访问测试
  curl http://主机ip:nodeport
  
  ```

#### 5.2 本地创建chart,安装自定义应用

- 创建my-nginx

  ```
  helm create my-nginx		
  ```

- 更改values.yaml文件

  ```
  --- 设置镜像名称，拉去策略，镜像版本号
  image:
    repository: nginx
    pullPolicy: IfNotPresent
    # Overrides the image tag whose default is the chart appVersion.
    tag: latest
  --- 设置service的type为NodePort，如果是ClusterIp只能在集群内访问，无法通过外部访问。
  service:
    type: NodePort
    port: 80
  ```

- 安装my-nginx到kubernetes

  ```
  helm install my-nginx -f values.yaml  .
  ```

- 更新

- my-nginx访问测试

  ```
  ---获取nodeport端口
  kubectl get svc
  ---测试访问
  curl http://主机ip:nodeport
  
  ```

#### 5.3 打包chart，上传chart到TCR私有仓库

- 打包my-nginx chart

  ```
  helm package my-nginx/
  ---
  Successfully packaged chart and saved it to: /root/helm/helm-demo/my-nginx-0.1.0.tgz
  ```

- 上传my-nginx-0.1.0.tgz到TCR

  ```
  添加TCR仓库
  helm repo add zmayu-tcr-dmeo-test-tcr https://zmayu-tcr-dmeo.tencentcloudcr.com/chartrepo/test-tcr --username 100002524462 --password xxxxxxxxxx
  ---注意  如果是公网访问的话需要开启公网访问入口，将源端的公网IP加入到白名单中
  上传tgz
  helm push my-nginx-0.1.0.tgz  zmayu-tcr-dmeo-test-tcr
  
  ---注意 helm push命令不是helm自带的，需要安装push插件
  helm plugin install https://github.com/chartmuseum/helm-push
  
  一
  
  ```

- 查看TCR Helm Chart仓库

  如图，刚才上传的chart已经可见

  ![image-20200924183236662](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924183236662.png)

#### 5.4 在TKE中通过Helm Chart部署应用

- #### 新建应用	![image-20200924185156813](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924185156813.png)

![image-20200924185242422](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924185242422.png)

选择对应的Chart版本，同时我们也可以自主更高values.yaml文件

为了方便在公网访问，我们将type: NodePort改为 type: **LoadBalancer**

![image-20200924185919188](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924185919188.png)

点击完成，成功创建我们的应用

![image-20200924190241429](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924190241429.png)

- 访问测试

  在对应集群找到service的LoadBalance Ip

![image-20200924190754503](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924190754503.png)

```
curl http://LoadbalanceIp:服务端口
```

- 更新应用

  当应用信息发生变动时（例如镜像版本发生变化），可以点击更新应用重新部署。

- 回滚应用

  ![image-20200924192415720](/Users/zmayu/Library/Application Support/typora-user-images/image-20200924192415720.png)

在应用的详情里面可以看到关于应用的各个历史版本，并且支持回滚到指定的历史版本。
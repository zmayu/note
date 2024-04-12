#  将kafka更优雅的部署在k8s上
## 1.选择operator https://strimzi.io/ 的方式部署kafka
### 1.1 介绍operator 
    我们知道k8s里面有很多的资源服务，例如pod、service、serviceaccount等等，通过这些资源我们就可以将我们的应用服务部署到k8s容器里面。
但是对于一些复杂的中间件服务，如果还用这些基础服务部署的话，部署过程会非常麻烦。对于计算机来说，任何一个问题都可以通过增加一个中间层来解决。
因为我们增加了一个中间层。通过自定义资源crd模式来实现快速部署复杂中间件。

![image](https://github.com/zmayu/note/assets/28054451/88486c23-2219-4fcf-8749-f7e30431fe6a)


我理解operator主要做两件事，定义crd和controller.
```
1.k8s支持自定义资源类型
2.通过kubectl写入crd文件
3.crd文件持久化到etcd
4.operatro controller监听并解析crd资源文件，并将其转换为k8s自身的资源文件
5.转换后的资源文件写入到etcd
6.kubelet检测到转换后的资源文件，执行pod的创建
```
operator所以解决的问题，即有状态的中间件部署在k8s上运维部署复杂，operator将复杂运维过程进行封装，对外以一种简单的资源类型暴露

其中所涉及到的k8s简单知识扫盲
```
1.kubelet监听apiserver，对调度到本node节点上的pod进行增删改操作。
2.pod里的服务想对apiserver接口进行调用，必须要有认证和授权。
    accountService即创建账户，accountService创建后会自动创建secret，即acccountService表示用户名、secret表示密码
    role: 有名称空间的限制，即表示对某一名称空间资源操作的角色
    clusterRole：没有名称空间的限制，即表示对整个集群中任意名称空间资源操作的角色
    rolebinding: 将角色和account账户关联绑定，即给账户赋予的角色。账户、role、rolebinging需要在同一个名称空间
    clusterRoleBinding：将集群角色和account账户关联绑定，即表示该账号有操作整个集群所有名称空间下的资源
    但是该账号是有隶属的名称空间，那么这个账户只能关联到该名称空间下的pod
  综上：operator需要拥有整个集群的权限，有了整个集群的权限就能够在其他不同的名称空间下创建资源。
```

### 1.2 介绍strimzi
strimzi通过operator的方式实现了快速在k8s上部署kafka
#### 1.2.1 kafka & zookeeper模式部署
认证模式：
一般internal内部应用访问可以不设置用户名和密码
如果服务需要暴露给外部应用，连接协议使用SASL_PLAINTEXT，认证策略使用scram-sha-512
授权模式：
授权模式选择最简单的simple模式即可，不用复杂的基于角色的权限访问控制RBAC.
```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 3.7.0
    replicas: 1
    ### 授权模式，只能选择这个
    authorization:
      type: simple
    ### 认证模式有多种
    listeners:
      ### 不设置认证，直接可以con/pro数据
      - name: plain
        port: 9092
        type: internal
        tls: false
      ### 设置scram-sha-512认证模式
      - name: external
        port: 9093
        type: nodeport
        tls: false
        authentication:
          type: scram-sha-512
        # configuration:
          # preferredNodePortAddressType: InternalDNS
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
      inter.broker.protocol.version: "3.7"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```
#### 1.2.2 user operator
```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: my-cluster
spec:
  ### 认证模式
  authentication:
    type: SCRAM-SHA-512
  ### 授权模式
  authorization:
    type: simple
    acls:
      # Example consumer Acls for topic my-topic using consumer group my-group
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations:
          - Describe
          - Read
        host: "*"
      - resource:
          type: group
          name: my-group
          patternType: literal
        operations:
          - Read
        host: "*"
      # Example Producer Acls for topic my-topic
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations:
          - Create
          - Describe
          - Write
        host: "*"
```
创建完用户后，会自动创建一个该用户的secert
```
kubectl get secret -n 名称空间 ｜grep 用户名称
kubectl get secret/secret名称 -n 名称空间
```
<img width="1325" alt="image" src="https://github.com/zmayu/note/assets/28054451/fdbd7879-903e-48ff-bfe6-8a4b7ba6f633">

将password base64解码即为用户的密码


#### 1.2.3 topic operator
```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 1
  replicas: 1
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
```


#### 1.2.4 consumer message test
```
SCRAM-SHA-512 认证模式
kubectl run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.40.0-kafka-3.7.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-external-bootstrap:9093 --topic my-topic --group my-group --consumer-property security.protocol=SASL_PLAINTEXT  --consumer-property  sasl.mechanism=SCRAM-SHA-512 --consumer-property sasl.jaas.config="org.apache.kafka.common.security.scram.ScramLoginModule required username=\"my-user\" password=\"XIR3yuPE54DizPUUY5cQcYn2MGmpwPlE\";"
增加认证信息
--consumer-property  sasl.mechanism=SCRAM-SHA-512 --consumer-property sasl.jaas.config="org.apache.kafka.common.security.scram.ScramLoginModule required username=\"my-user\" password=\"XIR3yuPE54DizPUUY5cQcYn2MGmpwPlE\";"
```
#### 1.2.5 producer message test
```
SCRAM-SHA-512 认证模式
kubectl  run kafka-producer -ti --image=quay.io/strimzi/kafka:0.40.0-kafka-3.7.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-external-bootstrap:9093 --topic my-topic --producer-property security.protocol=SASL_PLAINTEXT  --producer-property  sasl.mechanism=SCRAM-SHA-512 --producer-property sasl.jaas.config="org.apache.kafka.common.security.scram.ScramLoginModule required username=\"my-user\" password=\"XIR3yuPE54DizPUUY5cQcYn2MGmpwPlE\";"
增加认证信息
--producer-property security.protocol=SASL_PLAINTEXT  --producer-property  sasl.mechanism=SCRAM-SHA-512 --producer-property sasl.jaas.config="org.apache.kafka.common.security.scram.ScramLoginModule required username=\"my-user\" password=\"XIR3yuPE54DizPUUY5cQcYn2MGmpwPlE\";"
```

#### 

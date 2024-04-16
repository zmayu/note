#    k8s中的认证
## 一、双向认证
### 1.1 以k8s中的etcd集群和apiServer的交互为场景举例说明。
* k8s中为了独立区分etcd和apiServer,对于etcd和apiServer都有各自独立的ca证书
* etcd中的这证书可以分为ca证书、etcd集群通信证书、etcd server端证书、健康探测证书
```
root@tke-master1:/etc/kubernetes/pki/etcd# ll
total 44
drwxr-xr-x 2 root root 4096 Apr 16 14:47 ./
drwxr-xr-x 3 root root 4096 Apr 15 17:55 ../
-rw------- 1 root root 1058 Nov 14 14:46 ca.crt
-rw------- 1 root root 1675 Nov 14 14:46 ca.key
-rw-r--r-- 1 root root   41 Apr 15 18:02 ca.srl
-rw-r--r-- 1 root root 1139 Mar 20 11:18 healthcheck-client.crt
-rw------- 1 root root 1675 Mar 20 11:18 healthcheck-client.key
-rw-r--r-- 1 root root 1127 Apr 15 17:18 peer.crt
-rw------- 1 root root 1679 Apr 15 17:16 peer.key
-rw-r--r-- 1 root root 1151 Apr 15 18:02 server.crt
-rw------- 1 root root 1679 Apr 15 17:58 server.key
```
### 1.2 使用etcd ca证书签发etcd集群通信证书
#### 1.2.1 手动签发证书
* 生成CA私钥和证书
```shell
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=etcd-ca" -days 10000 -out ca.crt
```
* 生成etcd通信使用私钥
```shell
openssl genrsa -out peer.key 2048
```
* 创建etcd peer证书签名请求（CSR）crs.conf
```shell
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = etcd.k8s.local
IP.1 = <etcd-node-ip>
```
其中，<etcd-node-ip> 需要替换为对应 etcd 节点的 IP 地址。
使用conf配置生成csr
```shell
openssl req -new -key peer.key -out peer.csr -subj "/CN=<etcd-node-ip>" -config csr.conf
```
<etcd-node-ip> 需要替换为对应 etcd 节点的 IP 地址

* 使用ca签名etcd通信证书
```shell
openssl x509 -req -in peer.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out peer.crt -days 10000 -extensions v3_req -extfile crs.conf
```
### 1.3 双向认证过程中遇到的错误
### 1.3.1 由于签发时错误导致公私钥不匹配
*  错误信息： “etcdmain: tls: private key does not match public key”
* 解决办法
  * 重新签发证书 
  * 通过如下命令验证公私钥，这两个命令应该输出相同的哈希值。如果不是，那么你的证书和私钥就不匹配。
```shell
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa -noout -modulus -in server.key | openssl md5
```
### 1.3.2 双向认证错误，证书的使用者名称或使用者可选名称与etcd节点访问地址不匹配
* 错误信息：rejected connection from "10.149.10.210:41244"(error "tls: \"10.149.10.210\" does not match any of DNSNames [\"10.149.10.227\" \"localhost\"]
* 解决办法
  * 修改crs.conf配置里面alt_names的值以及生成csr命令中的CN值
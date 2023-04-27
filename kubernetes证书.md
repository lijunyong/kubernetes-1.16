# cfssl参数的讲解
## 说明:本博客的作者也没有完全搞懂cfssl的配置，只能作参考，不敢误导别人
#### 网上没有任何关于cfssl的视频教程，完全不知道cfssl配置文件可以配什么参数，学习成本太高了
#### 网上博客完全没有说json配置文件的哪些字段是必须的，只能在哪些范文内取值，搞得头痛几天，搞编程最怕的是那些自定义参数，没任何说明的参数

## 相关证书类型
```
client certificate： 用于服务端认证客户端,例如etcdctl、etcd proxy、fleetctl、docker客户端
server certificate: 服务端使用，客户端以此验证服务端身份,例如docker服务端、kube-apiserver
peer certificate: 双向证书，用于etcd集群成员间通信
```

### 步骤一：下载相关软件，共3个软件
```
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 > /usr/local/bin/cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 > /usr/local/bin/cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 > /usr/local/bin/cfssl-certinfo
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson /usr/local/bin/cfssl-certinfo
# chmod +x /usr/local/bin/cfssl
# chmod +x /usr/local/bin/cfssljson
# chmod +x /usr/local/bin/cfssl-certinfo

# 创建生成证书的目录
mkdir /root/ssl
cd /root/ssl
```

### 步骤二：通过ca-csr.json配置文件参数生成CA证书和私钥，此步骤只需执行一次，后续直接使用此步骤生成的文件，签发多个网站证书。
###### 用下面命令生成ca-csr.json配置文件的模板
```
cfssl print-defaults csr > ca-csr.json
```
###### 修改内容ca-csr.json配置文件模板的内容
```json
{
    "CN": "kubernetes",  
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```
> 参数说明  
* CN 这个字段key的名称是只能取名是 CN, CN的key对应的value取值可以任意，可以是域名，也可以是任意内容
* CN:Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name); 浏览器使用该字段验证网站是否合法
* names的O表示Organization ，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)
* CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法
* C: Country， 国家
* L: Locality，地区，城市
* O: Organization Name，组织名称，公司名称
* OU: Organization Unit Name，组织单位名称，公司部门
* ST: State，州，省

##### 执行下面命令生成CA证书和私钥
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca 
```

### 步骤三：通过ca-config.json文件来配置证书生成策略，此步骤只需执行一次，后续直接使用此步骤生成的文件，签发多个网站证书。
###### 配置证书生成策略，让CA软件知道颁发什么样的证书
###### cfssl证书有三种类型
```
client certificate： 用于服务端认证客户端,例如etcdctl、etcd proxy、fleetctl、docker客户端
server certificate: 服务端使用，客户端以此验证服务端身份,例如docker服务端、kube-apiserver
peer certificate: 双向证书，用于etcd集群成员间通信
```
###### 用下面命令生成ca-config.json配置文件的模板
```
cfssl print-defaults config > ca-config.json
```
###### 修改ca-config.json, 分别配置针对三种不同证书类型的profile, 其中有效期43800h为5年
```json
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {    # ca-config.jso: 可以定义多个profiles, 分别指定不同的过期时间， 使用场景等参数，下面
            "server": {  # 这个字段名称任意，把server字段改成 aaa, bbb, ccc都可以，但是网上没有一篇博客有说这个字段任意
                "expiry": "43800h",
                "usages": [
                    "signing",  # signing : 表示该证书可用于签名其它证书，生成的ca.pem证书中  CA=TRUE
                    "key encipherment",
                    "server auth" # server auth : 表示client可以使用该CA对server提供的证书进行验证
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"  # client auth : 表示server 可以用该CA对client提供的证书进行验证
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",   # server auth : 表示client可以使用该CA对server提供的证书进行验证
                    "client auth"    # client auth : 表示server 可以用该CA对client提供的证书进行验证
                ]
            }
        }
    }
}
```

### 步骤四：证书生成与签名
###### 截止目前，基于CFSSL的CA已经配置完成，该CA如何颁发证书呢？CFSSL提供了两个命令：gencert和sign。gencert将自动处理整个证书生成过程。该过程需要两个文件，一个告诉CFSSL本地客户端CA的位置以及如何验证请求，即config文件，另一个为CSR配置信息，用于填充CSR 即csr文件。
###### 举例：创建 etcd证书签名请求文件 etcd-csr.json （config文件采用之前的ca-config.json）
```
{
    "CN": "etcd",
    "hosts": [
        "172.20.0.117",
        "172.20.0.118",
        "172.20.0.119"
    ],
    "key": {
        "algo":"rsa",
        "size":2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}

```
###### 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表，由于该证书后续被 etcd 集群使用
###### hosts 中的内容可以为空，即使按照上面的配置，向集群中增加新节点后也不需要重新生成证书。

###### 执行下面命令, 生成 etcd.csr, etcd-server-key.pem, etcd-server.pem 文件
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server etcd-csr.json | cfssljson -bare etcd-server

```

###### 执行下面命令, 生成 etcd.csr, etcd-peer-key.pem, etcd-peer.pem 文件
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-csr.json | cfssljson -bare etcd-peer
```

###### 执行下面命令, 生成 etcd.csr, etcd-client-key.pem, etcd-client.pem 文件
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client etcd-csr.json | cfssljson -bare etcd-client
```
```
[root@localhost ssl]# ls -l
total 60
-rw-r--r--. 1 root root  819 Apr 27 09:30 ca-config.json
-rw-r--r--. 1 root root 1001 Apr 26 14:31 ca.csr
-rw-r--r--. 1 root root  266 Apr 26 14:31 ca-csr.json
-rw-------. 1 root root 1675 Apr 26 14:31 ca-key.pem
-rw-r--r--. 1 root root 1359 Apr 26 14:31 ca.pem
-rw-r--r--. 1 root root 1054 Apr 27 10:24 etcd-client.csr
-rw-------. 1 root root 1675 Apr 27 10:24 etcd-client-key.pem
-rw-r--r--. 1 root root 1411 Apr 27 10:24 etcd-client.pem
-rw-r--r--. 1 root root  349 Apr 27 10:18 etcd-csr.json
-rw-r--r--. 1 root root 1054 Apr 27 10:21 etcd-peer.csr
-rw-------. 1 root root 1675 Apr 27 10:21 etcd-peer-key.pem
-rw-r--r--. 1 root root 1428 Apr 27 10:21 etcd-peer.pem
-rw-r--r--. 1 root root 1054 Apr 27 10:19 etcd-server.csr
-rw-------. 1 root root 1679 Apr 27 10:19 etcd-server-key.pem
-rw-r--r--. 1 root root 1411 Apr 27 10:19 etcd-server.pem
[root@localhost ssl]# 

```

### 步骤五：部署etcd集群
```
[root@localhost bin]# cat /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=httpss://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
          --name=etcd2        \
          --data-dir=/var/lib/etcd \
          --listen-peer-urls=https://172.20.0.118:2380 \
          --listen-client-urls=https://172.20.0.118:2379 \
          --initial-advertise-peer-urls=https://172.20.0.118:2380 \
          --initial-cluster=etcd1=https://172.20.0.117:2380,etcd2=https://172.20.0.118:2380,etcd3=https://172.20.0.119:2380 \
          --initial-cluster-state=new \
          --initial-cluster-token=k8s-etcd-cluster \
          --advertise-client-urls=https://172.20.0.118:2379 \
          --cert-file=/root/ssl/etcd-server.pem \
          --key-file=/root/ssl/etcd-server-key.pem \
          --peer-cert-file=/root/ssl/etcd-peer.pem \
          --peer-key-file=/root/ssl/etcd-peer-key.pem \
          --trusted-ca-file=/root/ssl/ca.pem \
          --peer-trusted-ca-file=/root/ssl/ca.pem


Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
```
/usr/local/bin/etcd \\
  --cert-file=/etc/etcd/kube-etcd.pem \\                   # 对外提供服务的服务器证书
  --key-file=/etc/etcd/kube-etcd-key.pem \\                # 服务器证书对应的私钥
  --peer-cert-file=/etc/etcd/kube-etcd-peer.pem \\         # peer 证书，用于 etcd 节点之间的相互访问
  --peer-key-file=/etc/etcd/kube-etcd-peer-key.pem \\      # peer 证书对应的私钥
  --trusted-ca-file=/etc/etcd/cluster-root-ca.pem \\       # 用于验证访问 etcd 服务器的客户端证书的 CA 根证书
  --peer-trusted-ca-file=/etc/etcd/cluster-root-ca.pem\\   # 用于验证 peer 证书的 CA 根证书
  ...
```

### 步骤六：查看etcd集群情况
```
[root@localhost bin]# etcdctl --endpoints=https://172.20.0.117:2379    --ca-file=/root/ssl/ca.pem    --cert-file=/root/ssl/etcd-client.pem    --key-file=/root/ssl/etcd-client-key.pem    cluster-health
member a8e52540a0b3791 is healthy: got healthy result from https://172.20.0.117:2379
member 3efb17b994edd6ef is healthy: got healthy result from https://172.20.0.118:2379
member de35aa42e14ffaff is healthy: got healthy result from https://172.20.0.119:2379
cluster is healthy
[root@localhost bin]# 
```


























###### 一些术语
```
# 密钥用法：
数字签名 Digital Signature
认可签名 Non Repudiation
密钥加密 key Encipherment
数据加密 Data Encipherment
密钥协商 key Agreement
证书签名 Key CertSign
CRL 签名 Crl Sign
仅仅加密 Encipher Only
仅仅解密 Decipher Only

# OpenSSL密钥用法：
数字签名 digitalSignature
认可签名 nonRepudiation
密钥加密 keyEncipherment
数据加密 dataEncipherment
密钥协商 keyAgreement
证书签名 keyCertSign
CRL 签名 cRLSign
仅仅加密 encipherOnly
仅仅解密 decipherOnly
```

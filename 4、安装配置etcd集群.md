# 安装etcd集群


需要为 etcd 集群创建加密通信的 TLS 证书，这里复用以前创建的 kubernetes 证书

``` bash
cp ca.pem kubernetes-key.pem kubernetes.pem /etc/kubernetes/ssl
```

+ kubernetes 证书的 `hosts` 字段列表中包含上面三台机器的 IP，否则后续证书校验会失败；

## 下载二进制文件

到 `https://github.com/coreos/etcd/releases` 页面下载最新版本的二进制文件

``` bash
wget https://github.com/coreos/etcd/releases/download/v3.2.8/etcd-v3.2.8-linux-amd64.tar.gz
tar -xvf etcd-v3.2.8-linux-amd64.tar.gz
mv etcd-v3.2.8-linux-amd64/etcd* /usr/local/bin
```

## 创建 etcd 的 systemd unit 文件
```
[root@localhost ~]# vi /usr/lib/systemd/system/etcd.service
```
创建etcd工作目录：mkdir /var/lib/etcd/
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
          --name=etcd1        \
          --data-dir=/var/lib/etcd \
          --listen-peer-urls=http://172.20.0.113:2380 \
          --listen-client-urls=http://172.20.0.113:2379 \
          --initial-advertise-peer-urls=http://172.20.0.113:2380 \
          --initial-cluster=etcd1=http://172.20.0.113:2380,etcd2=http://172.20.0.114:2380,etcd3=http://172.20.0.115:2380 \
          --initial-cluster-state=new \
          --initial-cluster-token=k8s-etcd-cluster \
          --advertise-client-urls=http://172.20.0.113:2379

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

注意：配置文件只是其中一台机器的，部署到另一台机器时，记得更改ip地址

## 启动 etcd 服务

``` bash
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

## 验证服务

在任一 kubernetes master 机器上执行如下命令：

``` bash
[root@localhost ~]# etcdctl --endpoints=http://172.20.0.113:2379 cluster-health
member c7f7263aa440312f is healthy: got healthy result from http://172.20.0.113:2379
member c80ae7c350fcecb3 is healthy: got healthy result from http://172.20.0.114:2379
member ddad4013ec9242ff is healthy: got healthy result from http://172.20.0.115:2379
cluster is healthy

```

# 安装flannel网络插件

所有的node节点都需要安装网络插件才能让所有的Pod加入到同一个局域网中，本文是安装flannel网络插件的参考文档。

访问github，下载对应版本flannel，https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz。

```shell

[root@localhost opt]# tar -zxvf flannel-v0.11.0-linux-amd64.tar.gz
flanneld
mk-docker-opts.sh
README.md

[root@localhost opt]# cat /usr/lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \
          -etcd-endpoints=http://172.20.0.113:2379,http://172.20.0.114:2379,http://172.20.0.115:2379 \
          -etcd-prefix=/kube-centos/network
ExecStartPost=/usr/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```
查看生成的/run/flannel/subnet.env：

```
[root@localhost flannel]# cat subnet.env 
DOCKER_OPT_BIP="--bip=172.30.52.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NETWORK_OPTIONS=" --bip=172.30.52.1/24 --ip-masq=true --mtu=1450"
```

**启动flannel**

```shell
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```

## etcd查询flanneld配置
```
[root@localhost ~]# etcdctl --endpoints=http://172.20.0.113:2379 ls /kube-centos/network/
/kube-centos/network/config
/kube-centos/network/subnets
[root@localhost ~]# etcdctl --endpoints=http://172.20.0.113:2379 get /kube-centos/network/config
{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}
[root@localhost ~]# etcdctl --endpoints=http://172.20.0.113:2379 ls /kube-centos/network/subnets
/kube-centos/network/subnets/172.30.78.0-24
/kube-centos/network/subnets/172.30.74.0-24
/kube-centos/network/subnets/172.30.43.0-24
[root@localhost ~]# etcdctl --endpoints=http://172.20.0.113:2379 get /kube-centos/network/subnets/172.30.74.0-24/
{"PublicIP":"172.20.0.113","BackendType":"vxlan","BackendData":{"VtepMAC":"4a:6c:70:ab:d6:af"}}
[root@localhost ~]# 

```


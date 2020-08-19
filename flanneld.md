```
[root@node1 ~]# cat /usr/lib/systemd/system/flanneld.service 
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

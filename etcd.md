```
[root@node1 ~]# cat /usr/lib/systemd/system/etcd.service 
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

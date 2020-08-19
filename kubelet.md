```
[root@node1 ~]# cat /usr/lib/systemd/system/kubelet.service 
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --hostname-override=172.20.0.113 \
  --config=/var/lib/kubelet/config.yaml \
  --pod-infra-container-image=172.20.0.110/k8s/pause-amd64:3.1 \
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target


[root@node1 ~]# cat /var/lib/kubelet/config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 172.20.0.113
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
port: 10250
healthzPort: 10248
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS: ["10.254.0.2"]
clusterDomain: cluster.local
failSwapOn: false
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110


```

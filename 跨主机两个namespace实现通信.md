
## 分别在两台主机上放开ipv4转发
```aidl
root@lee-120:~# echo "1" > /proc/sys/net/ipv4/ip_forward
```
## 在第一台主机执行
```aidl
root@lee-120:~# echo "1" > /proc/sys/net/ipv4/ip_forward
root@lee-120:~# ip netns add ns1
root@lee-120:~# ip link add veth1 type veth peer name veth2
root@lee-120:~# ip link set veth1 netns ns1
root@lee-120:~# ip netns exec ns1 ip addr add 192.168.1.1/24 dev veth1
root@lee-120:~# ip netns exec ns1 ip link set veth1 up
root@lee-120:~# ip link set veth2 up
root@lee-120:~# ip link add name br0 type bridge
root@lee-120:~# ip link set dev br0 up
root@lee-120:~# ip link set veth2 master br0
root@lee-120:~# ip addr add 192.168.1.254/24 dev br0
root@lee-120:~# route add -net 192.168.2.0/24 gw 172.20.0.121
root@lee-120:~# ip netns exec ns1 ping 192.168.2.1
connect: Network is unreachable
root@lee-120:~# ip netns exec ns1 route add -net 192.168.2.0/24 gw 192.168.1.254
root@lee-120:~# ip netns exec ns1 ping 192.168.2.1
PING 192.168.2.1 (192.168.2.1) 56(84) bytes of data.
^C
--- 192.168.2.1 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3065ms

root@lee-120:~# ip netns exec ns1 ping 192.168.2.1
PING 192.168.2.1 (192.168.2.1) 56(84) bytes of data.
64 bytes from 192.168.2.1: icmp_seq=33 ttl=62 time=0.290 ms
64 bytes from 192.168.2.1: icmp_seq=34 ttl=62 time=0.797 ms
64 bytes from 192.168.2.1: icmp_seq=35 ttl=62 time=0.584 ms
64 bytes from 192.168.2.1: icmp_seq=36 ttl=62 time=0.456 ms
64 bytes from 192.168.2.1: icmp_seq=37 ttl=62 time=0.427 ms
64 bytes from 192.168.2.1: icmp_seq=38 ttl=62 time=0.353 ms
64 bytes from 192.168.2.1: icmp_seq=39 ttl=62 time=0.296 ms
64 bytes from 192.168.2.1: icmp_seq=40 ttl=62 time=0.294 ms
64 bytes from 192.168.2.1: icmp_seq=41 ttl=62 time=0.395 ms
```
## 在第二台主机执行
```aidl
root@lee-121:~# echo "1" > /proc/sys/net/ipv4/ip_forward
root@lee-121:~# ip netns add ns1
root@lee-121:~# ip link add veth1 type veth peer name veth2
root@lee-121:~# ip link set veth1 netns ns1
root@lee-121:~# ip netns exec ns1 ip addr add 192.168.2.1/24 dev veth1
root@lee-121:~# ip netns exec ns1 ip link set veth1 up
root@lee-121:~# ip link set veth2 up
root@lee-121:~# ip link add name br0 type bridge
root@lee-121:~# ip link set dev br0 up
root@lee-121:~# ip link set veth2 master br0
root@lee-121:~# ip addr add 192.168.2.254/24 dev br0
root@lee-121:~# route add -net 192.168.1.0/24 gw 172.20.0.120
root@lee-121:~# route -n
内核 IP 路由表
目标            网关            子网掩码        标志  跃点   引用  使用 接口
0.0.0.0         172.20.0.2      0.0.0.0         UG    0      0        0 ens33
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 ens33
172.20.0.0      0.0.0.0         255.255.255.0   U     0      0        0 ens33
192.168.1.0     172.20.0.120    255.255.255.0   UG    0      0        0 ens33
192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 br0
root@lee-121:~# route add -net 192.168.2.0/24 gw 192.168.2.254
root@lee-121:~# ip netns exec ns1 route add -net 192.168.1.0/24 gw 192.168.2.254
root@lee-121:~# ip netns add ns2
root@lee-121:~# ip link add veth3 type veth peer name veth4
root@lee-121:~# ip link set veth3 netns ns2
root@lee-121:~# ip netns exec ns2 ip addr add 192.168.2.3/24 dev veth3
root@lee-121:~# ip netns exec ns2 ip link set veth3 up
root@lee-121:~# ip link set veth4 up
root@lee-121:~# ip link set veth4 master br0
root@lee-121:~# ip netns exec ns2 ping 192.168.2.1
PING 192.168.2.1 (192.168.2.1) 56(84) bytes of data.
64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=0.057 ms
64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=0.045 ms
```

# 跨主机实现两个namespace通信

## 在第一台主机执行
```
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
```
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

# 跨主机vxlan配置
## 第一台主机
```
root@lee-120:~# ip link add vxlan0 type vxlan id 42 dstport 4789 remote 172.20.0.121 dev ens33
root@lee-120:~# ip addr add 172.17.1.2/24 dev vxlan0
root@lee-120:~# ip link set vxlan0 up
root@lee-120:~# bridge fdb
33:33:00:00:00:01 dev ens33 self permanent
01:00:5e:00:00:01 dev ens33 self permanent
33:33:ff:3b:cd:2f dev ens33 self permanent
33:33:00:00:00:fb dev ens33 self permanent
01:00:5e:00:00:fb dev ens33 self permanent
00:00:00:00:00:00 dev vxlan0 dst 172.20.0.121 via ens33 self permanent
```

## 第二台主机
```aidl
root@lee-121:~# ip link add vxlan0 type vxlan id 42 dstport 4789 remote 172.20.0.120 dev ens33
root@lee-121:~# ip addr add 172.17.1.3/24 dev vxlan0
root@lee-121:~# ip link set vxlan0 up
```

# 跨主机vxlan多播配置
## 第一台主机
```aidl
root@lee-120:~# ip link add vxlan0 type vxlan id 42 dstport 4789 group 224.1.1.1 dev ens33
root@lee-120:~# ip addr add 172.17.1.2/24 dev vxlan0
root@lee-120:~# ip link set vxlan0 up
root@lee-120:~# bridge fdb
33:33:00:00:00:01 dev ens33 self permanent
01:00:5e:00:00:01 dev ens33 self permanent
33:33:ff:3b:cd:2f dev ens33 self permanent
33:33:00:00:00:fb dev ens33 self permanent
01:00:5e:00:00:fb dev ens33 self permanent
01:00:5e:01:01:01 dev ens33 self permanent
00:00:00:00:00:00 dev vxlan0 dst 224.1.1.1 via ens33 self permanent
```

## 第二台主机
```
root@lee-121:~# ip link add vxlan0 type vxlan id 42 dstport 4789 group 224.1.1.1 dev ens33
root@lee-121:~# ip addr add 172.17.1.3/24 dev vxlan0
root@lee-121:~# ip link set vxlan0 up
```

# 跨主机vxlan多播配置实现namespace通信
## 第一台主机
```
root@lee-120:~# ip link add vxlan0 type vxlan id 42 dstport 4789 local 172.20.0.120 group 224.1.1.1 dev ens33
root@lee-120:~# ip link set vxlan0 up
root@lee-120:~# ip link add br0 type bridge
root@lee-120:~# ip link set vxlan0 master br0
root@lee-120:~# ip link set br0 up
root@lee-120:~# ip netns add ns1
root@lee-120:~# ip link add veth0 type veth peer name veth1
root@lee-120:~# ip link set veth1 master br0
root@lee-120:~# ip link set veth0 netns ns1
root@lee-120:~# ip netns exec ns1 ip addr add 172.17.1.2/24 dev veth0
root@lee-120:~# ip netns exec ns1 ip link set veth0 up
root@lee-120:~# ip link set veth1 up
root@lee-120:~# ip netns exec ns1 nc 172.17.1.3 1234
hello
123456
```

## 第二台主机
```
root@lee-121:~# ip link add vxlan0 type vxlan id 42 dstport 4789 local 172.20.0.121 group 224.1.1.1 dev ens33
root@lee-121:~# ip link set vxlan0 up
root@lee-121:~# ip link add br0 type bridge
root@lee-121:~# ip link set vxlan0 master br0
root@lee-121:~# ip link set br0 up
root@lee-121:~# ip netns add ns1
root@lee-121:~# ip link add veth0 type veth peer name veth1
root@lee-121:~# ip link set veth1 master br0
root@lee-121:~# ip link set veth0 netns ns1
root@lee-121:~# ip netns exec ns1 ip addr add 172.17.1.3/24 dev veth0
root@lee-121:~# ip netns exec ns1 ip link set veth0 up
root@lee-121:~# ip link set veth1 up
root@lee-121:~# ip netns nc -l 172.17.1.3 1234
Command "nc" is unknown, try "ip netns help".
root@lee-121:~# ip netns exec ns1 nc -l 172.17.1.3 1234
hello
123456
```
# ipvlan通信
```
root@lee-120:~# ip netns add ns1
root@lee-120:~# ip netns add ns2
root@lee-120:~# ip link add ipv1 link ens33 type ipvlan mode l3
root@lee-120:~# ip link add ipv2 link ens33 type ipvlan mode l3
root@lee-120:~# ip link set ipv1 netns ns1
root@lee-120:~# ip link set ipv2 netns ns2
root@lee-120:~# ip netns exec ns1 ip addr add 192.168.10.1/24 dev ipv1
root@lee-120:~# ip netns exec ns1 ip link set ipv1 up
root@lee-120:~# ip netns exec ns2 ip addr add 192.168.20.1/24 dev ipv2
root@lee-120:~# ip netns exec ns2 ip link set ipv2 up
root@lee-120:~# ip netns exec ns2 ifconfig
ipv2      Link encap:以太网  硬件地址 00:0c:29:3b:cd:2f  
          inet 地址:192.168.20.1  广播:0.0.0.0  掩码:255.255.255.0
          inet6 地址: fe80::c:2900:23b:cd2f/64 Scope:Link
          UP BROADCAST RUNNING NOARP MULTICAST  MTU:1500  跃点数:1
          接收数据包:0 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:0 错误:0 丢弃:2 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:0 (0.0 B)  发送字节:0 (0.0 B)

root@lee-120:~# ip netns exec ns1 ip route add default dev ipv1
root@lee-120:~# ip netns exec ns2 ip route add default dev ipv2
root@lee-120:~# ip netns exec ns1 ping 192.168.20.1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=0.053 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=0.070 ms
```

# 参考flannel vxlan网络模式，手工实现跨主机通信
## 第一台主机 172.20.0.120
```
 echo 1 > /proc/sys/net/ipv4/ip_forward
 ip link add flannel.1 type vxlan id 42 dstport 4789 dev ens33
 ip addr add 10.10.1.0/32 dev flannel.1
 ip link set flannel.1 up
 ip link add cni0 type bridge
 ip addr add 10.10.1.1/24 dev cni0
 ip link set cni0 up
 ip netns add ns1
 ip link add veth0 type veth peer name veth1
 ip link set veth0 netns ns1
 ip netns exec ns1 ifconfig
 ip netns exec ns1 ip addr add 10.10.1.3/24 dev veth0
 ip netns exec ns1 ip link set veth0 up
 ip netns exec ns1 ifconfig
 ip link set veth1 up
 ip link set veth1 master cni0
 route -n
 ip route add 10.10.2.0/24 via 10.10.2.0 dev flannel.1 onlink
 bridge fdb append 02:ae:fc:d0:50:d7 dev flannel.1 dst 172.20.0.121
 ifconfig
 ip neigh show
 ip neigh add 10.10.2.0 lladdr 02:ae:fc:d0:50:d7 dev flannel.1
 ping 10.10.2.0
 ip netns exec ns1 route -n
 ip netns exec ns1 route add default gw 10.10.1.1
 ip netns exec ns1 route -n
 ip netns exec ns1 ping 10.10.2.4
```
## 第二台主机
```aidl
root@lee-121:~# echo 1 > /proc/sys/net/ipv4/ip_forward
root@lee-121:~# ip link add flannel.1 type vxlan id 42 dstport 4789 dev ens33
root@lee-121:~# ip addr add 10.10.2.0/32 dev flannel.1
root@lee-121:~# ip link set flannel.1 up
root@lee-121:~# ip link add cni0 type bridge
root@lee-121:~# ip addr add 10.10.2.1/24 dev cni0
root@lee-121:~# ip link set cni0 up
root@lee-121:~# ip netns add ns1
root@lee-121:~# ip link add veth0 type veth peer name veth1
root@lee-121:~# ip link set veth0 netns ns1
root@lee-121:~# ip netns exec ns1 ip addr add 10.10.2.4/24 dev veth0
root@lee-121:~# ip netns exec ns1 ip link set veth0 up
root@lee-121:~# ip netns exec ns1 ifconfig
veth0     Link encap:以太网  硬件地址 4e:84:41:49:74:76  
          inet 地址:10.10.2.4  广播:0.0.0.0  掩码:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  跃点数:1
          接收数据包:0 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:0 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:0 (0.0 B)  发送字节:0 (0.0 B)

root@lee-121:~# ip link set veth1 up
root@lee-121:~# ip link set veth1 master cni0
root@lee-121:~# ifconfig
cni0      Link encap:以太网  硬件地址 26:3f:29:e0:6e:aa  
          inet 地址:10.10.2.1  广播:0.0.0.0  掩码:255.255.255.0
          inet6 地址: fe80::14a9:1aff:fe3f:ee75/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
          接收数据包:3 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:58 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:168 (168.0 B)  发送字节:6558 (6.5 KB)

ens33     Link encap:以太网  硬件地址 00:0c:29:36:04:a1  
          inet 地址:172.20.0.121  广播:172.20.0.255  掩码:255.255.255.0
          inet6 地址: fe80::20c:29ff:fe36:4a1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
          接收数据包:405 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:361 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:35564 (35.5 KB)  发送字节:40850 (40.8 KB)

flannel.1 Link encap:以太网  硬件地址 02:ae:fc:d0:50:d7  
          inet 地址:10.10.2.0  广播:0.0.0.0  掩码:255.255.255.255
          UP BROADCAST RUNNING MULTICAST  MTU:1450  跃点数:1
          接收数据包:0 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:0 错误:0 丢弃:17 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:0 (0.0 B)  发送字节:0 (0.0 B)

lo        Link encap:本地环回  
          inet 地址:127.0.0.1  掩码:255.0.0.0
          inet6 地址: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  跃点数:1
          接收数据包:5908 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:5908 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:439212 (439.2 KB)  发送字节:439212 (439.2 KB)

veth1     Link encap:以太网  硬件地址 26:3f:29:e0:6e:aa  
          inet6 地址: fe80::243f:29ff:fee0:6eaa/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
          接收数据包:9 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:23 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:726 (726.0 B)  发送字节:2853 (2.8 KB)

root@lee-121:~# ip route add 10.10.1.0/24 via 10.10.1.0 dev flannel.1 onlink
root@lee-121:~# bridge fdb append 86:50:b1:6c:1f:18 dev flannel.1 dst 172.20.0.120
root@lee-121:~# ip neigh add 10.10.1.0 lladdr 86:50:b1:6c:1f:18 dev flannel.1
root@lee-121:~# ip netns exec ns1 route -n
内核 IP 路由表
目标            网关            子网掩码        标志  跃点   引用  使用 接口
10.10.2.0       0.0.0.0         255.255.255.0   U     0      0        0 veth0
root@lee-121:~# ip netns exec ns1 route add default gw 10.10.2.1
root@lee-121:~# 
```

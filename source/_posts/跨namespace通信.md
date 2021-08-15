---
title: 跨namespace通信
date: 2021-08-15 18:14:31
tags: network、linux、veth pair
---

## netns
> - 对于每个 `network namespace` 来说，它会有自己独立的网卡、路由表、ARP 表、iptables 等和网络相关的资源

## ip netns
```
ip netns help


Usage: ip netns list
       ip netns add NAME
       ip netns set NAME NETNSID
       ip [-all] netns delete [NAME]
       ip netns identify [PID]
       ip netns pids NAME
       ip [-all] netns exec [NAME] cmd ...
       ip netns monitor
       ip netns list-id
```

## 实验

### veth pair通信

#### 配置
```
#创建ns2
ip netns add ns2 

#再次查看
ip netns l
ns1
ns2

#增加veth设备
ip link add type veth 

#增加带名字的veth设备
ip link add veth0 type veth peer name veth1

#把这对 veth pair 分别放到已经两个 namespace
ip link set veth0 netns ns1
ip link set veth1 netns ns2


#给这对 veth pair 配置上 ip 地址，并启用它们
ip netns exec ns1 ip link set veth0 up
ip netns exec ns1 ip addr add 172.16.1.1/24 dev veth0

ip netns exec ns2 ip link set veth1 up
ip netns exec ns2 ip addr add 172.16.1.2/24 dev veth1
```

#### 测试
```
ip netns exec ns2 ping 172.16.1.1 -c 4
```

#### 验证
```
#直接抓包，啥也抓不到
tcpdump host 172.16.1.1

#进入ns里面抓包
ip netns exec ns1 tcpdump host 172.16.1.1

10:49:46.218008 IP 172.16.1.2 > 10-9-153-165: ICMP echo request, id 10798, seq 1, length 64
10:49:46.218024 IP 10-9-153-165 > 172.16.1.2: ICMP echo reply, id 10798, seq 1, length 64
10:49:47.250282 IP 172.16.1.2 > 10-9-153-165: ICMP echo request, id 10798, seq 2, length 64
10:49:47.250304 IP 10-9-153-165 > 172.16.1.2: ICMP echo reply, id 10798, seq 2, length 64
10:49:48.274292 IP 172.16.1.2 > 10-9-153-165: ICMP echo request, id 10798, seq 3, length 64
10:49:48.274315 IP 10-9-153-165 > 172.16.1.2: ICMP echo reply, id 10798, seq 3, length 64
10:49:49.298327 IP 172.16.1.2 > 10-9-153-165: ICMP echo request, id 10798, seq 4, length 64
10:49:49.298349 IP 10-9-153-165 > 172.16.1.2: ICMP echo reply, id 10798, seq 4, length 64
```

### bridge（网桥）
> `veth pair` 可以实现两个 `network namespace` 之间的通信，但是当多个 `namespace` 需要通信的时候，就无能为力了。此时就需要`bridge`

#### 配置网桥
```
# 创建网桥
ip link add br0 type bridge
# 拉起网桥
ip link set dev br0 up

```

#### 配置net0
```
# 创建net0
ip netns add net0

# 创建veth
ip link add type veth

# veth1 移动至net0
ip link set dev veth1 netns net0

# 修改veth1的名字为eth0
ip netns exec net0 ip link set dev veth1 name eth0

# 给eth0配置ip
ip netns exec net0 ip addr add 192.168.10.1/24 dev eth0
# 拉起eth0
ip netns exec net0 ip link set dev eth0 up

# veth0连接到br0
ip link set dev veth0 master br0
# 拉起br0
ip link set dev veth0 up
```

#### 配置net1(同net0)
```
ip netns add net1
ip link add type veth
ip link set dev veth2 netns net1
ip netns exec net1 ip link set dev veth2 name eth0
ip netns exec net1 ip addr add 192.168.10.2/24 dev eth0
ip netns exec net0 ip link set dev eth0 up
ip link set dev veth3 master br0
ip link set dev veth3 up
```

#### 测试
```
ip netns exec net1 ping 192.168.10.1 -c 4

PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=0.087 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=0.082 ms
^C
--- 192.168.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 0.082/0.084/0.087/0.009 ms
```

### 参考
> - [Linux 网络虚拟化： network namespace 简介](https://cizixs.com/2017/02/10/network-virtualization-network-namespace/)
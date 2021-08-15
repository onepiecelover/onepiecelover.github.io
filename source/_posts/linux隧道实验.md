---
title: linux隧道实验
date: 2021-08-15 18:15:54
tags: gre、ipip、linux、network
---

### 隧道介绍
> - 为了在TCP/IP网络中传输其他协议的数据包。通过在源协议数据包上再套上一个IP协议头来实现，形成的IP数据包通过Internet后卸去IP头，还原成源协议数据包，传送给目的站点。

### 隧道模式
> ps:一共有5种模式 ipip | gre | sit | isatap | vti
> - ipip：即 IPv4 in IPv4，在 IPv4 报文的基础上再封装一个 IPv4 报文。
> - gre：即通用路由封装（Generic Routing Encapsulation），定义了在任意一种网络层协议上封装其他任意一种网络层协议的机制，IPv4 和 IPv6 都适用。
> - sit：和 ipip 类似，不同的是 sit 是用 IPv4 报文封装 IPv6 报文，即 IPv6 over IPv4。
> - isatap：即站内自动隧道寻址协议（Intra-Site Automatic Tunnel Addressing Protocol），和 sit 类似，也是用于 IPv6 的隧道封装。
> - vti：即虚拟隧道接口（Virtual Tunnel Interface），是 cisco 提出的一种 IPsec 隧道技术。

### 实践-准备两台云主机
> - 10.9.142.136
> - 10.9.29.68

### GRE模式(通用路由封装)
> - 在10.9.29.68机器上执行以下命令
```
modprobe ipip
modprobe ip_gre
ip tunnel add tun0 mode gre local 10.9.29.68 remote 10.9.142.136 
ip link set tun0 up
ip addr add 192.168.10.1 peer 192.168.10.2 dev tun0
ip route add 192.168.10.1/24 dev tun0
```

> - 在10.9.142.136执行以下命令
```
modprobe ipip
modprobe ip_gre
ip tunnel add tun0 mode gre local 10.9.142.136 remote 10.9.29.68 
ip link set tun0 up
ip addr add 192.168.10.2 peer 192.168.10.1 dev tun0
ip route add 192.168.10.1/24 dev tun0
```

### 查看隧道报文
> - 在`10.9.142.136`机器上启动nginx服务
> - 从`10.9.29.68`机器访问，命令 `curl192.168.10.2`
> - 抓包 `tcpdump host 10.9.142.136`

```
14:01:41.952193 IP 10-9-29-68 > 10.9.142.136: GREv0, length 64: IP 10-9-29-68.35892 > 192.168.10.2.http: Flags [S], seq 3964984022, win 27800, options [mss 1390,sackOK,TS val 4264166462 ecr 0,nop,wscale 7], length 0
14:01:41.954756 IP 10.9.142.136 > 10-9-29-68: GREv0, length 64: IP 192.168.10.2.http > 10-9-29-68.35892: Flags [S.], seq 1878434104, ack 3964984023, win 28960, options [mss 1460,sackOK,TS val 972483763 ecr 4264166462,nop,wscale 7], length 0
14:01:41.954865 IP 10-9-29-68 > 10.9.142.136: GREv0, length 56: IP 10-9-29-68.35892 > 192.168.10.2.http: Flags [.], ack 1, win 218, options [nop,nop,TS val 4264166464 ecr 972483763], length 0
14:01:41.955084 IP 10-9-29-68 > 10.9.142.136: GREv0, length 132: IP 10-9-29-68.35892 > 192.168.10.2.http: Flags [P.], seq 1:77, ack 1, win 218, options [nop,nop,TS val 4264166464 ecr 972483763], length 76: HTTP: GET / HTTP/1.1
14:01:41.955385 IP 10.9.142.136 > 10-9-29-68: GREv0, length 56: IP 192.168.10.2.http > 10-9-29-68.35892: Flags [.], ack 77, win 227, options [nop,nop,TS val 972483765 ecr 4264166464], length 0
14:01:41.955538 IP 10.9.142.136 > 10-9-29-68: GREv0, length 294: IP 192.168.10.2.http > 10-9-29-68.35892: Flags [P.], seq 1:239, ack 77, win 227, options [nop,nop,TS val 972483765 ecr 4264166464], length 238: HTTP: HTTP/1.1 200 OK
14:01:41.955546 IP 10-9-29-68 > 10.9.142.136: GREv0, length 56: IP 10-9-29-68.35892 > 192.168.10.2.http: Flags [.], ack 239, win 226, options [nop,nop,TS val 4264166465 ecr 972483765], length 0
14:01:41.955615 IP 10.9.142.136 > 10-9-29-68: GREv0, length 668: IP 192.168.10.2.http > 10-9-29-68.35892: Flags [P.], seq 239:851, ack 77, win 227, options [nop,nop,TS val 972483765 ecr 4264166464], length 612: HTTP
14:01:41.955625 IP 10-9-29-68 > 10.9.142.136: GREv0, length 56: IP 10-9-29-68.35892 > 192.168.10.2.http: Flags [.], ack 851, win 236, options [nop,nop,TS val 4264166465 ecr 972483765], length 0
14:01:41.956800 IP 10-9-29-68 > 10.9.142.136: GREv0, length 56: IP 10-9-29-68.35892 > 192.168.10.2.http: Flags [F.], seq 77, ack 851, win 236, options [nop,nop,TS val 4264166466 ecr 972483765], length 0
14:01:41.957046 IP 10.9.142.136 > 10-9-29-68: GREv0, length 56: IP 192.168.10.2.http > 10-9-29-68.35892: Flags [F.], seq 851, ack 78, win 227, options [nop,nop,TS val 972483767 ecr 4264166466], length 0
14:01:41.957055 IP 10-9-29-68 > 10.9.142.136: GREv0, length 56: IP 10-9-29-68.35892 > 192.168.10.2.http: Flags [.], ack 852, win 236, options [nop,nop,TS val 4264166466 ecr 972483767], length 0
```


#### 参考：
> - [Linux中IP隧道](https://sites.google.com/site/emmoblin/linux-network-1/linux-zhongip-sui-dao)
> - [什么是 IP 隧道，Linux 怎么实现隧道通信](https://cloud.tencent.com/developer/article/1432489)

---
title: openwrt 上搭建 ShadowVPN Server
date: 2016-07-15
tags:
  - ShadowVPN
  - VPN
  - openwrt
  - shadowsock
---
该文主要是针对 ShadowVPN 在 openwrt 上面配置成 Server 端的介绍。
<!-- more -->

### 关于ShadowVPN
关于 ShadownVPN 的介绍除了看其官方的 wiki 文档外，我个人有几点理解:
 - 使用的是 udp 协议，有利于速度的同时也给 GFW 抓取其特征增加难度
 - 使用了 libsodium 进行对称的点对点加/解密，CPU消耗上比较乐观
 - 基于 tun 的路由配置较简单，便于管理
### 开始安装
获取对应的 ipk 安装包，所有安装包见：[Sourceforge openwrt-shadowvpn][1] 以我的 HG556A 为例安装的是ShadowVPN_0.2.0-1_brcm63xx.ipk


```bash
root@OpenWrt:/tmp# wget https://sourceforge.net/projects/openwrt-dist/files/shadowvpn/0.2.0-e335cac/ShadowVPN_0.2.0-1_brcm63xx.ipk/download
root@OpenWrt:/tmp# opkg install ShadowVPN_0.2.0-1_brcm63xx.ipk
root@OpenWrt:/tmp# opkg files ShadowVPN
Package ShadowVPN (0.2.0-1) is installed on root and has the following files:
/etc/init.d/shadowvpn
/etc/hotplug.d/iface/30-shadowvpn
/etc/shadowvpn/client_down.sh
/usr/bin/shadowvpn
/etc/shadowvpn/client_up.sh
/etc/config/shadowvpn
```


### 配置
由于安装包默认是提供给 client 使用的，所以我们需要增加 server 用的配置跟去掉一下不需要的文件

- 增加配置 server.conf , 注意文件格式是 字段=值， = 号两边都不能有等号。因为这个是一个shell的配置。


```shell
root@OpenWrt:/tmp# cat /etc/shadowvpn/server.conf | grep -v ^# | grep -v ^$
server=0.0.0.0
port=1123
password=68b35f94c6436421ed697011029b43c7
mode=server
concurrency=1
mtu=1432
intf=tun0
net=10.8.0.1/16
up=/etc/shadowvpn/server_up.sh
down=/etc/shadowvpn/server_down.sh
pidfile=/var/run/shadowvpn.pid
logfile=/var/log/shadowvpn.log

```


- 增加 server_up.sh， 这个是 server 启动的时候执行的命令脚本.

```shell
root@OpenWrt:/tmp# cat /etc/shadowvpn/server_up.sh | grep -v ^# | grep -v ^$
sysctl -w net.ipv4.ip_forward=1
ip addr add $net dev $intf
ip link set $intf mtu $mtu
ip link set $intf up
if !(iptables-save -t nat | grep -q "shadowvpn"); then
  iptables -t nat -I POSTROUTING -s $net ! -d $net -m comment --comment "shadowvpn" -j MASQUERADE
fi
iptables -I FORWARD -s $net -j ACCEPT
iptables -I FORWARD -d $net -j ACCEPT
iptables -t mangle -A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
echo $0 done
# 注意上面的 FORWARD 规则，如果不加，那么 client 就走不出去，即上不了公网，只能跟 10.8.0.0/16 网络相通。
```


- 增加 server_down.sh， 这个是 server 关闭的时候执行的命令脚本.

```shell
root@OpenWrt:/tmp# cat /etc/shadowvpn/server_down.sh | grep -v ^# | grep -v ^$
iptables -t nat -D POSTROUTING -s $net ! -d $net -m comment --comment "shadowvpn" -j MASQUERADE
iptables -D FORWARD -s $net -j ACCEPT
iptables -D FORWARD -d $net -j ACCEPT
iptables -t mangle -D FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
echo $0 done
```


- 删掉不需要的文件  因为刚才说了这个 ipk 包主要是针对 client 的。我们需要的仅仅是 /usr/bin/shadowvpn 这个可执行的文件。



```shell
root@OpenWrt:/tmp# /etc/init.d/shadowvpn disable
root@OpenWrt:/tmp# mv /etc/init.d/shadowvpn /root/shadowvpn.bk
root@OpenWrt:/tmp# mv /etc/hotplug.d/iface/30-shadowvpn/root/30-shadowvpn.bk
```


- 开放 wan 口的shadowvpn的服务端口1123 具体多少看  server.conf 配置

```shell
root@OpenWrt:~# cat >> /etc/config/firewall <<_EOF
config rule
        option name 'Allow-ShadowVPN'
        option src 'wan'
        option proto 'udp'
        option dest_port '1123'
        option target 'ACCEPT'
        option family 'ipv4'
 _EOF
 root@OpenWrt:~# /etc/init.d/firewall reload
```
### 启动服务端


```shell
root@OpenWrt:~# /usr/bin/shadowvpn -c /etc/shadowvpn/server.conf -s start
```

启动成功后 ip a 可以看到一个 tun0 的网口

```shell
root@OpenWrt:~# ip addr show dev tun0
14: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1432 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.8.0.1/16 scope global tun0
       valid_lft forever preferred_lft forever
```

### 客户端 
我客户端是公司的电脑，系统是 Linux， 所以直接从 github clone 下来编译即可。由于种种原因作者已经从 github 上面把代码删掉了，但是有很多人都fork了，所以可以下载其中一个 比如： [Github][2]
编译就不多说，安装官方说明操作就行。注意一下 libsodium 这个需要从[Github][3]获取。
#### 配置客户端

```shell
yorks /etc/shadowvpn $ cat client.conf  | grep -v ^# | grep -v ^$
server=120.85.223.199
##上面的ip 是 ${您 openwrt wan口 外网ip地址}
port=1123
password=68b35f94c6436421ed697011029b43c7
mode=client
concurrency=1
mtu=1432
intf=tun0
net=10.8.0.2/24
up=/etc/shadowvpn/client_up.sh
down=/etc/shadowvpn/client_down.sh
pidfile=/var/run/shadowvpn.pid
logfile=/var/log/shadowvpn.log
```
-  修改 client_up.sh client_down.sh 的脚本

```shell
# client_up.sh 注释下面两行 因为我用的是局部路由，不是全局路由
# Shadow default route using two /1 subnets
##ip route add   0/1 dev $intf
##ip route add 128/1 dev $intf

# client_down.sh 注释下面两行
##ip route del   0/1
##ip route del 128/1
```

### 启动客户端

```shell
[root@c720 ShadowVPN]#  ./bin/shadowvpn -c etc/shadowvpn/client.conf -s start
Sat May 21 18:42:23 2016 warning: concurrency is temporarily disabled on this version, make sure to set concurrency=1 on the other side
started
[root@c720 ShadowVPN]# ip addr show dev tun0
5: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1432 qdisc noqueue state UNKNOWN group default qlen 500
    link/none 
    inet 10.8.0.2/24 scope global tun0
       valid_lft forever preferred_lft forever

```

### 测试

```shell
[root@c720 ShadowVPN]# ping 10.8.0.1
PING 10.8.0.1 (10.8.0.1) 56(84) bytes of data.
64 bytes from 10.8.0.1: icmp_seq=1 ttl=64 time=5.28 ms
64 bytes from 10.8.0.1: icmp_seq=2 ttl=64 time=4.85 ms
64 bytes from 10.8.0.1: icmp_seq=3 ttl=64 time=4.81 ms
# ping 通 openwrt 的服务端ip了 说明服务是正常的。
```
### 路由管理

举例说明，比如我要连回家上 ip.cn 这个网站，那么这样操作:

```shell
[root@c720 ShadowVPN]# dig ip.cn +short
42.159.28.168
[root@c720 ShadowVPN]# ip r add 42.159.28.168 dev tun0
[root@c720 ShadowVPN]# curl http://ip.cn
当前 IP：120.85.223.199 来自：广东省广州市 联通

出现的 ip 地址是 server 的外网. 这就说明是走了 vpn 通道了。
其他的路由也一样，比如 qq 的，想再公司上 qq 先查 qq 的 ip 地址范围，然后都加到 tun0 口出去即可。
```


   [1]: https://sourceforge.net/projects/openwrt-dist/files/shadowvpn/ "Sourceforge openwrt-shadowvpn"
   [2]: https://github.com/Long-live-shadowsocks/ShadowVPN "Github Long-live ShadownVPN"
   [3]: https://github.com/jedisct1/libsodium.git    "Github libsodium"

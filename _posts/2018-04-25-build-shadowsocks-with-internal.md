---
layout:  post
title:  "基于shadowsocks构建局域网翻墙"
date:  2018-04-25
lang: zh
---

### 理论概述

 大致的思路
 1. NAT实现局域网机器与internet联通
 2. dnsmasq+China DNS+ss-tunnel 解决DNS污染
 3. iptables+ipset+shadowsocks实现数据翻墙

大致的网络架构图如下：

![]({{ site.url }}/images/2018-04-25-build-shadowsocks-with-internal/topo.jpeg)

#### 基础环境

基本的网络结构是，用一台最小规格的centos服务器用于网关服务器，只是用于网络流量转发。购买EIP访问外部internet，局域网中的所有数据流量通过该服务器进行中转。

局域网中的机器通过NAT转发到网关服务器中的EIP网卡上，从而实现访问internet网络。外部的机器如果要访问局域网内的某个服务端口，采用DNAT映射的方式访问。

#### dns污染

大致原理，利用dnsmasq在网关服务器上搭建一个dns服务中转&缓存，局域网内所有机器均设置网络为第一dns服务器。

dns服务器的搭建模式，使用dnsmasq作为dns的缓存服务器，dnsmasq默认dns解析采用chinadns，chinadns在解析的使用默认使用两个地址解析，一个使用电信默认的dns服务器114.114.114.114. 另一个使用ss-tunnel转发到境外服务器上的shadowsocks服务器利用境外地址解析，对比后获得是否已经污染，从而获取没有污染的域名地址。

#### 数据穿墙

利用ipset构建一个国内地址池，将局域网内网地址及国内地址直接通过网关服务器转发至公网，其余地址数据通过shadowsocks转发境外的shadowsock服务器。

### 搭建基础网络环境



### 搭建dns

安装dnsmasq

```shell
yum update
yum install dnsmasq
```

配置dnsmasq，修改/etc/dnsmasq.conf

```
no-resolv
server=127.0.0.1#5353   #注意这里的端口是chinadns的监听端口
```

启动dnsmasq

```
service dnsmasq start
```

安装chinadns

下载并解压chinadns，下载[地址](https://github.com/shadowsocks/ChinaDNS/releases)

```
wget https://github.com/shadowsocks/ChinaDNS/releases/download/1.3.2/chinadns-1.3.2.tar.gz
tar -xvf chinadns-1.3.2.tar.gz
```

编译并安装

```
./configure && make
src/chinadns -m -c chnroute.txt
cp ./src/chinadns /usr/local/bin/
```

启动chinadns

```
chinadns -c ./chinadns-1.3.2/chnroute.txt \
 -m -p 5353 \ #这里的端口号要与前面配置dnsmasq一致 
 -s 114.114.114.114,127.0.0.1:5300 \
        1> /var/log/$NAME.log \
        2> /var/log/$NAME.err.log &
```

注：其中的5300是后续ss-tunnel的本地监听端口

安装ss-redir

安装依赖库

```
yum install epel-release -y
yum install gcc gettext libtool automake make pcre-devel asciidoc xmlto \
c-ares-devel libev-devel libsodium-devel mbedtls-devel -y
```

下载，[地址](ftp://ftp.gnu.org/gnu/autoconf/)

下载shadowsocks-libdev

```
git clone https://github.com/shadowsocks/shadowsocks-libev.git

cd shadowsocks-libdev
```

编译&安装

```
# Installation of Libsodium
export LIBSODIUM_VER=1.0.13
wget https://download.libsodium.org/libsodium/releases/libsodium-$LIBSODIUM_VER.tar.gz
tar xvf libsodium-$LIBSODIUM_VER.tar.gz
pushd libsodium-$LIBSODIUM_VER
./configure --prefix=/usr && make
sudo make install
popd
sudo ldconfig

# Installation of MbedTLS
export MBEDTLS_VER=2.6.0
wget https://tls.mbed.org/download/mbedtls-$MBEDTLS_VER-gpl.tgz
tar xvf mbedtls-$MBEDTLS_VER-gpl.tgz
pushd mbedtls-$MBEDTLS_VER
make SHARED=1 CFLAGS=-fPIC
sudo make DESTDIR=/usr install
popd
sudo ldconfig

# Start building
./autogen.sh && ./configure && make
sudo make install
```

启动ss-tunnel

```
nohup ss-tunnel -s <server-ip> -p <server-port> -b 0.0.0.0 -l 5300 \
-k <password> -m aes-256-cfb -L 8.8.8.8:53 -u &
```

注：server-ip: shadowsocks 服务端Ip地址
server-port: shadowsocks 服务端Ip地址
password:  shadowsocks 服务访问密码
5300是ss-tunnel本地监听端口

### 配置转发

#### 开启转发规则

修改/etc/sysctl.conf

```shell
net.ipv4.ip_forward=1
```

重启服务

```
sysctl -p
```

#### 配置IPSET

```
curl -sL http://f.ip.cn/rt/chnroutes.txt | egrep -v '^$|^#' > cidr_cn
sudo ipset -N cidr_cn hash:net
for i in `cat cidr_cn`; do echo ipset -A cidr_cn $i >> ipset.sh; done
chmod +x ipset.sh && sudo ./ipset.sh
rm -f ipset.cidr_cn.rules
sudo ipset -S > ipset.cidr_cn.rules
sudo cp ./ipset.cidr_cn.rules /etc/ipset.cidr_cn.rules
```

#### 配置IPTables

```
iptables -t nat -N shadowsocks
# 保留地址、私有地址、回环地址 不走代理
iptables -t nat -A shadowsocks -d 0/8 -j RETURN
iptables -t nat -A shadowsocks -d 127/8 -j RETURN
iptables -t nat -A shadowsocks -d 10/8 -j RETURN
iptables -t nat -A shadowsocks -d 169.254/16 -j RETURN
iptables -t nat -A shadowsocks -d 172.16/12 -j RETURN
iptables -t nat -A shadowsocks -d 192.168/16 -j RETURN
iptables -t nat -A shadowsocks -d 224/4 -j RETURN
iptables -t nat -A shadowsocks -d 240/4 -j RETURN
# 以下IP为局域网内不走代理的设备IP
iptables -t nat -A shadowsocks -s 10.0.0.111 -j RETURN
# 发往shadowsocks服务器的数据不走代理，否则陷入死循环
# 替换111.111.111.111为你的ss服务器ip/域名
iptables -t nat -A shadowsocks -d  111.111.111.111 -j RETURN   

# 大陆地址不走代理，因为这毫无意义，绕一大圈很费劲的
iptables -t nat -A shadowsocks -m set --match-set cidr_cn dst -j RETURN
# 其余的全部重定向至ss-redir监听端口1080(端口号随意,统一就行)
iptables -t nat -A shadowsocks ! -p icmp -j REDIRECT --to-ports 1080
# OUTPUT链添加一条规则，重定向至shadowsocks链
iptables -t nat -A OUTPUT ! -p icmp -j shadowsocks
iptables -t nat -A PREROUTING ! -p icmp -j shadowsocks
```

#### 配置默认网关

有两种方式都可以：

1. 修改默认的route关系

```
route del default
route add default gw 10.0.0.111 eth0
```

注： eth0是网卡名称，不同环境可能不一样

2. 增加一个NAT转发规则

```
iptables -t nat -A POSTROUTING -o eth0 -s 10.0.0.0/16 -j SNAT --to 10.0.0.111
```


### References:
[1] https://medium.com/@oliviaqrs/%E5%88%A9%E7%94%A8shadowsocks%E6%89%93%E9%80%A0%
E5%B1%80%E5%9F%9F%E7%BD%91%E7%BF%BB%E5%A2%99%E9%80%8F%E6%98%8E%E7%BD%91%E5%85%B3-fb82ccb2f729

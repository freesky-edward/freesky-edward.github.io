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
 -m -p 5354 \
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

### TODO

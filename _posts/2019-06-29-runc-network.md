---
layout: post
title: "runc source code——network"
date: 2019-06-29
type: tech
---

runc中本身太多的网络配置，如果非要介绍的话，那么只有三个点相关：
1. 在config.json中配置network namespace。
2. 使用PreStart hook来配置网络——实际上意义不大
3. 默认配置lo网卡

这三点中只有第三点是属于直接相关的代码逻辑，另两点只是可以进行配置。

首先看第三点，如果在config.json中配置network namespace，但是没有指定path那么系统会默认创建loopback回路网卡，默认配置项在[这里加入](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/specconv/spec_linux.go#L222)，网卡的创建是在init流程中，具体代码逻辑在https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L80
换句话说，如果使用runc的默认配置，那么attach到容器中执行ifconfig将只会看到lo一个网卡。

```
~ # ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

关于第二点，runc官网文档的介绍可以通过prestart，poststop来管理container的网络配置，可以将容器网络的创建脚本写在prestart中，网络的销毁配置在poststop中达到网络的配置，config.json配置方式
```
~ # ifconfig
"hooks": {
            "prestart": [
                {
                    "path": "/home/env/gopath/bin/netns",
                    "args": [
                        "netns",
                        "create",
                        "--ip",
                        "176.10.0.1/24"
                    ]
                }
            ],
            "poststop": [
                {
                    "path": "/home/env/gopath/bin/netns",
                    "args": [
                        "netns",
                        "rm"
                    ]
                }
            ]
        },
```

注上面的脚本用了网上的一个开源项目，项目路径：[https://github.com/genuinetools/netns](https://github.com/genuinetools/netns)
但是需要注意，prestart实际是在namespace挂载完成，rootfs的mount传播设置好，dev下所有的设备都挂载完成后，change root前执行。而且这个执行的调用init进程通过管道通知start进程执行调用，换句话说就是这个调用的namespace空间实际与runc相同的namespace中，而不是在init相同的namespace中。

这里是介绍第一点，因为他也是docker等网络设置的原理相似。假如host网络可以访问internet，那么runc通过默认的config.json配置出来的container实际与host间网络是隔离的，并且除了回路网卡外，没有网络配置，网络及host机器以及外界都是不通的，要想实现网络可访问，有两种选择：

1. 不挂载network namespace

修改config.json, 去掉namespace中的network配置
```
              "namespaces": [
                        {
                                "type": "pid"
                        },
                        {
                                "type": "ipc"
                        },
                        {
                                "type": "uts"
                        },
                        {
                                "type": "mount"
                        }
                ]
```

启动容器后，查看网卡信息

```
runc run container

ifconfig
eth0      Link encap:Ethernet  HWaddr FA:16:3E:44:2A:2D  
          inet addr:10.0.0.11  Bcast:10.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fe44:2a2d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:220113532 errors:0 dropped:0 overruns:0 frame:0
          TX packets:200823070 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:125843384251 (117.2 GiB)  TX bytes:35319134918 (32.8 GiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:14768870216 errors:0 dropped:0 overruns:0 frame:0
          TX packets:14768870216 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:3214996363063 (2.9 TiB)  TX bytes:3214996363063 (2.9 TiB)
```

我们可以看到，实际显示出来的网卡信息是host的网络信息并且网络是联通的，我们分别的容器里和启动容器的外部查看namespace情况

```
~ # ls -l /proc/self/ns/
total 0
lrwxrwxrwx    1 root     root             0 Jun 29 10:15 cgroup -> cgroup:[4026531835]
lrwxrwxrwx    1 root     root             0 Jun 29 10:15 ipc -> ipc:[4026532344]
lrwxrwxrwx    1 root     root             0 Jun 29 10:15 mnt -> mnt:[4026532342]
lrwxrwxrwx    1 root     root             0 Jun 29 10:15 net -> net:[4026531957]
lrwxrwxrwx    1 root     root             0 Jun 29 10:15 pid -> pid:[4026532345]
lrwxrwxrwx    1 root     root             0 Jun 29 10:15 user -> user:[4026531837]
lrwxrwxrwx    1 root     root             0 Jun 29 10:15 uts -> uts:[4026532343]
```

```
total 0
dr-x--x--x 2 root root 0 Jun 29 18:15 ./
dr-xr-xr-x 9 root root 0 Jun 29 18:15 ../
lrwxrwxrwx 1 root root 0 Jun 29 18:15 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Jun 29 18:15 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Jun 29 18:15 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Jun 29 18:15 net -> net:[4026531957]
lrwxrwxrwx 1 root root 0 Jun 29 18:15 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Jun 29 18:15 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Jun 29 18:15 uts -> uts:[4026531838]
```

我们可以发现net的ID是一样的，也就是说容器与host间没有网络隔离。容器的网络配置将和host上一样，这样只要主机上能访问的网络，容器内部都能访问。

2. 挂载一个已有网络已配置好的network namepace。配置好网络的访问。

这个相对来讲就要复杂一些，打通host与namespace间的网络，核心实现方式就是桥接。方法见下：

1. 创建一个网卡runc0，并配置其地址为176.10.0.1

 ```
 sudo brctl addbr runc0
 sudo ip link set runc0 up
 sudo ip addr add 176.10.0.1/16 dev runc0
 ```

2. 创建一个bridge，把其中的一段连接到runc0

```
sudo ip link add name veth-host type veth peer name veth-guest
 sudo ip link set veth-host up
 sudo brctl addif runc0 veth-host
```

3. 创建一个名为runc的network namespace, 把bridge的另一端加入到runc中，然后重命名另一端名字为eth1, 配置上ip地址

```
 sudo ip netns add runc
 sudo ip link set veth-guest netns runc
 sudo ip netns exec runc ip link set veth-guest name eth1
 sudo ip netns exec runc ip addr add 176.10.0.101/16 dev eth1
 sudo ip netns exec runc ip link set eth1 up
```

 到这里已经在host上有一个网卡，namespace下有另一个网卡，并且这两个网卡通过一个bridge进行了连通，他们之间已经可以进行通信了。

接下来要做得是将namespace中的数据能转发到host的runc0
网卡上，需要将namespace中的默认路由的网关设置成runc0的ip

```
 sudo ip netns exec runc ip route add default via 192.168.10.1
```
到这一步，host与namespace中的进程已经可以通信了。因为在添加ip设置的时候，系统实际默认向route表中加了一条地址转发规则。

```
ip netns exec runc route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         176.10.0.1      0.0.0.0         UG    0      0        0 eth1
176.10.0.0      *               255.255.0.0     U     0      0        0 eth1
```

```
route
176.10.0.0      *               255.255.0.0     U     0      0        0 runc0
```

最后需要做是让runc0中的外网访问数据能通过host的默认端口（比如eth0）发出去，并将该网段的数据能分发到runc0.

这里主要要做两件事情，1是配置SNAT，将176.10开头的源地址修改为eth0网卡的地址，2配置filter，使eth0与runc0间数据能否互相通信。

```
iptables -t nat -I POSTROUTING 1 --source 176.10.0.1/16 -o eth0 -j MASQUERADE
```

```
iptables -t filter -A FORWARD -o eth0 -i runc0 -j ACCEPT
iptables -t filter -A FORWARD -i eth0 -o runc0 -j ACCEPT
```

这样namespace中的进程就可以访问eth0能访问的空间了，接下配置好了这个链路以及network namespace后，需要将该network namespace配置给runc，该network namespace的路径为/run/netns/runc,config.json配置如下

```
              "namespaces": [
                        {
                                "type": "pid"
                        },
                        {
                                "type"： “network”,
                                "path": "/run/netns/runc"
                        },
                        {
                                "type": "ipc"
                        },
                        {
                                "type": "uts"
                        },
                        {
                                "type": "mount"
                        }
                ]
```

再通过runc启动容器后，在容器内部测试网络OK，整个网络原理和docker类似。

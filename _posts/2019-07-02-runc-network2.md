---
layout: post
title: "runc source code——network"
date: 2019-07-02
type: tech
---

前面[runc network](../runc-network/)介绍了如何打通容器与host机器的网络，关于容器网络的配置当然不止这一种，有兴趣可以研究下vxlan、ingress等

在完成上一篇的配置后，如果使用默认的config.json配置，就算通过bridge连通了网络，但是一些操作还是无法进行。比如 ping newto.me，系统会提示"bad address newto.me"

```
/ # ping newto.me
ping: bad address 'newto.me'
```
很明显这是因为无法解析域名导致的，查看/etc/resolv.conf文件，里面的配置都是空的，所以需要配置dns服务器

```
nameserver 10.0.0.1
nameserver 8.8.8.8
```
另一种方式是将host的文件挂载过来，在config.json的mounts下加入如下配置

```
                {
                        "destination": "/etc/resolv.conf",
                        "type": "none",
                        "source": "/etc/resolv.conf",
                        "options": [
                                 "rbind",
                                 "rw"
                        ]
                }
```

配置完成后，在执行ping命令

```
~ # ping newto.me
PING newto.me (35.201.166.48): 56 data bytes
ping: permission denied (are you root?)
```

还是不行，但是这次已经将host解析成了ip，只是权限不够，这是因为在config.json中配置的capabilities(后面单独介绍一下这个)中权限没有赋予，在每一项中添加上CAP_NET_RAW

```
             "capabilities": {
                        "bounding": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_NET_RAW"
                        ],
                        "effective": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_NET_RAW"
                        ],
                        "inheritable": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_NET_RAW"
                        ],
                        "permitted": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_NET_RAW"
                        ],
                        "ambient": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_NET_RAW"
                        ]
                },
```

再执行后一切便都OK了。

```
~ # ping newto.me
PING newto.me (35.201.166.48): 56 data bytes
64 bytes from 35.201.166.48: seq=0 ttl=44 time=24.037 ms
64 bytes from 35.201.166.48: seq=1 ttl=44 time=21.898 ms
64 bytes from 35.201.166.48: seq=2 ttl=44 time=22.798 ms
```

至此，container中的网络就可以连通外界了。但是请注意runc的example配置默认需要root权限执行的，其中的namespace是没有加载user namespace的，如果是rootless的配置或者是加载上user namespace那么情况又将完全不一样，后面我们专门就rootless container做一篇介绍。



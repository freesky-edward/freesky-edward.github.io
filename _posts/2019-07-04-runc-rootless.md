---
layout: post
title: "runc source code——rootless container"
date: 2019-07-03
type: tech
---

### 什么是rootless container
不同于privileged container，rootless container并不是说容器内部的进程权限是非root权限，而是指非root用户启动的容器，也就是说之前说的无论是privileged container还是unprivileged container都是root权限run出来的容器，只是容器内部进程的权限是否有特权（capabilities赋予）。

用非root用户运行的容器相比较于root运行的容器在一些领域是收到限制的，因为linux很多的操作是需要特权才能进行，比如cgroup，apparmor, overlay网络等等，对于runc来讲，由于cgroup需要特权所以跟其相关的特性都将不能工作，包括如下：
runc checkpoint
runc resume
runc pause
runc update
runc ps

详细runc支持rootless的patch在参见[https://github.com/opencontainers/runc/pull/774](https://github.com/opencontainers/runc/pull/774)

### runc rootless container 有哪些变化

除了上述提到的不能工作的部分，runc在其它部分也做了一些变化，首先spec生成的example不同，使用特殊参数--rootless生成rootless的config.json

```
runc spec --rootless
```

config.json相对于root container的不同点主要有以下不同，具体处理代码在[这里](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/specconv/example.go#L161)

a. 不新建network namespace，加上user namespace
b. 增加当前用户/用户组与容器内root用户/用户组的映射
c. 将sysfs更改为从host上进行mind mount
d. 去掉gid=5的mount 标志, 即pts挂载时不指定gid组为tty组（id =5）,这样在容器内部如果一个用户创建的pty，该pty的group将属于创建者相同的组。

### runc rootless container 变化的原因

在前面的[runc network](../runc-network/)中介绍了runc的网络实际在root模式下，默认是网络隔离的，容器内部只会新建lo 网络接口，容器与host间的网络通信是可以通过bridge的方式来实现，而这种方案很多都需要**CAP_SYS_ADMIN**权限，所以runc在rootless模式下默认使用了host网络，也就是不新建network namespace。

而由于是非特权用户运行容器，所以容器只能映射自己为容器内部的root，不能添加别的映射，所以runc在rootless模式下选择了新建namesapce，并将当前用户/用户组映射到容器内部的root，也就是uid/gid=0.

由于sysfs挂载需要**CAP_SYS_ADMIN**权限，在新建了user namesapce，而没有新建network namespace的情况下，虽然在容器内部是root，但是由于没有隔离网路，所以sysfs挂载会报权限不足，挂载不成功，需要将sys更改为mind mount。原因讨论参见[https://github.com/opencontainers/runc/issues/799](https://github.com/opencontainers/runc/issues/799)

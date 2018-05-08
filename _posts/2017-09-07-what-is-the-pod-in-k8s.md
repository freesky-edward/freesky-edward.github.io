---
layout: post
title: "从另一个角度理解pod"
date: 2017-09-07
lang: zh
type: tech
---

从另一个视角理解POD
===================

### 什么是POD

关于Pod的定义，在Kubernetes的官方文档中的[Pod章节](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)有比较详细的介绍，只是这里的介绍似乎有一些偏向概念和基本特性的介绍，如果只是概念的普及和理解，建议去阅读，不再对其做任何附加的介绍，这里将从它的设计理念、应用的角度去介绍它。

### 什么是Container

在很好认识pod之前，需要先来理解一下Container。

也许大家都知道，在linux中实则是没有Container的，Container实质是通过将linux的namespace和cgroup抽象出来的一个运行进程的环境的称呼，这个环境通过namespace提供了一个与外部隔离的效果，在linux中namespace的种类包括

- hostname    
- process id
- file system
- network
- ipc
- uts

这些namespace将应用到进程上进行资源隔离，可以通过/proc来查看某个进行运行的namespace，如：

```shell
ll -l /proc/self/ns
lrwxrwxrwx 1 root root 0 Sep  7 02:29 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Sep  7 02:29 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Sep  7 02:29 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Sep  7 02:29 net -> net:[4026531957]
lrwxrwxrwx 1 root root 0 Sep  7 02:29 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Sep  7 02:29 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Sep  7 02:29 uts -> uts:[4026531838]
```

如果我们说通过namespace隔离出来的是一个和其它进程完全隔离的环境，实则并不准确，这里的进程是可以使用外部全部的资源，如CPU，RAM等等，所以需要限制其对资源的无限占用，这就是cgroup的能力，cgroup的限制的资源类型同样有多种，具体cgroup的限制类型可以通过其挂载的文件系统来查看：

```shell
mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
```

这里需要说明的是，上述看到的这些namespace和cgroup都是独立工作的，并不是每一个container需要使用上述所有的namespace和cgroup，如：可以选择性的将一个进程加入到其中的某几种类型的namespace中，这样的话，没有加入的类型就意味着使用和其它进程共享资源；也可以将多个进程加入到同一个namespace中，这样加入到同一个namespace的所有进程将相互可见，共享资源。理解了这个机制会对我们认识pod有很大的帮助。

### 在同一个namespace下运行两个container

或许你会觉得，其实container的工作方式很简单，通过docker创建一个container，docker将为这个container创建一个独立的namespace和cgroup，把namespace和cgroup应用到container中的进程即可，最后把host上的卷和端口映射到这个namesapce下，既可实现隔离，也能与外界通信。

当然上述理解是OK的，但是在docker里并不完全是所有场景都是这样的，有时候，实际是可以通过命令参数把docker中的两个container运行在一个namesapce下，例如：

我们先启动一个nginx的容器：

```shell
docker run -d --name nginx -v ~/nginx.conf:/etc/nginx/nginx.conf -p  8080:80 nginx
```

然后可以再启动一个ghost容器，使其和nginx容器运行在同一个net namespace和pid namespace中：

```shell
docker run -d --name ghost --net=container:nginx --pid=container:nginx ghost
```

这样，nginx就能通过localhost直接与ghost进行通信，因为它们在同一个net namespace。这也是很多应用实际需要的一种场景，如： 监控，日志等都是需要旁挂的方式运行在业务进程边上。

实际上把多个container运行在同一个namespace或者cgroup下，正是抽象pod的目的，把多个容器加入到pod里就可以达到这个共享资源的效果，只是pod不只是简单的使用docker的参数来实现，它的实现比这个复杂。

一旦多个进程能够运行在相同的namesapce和cgroup里，那么他们之间就如同运行在同一个环境一样，可以共享存储，可以通过本地通信，可以使用TERM。。。

### 为什么需要pod

如上类似的场景，如果需要把一个监控进程和一个业务进程运行在同一个namespace里，在docker里的解决方案是将两个进程运行在一个container里，这样其实并不符合docker的最佳实践，同时由于只有一个container，没有办法对两个进程进行分开管理，那么其它的外部工具就无法通过docker api获取到各自进程的信息。
或许你会问为什么不使用两个container呢？对，没错！可以这样做，但是这样做有缺少一个统一的API来同时管理这两个进程，例如：同时重启，其中一个故障时，同时迁移等，这样实际带来新的问题。

有pod的情况下，可以使各自进程运行在独立的container里，两个container属于同一pod，既可以通过docker api获取各自信息，也可以统一进行服务管理，如：重启，故障迁移等等。极大的提升了应用服务管理的灵活性。

这就是Pod的魅力，但还不只是很low的一点，正如前面所说，通过组合可以将实际业务container和“辅助”container分离是其非常重要的一种应用，于是Pod另一类高级应用模型抽象就是能将很多公有container做成一个API服务供其它的container消费，社区称这类应用为——sidecar container. 它在很多领域应用优势明显，如容器存储，安全，监控等等，后续再做深入介绍。

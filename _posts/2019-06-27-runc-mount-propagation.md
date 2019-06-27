---
layout: post
title: "runc source code——mounts propagation"
date: 2019-06-27
type: tech
---
这篇文章我们主要来研究runc中的文件系统mounts操作，如前面的[rootfs流程介绍](../runc-rootfs/)，文件系统的准备主要是在
1.  [prepareRootfs](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L88)
2.  [finalizeRootfs](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L105)

我们使用runc debug，在关键步骤加上日志先看看都执行了那些mount操作，然后根据假设做一些测试。

在关键代码中加上日志，主要有两个地方，一个是prepareRoot，另一个是

```
func  prepareRoot(config  *configs.Config)  error  {
  flag  :=  unix.MS_SLAVE  |  unix.MS_REC
  if  config.RootPropagation  !=  0  {
      flag  =  config.RootPropagation
  }

  logrus.Debug(fmt.Sprintf("mount  root  dest=/  flag=%s",  flag))
  if  err  :=  unix.Mount("",  "/",  "",  uintptr(flag),  "");  err  !=  nil  {
      return  err
  }

  //  Make  parent  mount  private  to  make  sure  following  bind  mount  does
  //  not  propagate  in  other  namespaces.  Also  it  will  help  with  kernel
  //  check  pass  in  pivot_root.  (IS_SHARED(new_mnt->mnt_parent))
  if  err  :=  rootfsParentMountPrivate(config.Rootfs);  err  !=  nil  {
      return  err
  }

  logrus.Debug(fmt.Sprintf("mount  rootfs  dest=%s  source=%s  flag=%s  device=bind",  
     config.Rootfs,  config.Rootfs,  (unix.MS_BIND  |  unix.MS_REC)))
  return  unix.Mount(config.Rootfs,  config.Rootfs,  "bind",  unix.MS_BIND|unix.MS_REC,  "")
}
```

```
//  Do  the  mount  operation  followed  by  additional  mounts  required  to  take  care

//  of  propagation  flags.

func  mountPropagate(m  *configs.Mount,  rootfs  string,  mountLabel  string)  error  {
  
  ......
 
  logrus.Debug(fmt.Sprintf("start  mount  dest=%s  source=%s  flags=%s  data=%s  device=%s  
    Propagationflags=%s",  dest,  m.Source,  uintptr(flags),  data,  m.Device,  m.PropagationFlags))

  if  err  :=  unix.Mount(m.Source,  dest,  m.Device,  uintptr(flags),  data);  err  !=  nil  {
      return  err
  }

  
  for  _,  pflag  :=  range  m.PropagationFlags  {
      logrus.Debug(fmt.Sprintf("PropagationFlags  flag=%s  dest=%s",  pflag,  dest))
      if  err  :=  unix.Mount("",  dest,  "",  uintptr(pflag),  "");  err  !=  nil  {
          return  err
      }
  }
  return  nil
}
```

重新编译runc，并安装
```
cd $GOPATH/src/github.com/opencontainer/runc
make
make install
```

运行容器时配置log
```
runc --debug --log /var/log/runc.log run container1
```


查看执行日志：

```
cat /var/log/runc.log

"mount root  dest=/ flag=%!!(MISSING)s(int=540672)"
"mount rootfs  dest=/home/slob/runc/mycontainer/rootfs source=/home/slob/runc/mycontainer/rootfs flag=s(int=20480) device=bind"
"start mount dest=/proc source=proc flags=s(uintptr=0) data= device=proc Propagationflags=[]"
"start mount dest=/dev source=tmpfs flags=s(uintptr=16777218) data=mode=755,size=65536k device=tmpfs Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/dev/pts source=devpts flags=(uintptr=10) data=newinstance,ptmxmode=0666,mode=0620,gid=5 device=devpts Propagationflags=[]"
"start mount dest=/dev/shm source=shm flags=%!!(MISSING)s(uintptr=14) data=mode=1777,size=65536k device=tmpfs Propagationflags=[]"
"start mount dest=/dev/mqueue source=mqueue flags=%!!(MISSING)s(uintptr=14) data= device=mqueue Propagationflags=[]"
"start mount dest=/sys source=sysfs flags=%!!(MISSING)s(uintptr=15) data= device=sysfs Propagationflags=[]"
"start mount dest=/sys/fs/cgroup source=tmpfs flags=%!!(MISSING)s(uintptr=14) data=mode=755 device=tmpfs Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/systemd source=/sys/fs/cgroup/systemd/user.slice/user-0.slice/session-8126.scope/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/blkio source=/sys/fs/cgroup/blkio/user.slice/user-0.slice/session-8126.scope/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/pids source=/sys/fs/cgroup/pids/user.slice/user-0.slice/session-8126.scope/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/perf_event source=/sys/fs/cgroup/perf_event/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/memory source=/sys/fs/cgroup/memory/user.slice/user-0.slice/session-8126.scope/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/net_cls,net_prio source=/sys/fs/cgroup/net_cls,net_prio/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/cpu,cpuacct source=/sys/fs/cgroup/cpu,cpuacct/user.slice/user-0.slice/session-8126.scope/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/cpuset source=/sys/fs/cgroup/cpuset/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
"start mount dest=/home/slob/runc/mycontainer/rootfs/sys/fs/cgroup/freezer source=/sys/fs/cgroup/freezer/container2 flags=%!!(MISSING)s(uintptr=2117647) data= device=bind Propagationflags=[]"
```

从日志信息不难看出，系统首先将根挂载点及其所有子挂载点的创博关系修改为了SLAVE，其中flag十进制值540682正好是MS_SLAVE | MS_REC得值，参见[https://godoc.org/golang.org/x/sys/unix](https://godoc.org/golang.org/x/sys/unix)
设置以后，后续所有根下的挂载将不再传播给根系统相同的peer group，即不再影响原namespace。

然后将rootfs重新进行bind mount，这里的bind mount在原namespace中将不被传播，但是rootfs下的所有子挂载点，将会是原系统namespace的slave挂载点，原namespace中对rootfs挂载，将传播到容器空间中。

如上启动container后，我们host上执行如下操作：

```
mount -B /home/slob/test <rootfs>/tmp/test
```

在容器中实际能看到host上source的内容，在容器中能看到这个挂载点也能看到这个传播来的挂载。

反过来，在容器namespace中进行mount，在host namespace就不会被传播，我们做如下实验：
在config.json的mount字段中加入如下定义：

```
{
    "destination": "/tmp/terr",
    "type": "none",
    "source": "/home/slob/terraform",
    "options": [
           "rbind",
           "rw"
    ]
}
```

在重新运行container，挂载后在容器内部查看/tmp/terr目录已经是host的/home/slob/terraform下的内容。
```
ls -l /tmp/terr
ls -l /tmp/terr
total 108
drwxr-xr-x    3 root     root          4096 May  8 01:51 aksk_test
```

而在host下，查看rootfs/tmp/terr目录则并没有显示/home/slob/terraform下的内容：
```
ll rootfs/tmp/terr/
total 16
drwxr-xr-x 4 root root 4096 Jun 27 09:39 ./
drwxr-xr-x 6 root root 4096 Jun 27 16:07 ../
```

通过在container中查看挂载点信息：
```
cat /proc/self/mountinfo
393 297 253:1 /home/slob/runc/mycontainer/rootfs / rw,relatime master:1 - ext4 
    /dev/vda1 rw,errors=remount-ro,data=ordered
413 393 253:1 /home/slob/terraform /tmp/terr rw,relatime master:1 - ext4 
    /dev/vda1 rw,errors=remount-ro,data=ordered
```
我们可以看到/tmp/terr挂载是一个slave传播，该挂载的父挂载点是rootfs点，也是slave传播，他们具有相同peer group

现在如果在host上将其他目录bind mount 到rootfs/tmp/terr下，我们看下会发生什么事。
```
mount -B /home/slob/spark rootfs/tmp/terr
```
首先在host上肯定rootfs/tmp/terr目录下的内容会是/home/slob/spark下的内容，这个毋庸置疑。按照上面的分析因为根目录是slave传播，所以这个挂载会传播到container的/tmp/terr，那么/tmp/terr下内容会发生改变。

进入container，查看/tmp/terr

```
ls -l /tmp/terr
total 108
drwxr-xr-x    3 root     root          4096 May  8 01:51 aksk_test
```

然而内容并没有变化！！！到底发生了什么呢，我们在看下container中mount点情况
```
cat /proc/self/mountinfo
393 297 253:1 /home/slob/runc/mycontainer/rootfs / rw,relatime master:1 - ext4 
    /dev/vda1 rw,errors=remount-ro,data=ordered
413 296 253:1 /home/slob/terraform /tmp/terr rw,relatime master:1 - ext4 
    /dev/vda1 rw,errors=remount-ro,data=ordered
296 393 253:1 /home/slob/spark /tmp/terr rw,relatime master:1 - ext4 
    /dev/vda1 rw,errors=remount-ro,data=ordered
```

注意413挂载点的父挂载点发生了变化，不再是393，而是传播过来挂载点296. 也就是说/tmp/terr挂载点先进行了传播挂载mount  /home/slob/spark /tmp/terr, 再进行了/home/slob/terraform /tmp/terr，所以看到的还是原来的内容。


通过上面的分析，我们基本可以知道，container中的根挂载点事slave挂载，在容器外的rootfs下的子挂载会传播到容器内部，但是容器内部的挂载不会影响到容器外部，这个特性理论上可以在容器运行后，通过host挂载进行磁盘卷的挂载与卸载，不必一定要容器启动时进行。同时，容器内的同一挂载点，内部挂载的优先级会高于容器外部（内部的挂载的父挂载是外部挂载点）。


---
layout: post
title: "runc source code——rootfs init"
date: 2019-06-21
type: tech
---

### Rootfs prepare flow

前面接收了runc初始化分为两部分，一部分是nsenter通过cgo挂载namespace，第二部分是通过runc init准备其它环境。这里简要梳理下runc中容器文件系统相关的处理。文件系统的准备工作主要是在runc init流程中准备好，真正的逻辑入口为[https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/factory_linux.go#L283](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/factory_linux.go#L283)
创建一个newContainerInit， 然后执行[init方法](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L47), 其中跟文件系统相关的主要有两个地方
1. [prepareRootfs]([https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L88](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L88))
2. [finalizeRootfs]([https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L105](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L105))

其中第一个主要是挂载上config.json中配置的所有mounts，安装上devices, 准备好系统文件，切换root；第二个将整个root文件系统设置成只读，所以重点需要将第一个流程展开了解清楚，简化后大致流程如下：

```
//由于前面已经进入了新的namespace，主要是将根目录mount传播设置为slave和rootfs设置为private，
//避免影响其他namespace文件系统试图。然后将rootfs进行bind mount
if  err  :=  prepareRoot(config);  err  != nil {
    return  newSystemErrorWithCause(err, "preparing  rootfs")
}

//循环将config.json中的mount点挂载到rootfs。
//主要处理proc,sysfs；tmpfs；cgroup；mqueue；bind
for  _, m  :=  range config.Mounts {
    if  err  :=  mountToRootfs(m,  config.Rootfs,  config.MountLabel,  hasCgroupns);  
        err  != nil {
        return  newSystemErrorWithCausef(err, "mounting %q to  rootfs %q at %q",  
             m.Source,  config.Rootfs,  m.Destination)

    }
}

//加载设备，如果设备没有mknode则进行mknode，否则进行bind mount。
//重新加载pts/ptmx
//加载标准输入输出/proc/self/fd;/proc/self/fd/0;/proc/self/fd/1;/proc/self/fd/2 
if setupDev  {
    if  err  :=  createDevices(config);  err  != nil {
        return  newSystemErrorWithCause(err, "creating  device  nodes")
    }

    if  err  :=  setupPtmx(config);  err  != nil {
        return  newSystemErrorWithCause(err, "setting  up  ptmx")
    }

    if  err  :=  setupDevSymlinks(config.Rootfs);  err  != nil {
         return  newSystemErrorWithCause(err, "setting  up  /dev  symlinks")
    }
}

//change root
if  config.NoPivotRoot  {
    err  =  msMoveRoot(config.Rootfs)
}  else  if  config.Namespaces.Contains(configs.NEWNS)  {
    err  =  pivotRoot(config.Rootfs)
}  else  {
    err  =  chroot(config.Rootfs)
}
```

逻辑相对比较简单，只是因为进入了新的namespace，为了不传播给父namespace，mount前后需要做一些传播配置。

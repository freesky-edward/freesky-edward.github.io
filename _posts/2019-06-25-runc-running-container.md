---
layout: post
title: "runc source code——two ways of running container"
date: 2019-06-25
type: tech
---

我们知道runc启动容器有两种方式，一种方式是先创建一个容器再执行该容器：

```
runc create mycontainer1
runc start mycontainer1
```
另一种方式则是直接run容器：

```
runc run mycontainer1
```

对于create和run都是调用的[startContainer](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/utils_linux.go#L405)方法，只是传入的action参数不一样，分别是CT_ACT_CREATE和CT_ACT_RUN，而action参数的分支出现在runner的[run](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/utils_linux.go#L317)方法中, 分别调用的libcontainer的[start](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/container_linux.go#L233)和[run](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/container_linux.go#L250)方法，而run方法内部也是先调用start方法，然后执行exec，代码如下：

```
func  (c *linuxContainer) Run(process *Process) error {
    if  err  := c.Start(process);  err  != nil {
         return err
    }
    
    if process.Init {
        return c.exec()
    }
    return  nil
}
```

而实际上start命令的逻辑，也是首先找到container，然后如果状态是created，则执行libcontainer的[exec](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/start.go#L34)方法。这个exec方法实际是开启一个goroutinue去读取fifo管道，如下：

```
func  awaitFifoOpen(path  string)  <-chan  openResult {
    fifoOpened  :=  make(chan openResult)

    go  func()  {
        f, err  := os.OpenFile(path,  os.O_RDONLY, 0)
        if err  != nil {
            fifoOpened <- openResult{err: newSystemErrorWithCause(err, "open  exec  fifo  for  reading")}
            return
        } 
        fifoOpened <- openResult{file: f}
    }()

    return fifoOpened
}
```

在linux中fifo管道是一个双向管道，如果一端没有读取内容，那么写入端将会被阻塞，不难可以猜测在创建完成container后，实际进程已经生成，只是被fifo管道阻塞了，没有执行具体的用户指令，而这个阻塞的逻辑在之前介绍的init流程里执行config中cmd之前，代码在[这里](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/standard_init_linux.go#L188) 这个fifo文件句柄号是在调用runc init
时，通过环境变量_LIBCONTAINER_FIFOFD传递给子进程，而这个fifo文件默认在/run/runc/<container-name>/exec.fifo。

```
ll /run/runc/container1/
total 12
drwx--x--x 2 root root   80 Jun 25 15:06 ./
drwx------ 3 root root   60 Jun 25 15:06 ../
prw--w--w- 1 root root    0 Jun 25 15:06 exec.fifo|
-rw-r--r-- 1 root root 8365 Jun 25 15:06 state.json
```
注：start后这个fifo便被删除了。

由此，我们便明白了create后，start前其实容器进程已经存在，只是通过fifo管道阻塞，没有执行用户进程。


create&start与run还有一个区别在于终端，在代码中如果是使用create,默认是需要detach终端的，默认代码在这里[https://github.com/opencontainers/runc/blob/v1.0.0-rc8/utils_linux.go#L303](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/utils_linux.go#L303)，这样的话，如果config.json中的process>>terminal如果配置为true的话，默认是需要配置console-socket的，这部分逻辑处理见这里
[https://github.com/opencontainers/runc/blob/v1.0.0-rc8/utils_linux.go#L160](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/utils_linux.go#L160)
如果不配置的话会报错。

配置tty socket的方法：

```
$GOPATH/src/github.com/opencontainers/runc/contrib/cmd/recvtty/recvtty ./tty.sock
runc create --console-socket ./tty.sock container1
runc start container1
```

注意，第一个命令执行后，当前的bash将作为终端，故后面两个命令需要再开一个终端执行；
这也就解释了为什么需要这么做——因为run的时候，默认当前bash会直接作为终端启动，而create后，需要额外附加终端，该终端继续runc管理。

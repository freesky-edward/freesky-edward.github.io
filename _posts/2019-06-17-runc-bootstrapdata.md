---
layout: post
title: "runc source code——bootstrap分析"
date: 2019-06-17
type: tech
---

关于runc创建容器的大致进程，网络上有很多介绍得文章有提及，主要分为container start和container init两个重要的部分，本文主要是就container init部分对应源码做介绍，以便对细节有深入的理解。

### cgo初始化进程运行环境

首先借用网上的图片介绍了container初始化时的进程关系：
![](https://img-blog.csdn.net/20170814150514391?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhvbmdsaW56aGFuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
本篇文章主要是介绍右边两个蓝色框之间的交互已经执行流程。
在[container_linux.go](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/container_linux.go#L505)中初始化initProcess时，构建一个[netlink](http://man7.org/linux/man-pages/man7/netlink.7.html)格式的数据结构体，同时在这之前会创建一个[socketpair](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/container_linux.go#L441)用于start进程与init进程间通信，并把该socket对赋予initProcess. 该socket的父端是start进程持有，child端是init进程持有。

在真正启动parentProcess的时候，这个bootstrapdata会通过socketpair的父socket端发送给init进程[https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/process_linux.go#L298](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/process_linux.go#L298)

那么这个结构体里包含哪些内容呢，这些内容init进程又是如何使用最终构建出容器进程的基础运行环境的呢？

首先我们需要知道这个bootstrapdata在哪里被使用。如其他博客介绍，该Init进程实际就是调用runc init。如果只是看其中init涉及的代码，你会发现根本找不到处理的逻辑。

这是因为go runtime是多线程的，多线程进程不能通过setns来设置user namespace，所以必须要引用libcontainer的[nsenter](https://github.com/opencontainers/runc/tree/v1.0.0-rc8/libcontainer/nsenter)包使用cgo来设置namepace，引用地址：[https://github.com/opencontainers/runc/blob/v1.0.0-rc8/init.go#L8](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/init.go#L8)
cgo部分的主要逻辑在[nsexec](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/nsenter/nsenter.go#L9)方法里，该方法的源代码位于[https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/nsenter/nsexec.c#L540](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/nsenter/nsexec.c#L540)
也正是这里对bootstrapdata进行处理，处理的流程大致如下：

1. 从环境变量中获取到上文中提供socketpair的child的文件句柄号。

```
/*
*  If  we  don't  have  an  init  pipe,  just  return  to  the  go  routine.
*  We'll  only  get  an  init  pipe  for  start  or  exec.
*/
pipenum  = initpipe();
if (pipenum  ==  -1)
return;
```

2. 将bootstrapdata解析成nlconfig_t结构体

```
/* Parse  all  of  the  netlink  configuration. */
nl_parse(pipenum,  &config);
```

3. 刷新out of memory 的人工评分值

```
/* Set  oom_score_adj.  This  has  to  be  done  before  !dumpable  because
*  /proc/self/oom_score_adj  is  not  writeable  unless  you're  an  privileged
*  user  (if  !dumpable  is  set).  All  children  inherit  their  parent's
*  oom_score_adj  value  on  fork(2)  so  this  will  always  be  propagated
*  properly.
*/
update_oom_score_adj(config.oom_score_adj,  config.oom_score_adj_len);
```

4. 初始化两个socketpair用于与该进程的子进程以及孙进程进行通信

```
/* Pipe  so  we  can  tell  the  child  when  we've  finished  setting  up. */
if (socketpair(AF_LOCAL,  SOCK_STREAM, 0,  sync_child_pipe)  < 0)
bail("failed  to  setup  sync  pipe  between  parent  and  child");

/*
*  We  need  a  new  socketpair  to  sync  with  grandchild  so  we  don't  have
* race condition with child.
*/
if (socketpair(AF_LOCAL, SOCK_STREAM, 0, sync_grandchild_pipe) < 0)
bail("failed  to  setup  sync  pipe  between  parent  and  grandchild");
```

5. 使用setjmp, longjmp机制进行各项初始化。

其中第四、五步是整个过程的关键，这里一共进行了两次clone，共有parent进程，child进程（从parent clone而来）,init进程（实际是从child clone而来），每个进程的执行逻辑分别在对应switch case的三个case分支，关系如下：

```
switch  (setjmp(env))  {
case  JUMP_PARENT:  { //parent进程
   ...
   child  =  clone_parent(&env,  JUMP_CHILD);//复制生成child进程，执行逻辑跳转到JUMP_CHILD分支
   ...
}
case  JUMP_CHILD:  {//child进程
   ...
   child  =  clone_parent(&env,  JUMP_INIT);//复制生成init进程，执行逻辑跳转到JUMP_INIT分支
   ...
}
case  JUMP_INIT:  {//init进程
   ...
}
```

三个进程间，parent与child使用sync_child_pipe socketpair进行通信，parent与init使用sync_grandchild_pipe socketpair进行通信。child与init间没有通信。

parent进行：
1. clone生成child进程

```
/* Start  the  process  of  getting  a  container. */
child  = clone_parent(&env,  JUMP_CHILD);
if (child  < 0)
bail("unable  to  fork:  child_func");
```

2. 进入while循环处于与child间通信，直到child进行返回ready信号, 处理逻辑如下：

```
while (!ready)  {
    switch (s)  {
    case SYNC_ERR:{
        //接收到来自child的error信息，抛出错误
    }
    case SYNC_USERMAP_PLS:{
        //接收到child设置user map请求，设置uidmap gidmap.
        //并向child发送完成ACK
    }
    case SYNC_RECVPID_PLS:{
        //接收到child返回的PID信息，解析出PID
        //并向child发送接收成功ACK
    }
    case SYNC_CHILD_READY:{
        //接收到child返回的ready信号
        //向child发送接收成功ACK
        //向create进程child PID以及init PID
        //kill child进程
        //退出while循环
    }
}
```

3. 进入循环，处理与init进程交互，直到接收到ready信号

```
while (!ready)  {
    //向init进程发送SYNC_GRANDCHILD信号
    switch (s)  {
    case SYNC_ERR:{
        //接收到来自child的error信息，抛出错误
    }
    case SYNC_CHILD_READY:{
        //退出while循环
    }
}
```

Child进程：
4. 设置namespaces

```
if (config.namespaces)
    join_namespaces(config.namespaces);
```

5. unshare user

```
if (unshare(CLONE_NEWUSER)  < 0)
    bail("failed  to  unshare  user  namespace");
```

6. 向parent发送SYNC_USERMAP_PLS

```
s = SYNC_USERMAP_PLS;
if (write(syncfd,  &s, sizeof(s))  != sizeof(s))
    bail("failed to sync with parent: write(SYNC_USERMAP_PLS)  to  sync  with  parent:  write(SYNC_USERMAP_PLS)");
```

7. set resource uid & ushare cgroup
8. clone 生成init进程

```
child  = clone_parent(&env,  JUMP_INIT);
```

9. 向parent进程发送PID

```
s  =  SYNC_RECVPID_PLS;
if (write(syncfd,  &s, sizeof(s))  != sizeof(s))  {
    kill(child,  SIGKILL);
    bail("failed  to  sync  with  parent:  write(SYNC_RECVPID_PLS)");
}

if (write(syncfd,  &child, sizeof(child))  != sizeof(child))  {
    kill(child,  SIGKILL);
    bail("failed to sync with parent: write(childpid)");
}
``` 

10. 向parent 发送ready信号后退出

```
s  =  SYNC_CHILD_READY;
if (write(syncfd,  &s, sizeof(s))  != sizeof(s))  {
    kill(child,  SIGKILL);
    bail("failed  to  sync  with  parent:  write(SYNC_CHILD_READY)");
}

exit(0);
```

Init进程：
1. 等待parent进程的SYNC_GRANDCHILD信号

```
if (read(syncfd,  &s, sizeof(s))  != sizeof(s))
    bail("failed to sync with parent: read(SYNC_GRANDCHILD) to sync with 
    parent: read(SYNC_GRANDCHILD) to sync with parent: read(SYNC_GRANDCHILD)
    to  sync  with  parent:  read(SYNC_GRANDCHILD)");

if (s  !=  SYNC_GRANDCHILD)
    bail("failed  to  sync  with  parent:  SYNC_GRANDCHILD:  got %u",  s);
```

2. 设置sid uid gid
3. 向parent发送ready信号
4. 释放config空间

从上面可以看出，经过一系列的处理后，最后init进程将会一直运行下去去执行go runtime相关的runc init逻辑，而在这个过程中，namespace相关的设置已经完成，进程已经完成了与宿主的资源隔离。





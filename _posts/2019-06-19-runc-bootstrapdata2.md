---
layout: post
title: "runc source code——bootstrap分析2"
date: 2019-06-19
type: tech
---

### Why clone twice

如[前一篇博文](../runc-bootstrapdata)介绍，在nsenter包中，实际进行了两次进程clone，分别有parent, child, init三个进程进行相应的交互处理后，最后留下init运行go runtime。

这三个进程的关系是 parent --> child --> init，注意箭头只是clone关系，因为在clone时，clone flags参数为CLONE_PARENT | SIGCHLD
```
static  int  clone_parent(jmp_buf *env, int jmpval)  __attribute__  ((noinline));
static  int  clone_parent(jmp_buf *env, int jmpval)
{

    struct  clone_t ca  =  {
        .env =  env,
        .jmpval =  jmpval,
    };
    return  clone(child_func,  ca.stack_ptr,  CLONE_PARENT  |  SIGCHLD,  &ca);
}
```
所以这三个进程实际是具有相同ppid。

这里之所以要clone两次，一次是因为CLONE_NEWPID后，child进程才会进入该namespace，描述见下，这就是第二次clone生成init进程的原因, 当在child进程中设置namespace后，child进程的pid namespace并不起作用，需要在clone出init进程，使其与child进程同namespace，但是pid namespace生效。
```
**CLONE_NEWPID** (since Linux 2.6.24)
              If **CLONE_NEWPID** is set, then create the process in a new PID
              namespace.  If this flag is not set, then (as with [fork(2)](http://man7.org/linux/man-pages/man2/fork.2.html))
              the process is created in the same PID namespace as the
              calling process.  This flag is intended for the implementation
              of containers.
```
另一个clone的原因是，由于内核原因，一方面不能将USER namespace与其它namespace一起挂载，那样会导致namespace的所属不清楚的问题，另一方面对于rootless container应为没有CAP_SYS_ADMIN权限而无法挂载其它namespace（见下说明），所以首先需要先挂载user namespace，而有些操作系统挂载了user namespace后如果不做uid/gid map的话，后面操作也会报错，所以需要在挂载user namespace后先完成uid/gid map。 而一旦先挂载了user namespace，那么配置必须要由原来的namespace来做（见下说明2），于是这里必须得有一次clone，也就是parent clone出child进程。

```
       Starting in Linux 3.8, unprivileged processes can create user
       namespaces, and the other types of namespaces can be created with
       just the **CAP_SYS_ADMIN** capability in the caller's user namespace.

       If **CLONE_NEWUSER** is specified along with other **CLONE_NEW*** flags in a
       single [clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html) or [unshare(2)](http://man7.org/linux/man-pages/man2/unshare.2.html) call, the user namespace is guaranteed
       to be created first, giving the child ([clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html)) or caller
       ([unshare(2)](http://man7.org/linux/man-pages/man2/unshare.2.html)) privileges over the remaining namespaces created by the
       call.  Thus, it is possible for an unprivileged caller to specify
       this combination of flags
```

```
      The _uid_map_ file exposes the mapping of user IDs from the user
       namespace of the process _pid_ to the user namespace of the process
       that opened _uid_map_ (but see a qualification to this point below).
       In other words, processes that are in different user namespaces will
       potentially see different values when reading from a particular
       _uid_map_ file, depending on the user ID mappings for the user
       namespaces of the reading processes.
```

我们知道把user namespace与其它namespace分开挂载的话，将会有很多种方案：
1. 先clone user-ns，再clone others-ns
2. 先ushare user-ns，再clone others-ns
3. 先clone user-ns, 再unshare other-ns

第一种方案，必须要开启dump clone flags，这对于rootless container来讲将没法满足。
第二种方案，unshare user-ns后，原来的进程由于进入了新namespace，将没有权限设置多个uid/gid map。
所以这里runc采用先clone在unshare的方式. 然而事实并不是就调用一下clone然后再unshare那么简单，因为要考虑使用已有namespace的问题。最后的逻辑是先clone，然后挂载已有namespace（见下说明），接着进入user namesapce，然后再unshare，注释如下：

```
//挂载已有的namespaces
if (config.namespaces)

    join_namespaces(config.namespaces);

//先new user namespace
if (config.cloneflags &  CLONE_NEWUSER)  {

    if (unshare(CLONE_NEWUSER)  < 0)

        bail("failed to unshare user namespace");

    config.cloneflags &=  ~CLONE_NEWUSER;


if (config.namespaces)  {

    if (prctl(PR_SET_DUMPABLE, 1, 0, 0, 0)  < 0)

        bail("failed  to  set  process  as  dumpable");

}

//调用parent 配置uid/gid映射
s  =  SYNC_USERMAP_PLS;

if (write(syncfd,  &s, sizeof(s))  != sizeof(s))

    bail("failed  to  sync  with  parent:  write(SYNC_USERMAP_PLS)");

/* ...  wait  for  mapping  ... */
if (read(syncfd,  &s, sizeof(s))  != sizeof(s))

    bail("failed to sync with parent: read(SYNC_USERMAP_ACK)");
.........


//unshare namespaces
if (unshare(config.cloneflags &  ~CLONE_NEWCGROUP)  < 0)
    bail("failed  to  unshare  namespaces");
```

说明：
这里join_namespaces是加入runc配置的已有namespace，这个已有namespace是在bundle的config.json中的linux>>namespaces配置, 如果这里配置了path，则是使用已有namespace，没有配置则是进行cloneflags进行ushare。这部分处理逻辑在container初始化bootstrapdata时进行了判断[https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/container_linux.go#L500](https://github.com/opencontainers/runc/blob/v1.0.0-rc8/libcontainer/container_linux.go#L500)

```
"namespaces": [
                        {
                                "type": "pid"
                        },
                        {
                                "type": "network"
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
                ],
```

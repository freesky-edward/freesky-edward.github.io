---
layout: post
title:  "探秘namespace与/dev"
lang: zh
date: 2017-10-27
---

### 说明

一直有一个想法：能否实现给容器内部动态增删裸盘，今天就这个想法做了一个无聊的测试以验证是否可行（理论上是不可行的，无聊进行测试 ：）），简单分享一下这个测试，如果觉得无聊可以直接看最后面的总结。

### 环境准备 

这次用于测试相关的环境：     
Server：google 公有云上的VM Server   
OS：ubuntu 16.04    
Docker：17.06     
container fs: aufs    

首先看下当前系统的磁盘情况：

```shell
root@myinstance:~/workspace/kubernetes# fdisk -l
Disk /dev/sda: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x1a7d4c6a

Device     Boot Start       End   Sectors  Size Id Type
/dev/sda1  *     2048 209715166 209713119  100G 83 Linux
```

目前就一个100G的系统磁盘，该磁盘包括了一个主分区。

为了测试能否为容器动态添加裸盘，首先启动一个container，这里我采用了K8S启动一个pod，pod里只有一个container，启动pod的yaml文件如下：

```shell
root@myinstance:~# cat /root/k8s-configs/busybox-pod-no-volume.yaml 
apiVersion: v1
kind: Pod 
metadata:
    name: busybox-test
    labels:
        name: busybox-test
spec:
    restartPolicy: Never
    containers:
    - resources:
        limits:
          cpu: 0.5
      image: gcr.io/google_containers/busybox
      command:
         - "/bin/sh"
         - "-c"
         - "while true; do date >> /mnt/test/pod-out.txt; sleep 5; done"
      name: busybox
      volumeMounts:
      - name: vol
        mountPath: /mnt/test
    volumes:
        - name: vol
          emptyDir: {}
```

创建好容器之后先找到容器的rootfs在host上的位置：

查找容器的ID

```shell
root@myinstance:~# docker ps
CONTAINER ID     IMAGE      COMMAND     CREATED       STATUS    PORTS         NAMES
f215d1cf2e0f    gcr.io/google_containers/busybox     "/bin/sh -c 'while..."   7 hours ago    Up 7 hours 
  k8s_busybox_busybox-test_default_fc8632f0-bab9-11e7-96f5-42010a8c0002_0
```

查找到docker的ID为f215d1cf2e0f（如上图），然后通过image找到mount-id

```shell
root@myinstance:~# cat /var/lib/docker/image/aufs/layerdb/mounts/
f215d1cf2e0fce92d121274feb8fa0c8f3d0a8afd4aabad9c1352002ad26ac89/mount-id 
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c
```

找到mount-id以后，那么rootfs的host路径就找到了

```shell
root@myinstance:~# ll /var/lib/docker/aufs/mnt/200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/
total 104
drwxr-xr-x  29 root root  4096 Oct 27 07:38 ./
drwx------ 108 root root 28672 Oct 27 07:04 ../
drwxrwxr-x   2 root root  4096 May 22  2014 bin/
drwxr-xr-x   4 root root  4096 Oct 27 07:25 dev/
-rwxr-xr-x   1 root root     0 Oct 27 01:56 .dockerenv*
drwxr-xr-x   6 root root  4096 Oct 27 01:56 etc/
drwxrwxr-x   4 root root  4096 May 22  2014 home/
drwxrwxr-x   2 root root  4096 May 22  2014 lib/
lrwxrwxrwx   1 root root     3 May 22  2014 lib64 -> lib/
lrwxrwxrwx   1 root root    11 May 22  2014 linuxrc -> bin/busybox*
drwxrwxr-x   2 root root  4096 Feb 27  2014 media/
drwxrwxr-x   3 root root  4096 Oct 27 01:56 mnt/
drwxrwxr-x   2 root root  4096 Feb 27  2014 opt/
drwxrwxr-x   2 root root  4096 Feb 27  2014 proc/
drwx------   2 root root  4096 Oct 27 01:59 root/
lrwxrwxrwx   1 root root     3 Feb 27  2014 run -> tmp/
drwxr-xr-x   2 root root  4096 May 22  2014 sbin/
drwxrwxr-x   2 root root  4096 Feb 27  2014 sys/
drwxrwxrwt   4 root root  4096 Oct 27 01:56 tmp/
drwxrwxr-x   6 root root  4096 May 22  2014 usr/
drwxrwxr-x   4 root root  4096 May 22  2014 var/
```

这个路径也就等效于容器里的根目录，在该路径下增删文件，容器里是能看到，具体为啥请看总结。由于是测试裸盘，所以我们需要为这台虚拟机再挂载一块磁盘：

1. 创建磁盘

```shell
root@myinstance:~# gcloud compute disks create mydevice1 --size=10 --zone=asia-east1-a  --type=pd-standard
WARNING: You have selected a disk size of under [200GB]. This may result in poor I/O performance.
 For more information, see: https://developers.google.com/compute/docs/disks#pdperformance.
Created [https://www.googleapis.com/compute/v1/projects/slob-171609/zones/asia-east1-a/disks/mydevice1].
NAME       ZONE          SIZE_GB  TYPE         STATUS
mydevice1  asia-east1-a  10       pd-standard  READY

New disks are unformatted. You must format and mount a disk before it
can be used. You can find instructions on how to do this at:

https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting
```

2. 将磁盘attach到机器上

```shell
root@myinstance:~# gcloud compute instances attach-disk myinstance --disk=mydevice1
Did you mean zone [asia-east1-a] for instance: [myinstance] (Y/n)?  Y

Updated [https://www.googleapis.com/compute/v1/projects/slob-171609/zones/asia-east1-a/instances/myinstance].
```

3. 检查是否attach成功

```shell
root@myinstance:~# fdisk -l
Disk /dev/sda: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x1a7d4c6a

Device     Boot Start       End   Sectors  Size Id Type
/dev/sda1  *     2048 209715166 209713119  100G 83 Linux


Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

对比之前的结果，发现一个新的磁盘（/dev/sdb）已经在服务器上ready


### 测试为容器加盘

测试之前，先登录到容器里检测一下磁盘情况

```shell
root@myinstance:~# kc exec busybox-test -c busybox -it -- /bin/sh
/ #
/ # ls /dev/
core    full      null       pts           shm        stdin            termination-log  urandom
fd      mqueue    ptmx       random        stder      stdout           tty              zero
```

上图显示了目前在该容器的设备情况。

#### 测试1

首先我们尝试为该容器rootfs中的dev目前创建一个磁盘盘符，该盘符指向与新创建的磁盘，检测在容器中是否也能看到这块盘。

上面已经找到了rootfs在host中的路径，我们只需要通过mknod创建一个磁盘，该磁盘的主从设备号与/dev/sdb（新添加的磁盘）一样即可

1. 查询新磁盘的主从设备号

```shell
root@myinstance:/var/lib/docker# ls -la /dev/sd*
brw-rw---- 1 root disk 8,  0 Aug 14 12:01 /dev/sda
brw-rw---- 1 root disk 8,  1 Aug 14 12:02 /dev/sda1
brw-rw---- 1 root disk 8, 16 Oct 27 06:11 /dev/sdb
```

如上图所示主从设备号为8,16

2. 在<rootfs>/dev下创建磁盘盘符

```shell
root@myinstance:~# mknod /var/lib/docker/aufs/mnt/200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c
/dev/xvdb b 8 16
```

3. 检查创建结果

```shell
root@myinstance:~# ls -la /var/lib/docker/aufs/mnt/
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/dev/
total 16
drwxr-xr-x  4 root root  4096 Oct 27 07:25 .
drwxr-xr-x 29 root root  4096 Oct 27 07:38 ..
-rwxr-xr-x  1 root root     0 Oct 27 01:56 console
lrwxrwxrwx  1 root root    10 Feb 27  2014 log -> ../tmp/log
drwxrwxr-x  2 root root  4096 Feb 27  2014 pts
drwxr-xr-x  2 root root  4096 Oct 27 01:56 shm
brw-r--r--  1 root root 8, 16 Oct 27 07:30 xvdb
```

可以看到在该路径下已经有一个具有读写权限的block device，名字叫xvdb. 登录到容器中检查设备情况：

```shell
/ # ls /dev/
core     full         null      pts         shm       stdin      termination-log  urandom
fd       mqueue       ptmx      random      stderr    stdout     tty              zero
```

从上看并没有出现在容器里，说明这种方式没有成功。

#### 测试2

我们再将测试1中的盘格式化后bind mount到容器里的/test（创建test目录该处省略）目录下。

格式化磁盘：

```shell
root@myinstance:~# mkfs.ext4 /var/lib/docker/aufs/mnt/
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/dev/xvdb 
mke2fs 1.42.13 (17-May-2015)
Discarding device blocks: done                            
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: 92c515ed-63b0-46c4-9131-a3c425582541
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

将磁盘mount到test/vol1目录下：（为了测试是否可行，该处采用了share mount）

```shell
root@myinstance:~# mount --make-shared /var/lib/docker/aufs/mnt/
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/dev/xvdb test/vol1/
```

在host的test/vol1目录下写一个文件，然后在容器里检测是否可见

在host上写文件：
```shell
root@myinstance:~# echo "like this is good for me" >> /var/lib/docker/aufs/mnt/
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/test/vol1/cdef.txt

root@myinstance:~# cat /var/lib/docker/aufs/mnt/
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/test/vol1/cdef.txt 
like this is good for me
```

在容器里检测
```shell
/ # ls /test/vol1/
```

由此可见在容器里不可见，说明把这个磁盘格式化后，mount到rootfs里也是不可行的。


### 总结

正如 [从另一个角度理解pod](../what-is-the-pod-in-k8s)里介绍的一样，容器通过namespace将mount点进行了隔离，在容器里或者容器外进行的mount操作是相互不会影响的（注：这个说法不完全正确，后面要解释），在容器外部的/dev目录下创建一个磁盘盘符，虽然是在容器rootfs目录下执行，由于是不同的namespace，所以在容器里的VFS里是看不到这个磁盘的，之后进行mount当然也就无法传播到容器里。

但是在容器启动时，有一些目录是mind mount到rootfs里的，如容器里的根目录与该容器在host上的rootfs之间就是这种方式，正是由于这种mind mount关系，其子目录间的文件操作，包括mount操作是会传播的。也就是说如果在该路径下创建目录或者文件，在容器里是可以看到的。这就是为什么上一段说mount操作相互不会影响是不完全正确的。

在host的/test目录下创建一个目录vol2，再在其下写一个文件，在容器里是可以看到的

```shell
root@myinstance:~# mkdir /var/lib/docker/aufs/mnt/
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/test/vol2
root@myinstance:~# echo "hello i'm testing" >> /var/lib/docker/aufs/mnt/
200fb3caa83cb6be5888c08d63e3c88a2796c787f88aa5a57a719cbe530ab06c/test/vol2/aa.txt
```

再容器中查看：
```shell
/ # cat /test/vol2/aa.txt 
hello i'm testing
```

同理，相应的bind mount也是会生效的，今天在这里进行了这个无聊测试，从另一个角度也说明了为什么需要docker，k8s这样的引擎内核支持裸盘——在容器启动时，在rootfs下创建好盘符；内核支持裸盘要做什么具体工作——通过knod在容器所属的namespace下创建盘符。

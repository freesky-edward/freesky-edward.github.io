---
layout: post
lang: zh
title: "kubernetes存储介绍系列 ——存储架构总览"
date: 2017-09-18
---

Kuberenetes 存储架构总体介绍
==========================

### 说明

本文只针对Kubernetes中的存储做介绍，需要对Kubernetes有基本的了解，主要包括基础术语以及重要的核心组件等，如果你还不清楚，请先阅读[这里](https://kubernetes.io/docs/concepts/)

### 架构概述

Kubernetes (下面简称K8S)对容器存储（特指持久化存储，下同）做了一层自己的抽象，相比docker的存储来讲，K8S的存储抽象更全面，更面向应用，体现在如下几个方面：

 1. 提供卷生命周期管理
 2. 提供“声明”式定义，将使用者和提供者分离
 3. 提供存储类型定义
 
当然，要实现上诉特性，其实现逻辑相比docker的几个简单扩展接口来讲也要复杂不少，但是无论是docker存储也好，K8S存储也好，亦或者是其它类的容器相关的存储抽象，其本质都是一样的——让存储在容器里ready，核心做三件工作：1. provision/delete   2. attach/detach(可选).   3. mount/unmount.  

### 组件详解
 
K8S里存储相关的组件，从顶层来讲，主要包含4大组件：
   
 1. Volume Plugins  ---  存储提供的扩展接口, 包含了各类存储提供者的plugin实现
 2. Volume Manager --- 运行在kubelet 里让存储Ready的部件，主要做上诉介绍核心工作中的2、3
 3. PV/PVC Controller --- 运行在Master上的部件，主要做上诉介绍核心工作中的1
 4. Attach/Detach  --- 运行在Master上，顾名思义，主要做上诉介绍核心工作中的2
  
其中第一个部件是一个基础部件，后三个是逻辑部件，依赖于部件一，这四个部件在K8S集群中的大致关系如下图

![]({{ site.url }}/images/2017-09-18-k8s-storage-architecture/k8s_arthitecture1.png)

注：或许你已经注意到了，上诉介绍的核心工作2在部件二和部件四中均有涉及，对的，这就是说K8S支持两种attach/detach方式，一种是在master上通过调用底层逻辑来实现，另一种是在pod所在的minion节点上来实现。

上诉其实就是K8S内部的基本逻辑架构，扩展出去再加上外部与这些部件有交互关系的部件(调用者和实现者)和内部可靠性保证的部件，就可以得出K8S的架构全景。如下图：

![]({{ site.url }}/images/2017-09-18-k8s-storage-architecture/k8s_arthitecture2.png)

对于调用者，在master上主要是通过监听API Server来获取资源变化，从而触发卷的增删改查，在minion上，因为只有pod调度到这个node上才会有卷的相应操作，所以它的触发端是kubelet（严格讲是kubelet里的pod manager），根据Pod Manager里pod spec里申明的存储来触发卷的挂载操作。

对于实现者这里不用多讲，主要就是存储的提供者+plugin的实现。这里将存储的提供者大致分为这么几类：

1. 持久化存储    
	1.1. 公有云存储 --- google， aws， azure， cinder    
	1.2. 协议类存储 --- iSCSI， NFS， FC ...     
	1.3. 系统类存储 --- vSphere, ScaleIO, Ceph ....    
2. 临时存储    
	2.1. empty dir    
	2.2. 向下接口类存储 --- configmap, downwardAPI, secret    
3. 其它    
	3.1. 主机存储 --- host path, local storage     
	3.2. flex存储 --- flex volume      

剩下的就是可靠性保证部件，这个部件主要职责是保证调用可靠性，如避免同一操作多次提交，超时调用机制等。

### 顶层逻辑流

##### 卷管理流

实现该流程的主要组件是PV Controller，PV Controller和K8S其它组件一样监听API Server中的资源更新，对于卷管理主要是监听PV，PVC， SC三类资源，当监听到这些资源的创建、删除、修改时，PV Controller经过判断是需要做创建、删除、绑定、回收等动作（后续会展开介绍内部逻辑），然后根据需要调用Volume Plugins进行业务处理，大致调用逻辑如下图：

![]({{ site.url }}/images/2017-09-18-k8s-storage-architecture/k8s_arthitecture3.png)

##### 卷挂载

卷的挂载，主要分两个阶段，attach/detach卷和mount/umount 卷，其中卷的attach/detach，前面已经说过了，有两个组件做这个工作，分别是Master上的AttachDetach Controller 和Minion上的VolumeManager，这两者在顶级流程上有一些区别。

先来看看AttachDetach Controller（后简称ADController），ADController的处理流程和上面介绍的PV Controller基本类似，首先监听API Server的资源变化，主要监听的是node和pod资源，通过node和pod变更，触发ADController是否attach/detach操作，然后调用plugin做相应的业务处理，大致流程如下：

![]({{ site.url }}/images/2017-09-18-k8s-storage-architecture/k8s_arthitecture4.png)

VolumeManager相比ADController最大的区别是事件触发的来源，VolumeManager不会监听API Server，在Minion端所有的资源监听都是Kubelet完成的，Kubelet会监听到调度到该节点上的pod声明，会把pod缓存到Pod Manager中，VolumeManager通过Pod Manager获取PV/PVC的状态，并进行分析出具体的attach/detach, 操作然后调用plugin进行相应的业务处理，流程如下：

![]({{ site.url }}/images/2017-09-18-k8s-storage-architecture/k8s_arthitecture5.png)

注：为了在attach卷上支持plugin headless形态，Controller Manager提供配置可以禁用ADController。

对于mount/umount其流程和attach/detach类似，不再详细介绍，详细流程后续在专门介绍。这里需要说明一点的是如果是ControllerManager执行attach操作，当执行成功后需要通过API Server来刷新volume的实际状态，这样以保证Volume Manager的mount操作是在卷attach完成后执行。
   
### 结语

至此，K8S的存储大致组件架构和顶级流程就基本介绍完了，看似非常的简单，但是实际内部处理逻辑远比这个复杂，如：PV、PVC的绑定，Controller里期望状态与实际状态间的协调处理，多次操作归一化处理等等，后续分章节再展开进行介绍。

---
layout: post
title:  "Kubernetes存储介绍系列 —— AttachDetachController1"
lang: zh
date: 2017-09-26
---

### 说明

在[CSI plugin设计]({{site.url}}/k8s-csi-design) 中简单介绍了AttachDetachController（简称AD Controller）的实现原理，它主要包括两个缓存，一个populator，一个reconciler，一个status updater，现再深入介绍一下这些组件的内部设计和实现，这篇文章主要介绍其中的两个缓存：actual state of world 和desired state of world

本文适合对实现细节感兴趣的阅读。

### AttachDetachController中对象管理

在[CSI plugin设计]({{site.url}}/k8s-csi-design) 中已经介绍过了，AD Controller主要是将卷attach/detach相关的对象的两种对象保存在缓存中，以便reconciler对比来决定是否做attach/detach操作，基于这个定位，对于actual state of world 和desired state of world，主要是弄清楚：

1. 存储的对象及存储结构
3. 提供了哪些外部访问接口

相信弄清楚了上面三点，对于AD Controller的原理认识将会更深刻，以下就分别从这两方面介绍这两个缓存。

#### Actual State Of World

Actual State的主要代码实现主要是在[actual_state_of_world.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/volume/attachdetach/cache/actual_state_of_world.go) 文件，由于涉及到volume的接口操作，所以部分顶层接口是定义在[volume文件夹](https://github.com/kubernetes/kubernetes/tree/master/pkg/volume)里，如下面将要介绍的[ActualStateOfWorldAttacherUpdater](https://github.com/kubernetes/kubernetes/blob/master/pkg/volume/util/operationexecutor/operation_executor.go#L154)接口就是在volume相关定义的文件夹下。

1. 存储对象及存储结构

![]({{ site.url }}/images/2017-10-19-k8s-adcontroller-caches/actual_state_data_model.png)

顶级对象是actualStateOfWorld，在这个对象里包含两个核心的模型，一个是以卷名字索引的attachedVolume集合，另一个是以node名索引的nodeToUpdateStatusFor集合。

attachedVolume代表了一个卷的attach情况（从卷的角度索引信息），包括attach到哪些节点上，在这些节点上是否已经mount，以及attach的设备路径等等, 由于一个卷可以attach多个节点上，volume和node是1对多的关系，所以在attachVolume有一个以node名索引的nodeAttachedTo集合。

nodeAttachedTo代表一个卷attach到某个node上，以及在这个node上的一些信息。

nodeToUpdateStatusFor代表了一个node上卷的attach情况（从node的角度索引信息），包括了attach到某个node的所有卷的名称，以及是否要更新API-Server中node.Status.VolumesAttached这个状态。

2. 提供外部访问接口

![]({{ site.url }}/images/2017-10-19-k8s-adcontroller-caches/actual_state_interfaces.png)

实际接口主要是在[ActualStateOfWorld](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/volume/attachdetach/cache/actual_state_of_world.go#L38)，这个接口又继承了[ActualStateOfWorldAttacherUpdater](https://github.com/kubernetes/kubernetes/blob/master/pkg/volume/util/operationexecutor/operation_executor.go#L154)接口，这些接口可以分为如下几类：

1. 对attachedVolume、nodeToUpdateStatusFor基础信息的更新类接口
2. 面向业务处理的信息组合查询接口。

通过上述的模型和接口，基本实现了从node、volume两个维度维护、处理与attach相关的实际业务处理。

#### Desired State Of World

Desired State的主要代码实现主要是在[desired_state_of_world.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/volume/attachdetach/cache/desired_state_of_world.go) 文件

1. 存储对象及存储结构

![]({{ site.url }}/images/2017-10-19-k8s-adcontroller-caches/actual_state_data_model.png)

期望状态主要是由三个核心对象组织，node，pod，volume，顶层对象是desiredStateOfWorld，这个对象根据node名索引记录了一个nodeManaged集合。

nodeManaged代表了一个node希望要attach的卷集合，在nodeManaged中记录一个以volume唯一名字标识的volumeToAttach集合。

volumeToAttach代表了一个卷希望的attach状态以及调度到哪些pod里使用。

2. 提供外部访问接口

![]({{ site.url }}/images/2017-10-19-k8s-adcontroller-caches/desired_state_interfaces.png)

在API中，volume的定义实际是在pod里，所以Desired Status的接口就非常简单，主要就是node，pod的增删，以及卷的查询。

### 总结

对于Desired Status主要按照一个node对应多个volume，一个volume对应多个pod的关系进行模型存储，然后根据API接口中node，pod的增删提供相应的接口。

对于Actual Status则是从一个volume attach到哪些node，以及某个node attach哪些volume两个方便进行了信息索引，接口也根据业务需要进行了复杂的定义。

以上设计文档以发布在[这里](https://github.com/freesky-edward/k8s-research/tree/master/design-analysis/pkg/controller/volume/attach_dettach_controller)



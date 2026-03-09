---
title: 一致性哈希算法在Swift中的应用
date: 2017-01-16 18:25:31
categories:
- OpenStack
- Swift
tags:
- OpenStack Swift
- 一致性哈希
---

> Created by byliu@iflytek.com

## 前言 ##

在分布式对象存储中，一个关键问题是数据该如何存放。Swift是基于**一致性哈希**技术，通过计算可将对象均匀分布到虚拟空间的虚拟节点上，在增加或删除节点时可大大减少需移动的数据量。本文主要介绍一致性哈希在云存储Swift中具体应用。

## 正文 ##

### 普通哈希算法与场景分析 ###

先来看一个简单的例子，假设我们手里有N台存储服务器（以下简称node），打算用于图片文件存储，为了使服务器的负载均衡，需要把对象均匀地映射到每台服务器上，通常会使用哈希算法来实现，计算步骤如下：
 
1.	计算object的hash值Key
2.	计算Key mod N值

<!--more-->
      
有N个存储节点，将Key模N得到的余数就是该Key对应的值需要存放的节点。比如，N是2，那么值为0、1、2、3、4的Key需要分别存放在0、1、0、1和0号节点上。如果哈希算法是均匀的，数据就会被平均分配到两个节点中。如果每个数据的访问量比较平均，负载也会被平均分配到两个节点上。

但是，当数据量和访问量进一步增加，两个节点无法满足需求的时候，需要增加一个节点来服务客户端的请求。这时，N变成了3，映射关系变成了Key mod (N+1)，因此，上述哈希值为2、3、4的数据需要重新分配（2->server 2，3 -> server 0，4 -> server 1）。如果数据量很大的话，那么数据量的迁移工作将会非常大。当N已经很大，从N加入一个节点变成N+1个节点的过程，会导致整个哈希环的重新分配，这个过程几乎是无法容忍的，几乎全部的数据都要重新移动一遍。

举例说明，假设有100个node的集群，将10^7项数据使用md5 hash算法分配到每个node中,现在假设增加一个节点提供负载能力，不过得重新分配数据项到新的节点上。

通过简单的计算我们可以发现，为了提高集群1%的存储能力，我们需要移动9900989个数据项，也就是99.01%的数据项！显然，这种算法严重地影响了系统的性能和可扩展性。

![增加1%的存储能力=移动99%的数据](http://byliu.bj.openstorage.cn/byliu/普通HASH移动数据.gif "增加1%的存储能力=移动99%的数据")

一致性哈希算法就是为了解决这个问题而来。

### 一致性哈希算法 ###
      
一致性哈希算法是由D. Darger、E. Lehman和T. Leighton 等人于1997年在论文 [Consistent Hashing and Random Trees:Distributed Caching Protocols for Relieving Hot Spots On the World Wide Web](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf) 首次提出，目的主要是为了解决分布式网络中的热点问题。在其论文中，提出了一致性哈希算法并给出了衡量一个哈希算法的4个指标：

*	平衡性(Balance)
	*	平衡性是指Hash的结果能够尽可能分布均匀，充分利用所有缓存空间。
*	单调性(Monotonicity)
	*	单调性是指如果已经有一些内容通过哈希分派到了相应的缓冲中，又有新的缓冲加入到系统中。哈希的结果应能够保证原有已分配的内容可以被映射到新的缓冲中去，而不会被映射到旧的缓冲集合中的其他缓冲区。
*	分散性(Spread)
	*	分散性定义了分布式环境中，不同终端通过Hash过程将内容映射至缓存上时，因可见缓存不同，Hash结果不一致，相同的内容被映射至不同的缓冲区。
*	负载(Load)
	*	负载是对分散性要求的另一个纬度。既然不同的终端可以将相同的内容映射到不同的缓冲区中，那么对于一个特定的缓冲区而言，也可能被不同的用户映射为不同的内容。

Swift使用该算法的主要目的是在改变集群的node数量时（增加/删除服务器），能够尽可能少地改变已存在key和node的映射关系，以满足单调性。一致性哈希一般两种思路：

1.	迁移为主要特点(swift初期采用)
2.	引入虚结点，减少移动为特点(swift现采用)

具体步骤如下：

1.	首先求出每个节点(机器名或者是IP地址)的哈希值，并将其分配到一个圆环区间上（这里取0-2^32)。
2.	求出需要存储对象的哈希值，也将其分配到这个圆环上。
3.	从对象映射到的位置开始顺时针查找，将对象保存到找到的第一个节点上。

其中这个从哈希到位置映射的圆环，叫做哈希环(Ring)。哈希环空间上的分布如下图所示：

![哈希环空间](http://byliu.bj.openstorage.cn/byliu/哈希环空间.png "哈希环空间")

假设在这个环形哈希空间中，新增一个node5，node5被映射在node2和node4之间，那么受影响的将仅是沿node5逆时针遍历直到下一个node（node2）之间的对象（它们本来映射到node4上）。如下图所示：

![一致性哈希算法的数据移动](http://byliu.bj.openstorage.cn/byliu/一致性哈希算法的数据移动.png "一致性哈希算法的数据移动")

通过前面的分析可以知道，一致性哈希算法对于节点的增加(或减少)都只需重定位环空间中的一小部分数据，具有较好的容错性和可扩展性。

但是上边描述的一致性Hash算法还有两个潜在的问题:

1.	将节点hash后会不均匀地分布在环上，这样大量key在寻找节点时，会存在key命中各个节点的概率差别较大，无法实现有效的负载均衡。
2.	如有三个节点Node1,Node2,Node3，分布在环上时三个节点挨的很近，落在环上的key寻找节点时，大量key顺时针总是分配给Node2，而其它两个节点被找到的概率都会很小。

为了解决这个问题，我们引入**虚拟节点(Partition)**的概念

### 虚拟节点(Partition) ###

前面也说道，一致性哈希算法在服务节点太少时，容易因为节点分部不均匀而造成数据倾斜问题。例如系统中只有两台服务器，其环分布如下：

![两节点不平衡环](http://byliu.bj.openstorage.cn/byliu/两节点不平衡环.png "两节点不平衡环")

此时必然造成大量数据集中到Node A上，而只有极少量会定位到Node B上。

考虑到哈希算法在node较少的情况下，改变node数会带来巨大的数据迁移。为此一致性哈希引入了“虚拟节点”的概念： “虚拟节点”是实际节点在环形空间的复制品，一个实际节点对应了若干个“虚拟节点”，“虚拟节点”在哈希空间中以哈希值排列。如下图：

![虚拟节点](http://byliu.bj.openstorage.cn/byliu/虚拟节点.jpg "虚拟节点")

引入了“虚拟节点”后，映射关系就从[object--->node]转换成了[object--->virtual node---> node]。查询object所在node的映射关系如下图所示。

![对象、虚结点、节点间的映射关系](http://byliu.bj.openstorage.cn/byliu/对象、虚结点、节点间的映射关系.jpg "对象、虚结点、节点间的映射关系")

这样就解决了服务节点少时数据倾斜的问题。在实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。

### 权重(weight) ###

基于虚拟节点(partition)，引入weight的概念，目的是“能者多劳”：解决未来添加存储能力更大的node时，使得可以分配到更多的虚拟节点(partition)。例如，2T 容量的node的虚拟节点(partition)数为1T的两倍。

## 总结 ##

以上就是一致性哈希算法在云存储swift中的应用了，swift中的核心组件——Ring的实现就是基于此，当然Ring在实现上还有更多细节上优化的地方，这里略过不讲。

一致性哈希算法除了在云存储swift中解决存储数据节点均衡问题外，在很多场景下也得到了应用，比如分布式cache系统(参考资料1)、服务器均衡(参考资料3)等。

本文只是简要介绍了这个算法，更深入的内容可以参看论文[Consistent Hashing and Random Trees:Distributed Caching Protocols for Relieving Hot Spots On the World Wide Web](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)，同时提供一个[C语言版本的实现](http://www.codeproject.com/Articles/56138/Consistent-hashing)供参考。

## 参考资料 ##

1.	核心论文：[Consistent Hashing and Random Trees:Distributed Caching Protocols for Relieving Hot Spots On the World Wide Web](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)

2.	[memcache的一致性hash算法使用](http://blog.csdn.net/kongqz/article/details/6695417)

3.	[一致性Hash算法背景](http://www.cnblogs.com/haippy/archive/2011/12/10/2282943.html)
	
4.	[一致性哈希算法（用于解决服务器均衡问题）](http://blog.csdn.net/caigen1988/article/details/7708806)

5.	[每天进步一点点——五分钟理解一致性哈希算法(consistent hashing)](http://blog.csdn.net/cywosp/article/details/23397179/)

6.	[深入云存储系统Swift核心组件：Ring实现原理剖析](http://www.cnblogs.com/yuxc/archive/2012/06/22/2558312.html)

7.	[一致性哈希算法及其在分布式系统中的应用](http://www.openstack.cn/?p=303)

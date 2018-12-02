---
layout:     post
title:      "Cassandra数据库基础总结"
subtitle:   "NoSQL数据库，分布式数据库，Cassandra，大数据存储"
date:       2018-12-01
author:     "ZihaoRao"
header-img: "img/post-bg-bigdata.jpg"
tags: Cassandra，NoSQL，数据可视化
---





#### 总体概述

> 对于一位分布式存储系统的开发者，`Cassandra`无疑是非常引人注目的，它的无中心架构、高可用性、无缝扩展等继承自亚马逊`Dynamo`的特质，相对于其他主从架构的`NoSQL`系统更加简洁，也更具有美感！   
>
> ​                                                                                                                                             ——Cassandra权威指南



#### 基本特点

------

1. ##### 分布式与无中心

   `Cassandra`是分布式的，这意味着它可以运行在多台计算机上，并呈现给用户一个一致性整体。相比于其他主从分布式数据库系统，它具有无中心特点，这意味着它不存在单点失效。`Cassandra`集群中所有节点的功能都是一样的。

2. ##### 弹性可扩展

   弹性可扩展指集群可以在不间断服务的情况下完成扩展或缩减规模，在`Cassandra`集群里，只要加入或从中移除计算机，`Cassandra`集群会自动地发现并让其参与工作或离开。

3. ##### 面向“行”

   `Cassandra`一般会被看作“面向列”的数据库，确实，它的数据结构不是关系型的，而是一张**多维稀疏哈希表。“稀疏”**意味着任何一行都会有一列或多列，但每行都不一定有和其他行一样的列数。与关系型数据库类似，其每一行也有唯一的键值，用于进行数据访问。总的来说，`Cassandra`是一个有索引，面向行的存储系统。



#### Cassandra数据模型

------

- ##### 总体结构

  `Cassandra`数据库中表的实现结构：`Map<RowKey, SortedMap<ColumnKey, ColumnValue>>`

  从上述结构便可理解，为什么总说`Cassandra`有面向列的特点，也可以解释为什么`Cassandra`中表的各行可  以有不一样的列数。

- ##### 键值key

![key](/img/in-post/content/key.png)

​       由上图可知，在`Cassandra`中，每一行数据记录是以***key/value***的形式存储的，其中`key`是唯一标识。

​       **`Cassandra`中键的组成**：`Primary Key = Partition Key + Clustering Key`

​       **Primary Key：**唯一确定`Column Family`(即关系数据库中表)的唯一一行数据。

​       **Partiton Key：**可以由一列或多列组成，其确定行在集群中所存放的主机，不需要唯一，最好设计得使所有数  据能够较为均匀的存放在集群中的各主机上。

​       **Clustering Key：**也是由一列或者多列组成，其确定行在对应主机中的排列顺序，`Partition Key`相同的行，`Clustering Key`必须不同。



- #####  Column结构

`Column`是`Cassandra`所支持的最基本的数据模型，该模型由三个键值对构成。

```jade
{
    "name" :  "User Name" //列名，其实唯一的
    "value" :  "Steve"   //列值
    "timestamp" : "123456789" //更新时间
}
```



- ##### 特点总结与归纳

`Cassandra`数据库与关系型数据库在数据模型之间有着显著差异，由于其表是由`HashMap`数据结构封装，使其具有高速的查询能力和整体分布式架构，所以其非常适合于大数据的存储和处理。另外由于其为保证高性能，从而牺牲了许多关系型数据库所具有的查询灵活性（其`CQL`操作`API`中种种局限性就是最好体现）。所以我们在使用`Cassandra`数据库时，不能用关系型数据库的观点来理解和使用它，这样是没有意义的，因为两者的应用场景不一样！关系型数据库中强调“关系”，有范式等概念。而`Cassandra`是反范式的，为了性能其仅支持单表查询，通过一定的数据冗余提高查询性能，根据业务查询需求设计键等等都是采用与关系型数据库不同，甚至截然相反的思想来解决大数据存储问题。



#### Cassandra的CQL（Cassandra Query Language）操作基本语法

------

​ `CQL`是`Cassandra`目前主要的交互接口，其拥有类似于`SQL`语句的语法，但相对于`SQL`语句来说在数据查询方面很弱。分页、`Order By`等等很多关系型数据库中支持的功能其都支持得不好！也许它就不适合数据的查询显示而重点关注大数据的存储等。（由于其他人已经总结了，所以不再累述）

（1）[操作语法](https://www.cnblogs.com/youzhibing/p/6617986.html)

（2）[DataStax为java提供操作Cassandra的JDBC](https://www.cnblogs.com/youzhibing/p/6607082.html)



#### Cassandra稍底层介绍

------

​最后这一部分主要涉及`Cassandra`分布式集群的相关概念：集群中的节点间数据传输协议（`Gossip`）、确定数据节点分布`Hash`算法（`Consistent Hash`）等，[详见他贴](https://www.cnblogs.com/loveis715/p/5299495.html)



#### 参考资料

------

1. 《Cassandra权威指南》
2.  [Cassandra Query Language API](http://cassandra.apache.org/doc/latest/cql/index.html)
3. [Cassandra高级操作之索引、排序以及分页](https://www.cnblogs.com/youzhibing/p/6617986.html)
4. [DataStax为java提供操作Cassandra的JDBC](https://www.cnblogs.com/youzhibing/p/6607082.html)
5. [Cassandra简介](https://www.cnblogs.com/loveis715/p/5299495.html)
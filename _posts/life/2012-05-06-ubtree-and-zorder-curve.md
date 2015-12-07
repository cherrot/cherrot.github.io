---
layout: post
title: "UBTree & Z-Order Curve学习"
date: 2012-05-16 10:24:00
categories: life
tags: 
keywords: algorithm
description: 

---

最近太忙，花两天时间复习各种树结构，大致浏览了二叉搜索树（BST）、笛卡尔树和Treap、T树、AVL树、B树、B+树、B\*树、R树，然后便看到了我需要学习的主题——*UB树*。

关于B树、B+树、B\*树和R树的详细解释，强烈推荐[结构之法 算法之道的这篇博文][ref1]。

关于UB树的文献资料不多，我先从[WikiPedia的条目][ref2]看起，原来UB树实质是一棵B+树，只是通过某种space filling curves算法（Z-order curve）将多维访问降为一维访问以适用于B+树。这就使得UB树比其他用于多维访问的树结构（比如R树）更容易同现今的关系数据库产品整合，毕竟目前绝大多数的RDBMS都是使用B树和其变种作为数据存储结构的。

UB树的核心在于多维数据的降维方法，也就是Z-order curve，这里我参考了[这篇文章][ref3]和[WikiPedia][ref4]条目，简单来说就是将多维数据的二进制码交叉存储达到降维的目的。

明白了UB树降维的原理后，就需要了解范围查询(Range Queries，比如查询某位置附近5公里内的酒店。区别于一维查询中的点查询point queries和区间查询interval queries)中的核心问题：如何从当前查询到的结点计算得出范围内的下一个Z值(Z-Value)？这里请参考如下文章：

+ [B-tree and UB-tree][ref5] 该文章主要讲述数据库中的B树算法
+ [Integrating the UB-Tree into a Database System Kernel][ref6]

另外强推UB-tree的“官方网站”：<http://mistral.in.tum.de/>，这里有很多关于UB树的论文，而且有很多presentation，包括算法的执行演示，可以给我们更可视化的了解。

[ref1]: http://blog.csdn.net/v_JULY_v/article/details/6530142/ "从B 树、B+ 树、B* 树谈到R 树"
[ref2]: http://en.wikipedia.org/wiki/UB-tree "UB-tree"
[ref3]: http://cg2010studio.wordpress.com/2011/12/06/%E6%91%A9%E9%A0%93%E7%A2%BC-morton-code/ "摩頓碼 (Morton Code)"
[ref4]: http://en.wikipedia.org/wiki/Z-order_curve "Z-order curve"
[ref5]: http://www.scholarpedia.org/article/B-tree_and_UB-tree "B-tree and UB-tree"
[ref6]: http://www.vldb.org/dblp/db/conf/vldb/RamsakMFZEB00.html "Integrating the UB-Tree into a Database System Kernel"

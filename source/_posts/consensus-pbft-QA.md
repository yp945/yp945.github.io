---
title: pbft共识算法问题答案
date: 2019-07-10 10:01:27
categories:
    - 共识算法
tags:
    - pbft
author: ClimbYang
---

### pbft共识算法问题答案

- **pfbt共识为什么至少需要3f+ 1个节点？**

  最坏的情况，系统拜占庭节点为f个，由于消息到达顺序的问题，有可能f个有问题的节点先比f个正常的节点先返回消息，此时又要保证正确的消息比有问题的消息多，所以至少3f+ 1个节点
  $$
  N - f- f > f => N > 3f
  $$

<!-- more -->

- **pbft共识 parpare和commit 阶段为什么收到需要2f+ 1个相同的回复（包括自己的），f + 1个不行吗？**

  某副本收到了f+ 1相同的消息反馈,如果这个f+ 1个反馈中包含faulty 节点，此时消息是不能作数的，因为faulty可能会发送错误消息给不同的节点，所以需要必须要2f+1个相同的反馈确认才能保证f+1个non-faulty节点正常，这时候即便f个faulty节点给不同人发不同消息也没关系，f+1个non-faulty节点已经形成了统一战线，他们在人数上已经多于那些墙头草了，可以达成一致了。

- **pbft 共识 客户端为什么需要f + 1个节点的相同回复，f个不行吗？**

  假设只需要从f个不同的节点那里拿到相同的reply，但我们不得不考虑一种情况，即这f个相同的reply全是来自f个faulty节点【系统中至多有f个faulty节点】。如果真是这样的话，很有可能客户端就得到错误的结果。因此为了进一步增强reply的可信度，我们需要来自不同节点的总计(f+1)个相同reply。多出的那一个可以作为对比！

- **pbft共识 为什么需要三阶段？去除掉commit阶段可以吗？**

  假设我们去掉commit，所有节点收到2f+1(包括自己)的prepare之后就执行操作，会发生什么？

  其实如果顺利的话，即使有f个作恶节点，依然有f+1个正常节点所有节点都会收到正确的结果，最后所有的节点都能顺利的达成一致的结论。这样看来似乎我们完全不需要commit吗？

  但是如果主节点崩溃发生换主，其中只有一个或几个（不是大多数）已经收到了足够的prepare，其他节点因为网络原因没有收到本应该收到的足够多的prepare（异步网络环境没有任何通信保证，只有最终一定会收到的保证），那么那个执行了操作的节点就悲剧了，这个时候新主发起新一轮共识，sequence跟已经执行的操作一致，那个节点到底执行好还是不执行同样sequence的操作？

  那么commit是怎么做到的呢？假设节点收到足够多的prepare进入commit阶段，这个时候发生了一样的换主情形，由于节点还没执行，继续按照新一轮的流程走即可，这个时候sequence不变，但是view改变。

  如果已经收到了足够的commit，并且已经执行了操作呢？仿佛陷入了prepare一样的地步...但是实际上因为要产生commit消息，说明2f+1个节点已经prepare了，换主的时候主会去搜集要重放的pre-prepare（2f+1个节点的，必然存在一个诚实节点并且有对应的pre-prepare）,因此会把同样的digest对应的消息view改为自己重发一次，并且注意到commit只需要跟当前的view相同就可以接受，那么实际上commit是对view不敏感的。

  简而言之，prepare锁定同一个view下的sequence，commit锁定sequence。


- **在一个节点数为N的节点中，诚实节点的数量是多少个？**

  (non-faulty)=(2/3)*N+1

- **CAP定理在pbft中是如何取舍的？**

  PBFT算法将一致性（C）摆在首位，对可用性（A）作了妥协。一旦faulty节点的数量超过f，该系统就不能继续执行客户端的请求【系统会卡住，不能做写操作】。此外，分区容忍是必须要保证的。

- **设置waterline的目的是什么？**

  假设主节点是坏的，它在给请求编号时故意选择了一个很大的编号，以至于超出了序号的范围，所以我们需要设置一个低水位（low water mark）h和高水位（high water mark）H，让主节点分配的编号在h和H之间，不能肆意分配
  
- **PAREPARE 和commit阶段为什么需要保存消息在本地或者内存？PRE-PREPARE为什么不需要？**

  保存消息的主要目的是为了方便viewChange的时候能够恢复消息，重新在新的view上达成共识。PRE-PREPARE阶段各节点还没有发送消息给对方，所以不需要保存。

- **pbft通信时间复杂度是多少？如何计算的？**

  因为需要三阶段共识，每个阶段各个节点之间都需要通信，所以通信量还是很大的。

  假设系统中存在2个拜占庭节点，此时应该最少需要7个节点，下图展示了7个节点通信的过程

![](http://ww1.sinaimg.cn/large/c26c1fe3gy1g2yw43eqxfj20kd0f2win.jpg)
    请求消息总量为：
$$
1 + 3f + 3f(3f-f) + (3f-f+1)(3f+1) + 3f-1
$$
   在上述例子中我们可以进行一个简单计算

```java
 request messages: 1
 pre-prepare messages: 3f = 6
 prepare messages: 3f(3f-f) = 24
 commit messages: (3f-f+1)(3f+1)= 35
 reply messages: 3f-1 = 5
```
可以看出当有7各节点时，pbft需要的消息通信总量竟然达到了71次，这还是只有一次请求的情况下，如果副本更多，消息将会变得更多。
- **如果在commit阶段view change，会导致达成不了共识吗？会导致之前的view下的请求编号丢失吗？**

  如果commit阶段viewchange，会保留之前commit阶段的请求，不会达成不了共识，也不会丢失请求编号

  prepare阶段和commit阶段用来确保那些已经达到commit状态的请求即使在发生viewchange后在新的view里依然保持原有的序列不变，比如一开始在view 0中，共有req 0， req 1， req2三个请求依次进入了commit阶段，假设没有坏节点，那么这四个replicas即将要依次执行者三条请求并返回给Client。但这时主节点问题导致view change的发生，view 0 变成 view 1，在新的view里，原本的req 0， req1， req2三条请求的序列被保留，作数。那些处于pre-prepare和prepare阶段的请求在view change发生后，在新的view里都将被遗弃，不作数。

  简单来说就是 如果每个节点都进入了commit阶段（这里要强调的是每个节点都进入这个commit阶段才算是整体进入了commit阶段），这时即使view change，也会保留之前的view里进入commit阶段的请求信息，view change会继续之前的commit阶段请求，不会再重新进入pre-prepare和prepare阶段。

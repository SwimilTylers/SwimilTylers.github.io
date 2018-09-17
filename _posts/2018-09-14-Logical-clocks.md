---
layout:     post
title:      "Distributed Algorithm Basics (V) - Scenario: Logical Clock"
subtitle:   "Notes on DA study"
date:       2018-09-14 19:08:40
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - logical clock
    - Lamport
    - Neiger-Toueg-Welch
    - Vector clock
    - Notes
---

> 这是分布式算法笔记的第五章，讲述两种标量(_scalar_)时钟和一种向量(_vector_)时钟。

逻辑时钟主要解决的是在现实情况非同步的下如何确定时间戳(_timestamp_)，本章介绍的三种算法各有优劣。参考Yale讲义。

## Causal ordering

__happens-before__ $e\Rightarrow_se'$:

1. $e$ precedes $e'$ in $S$ and $e$ and $e'$ are events of the same process;
2. $e$ is a send-event and $e'$ is the receive-event for the same message;
3. there exists a third event $e''$ such that $e\Rightarrow_s e''$ and $e''\Rightarrow_s e'$.

__causal shuffle__: causal shuffle of a schedule $S$ is a permutation of $S$ that is consistent with the happens-before relation on $S$

___Lemma 1___: Let $S'$ be a permutation of the events in $S$. Then, the following statements are equivalent:

 	1. $S'$ is a causal shuffle of $S$ 
		2. $S'$ is the schedule of an execution fragment of a message-passing system with $\forall p\rightarrow S|p=S'|p$

In general, we will have a relation $<_L$ between timestamps such that $e\Rightarrow_S e'$ implies $e<_L e'$, but it may be that there are some pairs of events that are ordered by the logical clock despite being incomparable in the happens-before relation.

Both Lamport clocks and Neiger-Toueg-Welch clocks are scalar clocks. The major difference is that Lamport clocks do not alter the underlying executions with arbitrarily large jumps in clock values, while NTW clocks guarantee small increment at the cost of possibly delaying parts of the system.

## Lamport clocks

Lamport clocks是运行在现有协议之上的，对于每个进程和消息都添加相应的标签，这些标签信息对于底层协议是透明的，每个进程都维护一个本地的$clock$。

1. 每当该进程发送一次信息或者进行一次内部步骤时，$clock\leftarrow clock+1$，并且将更新好的$clock$戳在信息上
2. 每当该进程接收一次消息（该消息携带时间戳$t$）的时候，$clock\leftarrow \max(clock, t)+1$, 更新过的$clock$作为信息接收时间

## Neiger-Toueg-Welch clocks

NTW clocks的时间戳由多个分量构成$\left<clock, id, eventCount\right>$，其中$eventCount$是收发信息时间的计数，与Lamport clocks主要的区别在于，传送协议不能改变本地时间，并且当接受到的信息携带的时间戳迟于当前本地时间的时候，一直等待直到本地时间超过信息携带的时间戳，并且此时的本地时间作为信息接收时间。

这种logical clocks适用于对于时间计数有着有限增量要求的系统，比如需要经过1000轮定期检查的系统，需要不同的进程中的时间是近似的，否则将会出现过长时间的等待。

## Vector clocks

Logical clocks give a superset of the happens-before relation. If we want to compute $\Rightarrow_S$ exactly, we can use a vector clock: each event is stamped with a vector of values, one for each process. 

1. 每当该进程发送一次信息或者进行一次内部步骤时，仅仅增加该进程对应的分量
2. 每当该进程接收一次消息（该消息携带时间戳$t$）的时候，增加该进程对应的分量，并且将其他进程对应的分量修改为$\max(x_p, t)$

Vector clock满足如下的性质

$$\forall e,e',i,\rightarrow \left(VC(e)_i \leq VC(e')_i\;\Leftrightarrow \;e\Rightarrow_S e'\right)$$

##Applications

### Consistent snapshot

__Definition__: a description of the states of the processes that gives the global configuration at some instant of a schedule that is a consistent reordering of the real schedule. Logical time can be used to obtain a consistent snapshot, otherwise we can employ Chandy-Lamport algorithm.

__Chandy-Lamport algorithm__: equivalent to the logical-time snapshot based on Lamport clocks

1. ___snap___ _message_: the central initiator broadcasts a __snap__ message, and each process records its state and immediately forwards the __snap__ message to all neighbors when it first receives a __snap__ message.
2. ___marker___ _message_: it's used to sweep messages in transit out of com-channel, avoiding the needing to keep logs if we want to reconstruct messages in transit.
   - when a process records its state after receiving the __snap__ message, it issues a __marker__ message on each outgoing channel
   - for incoming channels, the process records all the messages received between the snapshot and receiving a __marker__ message on that channel (or nothing if it receives __marker__ before receiving __snap__). 
   - A process only reports its value when it has received a __marker__ on _each_ channel.
3. ___snap___ and ___marker___ can be combined if the broadcast algorithm for __snap__ resends it on all channel anyway.
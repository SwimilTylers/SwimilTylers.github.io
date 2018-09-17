---
layout:     post
title:      "Distributed Algorithm Basics (I) - Models"
subtitle:   "Notes on DA study"
date:       2018-08-27 16:07:10
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - Message-Passing model
    - Shared Memory model
    - Notes
---

> 这是分布式算法笔记的第一章，讲述两个基础的分布式计算的理论模型。

新的一学期，导师想让我了解一下Paxos，因此让我先入门一下分布式算法，打一下基础，入门的书选择的是

- Attiya: _Distributed Computing-Fundamentals Simulations and Advanced Topics_
- _2017-Yale-Notes on Theory of Distributed Systems_

大概将这几周的学过的进行整理，做成笔记，笔记主要是围绕Attiya展开，Yale笔记进行一些补充，主要是对一些概念的整理和对算法的概述有效性证明，如果你想更加深入地了解分布式算法，不妨看一看原文。本章主要整理了分布式算法的几个基本的模型：Message-Passing model和Shared Memory model。此外这些模型还分同步和非同步，但是我们不再对同步的共享内存模型进行讨论。

| Computation models                 | Event                            | Complexity                           | Admissible                                                   |
| ---------------------------------- | -------------------------------- | ------------------------------------ | ------------------------------------------------------------ |
| Asynchronous shared memory model   | computation step                 | space complexity                     | 1) infinite execution; 2) each processor has an infinite number of computation steps |
| Asynchronous message-passing model | computation step & delivery step | message complexity & time complexity | 1) each processor has an infinite number of computation events; 2) every message sent is eventually delivered |
| Synchronous message-passing model  | computation step & delivery step | message complexity & time complexity | infinite execution                                           |


## Message-passing system

- 每个双向通道(_channel_)连接两个处理器(_processor_)，所有通道的分布模式构成了这个系统的拓扑结构(_topology_)，通常用无向图表示，这些通道的总和就是网络(_network_)

- 通常在这样的系统下，算法由每个处理器的本地程序(_local programs_)够成，该本地程序主要负责本地的运算(_local computation_)以及根据拓扑结构组建的邻里关系(_neighbor_)收发报文(_message_)

- 每个处理器$p_i$在无向图中表示为一个节点，出入度为$r_i$，处理器本身可以表示为状态集合为$Q_i$的状态机(_state machine_)，每个状态都有$2r$个特别的组件(_component_)，也就是为每个连接的通道准备一个$outbuf_i[l]$和一个$inbuf_i[l]$存放未处理报文。$Q_i$会包含一个特别初始状态(_initial states_)，在该状态下每一个$inbuf_i[l]$皆为空

- 每个处理器有一个转换函数(_transition function_)，接收一个值作为$p_i$的一个可到达状态(_accessible state_)的输入，并且在__清空每一个$inbuf_i[l]$__之后产生一个值作为输出，同时在每个通道上产生__最多一个__待发送(_in transit_)报文（不会出现在接收方的$inbuf$中）

- 一个系统的配置(_Configuration_)是$C=\left(q_0,q_1,\dots,q_{n-1}\right)$由所有处理器的当前状态组成的，配置中的$outbuf$存放着所有待发送的报文，配置的初始状态由所有处理器的初始状态组成

- 所有在系统中发生的事情称为事件(_event_)，对于MP系统，所有的事件可以分为两种：计算(_computation event_)和传送(_delivery event_)。计算意味着某些或者某个处理器正在通过转换函数到达下一个可到达状态，而传送意味着$outbuf$中的报文转移到目标$inbuf$中

- 配置和事件交替、并且满足一定条件(_condition_)的序列称为执行(_execution_)，条件可分为两种：

  - _safety condition_: a condition that must hold in every finite prefix of the sequence $\rightarrow$ nothing bad has happened yet
  - _liveness condition_: a condition that must hold a certain number of times, possibly an infinite number of times $\rightarrow$ eventually something good happens

  满足某个特定系统所有_safety condition_的序列才能称为执行，如果一个执行满足所有_liveness condition_就称为_admissible_

- 处理器包含终止状态(_terminated state_)，并且在该状态下转换函数始终指向终止状态。当一个系统（或者算法）终止的时候意味着所有的处理器都处在终止状态，并且已经没有待发送的报文

### Asynchronous MP System

+ no fixed upper bound on how long it takes for a message to be delivered or how much time elapses between consecutive steps of a processor: algorithm independent of any particular timing parameters
+ 在传送事件，只有待发送的报文才会且只会从发送方的$outbuf$到接收方的$inbuf$
+ 在计算事件，某个处理器会经过转换函数到达下一个可到达状态，转换函数会保证该处理器在到达可到达状态的时候$inbuf$会清空
+ 调度(_schedule_)是执行中所有有效事件的序列，当本地程序是确定的(_deterministic_)，那么一个执行就是被初始配置和调度唯一确定的，记作$exec(C_0,\sigma)$
+ In the asynchronous model, an execution is _admissible_ if each processor has an infinite number of computation events and every message sent is eventually delivered $\rightarrow$ processors do not fail; It does not imply that local program must loop infinitely, but have the transition function not change the processor's state (_dummy steps_).
+ 同一个算法在一个非同步MP系统实际可能有很多不同的执行序列

### Synchronous MP System

+ 执行被划分为多个轮次(_round_)，在每个轮次中，每个处理器可以给每个邻居发送一个报文，并且这些报文被及时发送，基于这些报文进行运算的处理器都能够接受到所需报文。严格地说，执行可以被划分为互不相交(_disjoint_)的轮次，囊括了：1）每一个$outbuf$中的报文对应的发送事件；2）每一个$outbuf$被清空；3）每个处理器的计算事件。
+ In the synchronous model, an execution is _admissible_ if it is infinite.
+ 同一个算法在一个同步MP系统只有一个执行序列，只有初始条件的改变才能产生不同的执行序列

### Complexity Measures

+ Admissible execution must still be infinite, but once a processor has entered a terminated state, it stays in that state, taking _dummy steps_.
+ __报文复杂度__(_message complexity_)指的是算法的admissible execution中的报文总和的最大值（在非同步MP中存在多个等价的执行）
+ __时间复杂度__(_time complexity_)在同步MP系统中直接可以计算轮次的个数。而在非同步MP系统中，我们假设报文传播的延迟(_delay_)不超过一个单元，然后计算到终止时的运行时间，也就是延迟的总和。延迟指的是报文从在$outbuf$等待发送时间+在$inbuf$等待处理的时间。这里，引入可算时间(_timed_)执行的概念：
  + 每个事件的起始时间都可以通过一个非负实数表示
  + 整个执行的时间起点为0并且不会下降，无限执行的时间的增长不存在上限
  + 对于每个处理器都是严格单增的

## Shared memory system

+ $n$ processors + $m$ registers $\Rightarrow$ __no__ $inbuf$ or $outbuf$
+ each register has a _type_, which specifies: 1) The value that can be taken on by the register; 2) The operations that can be performed on the register; 3) The value to be returned by each operation; 4) The new value of the register resulting from each operation.
+ 整形寄存器的基本操作包括读取$\mathrm{read}\left(R,v\right)$和写入$\mathrm{write}\left(R,v\right)$
+ 一个系统的配置$C=\left(q_0,q_1,\dots,q_{n-1},r_0,r_1,\dots,r_{m-1}\right)$是由所有处理器的当前状态和所有寄存器状态组成的，配置的初始状态由所有处理器的初始状态和寄存器的初始状态组成
+ 对于SM系统，事件只有计算(_computation step_)，相当于一次性地(_atomically_)完成如下事件
  + $p_i$基于当前状态选择一个共享变量(_shared variable_)进行制定操作
  + 在该共享变量上进行指定操作
  + 根据$p_i$的状态转换函数、$p_i$原先状态和共享变量操作的返回值变更状态
+ In asynchronous shared memory systems the only requirement for _admissible_ executions is that in an infinite execution, each processor has an infinite number of computation steps.
+ 调度可以简化为寄存器下标的序列，代表着相对应的执行段(_execution segment_)。同样的，一个系统的配置也是被初始配置和调度唯一确定的，记作$exec(C_0,\sigma)$。如果一个配置$C'$可以到达(_reachable from_)另一个配置$C$，这就意味着存在一个有限的调度，使得$C'=\sigma(C)$
+ $P$是一个处理器集合，如果在配置$C$中每一个在$P$中的处理器的状态和在配置$C'$中的一样，并且$mem(C)=mem(C')$，那么两个状态是相似的(_similar_)

### Complexity Measures

+ __空间复杂度__(_space complexity_)指的是解决摸个问题需要的shared memory的个数。通常计数通过需要的共享变量(_shared variables_)的个数，或者需要的共享空间(_shared space_)
+ __时间复杂度__(_time complexity_)在SM模型中通常以最坏情况下处理器解决问题需要的步数代替，通常讨论该数是无限，有限还是有界的


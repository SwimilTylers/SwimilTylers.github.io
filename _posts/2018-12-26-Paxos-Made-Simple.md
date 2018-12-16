---
layout:     post
mathjax:  true
title:      "Distributed Algorithm Basics (IX) -  Paxos made simple"
subtitle:   "Notes on DA study"
date:       2018-12-26 23:25:00
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - Consensus
    - Paxos
    - Notes
---

> 这是分布式算法笔记的第九章，讲述经典Paxos算法。

Paxos是解决一部环境下的数据强一致性的共识算法，在工业界有着广泛的应用。最早的实现就是Google的chubby系统。目前比较著名的Paxos实现包括zookeeper、DynamoDB以及阿里的X-Paxos。Paxos算法是由Lamport大神提出的，强烈建议大家去看《Paxos Made Simple》这篇文章，Lamport讲的还是比较清楚的，层层深入。也可以结合Yale的讲义。这次笔记就主要围绕着这篇文章，做一个简单的总结。

## Paxos Protocol

Paxos是为了解决异步系统的共识问题，本质上是通过足够多的machine来对抗可能发生的failure。由于前面已经证明了纯异步共识算法的不存在性，因此Paxos在节点重启时候需要额外的信息，从而打破这个困局。《Paxos Made Simple》描述的Paxos算法可以总结为

- $N/2$-resilience
- Support: **crush** failure, delayed/ missing message
- Proposer (1), acceptor ($N$), learner (1)

整个Paxos算法主要由3种agent组成(Proposer, Acceptor and Learner)，但是实际实现的过程中可以不需要严格割裂三者的功能，在一个典型的Replica State Machine案例中，每一个服务器都可以干3个不同agent的功能。

Proposer主要的任务就是分发propose，推动整个系统达成共识。在一个系统中，存在多个proposer可能会影响progress和liveness，因此在初步的Paxos中我们只要求只有一个proposer——这个性质是由leader election算法的liveness来解决——如果出现多个proposer，可以通过类似IP协议的指数随机后退，或者是failure detector来甄别正确的唯一proposer。Proposer重启的时候，需要记住之前最新的进展$\left<n,v\right>​。$

Acceptor就是要求达成共识的对象，故障个数严格小于$N/2$。需要注意的是，本篇文章中的acceptor是machine (cpu+memory)，本身是可以做出比较复杂的动作（与disk paxos的主要区别）。Acceptor重启的时候，需要记住之前最新的进展$\left<n_{\max},v,n_v\right>$。

Learner是获知当前共识的对象，通常只需要指定一个地位特殊(distinguished)的learner子集（通常为1）作为中继节点，每次当acceptor确认了一个输出之后都会向这些distinguished learners通知，当这些learner收获到超过一半的acceptor的信息之后（达成quorum），distinguished learners通知其他平凡learner。由于learner算法不是重点，因此之后的讨论不再围绕该类agent。Learner的重启只需要求Proposer重发一轮proposal。

具体流程请参考《Paxos Made Simple》原文，这里主要罗列一些重要信息。

- Phase 1
  - $prepare\left<n\right>$: ask for promise
  - $ack\left<n,v,n_v\right>$
- Phase 2
  - $accept\left<n,v\right>$
  - $accepted\left<n,v\right>$

## Proof: Safety

- ***Validity***: $output$来源于$input[N]$

  - **P1.** Acceptor应当接受第一个收到的Proposal
  - **P1.a**  An acceptor can accept a proposal numbered $n$ iff it has not responded
    to a prepare request having a number greater than $n$.

- ***Agreement***: 共识 $v$ 一旦达成，今后就一定不会有另外一个output $\hat{v}\neq v$ 成为主流 (*chosen*)

  - **P2** [*Global* view] If a proposal with value $v$ is chosen, then every higher-numbered <u>proposal</u>
    <u>that is chosen</u> has value $v$.

  - $\Leftarrow$**P2.a** [*Acceptor's* view] If a proposal with value v is chosen, then every higher-numbered <u>proposal</u>
    <u>accepted by any acceptor</u> has value v.

  - $\Leftarrow$**P2.b** [*Proposer's* view] If a proposal with value v is chosen, then every higher-numbered <u>proposal</u>
    <u>issued by any proposer</u> has value v.

    - **I.H.** $\forall n'\in[m,n), v_m=v_{n'}$

    - **Ind.** 这里$C$和$S$都是关于acceptor的集合

      1. $m\rightarrow C,\; m+1\rightarrow C,\;\dots,\;n-1\rightarrow C\;(\mid C\mid\geq N/2)$

      2. $S\rightarrow n\;(\mid S\mid\geq N/2)$ 

      3. **[*Quorum*.]** $C\cap S\neq\varnothing\Rightarrow \max\{n_v\mid\left<n_{\max},v,n_v\right>\in S\}\geq m$
      4. 根据算法(**P2.c**)，我们得到标号为$n$的proposal对应的$v_n=v_m$

    $\Box​$

## Application

- Replicated State Machine
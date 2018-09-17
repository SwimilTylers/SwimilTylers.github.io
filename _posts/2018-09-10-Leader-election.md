---
layout:     post
title:      "Distributed Algorithm Basics (IV) - Scenario: leader election"
subtitle:   "Notes on DA study"
date:       2018-09-10 17:20:15
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - Leader election
    - Le-Lann-Chang-Roberts
    - Hirschberg-Sinclair
    - Peterson
    - Notes
---

> 这是分布式算法笔记的第四章，讲述选举问题。

选举问题(_Leader election_)就是在一群处理器中选举出中央的处理器，这种特殊的处理器在集群中只有一个，而且选举人其实可以不知道中央处理器是谁，只需要知道自己是否是中央处理器。选举问题可以变得非常复杂，但是在本章中，主要考虑在环状网络下通过MP系统进行选举。可以查阅Yale的详细表述。
本章主要举出了三种不同场景下的选举算法（Peterson算法之后补充完整），并在最后讨论了该问题的复杂度下界并给出了构造性证明，还是比较有意思的。

## Symmetry

- __Symmetry__: 节点顺序的改变不会影响整个系统的表现
- __Equivalence Relation__:  所有等价关系的进程都运行着相同的代码，并且如果$p$等价于$p'$, 那么$p$的每一个近邻都等价于$p'$的每一个近邻
- __Anonymous Ring__: 每一个进程都运行__相同__的代码的__环状__网络结构
- __Sense of Direction__: 如果一个节点能够分辨左右近邻，那么就可以通过距离最左侧的节点的距离来唯一的确定每一个节点
- __Symmetry Breaking__: 偏离初始对称配置的message-passing system。打破对称性可以通过让每个节点进行随机选择，以一定的概率打破原先的对称性配置，但是这种方法不能保证收敛到唯一一个leader；或者为每个节点设置身份标识，只需申明身份就能打破对称性。
- ___Lemma 1.1___ : A symmetric deterministic message-passing system that starts in an initial configuration in which equivalent processes have the same state has a synchronous execution in which equivalent processes continue to have the same state.

## The Le-Lann-Chang-Roberts Algorithm

__Configuration__: unidirectional ring, nodes with identities, not necessarily synchronous

__Complexity__: $O(n^2)$ messages and $O(n)$ time

```pseudocode
Class node:
	Procedure initialize(assigned_id) :
		this.leader := false
		this.id := assigned_id
		this.maxId := assigned_id
		this.sendMessageTo(content:=assigned_id, to:=this.next)
	End Procedure
    
	Procedure receiveMessage(message) :
		if message.content == this.id
			this.leader := true
		if message.content > this.maxId
			this.maxId = message.content
			this.sendMessageTo(content:=message.content, to:=this.next)
	End Procedure
End Class
```

## The Hirschberg-Sinclair Algorithm

__Configuration__: unidirectional ring, nodes with identities, not necessarily synchronous

__Complexity__: $O(n\log n)$ messages and $O(n)$ time

__Improvement__: [LRC] each process probe locally: double range when locally max, otherwise drop out

```pseudocode
Class agent:
	Field original_sender: node.pid.atype
	Field direction: {LEFT, RIGHT}
	Field hop_count: int
	Field max_seen: int
End Class

Class node:
	Procedure initialize(assigned_id) :
		this.leader := true
		this.id := assigned_id
		this.hop := 1
		this.carry := new agent(this.pid, LEFT, this.hop, this.id)
		this.sendMessageTo(
			content:=this.carry,
			to:=this.neighbor(this.carry.direction)
		)
		this.carry.direction := RIGHT
		this.sendMessageTo(
			content:=this.carry,
			to:=this.neighbor(this.carry.direction)
		)
	End Procedure
    
	Procedure receiveMessage(message) :
		if this.carry.original_sender == this.pid
			if message.content.max_seen == this.id
				this.hop := this.hop * 2
				this.carry.hop_count := this.hop
				this.carry.direction := LEFT
				this.carry := message.content
				this.sendMessageTo(
					content:=this.carry,
					to:=this.neighbor(this.carry.direction)
				)
				this.carry.direction := RIGHT
				this.sendMessageTo(
					content:=this.carry,
					to:=this.neighbor(this.carry.direction)
				)
			else
				this.leader := false
				this.carry := message.content
		else
			if message.content.hop_count != 0
				if message.content.max_seen > this.carry.max_seen
					this.leader := false
					this.carry := message.content
				else if message.content.max_seen < this.carry.max_seen
					message.content.max_seen := this.carry.max_seen
					--message.content.hop_count
			this.sendMessageTo(content:=message.content, to:=this.next)
	End Procedure
End Class
```

## The Peterson's Algorithm

__Configuration__: unidirectional ring, nodes with identities, asynchronous

__Complexity__: $O(n\log n)$ messages and $O(n)$ time

## Randomized $O(n\log n)$-message Algorithm

__Argument__: the $i$-th largest id only propagates an expected $n/i$ hops -> $O(nH_n)=O(n\log n)$ hops in total.

__Improvement__: Run LCR where each id is constructed by prepending a long random bit-string to the real id. -> something close to a random permutation on the constructed id's

__Backwards__: 相较于Peterson算法，随机化算法需要提前知道$n$的大小，从而确定随机标签的范围，此外其message的字长比较大

## Lower bounds for message complexity

### Asynchronous + uniform: $\Omega (n\log n)$

__BASIC IDEA__: 

- recursively, by first constructing two bad $(n/2)$-process executions and pasting them together in a way that generates many extra messages
- learning the identity of the process with the smallest id is a slightly stronger problem than merely leader election, with only additional $O(2n)$ message complexity

__ADVERSARY__:

- _open executions_: partial executions where no message is delivered across some edge (called open edge)
- if no message is delivered across this edge, the processes can't tell if there is really a single edge there or some enormous unexplored fragment of a much larger ring

___Proof___: 

1. $n=2$时，一定有人最终要进行通讯，因此$T(2)\geq 1$
2. for larger $n$
   + we have two open executions $\sigma_1, \sigma_2$on $n/2$ processes that each send at least $T(n/2)$ messages
   + break the open edges on both executions and paste the resulting line together to get a ring of size $n$, then we get a combined schedule $\sigma_1\sigma_2$ with at least $2T(n/2)$ messages. Note that in the combined schedule $\sigma_1\sigma_2$ no messages are passed between the two sides, so the processes continue to behave as they are separated. 
   + Denote $e$ and $e'$ as the edges that we used to paste the two ring together. Extend $\sigma_1\sigma_2$ by the longest suffix $\sigma_3$.  Since $\sigma_3$ is as long as possible, after $\sigma_3$ there are no messages waiting to be delivered across any of $e$ and $e'$ and all processes are quiescent.
   + Consider some complete execution $\sigma_1\sigma_2\sigma_3\sigma_4$ in which all messages delayed on both $e$ and $e'$ are delivered.
   + Now consider the processes in the half of the ring with the larger minimum id. Partition the $n/2$ processes o4en the this side into two groups based on whether the first message they get is triggered by opening $4$ or $e'$.
   + Since the losing side need at least an additional $n/2-2$ messages (excluding the two processes connection $e$ and $e'$) to learn to minimum id of its counterpart, one of the two group divided on the last step must contain at least $\frac{1}{2}(n/2-2)$ processes who receive new messages, meaning that there is an open execution $\sigma_2\sigma_2\sigma_3\sigma_4'$ in which we open up only one edge and still get and additional $\Theta(n)$ messages.
   + Thus, we form an open execution on $n$ processes. $T(n) \geq 2T(n/2) + \Theta (n)$ 

### Synchronous + comparison-based:  $\Omega (n\log n)$

__BASIC IDEA__:

+ _order-equivalent_: for two fragment $i\dots i+k$ and $j\dots j+k$, $\forall b = 0\dots k,\,\mathrm{id}_{i+a}>\mathrm{id}_{i+b} \;\Leftrightarrow\;\mathrm{id}_{j+a}>\mathrm{id}_{j+b}$
+ _similar_: executions of $p_1$ and $p_2$ is similar if both processes send messages in the same direction(s) in the same rounds and both processes declare themselves leader (or not) at the same round.
+ _full-information protocol_: each message is replaced by the ID and a complete history of the sending process, including all messages it has ever received.
+ _comparison-based_: the algorithm can only copy IDs and test for $<$. The state of such an algorithm is modeled by 1) some non-ID state + a big bag of IDs, 2) messages have a pile of IDs attached to them, etc.. Two states are equivalent under some mapping of IDs if you can translate the first to the second by running all IDs through the mapping.
+ _active round_: a round in which ate least 1 message is sent.
+ actions of $i$ after $k$ active rounds depends only on the order-equivalence class of ids in the $k$-neighborhood of $i$, up to an order-equivalent mapping of ids.
+ If we have an order of ids with a lot of order-equivalent $k$-neighborhoods, then after $k$ active rounds if one process sends a message, so do a lot of other ones.
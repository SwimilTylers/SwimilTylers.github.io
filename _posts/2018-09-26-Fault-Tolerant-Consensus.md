---
layout:     post
mathjax:  true
title:      "Distributed Algorithm Basics (VIII) -  Fault-Tolerant Consensus"
subtitle:   "Notes on DA study"
date:       2018-09-26 19:57:49
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - Simulations
    - Consensus
    - Message passing system
    - Synchronous
    - Notes
---

> 这是分布式算法笔记的第八章，讲述了节点故障情况下同步系统中的共识算法。

共识算法是分布式算法中非常重要的算法，涉及的方面非常广泛，之前的协调进攻算法可以当作在 _link failure_ 情况下的共识算法问题。但是由于主要参考Attiya教材，因此本章讨论的仅限于节点故障情况下的。对于非同步系统中的共识问题，将在之后的笔记中单独阐述。

+ 共识算法考虑两种情况下的节点故障
  + 停机故障(_crash failure_)：指的是由于处理器故障，导致不能发送报文
  + 拜占庭故障(_Byzantine failure_)：指的是存在一些处理器，他发送的报文和当前的状态无关，这些处理器可以是抽风的，也可能是恶意的，甚至可以伪装成停机故障
+ 本次问题中的网络结构是完全图(_complete graph_)，每一个节点都与其他节点为邻
+ 只有在同步系统中才能完美地解决共识问题，在非同步的系统中共识算法并不存在，查看[Simulations笔记](Simulations.md)

## Synchronous System with crash failure

### Formal model

+ 共识算法中，对于原先的同步MP系统进行相应的补充，达到形式化停机(_crash failure_)描述的目的。在本次问题中。一个系统描述的重要特性是$f$-抗性(_resilient_)，指的是一个系统中最多发生停机的处理器的个数。
+ 共识算法中，追加了对于执行的描述。在可靠(_reliable_)的同步MP系统中，一系列的轮次组成了执行，每一个轮次涵盖了报文传送和单步计算。在$f$-_resilient_系统中
  + 存在一个由故障(_faulty_)处理器组成的子集$F​$，最大元素个数是$f​$。这个子集在每一步的执行中都可能是不同的，因此无法提前知道哪些处理器出现故障
  + 每一个轮次中，对于不在$F$中的处理器，都**有且只有一次**计算事件；而对于在$F$中的处理器，**最多有一次**计算事件，并且本轮次没有计算事件的处理器往后都不再会有计算事件
  + 在某个故障处理器最后一次进行计算事件的那个轮次之后，该处理器待发送报文中有随机一部份会被发送，这个特性将会增加停机模型的难度，这种随机性的效应要求共识算法更加健壮
+ 在上述的系统中，每个处理器都存在两个特殊的组件(_component_)，一个是作为输入的$x_i$，另一个是作为输出的$y_i$（也被称为主意_decision_）。开始的时候，$x_i$的取值是良序集(_well-ordered set_)的某个可能的取值，一般来说是事先指定的并且不会更改，而$y_i$的取值是待定的。值得注意的是，$y_i$只能取一次值(_irreversible_)。共识问题应当满足一下的条件：
  + __Termination__： In every admissible execution, $y_i$ is eventually assigned a value, for every non-faulty processor $p_i$
  + __Agreement__：In every execution, if $y_i$ and $y_j$ are assigned, then $y_i = y_j$ (no conflicts)
  + __Validity__：if all the processors have the same input, then any value decided upon must be that common input. In another word, the value of $y$ **must not be a trivial or static one**, it has something to do with the inputs $x_1, x_2,\dots$, and can respond to the incoming situation.

### Algorithm

整个算法的过程中，每个处理器都在维护一个集合，涵盖了这个处理器所知道存在着的值，初始的时候是自己的输入(_input_)。在算法迭代的过程中，处理器向周围人发送之前没有发送过的集合元素（即`v has not been sent yet`），并且利用将来自其他处理器的集合$S_j$更新本地维护的集合。当到最后一轮的时候，选取集合中最小的元素作为输出。

```pseudocode
Initially V = {x}

for(k = 1; k <= f+1; ++k):
	send [(v has not been sent yet) for v in V] to this.neighbors
	receive [(S[j] from p_j) for j in this.neighbors]
	V = V.conjunction(S[0], S[1], ..., S[this.neighbor.length-1])
	if (k == f+1)
		y = min(V)
```

由于失效节点在最后一轮计算事件之后，会随机的发送部分的待发送报文，因此在crash failure条件下共识算法需要不少于$f+1$轮次才能完成。

#### 有效性

+ _Termination_：显然，只有$f+1$轮
+ _Agreement_：在每一个执行的第 $f+1$轮次结束的时候，对于每一对非故障的处理器来说$V_i=V_j$。下面证明上述引理：设第$r$轮的时候$x$第一次加入$V_i$，如果$x$初始化的时候就已经在$V_i$中，那么$x=0$
  + 当$r\leq f$时，显然在第$r+1\leq f+1$轮的时候能够$V_i=V_j$
  + 当$r=f+1$的时候，不妨设$p_j$是在$f+1$的时候第一次接收到$x$的非故障处理器，因此$x$到达$p_j$一定经过一条长度为$f+1$的传送链$p_{i_1},\dots,p_{i_{f+1}},p_j$。因为每个处理器不会重复发送同一个输入，因此$p_j$之前$x$一定经过$f+1$个不同的处理器，因此这其中至少有一个是故障的处理器
+ _Validity_：显然，每一个正常处理器的输出都是某一个处理器的输入

## Synchronous system with Byzantine failure

### Formal model

- 共识算法中，对于原先的同步MP系统进行相应的补充，达到形式化停机(_crash failure_)描述的目的。在本次问题中。一个系统描述的重要特性是$f$-抗性(_resilient_)，指的是一个系统中最多发生停机的处理器的个数。
- 共识算法中，追加了对于执行的描述。在可靠(_reliable_)的同步MP系统中，一系列的轮次组成了执行，每一个轮次涵盖了报文传送和单步计算。在$f$-_resilient_系统中
  - 存在一个由故障(_faulty_)处理器组成的子集$F$，最大元素个数是$f$。这个子集在每一步的执行中都可能是不同的，因此无法提前知道哪些处理器出现故障
  - 每对于在$F$中的处理器，每次计算事件之后的处理器状态将不再限制(_constrain_)报文的内容，也就是说，这个故障的处理器表现得乖张(_arbitrarily_)，甚至是恶意的(_maliciously_)。当然，这个故障的处理器也可以伪装成(_mimic_)停机故障
- 在上述的系统中，每个处理器都存在两个特殊的组件(_component_)，一个是作为输入的$x_i$，另一个是作为输出的$y_i$（也被称为主意_decision_）。开始的时候，$x_i$的取值是良序集(_well-ordered set_)的某个可能的取值，而$y_i$的取值是待定的。值得注意的是，$y_i$只能取一次值(_irreversible_)。共识问题应当满足一下的条件：
  - __Termination__： In every admissible execution, $y_i$ is eventually assigned a value, for every non-faulty processor $p_i$
  - __Agreement__：In every execution, if $y_i$ and $y_j$ are assigned, then $y_i = y_j$ (no conflicts)
  - __Validity__：if all the processors have the same input, then any value decided upon must be that common input

### Exponential Algorithm

整个算法分为两个阶段，第一阶段是收集信息，第二阶段是根据收集到的信息，逐层归纳。因此，我们通过构建前缀树来描述整个算法的过程。抗性为$n\geq 3f+1$，是根据对于某个报文所经过的处理器中非故障处理器占主要部分($n-f\geq 2f+1$)产生的。

#### 前缀树的构建

+ 每个处理器都在本地维护一棵前缀树
+ 深度为$f+1$，并且从树根到叶子都要经过$f+2$个节点
+ 每个节点都有标签串，树根是空串$\empty$。记中间节点$v$
  + $v$的标签$i_1,i_2,\dots,i_r$，$r$是$v$所在的深度，该节点上的取值来自于处理器$i_r$。
  + 对于每一个**未在**$i_1,i_2,\dots,i_r$中出现的处理器$j$，节点$v$都有对应的子节点$i_1,i_2,\dots,i_r, j$【没有一个处理器会在标签串中出现两次】

####Gathering stage: exactly $f+1$ rounds

+ 开始阶段
  + 每个处理器向所有人（包括自己）发送初始输入$x$，并将初始输入保存在根节点
  + 当收到来自$p_j$的报文$x_j$，将其保存在标签为$p_j$的节点上
  + 如果从$p_j$收到的$x$不合法(_not legitimate_)或者是没有收到来自$p_j$的报文，给标签为$p_j$的节点写上$v_\perp$
  + 将$x$打上标签$p_j$，并发送给除了$p_j$的其他处理器
+ 在第$r$轮的时候
  + 处理器$p_i$接受到来自$p_j$的报文$x$，其标签为$i_1,i_2,\dots,i_r$。将$x$存储在标签为$i_1,i_2,\dots,i_r,j$的节点上，记该节点的值为$tree_i\left(i_1,i_2,\dots,i_r,j\right)$
  + 报文不合法或者未接受的，对应节点写上$v_\perp$
  + 将$x$打上标签$i_1,i_2,\dots,i_r,j$，并发送给标签上未出现过的处理器。
+ 在第$f+1$轮的时候，等到本地维护的前缀树每个节点都有赋值时，进入归纳阶段

#### Reduction stage

归纳阶段是建立在之前建立好的前缀树上，记在以$\pi$为根节点的子树上的归纳为$\mathrm{resolve_i}(\pi)$，而处理器$y_i$的取值就是对于根节点的归纳$\mathrm{resolve_i}(\empty)$。归纳函数$\mathrm{resolve}$的递归定义是：

1. 若$\pi$为叶子节点，那么$\mathrm{resolve}(\pi)=tree(\pi)$
2. $\mathrm{resolve}(\pi)$取$\pi$的所有子节点归纳结果中最为频繁的一个，如果没有就取$v_\perp$

#### 复杂度

+ 时间复杂度$O(f+1)$
+ 报文复杂度$O\big(n^2(f+1)\big)$，但是报文的长度指数级上升，最长的报文包含的标签长度可以达到$n(n-1)(n-2)\dots(n-(f+1))=\Theta(n^{f+2})$

#### *有效性前引理

1. __Lemma 5.9__ 对于每个非故障的处理器$p_i$的前缀树，树上每个标记为$\pi=\pi'j$且$p_j$为非故障处理器的节点，该节点上的归纳都满足$\mathrm{resolve_i}(\pi)=tree_j(\pi')$

   1. 如果$\pi$是叶子节点，那么$\mathrm{resolve_i}(\pi)=tree_i(\pi)$。由于$tree_j(\pi')$的值是上一轮由$p_j$发送到$p_i$的，并且根据算法被$p_i$保存到$tree_i(\pi'j)=tree_i(\pi)$的，因此，如果$p_j$不是故障节点，那么就满足$\mathrm{resolve_i}(\pi)=tree_i(\pi)=tree_j(\pi')$（对于拜占庭式故障节点，由$p_j$发送的报文可能是任何值，因此存在$tree_i(\pi)\neq tree_j(\pi')$的情况），得证

   2. 如果$\pi$是中间节点，我们假设在它任意的非故障子节点$\pi k$中（即$p_k$为非故障处理器），满足$\mathrm{resolve_i}(\pi k)=tree_k(\pi)$。

      + 因为$p_j$也是非故障的，因此$p_j$能够准确的将$tree_j(\pi')$的值发送给$p_k$，因此$tree_k(\pi'k)=tree_j(\pi')$，因此，对于每个非故障节点$p_k$，都能满足$\mathrm{resolve_i}(\pi k)=tree_j(\pi')$

      + 它的深度不会超过$f$。**每个节点的度数(*degree*)与深度有关**，满足$d=n-r$。因此，$\pi$的度数至少为$n-f\geq 2f+1$，因此**非故障子节点总能占到大多数**
      + 根据定义$\mathrm{resolve_i}(\pi)$取子节点归纳结果中的大多数。由于正常的节点归纳结果都为$tree_j(\pi')$，并且正常节点占子节点中的大多数，因此$\mathrm{resolve_i}(\pi)=tree_j(\pi')$

2. __Definition__ 

   1. 公共节点(_common node_)：对于任意两个非故障处理器$p_i,p_j$，有$\mathrm{resolve_i}(\pi)=\mathrm{resolve_j}(\pi)$，那么$\pi$就是个公共节点
   2. 公共阵线(_common frontier_)：在子树中，如果从根到叶的每一个路径上都存在一个公共节点，那么这棵子树局势公共阵线

3. __Lemma 5.10__ 如果以$\pi$为根节点的子树是个公共阵线，那么$\pi$就是个公共节点

   1. 如果$\pi$是叶子节点，显然
   2. 如果$\pi$是中间节点，根据假设$\pi$的所有子节点都满足这样的性质。反设如果$\pi$-子树是公共阵线但$\pi$不是公共节点，那么根据公共阵线定义可以推出，所有子节点的子树都是公共阵线，因此所有的子节点都是公共节点。根据归纳定义，$\mathrm{resolve}_i(\pi)$取自最常见子归纳；根据公共节点定义，这些子归纳在任何正常处理器上都相同，因此在任意正常处理器上的$\pi$归纳也相同，得证

####有效性

+ _Termination_: 显然$f+1$轮结束
+ _Agreement_: 前缀树表示了报文传播路径，而每一条从根节点到叶节点的路径长度都是$f+1$，意味报文经过了$f+1$个不同的处理器，那么至少有一个处理器一定是正常的，因此在这个正常的处理器对应的节点应当是公共节点（在同一轮的发送阶段，正常处理器对所有处理器发送的报文应该是一致的），那么意味着这整棵都是公共阵线，因此对于所有正常的处理器，做出相同的决定
+ _Validity_: 如果所有处理器的输入都是$v$，那么对于任意一个处理器$p_i$，由于$y_i=\mathrm{resolve_i}(\empty)$，因此$y_i$应当是子节点归纳中最频繁的一个。而对于任意的子节点$j$，$\mathrm{resolve_i}(j)=tree_j(\empty)=x_j=v$，这样根据定义$p_i$的决定就是$v$，满足Validity

### Polynomial Algorithm

该算法使用多项式长度的报文，但是带来了更多的轮数和更弱的抗性(_resilience_)，为$n\geq4f+1$，是根据收到的报文始终占所有处理器中的大部分($n-f>n/2+f$)产生的。每个处理器都会维护一个本地的$pref$数组记录当前最受欢迎的选择。该算法可以将运算的过程分为$f+1$个阶段(_phase_)，每个阶段分为有两轮：

+ 第一轮是选择(_preference_)阶段，选择出现最多次数的选择(`<maj,m_count>`)，如果找不到，就拿$v_\perp$标记
+ 第二个阶段就要判断下一轮的选择，如果接受到的最多次选项出现的频率超过$n/2+f$个，那么就将第一轮中最多出现次数的选择(`this.maj`)作为下一轮的选择(`pref[i]`)，否则就将来自主宰(_king_)处理器中最多出现的选择(`king_maj`)作为自己的选择。

```pseudocode
Initially pref[i] = x and pref[j] for any j != i

in the round 2k-1 (1<=k<=f+1):
	send pref[i] to this.neighbors
	receive v[j] from other proccessors and store at pref[j]
	from pref[0...n-1] select the majority <maj, m_count> except <nil, 0>

in the round 2k (1<=k<=f+1):
	if i == k
		send this.maj to this.neighbors
	receive king_maj from this.kings[k].pid
	if this.m_count > n/2 + f
		pref[i] = this.maj
	else
		pref[i] = king_maj
	if k == f+1
		y = pref[i]
```

#### *有效性前引理

1. _persistence of agreement_: if all non-faulty processors prefer $v$ at the beginning of phase $k$, then they all prefer $v$ at the end of phase $k$, for all $k:\;1\leq k\leq f+1$
   + 在第一阶段，每个处理器（加上自己发送的）会接受到至少$n-f$个相同的报文$v$
   + 在第二阶段，由于$n\geq 4f$，因此$n-f\geq n/2+f$，因此会坚持第一阶段中的众数作为`pref[i]`
2. **Lemma 5.13**: Let $g$ be a phase whose king $p_g$ is non-faulty. Then all non-faulty processors finish phase $g$ with the same preference
   + 假设所有的非故障节点都不能满足`this.m_count>n/2+f`的条件，那么自然的都会选择`king_maj`
   + 假设一些非故障节点满足`this.m_count>n/2+f`的条件，比如说$p_i$，那么说明$p_i$接收到的报文$v$的个数超过$n/2+f$，那么说明**其他处理器接收到至少有$n/2$个该报文$v$，包括*king***，因此`king_maj`$=v$，从而$p_i$的选择也和*king*相同

#### 有效性

+ _Termination_: 显然$2(f+1)$轮结束
+ _Agreement_: 整个系统中最多有$f$个故障节点，而在算法的$f+1$轮中，总有一个非故障节点能够成为*king*。根据**Lemma 5.13**引理，所有的非故障节点都能得到相同的$v$，再根据*persistence of agreement*这些相同的$v$将成为最终的一致的选择
+ _Validity_: 借助*persistence of agreement*性质，我们可以知道，如果所有的非故障处理器拥有相同的初始$v$，那么整个算法的过程中他们会一直选择$v$，最终在$f+1$阶段选择$v$作为输出
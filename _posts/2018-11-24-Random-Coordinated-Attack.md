---
layout:     post
mathjax:  true
title:      "Distributed Algorithm Basics (VII) - Scenario: Coordinated Attack"
subtitle:   "Notes on DA study"
date:       2018-11-24 22:01:49
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - Coordinated Attack
    - TODO
    - Notes
---

> 这是分布式算法笔记的第七章，讲述了协调进攻算法。

## Requirements

1. __Randomized Agreement__: For any adversary $A$, the probability that some process decides 0 and some other process decides 1 given $A$ is at most $\epsilon$
2. __Validity__: If all processes have the same input $x$, and no messages are lost, all processes produce output $x$
3. __Termination__: All processes terminate in abounded number of rounds

## Algorithm

Given $\epsilon = 1/r$  (lower bound: $\Pr[\mathrm{Disagreement}]=\frac{1}{(1+r)}$), Complete Network ( or SCN with $r\geq D$ )

1. 追踪信息层次步骤

   1. 每一个进程保存一个由$(input, level)$组成的 __向量__ ，该进程本身的分量的初始状态为$(input,0)$，其他分量的状态是$(\perp,-1)$ 

   2. 在每一轮中，每一个进程将整个向量告诉其他所有进程

   3. 当接收到别人的向量的时候

      ```pseudocode
      Procedure update (my_vector, rec_vector)
      	
      	integer min_level =-1 
      	
      	For my_pair, rec_pair in my_vector, rec_vector
      		If rec_pair.process  is not my_vector.self_pair.process
      			my_pair.input = rec_pair.input 
      			my_pair.level = max(my_pair.level, rec_pair.level)
      			min_level = min(min_level, my_pair)
      	
      	my_vector.self_pair.level = min_level +1 
      
      End Procedure 
      ```

2. 输出步骤

   1. Process-1选择从$[1,r]$中随机选择一个$key$
   2. 将$key$传播，从而每一个$level_i[1]\geq0$都能知道这个$key$
   3. 对于任意一个进程来说，当且仅当它收到该$key$，并且他自己的$self.level\geq key.level$ 并且输入满足$self.input=[1\,1\,\dots\,1]$的时候，该进程在本轮的输出就是$1$

3. 重复上述两个步骤，迭代$r$轮

## Proof

1. __Termination__ : 显然

2. __Validity__ :

   1. 如果所有的输入都是$0$，那么没有进程的输入满足$self.input=[1\,1\,\dots\,1]$，因此没有进程会输出$1$
   2. 如果所有的输入都是$1$并且没有信息丢失 ，那么对于每一个进程来说，在第$k$轮的时候都满足$self.level=k$，所以在输出步骤的时候每一个进程都能够满足$self.level=k\geq key.level$，因此每一个进程都会输出$1$

3. __Randomized Agreement__

   1. 首先证明引理

      > 记$level_i[k]^t$表示进程Process-$i$ 中，记录的进程Process-$k$的在第$t$轮输入，对于任意的$i\,,j\,,k\,,t$满足
      >
      > 1. $level_i[j]^t \geq level_j[j]^{t-1}$
      > 2. $|\,level_t[k]^t-level_j[k]^t\,|\leq1$

   2. ​
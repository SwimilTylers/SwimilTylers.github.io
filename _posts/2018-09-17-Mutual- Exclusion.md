---
layout:     post
title:      "Distributed Algorithm Basics (VI) - Scenario: Mutual Exclusion"
subtitle:   "Notes on DA study"
date:       2018-09-17 21:18:00
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - Mutual Exclusion
    - Notes
---

> 这是分布式算法笔记的第六章，讲述了建立在共享内存模型下的互斥算法。

互斥算法的重要性不言而喻，因此本章的内容也比较繁杂，从问题阐述到算法过程，最后讨论互斥问题的复杂度下界。有不理解的地方可以参考Attiya的原文。

+ 互斥问题的程序可以分为4个阶段
  + 尝试阶段(_Entry_)：代码尝试进入critical section
  + 保护阶段(_Critical_)：代码的执行流与其他并发操作隔绝开来，进行保护
  + 退出阶段(_Exit_)：代码离开critical section
  + 剩余阶段(_Remainder_)：剩余的代码部分

+ 互斥问题应当满足下面的需求
  + __Mutual exclusion__: at most one process is in the critical state at a time
  + __No deadlock__: if there is at least one process is in a trying state, then eventually some process enters a critical state
  + __No lockout (lockout freedom)__: if a processor wishes to enter the critical section, then it will eventually succeed as long as no processor stays in the critical section forever (in another words, no _starvation_).

+ 在$n$个处理器的SM系统中，满足Lockout freedom至少需要$\sqrt{n}$个不同的寄存器状态。如果一个算法满足，那么至少需要$n$个不同的寄存器状态
+ 互斥算法如果能满足在每一个执行中，没有一个处理器能够在其他处理器仍然在尝试阶段的时候进入critical section超过$k$次，那么该算法能够保证$k$-bounded waiting
+ 如果所有的处理器都进入了剩余阶段，那么互斥算法的配置就陷入沉寂(_quiescent_)
+ 如果一个互斥算法能保证no deadlock和$k$-bounded waiting特性，那么这个算法的共享内存需要至少$n$个不同的状态 ：
  + 记$C$是该算法的初始配置，目前该算法是沉寂的。让$\tau_0 '$是一个无限的$p_0$-only的调度，由于$exec(C,\tau_0')$是满足admissible的特性，那么存在一个$\tau_0'$的有穷前缀$\tau_0$使得$p_0$处在保护阶段$C_0=\tau_0\left(C\right)$，相似的为其他处理器构建$p_i$-only的调度使得$p_i$处在尝试阶段$C_i=\tau_i\left(C_{i-1}\right)$得到$C_{n-1}=\tau_0\tau_1\dots\tau_{n-1}\left(C\right)$
  + 我们假设共享内存只需要严格小于$n$个不同的状态，那么根据鸽笼原理，存在两个不同的状态$C_i,C_j,\,0\leq i<j\leq n-1$的共享内存的状态是完全一致的，也就是说对于没有在$\tau_{i+1},\dots,\tau_j$执行中进行操作的处理器$p_0,\dots,p_i$（$\tau_i$是$p_i$-only调度）来说，$C_i\sim C_j$
  + 在$C_i$施加一个调度$\rho'$，该调度只包含了$p_0$到$p_i$的无限步操作。因为$exec(C, \tau_0\tau_1\dots\tau_i\rho')$本身是admissible，并由无死锁特性推出，某一个处理器$p_\ell,\,0\leq\ell\leq i$曾经进入critical section无数次
  + 我们选择$\rho'$的一个有穷前缀$\rho$，在$exec(C, \tau_0\tau_1\dots\tau_i\rho)$中$p_\ell$进入critical section区域$k+1$次（注意，此时仍然满足$k$-bounded waiting）。由于对于$p_0,p_1,\dots p_i$而言$C_i\sim C_j$，并且$\rho$本身就是$\{p_0,\dots p_i\}$-only的，因此$p_\ell$在$exec(C_j,\rho)$仍然是进入了$k+1$次
  + 在$C_j$施加一个调度$\sigma$，该调度只包含了$p_0$到$p_j$的无限步操作。因为$p_0$到$p_j$进行了无限步步骤而剩余的处理器依然在剩余阶段，因此$exec(C, \tau_0\tau_1\dots\tau_i\rho\sigma)$本身是admissible。但是在执行片段$\rho$中，$p_j$一直在尝试阶段而$p_\ell$已经进入critical section中$k+1$次，违反了$k$-bounded waiting的特性。因此，共享内存至少存在$n$个不同的状态

## Mutual exclusion using strong primitives

### Stronger Atomic Operations

+ __read-modify-write__ register: RMW operation computes a new value based on the old one and write the new one back as a single atomic operation, usually returning the old value to the caller as well.

+ __test-and-set__ operation: set the value (to $1$) in the register and return the old one ($0$ or $1$)

+ __reset__ operation: set the bit back to zero

### Test-and-Set Registers

```c
// without lockout-freedom

while (true){
    // trying
    while (test_and_set(lock) == 1);
    
    // critical
    
    // exiting
    reset(lock);
    
    // remainder
}
```

### Read-Modify-Write Registers

上述的算法无法保证lockout-free，为了确保这种特性，我们需要一个可以支持原则操作的共享队列(_atomic shared queue_)。再尝试阶段的时候，每个处理器将自己加入(_enqueue_)这个共享队列的尾部。对于这个队列来说，每次去取其头部的处理器，允许他进入保护阶段，而对于其他处理器来说就需要不停的访问队列是否轮到了他（这个方式叫做_spinning_）当一个进入保护阶段的处理器退出的时候，队列也随之弹出(_dequeue_)

```c
/*
* with lockout-freedom, 
* using an atomic shared queue Q 
* with atomic operations: enq, head & deq
*/

while (true){
    // trying
    enq(Q, my_id);
    while (head(Q) != my_id);
    
    // critical
    
    // exiting
    deq(Q);
    
    // remainder
}
```

RMW寄存器保存两个值，_first_和_last_， 近队列的时候增加_last_，选取进程的时候增加_first_，而进程自己检查当前队列头是否相同于近队列时候拿到的ticket。当_first_等于_last_的时候，整个critical section control暂停直到新的进程需要进入。$\mathrm{enq}$和$\mathrm{deq}$操作都是通过RWM寄存器操作，$\mathrm{head}$操作是通过普通读(_Read_)实现。

#### 复杂度

假设系统中有$n$个处理器，则需要$\log n$bit表示，因此空间复杂度为$O(2\log n)$

There is an improvement in reducing space complexity of the shared queue, that is we can hand out numerical tickets to each process and have the process take responsibility for remembering where its place in line is.

#### 有效性

+ _mutual exclusion_：只有队列的头部的处理器能够进入critical section，并且他一直停留在队列的头部直到他离开critical section，因此防止其他处理器进入critical section
+ _no deadlock_ and _no lockout_：先进先出(FIFO)的进队顺序加上处理器不会用就呆在critical section的假设，因此不会有人无法进入critical section，避免了了饥饿(_starvation_)，继而避免了死锁

### Local Spinning RMW Registers

之前的算法需要一直访问 __单个__ 变量来确定是否能够进入critical section。这种方法叫做 _spinning_ ，但是这种方式增加了访问shared memory的时间消耗，因此，本算法让不同的处理器去 _spinning_ 访问 __不同__ 的变量，这个算法需要的空间开销是$O(n)$，使得$n$个处理器能够同时 _spinning_ 减少时间开销

```pseudocode
Initially _Last_ = 0, _Flags_[0] = __has_lock__, _Flag_[i] = __must_wait__ for other i

<Entry>:
	this.place = rmw(_Last_, (_Last_+1) % n)
	wait until (_Flags_[this.place] == __has_lock__)
	_Flags_[this.place] = __must_wait__
<Critical Section>:
<Exit>:
	_Flags_[(this.place+1) % n] = __has_lock__
<Remainder>:
```

该算法保持如下不变量：

1. 最多只有一个元素保持`__has_lock__`状态
2. 在进入critical section的时候，没有元素是保持`__has_lock__`状态
3. 如果`_Lock_[k]==__has_lock__`，那么有整整`(k-_Last_-1) % n`个处理器在尝试状态(_entry section_)，每一个都在盯着(_spinning_)不同的变量

## Mutual exclusion using only atomic registers

### the Bakery Algorithm

主要的想法在于，想要进入critical section的处理器持有一个比当前所有在排队的处理器更大的ticket，如果发现自己是最小的ticket的时候才进入。这里，我们约定当一个进程不在等待队列的时候，持有的ticket设为0。`_Number_`标识对应处理器持有的ticket，`_Choosing_`表示处理器是否正在选择ticket。

处理器$p_i$在选择自己的数字的时候，需要查看所有的处理器，找到当前最大的ticket（查找过程的时候`_Choosing_[i] = true`），持有该数字更大的数作为自己的ticket写入`_Number_[i]`。但是在寻找最大数的时候，可能会出现多个处理器获得相同的ticket，这样`_Number_[i]`就有可能相同。为了打破这样的情况，我们加入处理器标号作为处理器的凭据对(_ticket pair_)

```pseudocode
Initially _Number_[i] = 0 and _Choosing_[i] = false for all i

<Entry>:
	_Choosing_[i] = true
	_Number_[i] = max(_Number_[0], ..., _Number_[n-1]) + 1 // read, then write
	_Choosing_[i] = false
	for j in range(0, n)
		wait until _Choosing_[j] = false
		wait until _Number_[j] = 0 || (_Number_[j], j) > (_Number_[i], i)
		
<Critical Section>:
<Exit>:
	_Number_[i] = 0
<Remainder>:
```

这个算法的问题在于，除非所有的处理器都能稳定在剩余阶段，ticket的数值可能会无限的增长，超出实际存储类型的表达上限

#### 有效性

+ _mutual exclusion_：
  + 当处理器$p_i$进入critical section，对于$k\neq i,Number[k]\neq 0$满足$\left(Number[k],k\right)>\left(Number[i],i\right)$
  + 当$p_i$进入critical section的时候，$Number[i]>0$
  + 如果有两个处理器$p_i$和$p_j$同时进入critical section，那么$Number[i]\neq0\and Number[j]\neq0$，并且$\left(Number[j],j\right)>\left(Number[i],i\right)\and \left(Number[k],k\right)<\left(Number[i],i\right)$矛盾，因此不会同时有两个处理器进入critical section
+ _no deadlock_ and _no lockout_：反设存在一个处理器$p_i$，其ticket是饥饿处理器中最小的。由于选取ticket的过程不会出现阻隔，因此在$p_i$之后获得的ticket都比$p_i$的大，因此这些处理器不会早于$p_i$进入critical section。当所有比$\mathrm{ticket}(p_i)$小的处理器进入critical section（这些处理器并未饥饿）并且退出，这个时候$p_i$就能够进入critical section，这与$p_i$出现饥饿矛盾

### Bounded Mutual Exclusion Algorithm for $2$ processors

首先是一个简单的**双处理器**互斥算法，保证了_mutual exclusion_和_no deadlock_特性。但是由于该算法$p_0$的优先级高于$p_1$，因此可能造成饥饿。

```pseudocode
Initially _Want_[0] and _Want_[1] are 0

// p_0 code
<Entry>:
_Want_[0] = 1
wait until (_Want_[1] == 0)
<Critical Section>:
<Exit>:
_Want_[0] = 0
<Remainder>:

// p_1 code
<Entry>:
do
	_Want_[1] = 0
	wait until (_Want_[0] == 0)
	_Want_[1] = 1
while (_Want_[0] == 1)

<Critical Section>:
<Exit>:
_Want_[1] = 0
<Remainder>:
```

可以看到在进入Critical Section的时候，`_Want_[i] == 1`。上述的算法不能保证_no lockout_的特性，我们加入一个共享变量`Priority`来决定当前情况下哪个处理器具有优先权，具有优先权的处理器处理流类似于上述算法中的$p_0$，而没有优先权的处理器则表现类似$p_1$。这样，我们就能防止出现_lockout_的情况。

```pseudocode
// Peterson's tournament algorithm

Initially _Want_[0], _Want_[1] and _Priority_ are all 0

// code for p0
<Entry>:
_Want_[0] = 0
wait until (_Want_[1] == 0 || _Priority_ == 0)
_Want_[0] = 1
if (_Priority_ == 1)
	if (_Want_[1] == 1)
		goto <Entry>
else
	wait until (_Want_[1] == 0)
<Critical Section>:
<Exit>:
	_Priority_ = 1
	_Want_[0] = 0
<Remainder>:

// code for p1
<Entry>:
_Want_[1] = 0
wait until (_Want_[0] == 0 || _Priority_ == 1)
_Want_[1] = 1
if (_Priority_ == 0)
	if (_Want_[0] == 1)
		goto <Entry>
else
	wait until (_Want_[0] == 0)
<Critical Section>:
<Exit>:
	_Priority_ = 0
	_Want_[0] = 0
<Remainder>:
```



#### 有效性

+ _mutual exclusion_：反设存在两个处理器同时进入critical section，我们不妨设当$p_1$最后一次写入`_Want_[1] = 1`之后读取的`_Want_[0]`仍然是0（这样$p_1$就能够进入critical section），然后$p_0$才开始写入`_Want_[0] = 1`，在进入critical section之前先读入`_Want_[1] == 0`（这样$p_0$才能够进入critical section）。但是当$p_1$在critical section的时候`_Want_[1]`应当为$1$，这样在$p_0$进入critical section之前$p_1$已经不在critical section了，因此矛盾

  ```sequence
  Note left of p1: write 1 to Want[1]
  Note right of p0: write 1 to Want[0]
  Note right of p0: read 0 from Want[1]
  ```

+ _no deadlock_：反设至少一个处理器一直在尝试阶段并且没有处理器进入critical section

  + 当两个处理器皆在尝试阶段，由于没有进入critical section因此`_Priority_`不会发生变化。不妨令`_Priority_ == 0`，因此$p_0$不会受到`wait until (_Want_[1] == 0 || _Priority_ == 0)`的阻碍。但由于$p_0$一直呆在尝试阶段，因此$p_0$只能是一直呆在`wait until (_Want_[1] == 0)`这一行，因此`_Want_[1] == 0`。但是由于`_Priority_ == 0`，因此$p_1$只能停留在`wait until (_Want_[0] == 0 || _Priority_ == 1)`，也就是说此时`_Want_[1] == 0`，$p_0$不能一直呆在尝试阶段了，矛盾
  + 当一个处理器在尝试阶段，另一个处理器永远待在剩余阶段，不妨设$p_0$在尝试阶段。由于$p_1$永远呆在剩余阶段，因此`_Want_[1] == 0`一直成立，此时$p_0$没有任何可以循环的条件并进入critical section，矛盾

+ _no lockout_：反设某一个处理器一直呆在尝试阶段，不妨设其为$p_0$

  + 如果出现`_Priority_ == 0`，那么唯一阻碍$p_0$进入critical section就是`wait until (_Want_[1] == 0)`，因此$p_1$应当一直在从`_Want_[1] = 1`到`_Priority_ = 0`，但是两个处理器一直都在尝试阶段违反了_no deadlock_特性，矛盾
  + 如果直一`_Priority_ == 1`，那么$p_1$一直在剩余阶段（如果在尝试阶段那么根据_no deadlock_一定会到达退出阶段改变`_Priority_`的值，回到第一种情况），此时`_Want_[1] == 0`一直保持，此时$p_0$一定会进入critical section，矛盾

### Bounded Mutual Exclusion Algorithm for $n$ processors

$2$-processor的互斥算法建立了一种pairwise的关系，因此$n$-processors的互斥算法就是在这种pairwise的关系上建立tournament tree，每个处理器从指定的树叶开始竞争，到达树根获得进入critical section的许可。在tournament tree中我们依次给每个节点进行标号，在每个节点进行本地的竞争，进入下一层，递归下去。

```pseudocode
procedure Node(NODE_NUM: int, side: int[]{0, 1})

<Entry>:
	_Want_[NODE_NUM][side] = 0
	wait until (_Want_[NODE_NUM][1-side] == 0 || _Priority_[NODE_NUM] == side)
	_Want_[NODE_NUM][side] = 1
	if (_Priority_[NODE_NUM] == 1-side)
		if (_Want_[NODE_NUM][1-side] == 1)
			goto <Entry>
	else
		wait until (_Want_[NODE_NUM][1-side] == 0)
		
	// at the root
	if (v == 1)
		<Critical Section>:{}
	else
		Node(floor(v/2), v % 2)
		
<Exit>:
	_Priority_[v] = 1-side
	_Want_[NODE_NUM][side] = 0

end procedure
```


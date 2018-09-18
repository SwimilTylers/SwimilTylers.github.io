---
layout:     post
mathjax:  true
title:      "Principle of Complier (I) - Lexical Analysis"
subtitle:   "Notes on Principle of Compiler"
date:       2018-09-18 15:00:31
author:     "swimiltylers"
header-img: "img/post-bg-re-vs-ng2.jpg"
tags:
    - Complier
    - Lexical Analysis
    - Regular Expression
    - Regex
    - DFA
    - NFA
    - Automata
    - Notes
---

> 这是编译原理笔记的第一章，讲述了词法分析。

编译原理是计算机科学的一个基础课程，但是这门课比较难，因此上课的同时做一下笔记，方便之后查阅复习。本篇主要是对一些概念进行提纲，对一些算法进行简单的阐释。参考书是龙书。

## 词法分析的作用

1. 编译器划分为两个阶段的原因
   + 简化编译器的设计，任务分解
   + 提高编译器的效率
   + 增强编译器的可移植性
2. 词法分析单位
   1. 词法单元(_Token_)：包含单元名(_Token name_)和可选的属性值(_Attribute value_)，单元名是表示某种词法单位 __抽象__ 符号
   2. 词素(_Lexeme_)：源程序的字符序列，和某个词法单元的模式匹配，被识别为词法单元的实例
   3. 模式(_Pattern_)：词法单元的词素所有可能的形式，通常用正则表达式标识

## 词法单元的规约

1. 基本概念
   + 字母表：有限的符号集合，ASCII
   + 串：字母表中的符号构成的__有穷__序列
     + $s$的长度$\mid s\mid$
     + 空串$\varepsilon$，长度为0的串
     + （真）前缀，（真）后缀，（真）字串，子序列
     + 连接运算(_concatenation_)，指数运算
   + 语言：给定字母表上任意一个可数的串的集合
     + 并运算
     + 连接
     + Kleene闭包
     + 正闭包
2. 正则表达式
   + 定义
     + $\varepsilon$是一个正则表达式
     + 优先级从低到高：选择($r\mid s$)，连接($rs$)，闭包($r^*$)，括号($\left(r\right)$)
     + 扩展运算符：一个或多个($r^+$)，零个或一个($r?$)，字符类($\left[abc\right],\,\left[a-z\right]$)
   + 性质
     + 等价性：如果两个正则表达式表示相同的语言，则这两个表达式等价
     + 交换律，结合律，分配律，幂等律
3. 正则定义：命名一些正则表达式，基于这些名字进行进一步的定义，该定义的正则表达式可以通过递归展开

##词法单元的识别

1. 状态转换图

   + 正则表达式可以转换为状态转换图
   + 节点：状态
     + 状态看作已处理部分的总结
     + 某系状态为最终状态，表名已经找到了词素
     + 加上*的接受状态表示最后读入的字符不在词素中
     + 由"start"边初始化
   + 边：状态转换

2. 多个模式集成方式

   1. 词法分析器需要匹配多个模式（有多个状态转换图）

      顺序尝试各个语法单元的状态转换图，如果引发fail，就回退并启动下一个状态转换图

   2. 并行的运行各个状态转换图

   3. 所有的状态转换图合并为一个图

3. 有穷自动机

   + 不确定的有穷自动机(_Nondeterministic Finite Automata_)
   + 确定的有穷自动机(_Deterministic Finite Automata_)
   + 自动机与状态转换图的区别在于，自动机是识别器，对每个输入串回答yes or no
   + NFA与DFA
     + 不同
       1. 一个符号标记离开同一状态的多条边
       2. 可以有边的标记是$\varepsilon$
     + 相同：两者之间存在等价性
   + NFA
     + 组成元素：1）有穷的状态集合$S$；2）输入符号集合$\Sigma$；3）转换函数。/；4）开始状态$S_0$；5）接受状态$F$
     + 可以表示为一个转换表
     + 只要存在一条从开始状态到结束状态的路径，就表示能接受。有开始状态到某个接受状态的所有路径上的符号串集合，称为$L(A)$
   + DFA
     + 没有$\varepsilon$之上的转换动作，对于每个状态和每个输入符号，有且只有一条边
     + 每个NFA都可以转换为等价的DFA

4. 正则表达式转为NFA

   1. $r=s$

      ```mermaid
      graph LR
      A(A)-->|r|B(B)
      ```

   2. $r=s\mid t$

      ```mermaid
      graph LR
      A(A)-->|ε|B(B)
      A-->|ε|C(C)
      B-->|s|D(D)
      C-->|t|E(E)
      D-->|ε|F(F)
      E-->|ε|F(F)
      ```

   3. $r=st$

      ```mermaid
      graph LR
      A(A)-->|s|B(B)
      B-->|t|C(C)
      ```

   4. $r=s^*$

5. NFA转为DFA：**子集构造法**，DFA一个状态对应NFA状态集合的子集

   | 操作                       | 描述                                                         |
   | -------------------------- | ------------------------------------------------------------ |
   | $\varepsilon$-$closure(s)$ | $s$通过$\varepsilon$边能到达的状态的集合$\bigcup \{s\}$      |
   | $\varepsilon$-$closure(T)$ | $T$中的每个元素$s$能通过$\varepsilon$边到达的状态的集合$\bigcup T$ |
   | $move(T,a)$                | $T$中的每个元素$s$能通过$a$边到达的状态的集合                |

   1. 构造方法：NFA产生的集合作为DFA的一个状态。初始的集合是$\varepsilon$-$closure(s_0)$，$T$通过$a$到达的下一个集合应当是$\varepsilon$-$closure(move(T,a))$

   2. 子集构造法对NFA的运行进行模拟

      ```c
      S = epsilon_closure(s_0);
      c = nextChar();
      
      while (c != __eof__){
          S = epsilon(move(S, c));
          c = nextChar();
      }
      if (S and F != empty_set)
      	return "yes";
      else
      	return "no";
      ```

6. 最小化DFA状态个数：将一个DFA的状态集合划分为多个组，每个组中的各个状态之间互相不可区分，然后每个组的状态合并为一个状态

   + 任何一个正则语言都有一个唯一的（不计同构）状态数目最少的DFA
   + 可区分：如果分别从$s$和$t$出发，存在某个路径，沿着该路径到达的两个状态只有一个是接受状态
   + 划分算法
     1. 设置初始化分$\Pi=\{S-F,F\}$，$F$表示结束状态的集合。如果是多个不同的结束状态，那么就将相同结论(_output_)的结束状态划分为同一组
     2. 迭代，更新划分$\Pi'=\big[\left[\mathrm{G.splitIf}\left(\exists a\in\Sigma\wedge\mathrm{though\,}a\,\mathrm{reach\,} s\notin \mathrm{G}\right)\right]\mathrm{for\,each\,G\, in\,}\Pi\big]$，直到$\Pi^{k+1}=\Pi^{k}$收敛，此时记为$\Pi_{\mathrm{final}}$
     3. 在$\Pi_{\mathrm{final}}$中选出分组的代表
        + 有开始状态或者结束状态的，将其中一个开始/结束状态作为代表
        + 如果$r$是某个组$G$的代表，假设原DFA中如果存在一条边$a$到达$s$，如果$s$所在的组$H$代表为$t$，那么新DFA中就存在从$r$到$t$在输入$a$上的转换


---
layout:     post
mathjax:  true
title:      "Distributed Algorithm Basics (II) - Broadcast and Converge-cast"
subtitle:   "Notes on DA study"
date:       2018-09-01 09:10:10
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - Flooding
    - Converge-cast
    - MST
    - Message-Passing model
    - Notes
---

> 这是分布式算法笔记的第二章，讲述洪泛(_Flooding_)算法和汇合(_Converge-cast_)算法。

从本章开始，分布式算法的笔记也将围绕算法展开。本章主要讲述两个非常基础的生成树算法。本章介绍的算法非常基础，尤其是洪泛，但也存在在效率的问题。这一章的内容主要围绕Yale讲义，由于本章算法比较简单，因此只是简述算法流程，不进行详细的讨论。

## Simple Flooding

得到网络的MST，在同步信息系统中，得到的是BFS遍历树

```pseudocode
Class node:
	Procedure initialize:
		if this.pid == root
			this.parent := root
			this.sendMessageTo(this.neighbors)
		else
			this.parent := null
	End Procedure
	
	Procedure receiveMessage(message) :
		if this.parent == null
			this.parent := message.from.pid
			this.sendMessageTo(this.neighbors)
	End Procedure
End Class
```

## Converge-cast

Flooding的逆运算，从底向上汇总信息传递给`root`，这个运算是建立在已经建立好spanning tree的基础上进行传播的

```pseudocode
Class node<f>:
	Procedure initialize:
		if this.pid != root
			this.sendMessageTo(this.parent)
	End Procedure

	Procedure receiveMessage(message) :
		this.buffer.append(message)
		this.receivedProcessId.push(message.from.pid)
		if this.receivedProcessId.equal(this.children)
			this.message := f(this.buffer, this.message)
			if this.pid == root
				return this.message
			else
				this.sendMessageTo(this.parent)
	End Procedure
End Class
```
---
layout:     post
mathjax:  true
title:      "Distributed Algorithm Basics (III) - Distributed BFS"
subtitle:   "Notes on DA study"
date:       2018-09-04 11:40:05
author:     "swimiltylers"
header-img: "img/home-bg-o.jpg"
tags:
    - Distributed Algorithm
    - BFS tree
    - Message-Passing model
    - Synchronizer
    - MST
    - Notes
---

> 这是分布式算法笔记的第三章，讲述分布式BFS生成算法。

本章主要讲述三种BFS生成算法，通过衡量到达树根的距离，处理器确定自己在BFS tree中的层数以及对应的子节点和父节点。后两种算法涉及到不同类型的同步机制(_Synchronizer_)，这里就不细细展开了。


## Explicit Distance

__Complexity__: $O(VE)$ messages and $O(D)$ time

__Backward__: Messages pile up on the channel.

```pseudocode
Class node:
	Procedure initialzie :
		if this.pid == root
			this.distance := 0
			this.sendMessageTo(this.neighbors)
		else
			this.distance := POSTIVE_INFINITE
	End Procedure
	
	Procedure receiveMessage(message) :
		if message.from.distance+1 < this.distances
			this.distance := message.from.distance+1
			this.parent := message.from.pid
			this.sendMessageTo(this.neighbors)
	End Procedure
End Class
```

## Layering: Building Beta Synchronizer

__Complexity__: $O(E+VD)$ messages and $O(D^2)$ time

__Backward__: Too much time on propagation permission.

```pseudocode
Procedure LayeredBFS
	Initialize __bfs_tree__, Set __root__.dist = 0 while __root__.bound = 1
	Send __root__.generateMessage to __root__.neighbors
	
	Define __node__: 
		message->{
			Register this to __bfs_tree__ Unless __bfs_tree__.find(this)
			Set this.dist = message.dist + 1 If message is __dist_update__
			Send message to __bfs_tree__.children(this) If message is 			__bound_update__
			Send this.generateMessage to this.neighbors If message.bound > this.bound
		}
	
	do
		Wait for __bfs_tree__.stableS
		Break Unless __bfs_tree__ changes
		Increment __root__.bound
		Notify the update to all nodes through __bfs_tree__.path
	loop
End Procedure
```

## Local Sync: Building Alpha Synchronizer

__Inspiration__: Layering algorithm needs a global permission to continue propagation,  which is too costly. Instead, that each node at distance $d$ delays sending out a recruiting message until it has confirmed that none of its neighbors will sending it a smaller distance is a better option.

__Complexity__: $O(\mid E\mid\bullet D)$ messages and $O(D)$ time

```pseudocode
Class node:
	Procedure initialzie :
		if this.pid == root
			this.distance := 0
			this.sendMessageTo("exactly", this.neighbors)
	End Procedure
	
	Procedure receiveMessage(message) :
		if this.pid != root
			if this.distance == 0
				this.sendMessageTo("more than", this.neighbors)
				return
			if this.neigbhors.equal(this.receivedMessages.from)
				scenarios_dict := {
                	"scenario one":
                		x.equal("exactly", $d) -> more than 1
                		x.equal("more than", $d-1) -> rest
                	"scenario two":
                		x.equal("more than", $d) -> all
				}
				switch checkScenario(this.receivedMessages, scenarios_dict)
					case "scenario one":
						this.distance := $d + 1
						this.sendMessageTo("exactly", this.neighbors)
					case "scenario two":
						this.distance := $d + 1
						this.sendMessageTo("more than", this.neighbors)
			else
				this.receivedMessages.push(message)
	End Procedure
End Class
```


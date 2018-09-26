---
layout:     post
mathjax:  true
title:      "Coq Basics (I) Inductives, Fixpoints, Tactics"
subtitle:   "Notes on Coq"
date:       2018-09-23 19:59:31
author:     "swimiltylers"
header-img: "img/post-bg-e2e-ux.jpg"
tags:
    - Coq
    - Coq Tactics
    - Notes
---

> 这是Coq笔记的第一张，记录了SF教材的第一卷前三章


## `Inductive`

`Inductive`是Coq用来定义数据类型，实际上Coq的数据类型十分精简，但是我们可以通过这样的方法来自定义数据类型，Coq中最基本的数据类型就是`Type`，相当于Java的`Object`。语句的结构为`Inductive [TYPE_NAME] : [INHERITED_TYPE] := {IND_DEF}`

### Enumerate

```coq
Inductive day : Type :=
  | monday : day
  | tuesday : day
  | wednesday : day
  | thursday : day
  | friday : day
  | saturday : day
  | sunday : day.
```

```coq
Inductive bool : Type :=
  | true : bool
  | false : bool.
```

枚举的Data Type可以直接通过枚举所有的构造器(_constructor_)来确定，构造器的语法类似于函数的定义，在按照`[cons_name]: [intype_1 ->...->outtype]`的形式进行定义，一般来说枚举类型的构造器是无输入类型。尽管表达形式类似函数，但是实际上只是数据的一种标记方式，在内存中直接以构造器的形式进行存储。

Coq's conditionals are exactly like those found in any other language, with one small generalization. Since the boolean type is not built in, Coq actually supports conditional expressions over *any* inductively defined type with exactly two constructors. The guard is considered true if it evaluates to the first constructor in the Inductivedefinition and false if it evaluates to the second.

### Compound

```coq
Inductive rgb : Type :=
  | red : rgb
  | green : rgb
  | blue : rgb.

Inductive color : Type :=
  | black : color
  | white : color
  | primary : rgb → color.
```

### Natural Number

```coq

```

在Coq中，可以直接使用数字来表示`S`这种一元构造器，因此如果是`nat`类型，那么$4=S(S(S(S(O))))$

### Pair

```coq

```

这种构造器的输入类型有两个`(nat,nat)`。如果证明中需要显式通过`natprod`中的构造器输入类型，除了在证明中直接指出，或者参考`destruct`中构造器析构方法进行证明中命名。

## `Definition`

## `Fixpoint`

Fixpoint允许递归的调用本身，但是这种调用必须是一种有界的。因此，Fixpoint函数中所匹配(_match_)的参数必须是以降序的方式进行展开，确保不会无限递归。

```coq
Fixpoint plus (n : nat) (m : nat) : nat :=
  match n with
    | O ⇒ m
    | S n' ⇒ S (plus n' m)
  end.
```

```coq
Fixpoint beq_nat (n m : nat) : bool :=
  match n with
  | O ⇒ match m with
         | O ⇒ true
         | S m' ⇒ false
         end
  | S n' ⇒ match m with
            | O ⇒ false
            | S m' ⇒ beq_nat n' m'
            end
  end.
```



## Tactics

### `simpl`和`reflexivity`

#### 化简

```coq
Theorem plus_O_n: forall n : nat, o + n = n
Proof.
	intros n. simpl. reflexitivity. Qed.
```

这个证明只用了`simpl`和`reflexivity`两种方法。`simpl`可以将等式的两边同时化简(_simplify_)，而`reflexivity`是用来检查等式的两边是否完全一致(_identical_)。一般情况下，不需要加`simpl`，因为`reflexivity`本身在检查一致性的时候，就会自动的进行展开，如果需要在调试的时候查看化简之后的效果才会加上`simpl`。

`reflexivity`除了进行化简的工作，还会根据定义将产生式展开(_unfolding_)。化简和展开的区别在于，化简是将定义的函数（比如递归函数）化简到稳定的状态（对于递归函数，就是Base case）；展开则是利用产生式的构造器完整的写出数据的形式。

能够化简的一个前提条件是，函数能够无歧义的化简（上文中`n + o = n`就不能够化简证明，需要分情况证明，具体原因在于`plus`的第一个参数决定了函数的分支）。

### `rewrite`

#### 重写

```coq
Theorem plus_id_example : forall n m:nat,
  n = m ->
  n + n = m + m.
Proof.
  (* move both quantifiers into the context: *)
  intros n m.
  (* move the hypothesis into the context: *)
  intros H.
  (* rewrite the goal using the hypothesis: *)
  rewrite -> H.
  reflexivity. Qed.
```

重写(_rewriting_)指的是，在证明的过程中，按照给出的前提假设(_Hypothesis_)，在要证明的式子两边对变量进行改写，辅助证明。`->`表示的意思是使用前提假设的右侧替换左侧，`<-`表示左侧替换右侧。对于上面的证明过程，我们无法指出所有可能的取值，因此证明的时候无法进行相应的确切的化简，因此需要改写变量满足假设进行证明，该重写过程就是`n + n = m + m (*-->*) m + m = m + m `

### `destruct`

#### 构造器析构

```coq
Theorem surjective_pairing : ∀ (p : natprod),
  p = (fst p, snd p).
Proof.
  intros p. destruct p as [n m]. simpl. reflexivity. Qed.
```

#### 分类讨论

```coq
Theorem plus_1_neq_0 : forall n : nat,
  beq_nat (n + 1) 0 = false.
Proof.
  intros n. destruct n as [| n'].
  - reflexivity.
  - reflexivity. Qed.
```

可以简写成

```coq
Theorem plus_1_neq_0' : forall n : nat,
  beq_nat (n + 1) 0 = false.
Proof.
  intros [|n].
  - reflexivity.
  - reflexivity. Qed.
```

使用`destruct`对引入的参数进行分情况讨论，设置子目标(_sub goal_)，分别进行证明。分情况讨论允许多层的嵌套，在需要进行分情况讨论的地方进行`destruct`。每个子目标前面的`-`没有语法含义，可以替换为`+`或者`*`。

`destruct`后面的中括号参数是根据类型的定义设置的，如果出现多种情况就是用`|`隔开。对于`nat`的定义，第一个构造器(_constructor_)并无参数，因此为空，第二个构造器的参数也是`nat`，因此需要指明参数名称。如果构造器都没有参数，那就可以省略`as`子句。

### `induction`

#### 假设归纳

```coq
Theorem plus_n_O : forall n:nat, n = n + 0.
Proof.
  intros n. induction n as [| n' IHn'].
  - (* n = 0 *) reflexivity.
  - (* n = S n' *) simpl. rewrite <- IHn'. reflexivity. Qed.
```

那上述的证明来说明，$n$可以是任意大的自然数，如果按照分情况讨论的话，可能会一直化简下去，并不能验证任意大小的$n$。这种情况下，使用类似于数学归纳法进行证明，即`induction`，归纳证明命题的正确性。

归纳`induction`的语法结构同样类似与`destruct`，产生的不同的子目标，进行归纳证明。产生的子目标也同样根据参数的构造方法有关，并使用`|`区分不同构造器情况，并在之后子目标的论证用`-`、`+`或者`*`加以区别。`induction`可以提供关于子目标环境下的参数命名，也可以对子目标环境下的假设(_hypothesis_)进行命名（在上述论证中就是`IHn'`，代表着`n' = n' + 0`），在该子目标下进行重写，进行进一步的论证。

### `assert`

#### 行间证明

```coq
Theorem mult_0_plus' : forall n m : nat,
  (0 + n) * m = n * m.
Proof.
  intros n m.
  assert (H: 0 + n = n). { reflexivity. }
  rewrite → H.
  reflexivity. Qed.
```

如果在证明的过程中需要加入临时的、显而易见的中间引理证明，那么可以考虑`assert`进行证明内的小引理证明(_sub-theorem_)。`assert`这种方式引入了两个子目标，第一个便是断言(_assertion_)本身（在上述的论证中，断言被命名为$H$，指的是$0 + n = 0$），这个子目标的论证囊括在大括号$\{\dots\}$中。另一个子目标就是在既有断言基础上进行原先的论证，通常也是通过重写来完成。

断言的方式主要是提供一种类似于匿名函数的功能。




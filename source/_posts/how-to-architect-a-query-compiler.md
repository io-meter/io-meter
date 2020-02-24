---
title: SIGMOD 16 | How to Architect a Query Compiler
date: 2020-02-24 18:29:55
tags: [database, compiler, scala, 'functional programming']
---

这篇发表在 SIGMOD 16 的论文来自洛桑联邦理工学院（EFPL），这所学校在计算机领域以Scala编程语言的发源地而闻名于世。不出意外，这篇论文虽然发表于数据处理相关顶会，却弥漫着浓厚的函数式编程和PL的气氛。连系统实现和代码样例都使用 Scala 描述。

<!-- more -->

从研究的依赖关系上，本文是之前介绍过的同样来自EFPL的研究[LegoBase](https://zhuanlan.zhihu.com/p/89644696)的延伸。本文在之前从Scala编译到C语言的系统之上，进一步地建立起了多层次的查询编译器架构，并提出了一套实现易操控和可维护的SQL查询编译系统的方法论。


## 简介

SQL编译查询不是新鲜事物。在Hekaton，Impala，Presto，MemSQL和Calcite当中都有使用。由于现阶段OLAP任务的主要瓶颈还在于IO，各个系统对相关领域的研究不算深入。其中Java系的大数据处理平台很多使用[Janino](https://janino-compiler.github.io/janino/)作为Bytecode编译器，也有很多系统将查询翻译成C/C++语言，并使用GCC/Clang或直接使用LLVM进行处理。本文指出，现阶段很多查询编译系统仍然停留在模板展开（template expander）的层次，简单来说也就是拼接代码和进行Operator调用内联（inlining）。本文指出，这类查询编译结局方案缺乏应用多种编程语言优化技术的能力，且需要的规则数和代码量随着算子数量的上升，至少以 N^2 的速度上升。此外，目前的解决方案也很难利用编译器来直接支持 Pipeline。

本文的解决方案在于为查询编译实现专门的DSL，而且是实现一系列堆叠起来的具有不同表现力和底层操控能力的DSL。在这一堆栈的最上层是最接近于关系模型的QPlan和QMonad DSL，这一层次的DSL非常的声明式（Declarative）。相反，在这一堆栈的最下层是C.Scala——一个几乎可以直接逐句翻译成C语言的DSL。显然，在最上层，用户得到的是抽象程度高的编程语言，在这一层次用户可以使用非常简洁地表示查询，也方便应用很多PL优化算法，但缺乏对程序细节（内存分配、数据结构、执行顺序等）的控制，也难以进行性能调优。在最下层，用户得到了一个非常接近硬件的命令式的DSL，因此可以对执行细节进行详细的操控，但是程序的表现力减弱且应用各种编译优化技巧变得困难起来。实际上，为了平衡这两者，本论文提出在两个极端之间插入多种中间DSL语言，在每个层次应用不同层次的PL优化技术。同时，在编译时，使用逐层下降的方法将高层次的语言（SQL）翻译成低层次的语言（C）从而实现模块化并提高可维护性。下图展示了本文提出的DSL堆栈。



![](img/DSL-stack.png)


总体来说，在堆栈的每个层次，系统都会完成两项任务：



*   应用适合于当前层次的编程语言优化规则和算法，在当前层次对执行方案进行优化
*   进行数据结构特化，即将某种抽象的数据结构（List，Map，MultiMap）等，具体化更低层次的抽象（ArrayList，LinkedList，HashMap，TreeMap等）

经过这两步处理，高层次的DSL会被下降为低层次的DSL，直到最终的C语言实现，所有的内存管理和数据结构就都被具体化了，可以通过编译来直接执行。


## DSL 堆栈

本文使用Scala语言实现了名为ScaLite的DSL，这一DSL吸取了函数式编程思想。本文指出，函数式编程的语言介于纯声明式语言（SQL）和纯命令式语言（C）之间，是作为插入DSL层的理想选择。而且这类语言易于理解和推理，可以复用PL社区的研究成果，也便于计算机进行抽象处理并应用一些编程语言优化算法。在这里，论文着重介绍了函数式编程的不可变性（Immutability）为程序分析带来的好处——具有副作用的编程语言语句有时很难使用算法进行分析和优化。

从抽象到具体，本文提出的DSL堆栈主要有一下几个层次：

### SQL 层

SQL层就是最上层用户交互的层次，它包括传统数据库优化的技术栈，如使用算子和关系代数描述语句，并可以使用Volcano/Cascades模型对关系代数进行优化。在本文中，以下述SQL作为输入案例：


```scala
SELECT COUNT(*)
FROM R, S
WHERE R.name = "R1"
  AND R.sid = S.rid;
```


### QPlan/QMonad 层

QPlan和QMonad是两个可以相互替换的高层次声明式DSL，它们非常接近SQL的关系代数表示，但是在这个层次，SQL已经可以被翻译为能被Scala理解的语法了。相比起来，QPlan的表示方法更接近关系代数，而QMonad更接近于函数式编程和Collection Programming。

上述SQL在QPlan中的表示是：


```scala
AggOp(
  HashJoinOp(
    SelectOp(R, "name", EQ, "R1"),
    S, "sid", "rid"
  ),
  COUNT
)
```


上述 SQL 在 QMonad 上的表示是：


```scala
R.filter(r => r.name == "R1")
  .hashJoin(S)(r => r.sid)(s => s.sid)
  .count
```


随后，QPlan/QMonad会被翻译成 `ScaLite[Map, List]`。值得注意的是，在这一步系统会进行Pipelining的优化。比方说上述QMonad代码中的 filter 和 hashJoin 两个操作，其实可以将 hashJoin 操作的前半部分（为其中一个表构建Hash表）的循环和 filter 当中过滤元素的循环合并起来。因为我们在构建 Hash 表的时候，显然不需要无关的来自 R 表中的记录。下面层次的 DSL 表示已经执行了这一优化。

### ScaLite[Map, List] 层

这一层次，DSL可以使用抽象的Map和List数据结构来进行描述，而无需考虑这些Map和List的具体实现。下面是上述QPlan和QMonad翻译成的 ScaLite[Map, List]。


```scala
val hm = new MultiMap[Int, R]
for (r <- R) {
  if (r.name == "R1") { // Pipelining
    hm.addBinding(r.sid, r)
  }
}
var count = 0
for (s <- S) {
  hm.get(s.rid) match {
    case Some(rList) => 
      for (r <- rList) {
        if (r.sid == s.rid) count +=1
      }
    case None => ()
}
return count
```


在这一层次，系统会对 Map 数据结构进行特化，将其翻译成使用List或Array来描述的形式。在这个例子中，根据系统掌握的信息，他有可能把 MultiMap 翻译成 Array[List] 或者直接使用 Array （当 r.sid 是 Unique 主键时）。也就是说，在这一层次会进行数据结构的特化，将 MultiMap 转变成最合适的底层实现。经过这样的操作，所有的 Map 都被底层的 Array 或 List 替代，从而下降到 ScaLite[List]。

值得注意的是，在这一 DSL 层次，所有Map的Key和Value都假定是不可变的。因此在当前层次，可以应用对 FOR 循环的多种优化规则，因为已经插入 Map 的值不可变，程序分析时更容易对循环的Access Pattern进行推断。值得注意的是，第一个循环其实是上层 QMonad DSL 表示中 filter 和 hashJoin 算子 Fusion 之后的实现。本文还提到，在对 Map 类数据结构进行下降的时候，可以在合适的情况下使用 String Dictionary 来加速对字符串相关列的处理。

### ScaLite[List] 层

所有的Map数据结构都被特化，用Array等其他底层数据结构取代。在这一层次，我们放松了上一层次对不变性的要求，允许程序修改Array中List的值，这样，上述代码中第一个循环的MultiMap的构建过程，就可以更为具体地以命令式的方式表达出来。下面是DSL下降到这一层次的代码：


```scala
val MR: Array[List[R]] = new Array[List[R]](R_GROUPS);
for(r <- R) {
  if (r.name == "R1") MR(r.sid) += r // Materialized here
}
var count = 0
for (s <- S) {
  val rList = MR(s.rid)
  for (r <- rList) {
    if (r.sid == s.rid) count += 1
  }
}

return count
```


可以看到，对于 MultiMap 的构建和访问都已经以命令式的形式翻译成对 Array[List] 的操作了。

从这一层次进行下降，系统进行的变换是具像化 List。比方说，根据系统掌握的信息，可以将 List 翻译成 LinkedList 或者Array。在翻译成 LinkedList 的时候，与其使用一个封装类，系统可以将next指针内联到Record之中。也就是生成一个新的结构体，这个结构体除了保存原来数据条目的每个记录之外，还包含一个指向下一条记录的指针。这样就避免了使用封装链表访问数据字段的一次指针跳转。通过对 ScaLite[List] 进行下降，我们就得到了 ScaLite。

### ScaLite

ScaLite 的核心是一个简单类型的 Lambda Calculus，它不支持递归且主要由数据结构的构造和函数的调用构成。这一层次的DSL已经距离纯函数式编程相当远，数据结构的可变性和副作用都被允许。它支持定长数组、变长数组和有序列表等基本数据结构。下面是 ScaLite 的代码


```scala
val MR: Array[R] = new Array[R](R_GROUPS)
for (r <- R) {
  if (r.name == "R1") {
    if (MR(r.sid) == null) {
      MR(r.sid) = r
    } else {
      r.next = MR(r.sid)
      MR(r.sid) = r
    }
  }
}
var count = 0
for (s <- S) {
  var r = MR(s.rid)
  while (r != null) {
    if (r.sid == s.rid) count += 1
    r = r.next
  }
}

return count
```


可以看到，在这个层次，连 List 抽象数据结构也不存在了，对数据的访问已经转换为对 next 指针命令式的赋值和读取。在这个层次，系统会进行一个重要的下降转换：数据结构布局的变换和内存控制的变换。也就是说，在从这个层次进行下降的过程中，会加入内存分配和访问相关的代码，并可能根据系统信息将数据结构转换为不同的Layout。常见的Layout有：



*   Boxed layout：指向结构体指针的数组
*   Row layout：结构体排列在一起形成的数组
*   Columnar layout：将结构体的每个字段组合在形成多个列式的数组

下图展示了不同的数据布局表示：

![](img/data-layout.png)

在 ScaLite 下降之后，就形成了 C.Scala。

### C.Scala

这一DSL层次是一个几乎可以逐行翻译成C语言的DSL，它已经包含了所有的内存分配等操作，完全可以直接用于执行了。下面是 C.Scala 的例子：


```scala
val MR: Array[Pointer[R]] = malloc[Pointer[R]](R_GROUPS)
for (r <- R) {
  if (r -> name == "R1") {
    if (MR(r->sid) == null) {
      MR(r->sid) = r
    } else {
      r->next = MR(r->sid)
      MR(r->sid) = r
    }
  }
}
var count = 0
for (s <- S) {
  var r: Pointer[R] = MR(s->rid)
  while(r != null) {
    if (r->sid == s.rid) count += 1
    r = r-> next
  }
}

return count
```


在这一层次之后，DSL会用来生成真的C代码，从而交付给编译器进行编译。在编译的过程当中，C语言常用的编译优化规则自然也可以被应用，从而进一步加快执行。

值得注意的是，在本文系统架构翻译出的可执行代码，和 HyPer 等系统一样使用的是 Push 模型。


## 总结

本文在 LegoBase 的基础上提出了构建查询编译引擎的一些有益思路。通过将PL领域的相关研究和数据库领域的一些发展结合起来，提出了一个用来简化查询编译系统开发、推理和维护的框架。可以说是跨领域学科研究的一个不错的案例。

尽管现阶段常见OLAP系统当中主要的性能瓶颈仍是IO，因此从现实角度来说，In-memory 数据库从这个研究中获益更大。但是随着硬件的不断发展和RDMA系统的逐渐普及，也许在未来类似的查询编译系统将会成为主流。

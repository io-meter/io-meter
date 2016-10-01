---
title: 'Functional Go: Transient 及持久化'
date: 2016-10-01 22:21:46
tags: [Golang, go, functional programming, fp, persist datastructure, immutable, essay, vector trie]
category: 'Functional Go'
---

在之前的文章中，我们介绍了一些持久化数据结构实现的基本原理和 Vector Trie 这一数据结构在 Golang 下的实现过程。
这篇文章终于来到了实现持久化 List 的最后一步: 实现 Transient 和持久化的功能。

<!-- more -->

这篇文章是系列文章的一部分，如果还没有浏览过文章的其它部分请参考：

1. [持久化数据结构简介](https://io-meter.com/2016/09/03/Functional-Go-persist-datastructure-intro/)
2. [Vector Trie 的实现](https://io-meter.com/2016/09/15/functional-go-implement-vector-trie/)
3. [Transient 及持久化](https://io-meter.com/2016/09/15/functional-go-implement-vector-trie/) (本文)

在之前的文章中，我们已经看到了如何实现一个 Vector Trie，也知道如何使用 Vector Trie 来实现共享数据结构的持久化 List：
在每次修改时，我们复制从根节点到被修改节点路径上的所有节点，并使用得到的新的 Root 节点构造一个新的 List 的 HEAD
数据结构。这样通过新的 HEAD 我们就可以访问到新的数据，通过旧的 HEAD 我们可以得到旧的数据。

按照这样的思路，我们需要更改 List 的接口，对于每一个会修改 List 元素的操作，我们都要返回一个新的 List
对象而不是在原来的对象上修改。比如说:

```go
type List interface {
        Get(n int) (interface{}, bool)
        Set(n int, value interface{}) (List, bool)
        PushBack(value interface{}) List
        RemoveBack() (List, interface{})
        Len() int
}
```

为了实现这样的接口，我们有两种选择：

1. 对于 `Set`、`PushBack` 和 `RemoveBack` 函数，我们把它们修改成返回新的 `listHead` 的形式
2. 先实现 `TransientList`，对于`Set`、`PushBack` 和 `RemoveBack` 函数，我们把它改造成先把自己转换成 `TransientList`，
再在`TransientList` 上修改，最后返回将 `Transient` 持久化的结果

由于我们之前的代码设计上预先做了准备，两种方案的实现难度其实差别不大。但是因为 Transient
支持对数据结构一连串操作的高效执行，我们决定采用第二种方案。第二种方案也会使得代码更简洁、复用的程度更高。

那么什么是 Transient 呢？下面我们就来介绍它。

## Transient 的原理与实现

前面说道，持久化数据结构的实现原理是复制一条路径上的节点。在我们的设计中，每个节点的宽度是32，
那么如果我们连续地修改几个相邻的元素，即使这些元素都在一个叶子节点上，它也会被复制很多遍。
这样的行为是非常低效的。为了能让我们高效地进行一连串的修改，一种解决方案就是允许一个持久化数据结构临时地变成非持久的，
在我们一连串的修改之后，再转变回来。这样每次修改都会在原地进行，从而极大地改善了性能。
这里临时产生的非持久化数据结构就是我们所说的 Transient。

但是同样我们也要知道，使用 Transient 是有一定风险的。首先作为一种可变数据结构，它一般来说会被实现为非线程安全的类型，
因此如果并发地操作它，就可能产生 Race condition 等问题。另外，如果用户在使用时保留了对于 Transient 的引用，
直到把 Transient 转变为持久化之后仍然对它进行了修改，那么之前生成的持久化对象实际上也会被改变。
也就是说，引入 Transient 可能会导致无效的持久化。

尽管 Transient 带来了一些风险，但是考虑到性能上的提升，实现它还是值得的。Transient 的实现有两个关键点:

1. 为每个 Transient 分配一个全局唯一的`id`，当 Transient 每次对内部结构进行修改时，保证修改过的节点都被打上这个`id`
作为标记
2. 当 Transient 每次需要对节点进行修改时，它先检查目标节点是否和自己有相同的`id`，如果相同，
那么这个节点是自己之前曾经修改或复制过的，因此可以在节点上直接进行修改。否则这个节点可能是之前的 Transient 生成的，
为了防止改变原来的数据，我们应该复制当前节点一份。

这两条策略保证我们可以安全地修改 Transient 而不会改变原来的数据。关键在于，通过为 Vector Trie 的节点打上`id`
标志，Transient 可以判断一个节点的内存是不是由自己分配出来的。对于`id`不一样的节点，它是由当前 List 
修改历史上出现过的某个 Transient 产生的，而之前那个 Transient 已经被转换为持久对象，因此那些节点不应该被直接修改。
而如果`id`一致，则表明当前 Transient 新近修改过这一节点，我们就可以再次修改。这一步是基于持久化过的 Transient
不再会被使用者修改的假定。这也是为什么如果 Transient 持久化之后仍被修改，则生成的持久化对象的不可变性就会被破坏的原因。

下图对比了持久化 List 和 Transient 在执行修改时的不同。

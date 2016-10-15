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
3. [Transient 及持久化](https://io-meter.com/2016/10/01/Functional-Go-Transient-and-Persistent/) (本文)

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
2. 先实现 `TransientList`，对于`Set`、`PushBack` 和 `RemoveBack` 函数，我们把它改造成先把自己转换成 `TransientList`
并修改，最后返回将 `Transient` 持久化的结果

由于我们之前的代码设计上预先做了准备，两种方案的实现难度其实差别不大。但是因为 Transient
支持对数据结构一连串操作的高效执行，我们决定采用第二种方案。第二种方案也会使得代码更简洁、复用的程度更高。

那么什么是 Transient 呢？下面我们就来介绍它。

## Transient 的原理

前面说道，持久化数据结构的实现原理是复制一条路径上的节点。在我们的设计中，每个节点的宽度是32，
那么如果我们连续地修改几个相邻的元素，即使这些元素都在一个叶子节点上，它也会被复制很多遍。
这样的行为是非常低效的。为了能让我们高效地进行一连串的修改，一种解决方案就是允许一个持久化数据结构临时地变成非持久的，
在我们一连串的修改之后，再转变回来。这样每次修改都会在原地进行，从而极大地改善了性能。
这里临时产生的非持久化数据结构就是我们所说的 Transient。

但是同样我们也要知道，使用 Transient 是有一定风险的。首先作为一种可变数据结构，它一般来说会被实现为非线程安全的类型，
因此如果并发地操作它，就可能产生 Race condition 等问题。另外，如果用户在使用时保留了对于 Transient 的引用，
把 Transient 转变为持久化之后仍然对 Transient 进行了修改，那么生成的持久化对象实际上也会被改变。
也就是说，引入 Transient 可能会导致无效的持久化。

尽管 Transient 带来了一些风险，但是考虑到性能上的提升，它还是值得的。Transient 的实现有两个关键点:

1. 为每个 Transient 分配一个全局唯一的`id`，当 Transient 每次对内部结构进行修改时，保证修改过的节点都被打上这个`id`
作为标记
2. 当 Transient 每次需要对节点进行修改时，它先检查目标节点是否和自己有相同的`id`，如果相同，
那么这个节点是自己之前曾经修改或复制过的，因此可以在节点上直接进行修改。否则这个节点可能是之前的 Transient 生成的，
为了防止改变原来的数据，我们应该复制当前节点一份。

这两条策略保证我们可以安全地修改 Transient 而不会改变原来的数据。关键在于，通过为 Vector Trie 的节点打上`id`
标志，Transient 可以判断一个节点的内存是不是由自己分配出来的。对于`id`不一样的节点，它是由当前 List 
修改历史上出现过的某个 Transient 产生的，而之前那个 Transient 可能已经被转换为持久对象，因此那些节点不应该被直接修改。
而如果`id`一致，则表明当前 Transient 新近修改过这一节点，我们就可以再次修改。这一步是基于持久化过的 Transient
不再会被使用者修改的假定。这也是为什么如果 Transient 持久化之后仍被修改，则生成的持久化对象的不可变性就会被破坏的原因。

下图对比了持久化 List 和 Transient 在执行修改时的不同。

![Without vs. With transient](/img/posts/with-without-transient.png)

图中上方是不使用 Transient 时的情况，右边三种不同的颜色代表连续的三次修改。在这种情况下，
我们的每次修改都会产生一个新的 Root 节点和一份新的叶子节点。这样显然是没有效率的。

在图中的下方是使用了 Transient 时的情况，每个节点都包含了一个 ID (图中紫色标记)，当第一次修改进行的时候我们为 Transient
分配了一个新的 ID `b`，在修改的过程中检查需要更新的节点，发现他们都具有 ID `a`，与当前 ID 不同，因此需要进行一次复制。
在接下来的两次修改中，由于 Transient 在其生命周期中 ID 不变，进行修改时发现目标节点的 ID 与当前 Transient 一致，
因此我们不需要再复制节点，可以直接进行 In-place 的更新。

以上就是 Transient 的基本原理。由这个基本原理可以看出，实际上 Transient 和我们的持久化 List
可以共享一套底层的数据结构，其差别仅在于 Transient 拥有一个 ID 而 List 没有。实际上，为了区分这两种情况，
我们为所有的 List 的 `HEAD` 分配一个特殊的 ID，譬如`0`。在 List 和 Transient 之间转换可以使用下列手段:

1. 将 List 转化为 Transient，我们使用某种 ID 生成器生成一个唯一且不同于 List ID 的 ID (如正整数)并分配给 List
2. 将 Transient 转为 List，我们将 Transient 的 ID 重置为 List ID (如`0`)

在我们的情况下， 由于 Golang 特殊的面向对象设计，我们实际上可以将 List 内部数据结构实现为 Transient 
内部数据结构的一个 alias。

## Transient 的实现

### Unique ID 生成器的实现

对于 Transient 来讲，如何为每次修改生成独一无二的 ID 是一件重要的问题。在现实中存在很多功能各异的独特 ID 生成算法，
他们有的只能工作在单机情况下，有的可以保证分布式情况下的唯一性，有的生成成本比较高，有的则非常轻量。
在这里，我们选择最简单的一种方式：累计`uint64`类型的正整数。

通过在单例情况下累计正整数的方式，我们可以保证生成的 Unique ID 在当前进程中是具有唯一性的。
其原理是通过`sync/atomic`包下的原子操作`AddUint64`来实现递增操作。这一操作既快速又线程安全。

以下是实现这一功能的内部包`counter`:

```go
package counter

import "sync/atomic"

var id uint64 = 0

func Next() uint64 {
    return atomic.AddUint64(&id, 1)
}
```

### List 接口的更新

前面我们说道，可以将 List 实现为 Transient 的一个 alias。在这一步，我们先将之前博客里实现的 List 内部数据结构重命名为
`tListHead`，代表他是一个 Transient List 的 Head，之前实现的方法也都一并转移过来。除此之外，
我们还要在新的`tListHead`和它内部的 Trie 树节点上都添加 ID 字段:

```go
// Transient List Head
type tListHead struct {
    id     uint64
    len    int
    level  int
    offset int
    root   *trieNode
    tail   *trieNode
}

// Trie Node
type trieNode struct {
    id       uint64
    children []interface{}
}
```

然后我们重新定义 List 的接口和实现方法:

```go
type List interface {
    Get(n int) (interface{}, bool)
    Set(n int, value interface{}) (List, bool)
    PushBack(value interface{}) List
    RemoveBack() (List, interface{})
    TransientList() TransientList
    Len() int
}

type listHead tListHead
```

新的接口不再是在原来的基础上进行修改，而是每次操作都返回新的 List 对象。我们还添加了一个方法用于将当前 List
转换为一个 Transient。注意到`listHead`只是`tListHead`的一个 alias，因此在 Go 语言中他们之间可以相互类型转换。
接下来我们定义一个全局公共的 `empty` 变量代表空的 List，由于我们希望所有的空 List 都一样而持久化 List 是不会被改变的，
因此我们并不会在 New 时创建新的空对象而是每次都返回这一个对象。这样也节约了新建 List 时的内存消耗。

```go
var empty = &listHead{0, 0, 0, 0, nil, nil}

func New() List {
    return empty
}
```

List 的 `Get` 因为不会改变元素的值，我们直接通过类型转化的方法将`listHead`转换为
`tListHead`并调用后者的对应方法获得结果:

```go
func (head *listHead) Get(n int) (interface{}, bool) {
    return (*tListHead)(head).Get(n)
}
```

对于其它修改操作，我们都先将其转换为 Transient 执行完修改操作之后再持久化回来。这样就可以获得新的 List 了。

```go
func (head *listHead) Set(n int, value interface{}) (List, bool) {
    t := head.TransientList()
    if t.Set(n, value) {
        return t.Persist(), true
    } else {
        return head, false
    }
}

func (head *listHead) PushBack(value interface{}) List {
    t := head.TransientList()
    t.PushBack(value)
    return t.Persist()
}

func (head *listHead) RemoveBack() (List, interface{}) {
    if head.len == 1 {
        value, _ := head.Get(0)
        return empty, value
    } else {
        t := head.TransientList()
        value := t.RemoveBack()
        return t.Persist(), value
    }
}
```

下面给出了`TransientList`方法的实现。由于 List 的不可变性质需要被保留，因此转化为 Transient 
实际上需要新建立一个 `tListHead`，这样对于 Transient 的修改就不会影响到原来的 List。这里调用了之前实现的 `counter`
包来获得 Unique ID。

```go
func (head *listHead) TransientList() TransientList {
    id := counter.Next()
    return &tListHead{id, head.len, head.level, head.offset, head.root, head.tail}
}
```

### Transient 修改操作的实现

接下来还需要更新 Transient 修改操作的实现来保证不会影响到其它 Transient 以及之前的持久化 List。
之前的 List 在实现的过程中我们已经部分考虑到这种问题了，大部分操作被设计为递归执行，同时对 Trie 
树的递归操作会赋值给原来节点。在这一基础上我们首先为 `trieNode` 添加`clone`和`setChild`两个方法。

`clone` 方法会将当前节点的内容复制一边并返回新的节点，它接受一个`id`参数，新复制出来的节点的`id`属性会被设定为这一参数。

```go
func (node *trieNode) clone(id uint64) *trieNode {
    children := make([]interface{}, NODE_SIZE)
    copy(children, node.children)
    return &trieNode{id, children}
}
```

`setChild` 则是方便实现修改节点功能的函数，它的第一个参数也是`id`。如果传入的`id`与节点原来的`id`相同，
则这一方法直接在原来的节点上进行修改并返回原来的节点，否则它将会`clone`节点并在新的节点上进行操作。

```go
func (node *trieNode) setChild(id uint64, n int, child interface{}) *trieNode {
    if node.id == id {
        node.children[n] = child
        return node
    } else {
        newNode := node.clone(id)
        newNode.children[n] = child
        return newNode
    }
}
```

之前 List 修改操作的各个内部函数也都被加上了`id`作为参数。除此之外，如果`Set`前后 List 包含值相同，
我们希望实际效果是对象没有被修改，在这一步我们也做了一些小心的操作来尽可能保证。
具体的代码不再赘述，完整的代码请参考[这个文件](https://github.com/shanzi/persist/blob/master/list/transient_list.go)。

下面是将 Transient 转化为持久化的函数，由于我们预期用户在将 Transient 持久化之后不会再修改原来的 Transient
(尽管无法从代码上保证)，所以我们可以简单地使用类型转换来将 `tListHead` 转换为 `listHead`。

```go
func (head *tListHead) Persist() List {
    perisitHead := (*listHead)(head)
    perisitHead.id = 0
    return perisitHead
}
```

## 总结

这篇文章介绍了 Transient 的实现原理和最终实现持久化 List 的方法。可以看出 Transient 是为了提高持久化 List
在连续修改操作下的效率而引进的数据结构，同时引入 Transient 也会简化持久化 List 实现的复杂度。
但是如果用户以不正确的方式使用 Transient ，可能会破坏持久化 List 的持久性。
在 Transient 存在的情况下，持久化 List 的修改操作被实现为先转换为 Transient 并修改，最终将 Transient 持久化这样的方法。

至此，我们就实现了一个功能较为完整的持久化 List 类。持久化 List 类是持久化数据结构当中最容易实现的一种，
但是通过研究它的实现过程，我们可以体会到实现持久化数据结构的一些主要思路。这篇文章的结束宣告 Functional Go
这一系列 Blog 暂时告一段落。下一个系列将会开始探讨另一类非常重要的数据结构 Map 的持久化实现方法(Hash Array Mapped Trie)。

本文实现的代码已经[开源在 GitHub 上](https://github.com/shanzi/persist/)。按照计划，
配合 Blog 的更新，我也会继续将进一步实现的持久化数据结构添加到这一仓库中。
也欢迎各位读者对我实现的代码提出意见建议或反馈 Bug 以及贡献代码。

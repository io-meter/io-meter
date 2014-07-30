title: Go 语言的包依赖管理
date: 2014-07-30 16:11:44
tags: [Go, package]
category: essay
---

对于从 Ruby、Python 或者 Node 等编程语言转向 Go 语言的开发者，可能会有一个疑问：
Go 语言中的包依赖关系是怎么管理的？有没有什么方便使用的工具呢？
我最近研究了一下这个问题，以下是我的研究报告。

<!-- more -->

![(图片来源：nathany.com)](/img/posts/go-package.jpg)

## Go 语言本身提供的包管理机制

在 Go 语言中，我们可以使用`go get`命令安装远程仓库中托管的代码，不同于 Ruby Gem、pypi 等集中式的包管理机制，
Go 语言的包管理系统是去中心化的。简单来讲，`go get`命令支持任何一个位置托管的 Git 或 Mercurial
的仓库，无论是 Github 还是 Google Code 上的包，都可以通过这个命令安装。

我们知道，在 Go 语言中的`import`语句对于已经使用`go get`安装到本地的包，依然要使用其去绝对路径引入。
比如对于从 Github 上安装的 [goji](https://goji.io/)，其在 Github 上的路径 URL 是
`https://github.com/zenazn/goji`，因此在`import`它的时候需要使用下面的代码：
```go
import "github.com/zenazn/goji"
```

正因为如此，Go 语言可以通过直接分析代码中的`import`语句来查询依赖关系。
`go get`命令在执行时，就会自动解析`import`来安装所有的依赖。

除了`go get`，Go 语言还提供了一个 Workspace 的机制，这个机制也是很容易让人困惑的设计。简单来说就是通过设定
`GOPATH`环境变量，指定除了`GOROOT`所指定的目录之外，Go 代码所在的位置(也就是 Workspace 的位置)。
一般来说，`GOPATH`目录下会包含`pkg`、`src`和`bin`三个子目录，这三个目录各有用处。

* `bin` 目录用来放置编译好的可执行文件，为了使得这里的可执行文件可以方便的运行，
在 shell 中设置`PATH`变量。
* `src` 目录用来放置代码源文件，在进行`import`时，是使用这个位置作为根目录的。自己编写的代码也应该放在这下面。
* `pkg` 用来放置安装的包的链接对象(Object)的。这个概念有点类似于链接库，Go 会将编译出的可连接库放在这里，
方便编译时链接。不同的系统和处理器架构的对象会在`pkg`存放在不同的文件夹中。

我的`GOPATH`目录树如下所示：
```plain
├── bin
├── pkg
│   └── darwin_amd64
│       └── github.com
│           └── zenazn
│               └── goji
└── src
    ├── code.google.com
    │   └── p
    │       └── go.crypto
    └── github.com
        └── zenazn
            └── goji
```

一般来说，你自己的代码不应该直接放置在`src`目录下，而应该为其建立对应的项目文件夹。
`go get`也会把第三方包的源代码放到这个目录下，因此一般推荐设置两个`GOPATH`，比如：

```bash
export GOPATH="/usr/local/share/go:$HOME/codes/go"
```

这样第三方包就会默认放置在第一个路径中，而你可以在第二个路径下编写自己的代码。
虽然 Go 语言本身已经提供了相当强大的包管理方式了，但是仍然有一些不足：

1. 不能很方便地隔离不同项目的环境
2. 不能很方便地控制某个依赖包的版本
3. 不能管理 Go 本身的版本

因此我们还需要一些第三方的工具来弥补这些缺陷。

## 第三方的管理工具

### GOPATH 管理和包管理

由于存在`GOPATH`的机制，我们可以使用多个`GOPATH`来实现项目隔离的方法。
譬如，对于每个项目，都分配一个不同的路径作为`GOPATH`。
可以实现这样的目的的工具有[gvp](https://github.com/pote/gvp)等。

对于 gvp 来说，想要针对当前目录建立一个`GOPATH`，只需要执行`gvp init`即可。
gvp 会在当前项目的目录下新建一个隐藏的文件夹作为`GOPATH`指向的位置。
切换环境时使用下面两个命令来修改环境变量。这种做法跟 Python
中的[virtualenv](https://pypi.python.org/pypi/virtualenv)比较类似。

```bash
source gvp in   # 进入当前目录对应的 GOPATH 环境
source gvp out  # 登出当前目录对应的 GOPATH 环境
```

至于对依赖包更版本更细致的管理，可以配合的工具还有 [gpm](https://github.com/pote/gpm)。
`gpm`有点类似于 Python 中的[pip](http://pip.readthedocs.org/en/latest/)工具。他可以生成一个名为 `Godeps` 的文件，
其中记录了每个依赖包的 URL 以及使用的版本(hash tag)。
之前的[一篇文章](http://dev.af83.com/2013/09/14/a-journey-in-golang-package-manager.html)提到
`gpm`只能管理来自 Github 的依赖，不过当前的版本已经支持了非 Git 方式托管的依赖包了。

基于同样原理管理依赖包版本的工具还有[Godep](https://github.com/tools/godep)。
这个工具在 Github 上具有相当高的关注度。它所生成的`Godeps`文件采用 JSON 格式储存，
是一个跟 Node.js 中 [NPM](https://www.npmjs.org://www.npmjs.org/) 相仿的工具。

总体来说以上几个工具已经可以解决隔离项目环境和控制依赖包版本的问题了。但是使用上还不算方便，
为了能在我们 cd 到某个目录时自动的切换环境变量，我们可能还需要在 shell
做一些配置使其在`cd`到项目目录下时自动切换环境变量。

这方面做的比较好的一个选择是 [Go Manager(gom)](https://github.com/mattn/gom)，
它生成的`Gomfile`格式上几乎跟 Ruby Gem 一样。gom 可能是这些工具当中使用最方便的一个，
只要使用`gom build`命令代替原来的`go build`命令进行编译，你基本不需要配置 Shell 或者和环境变量打交道。

### Go 语言版本管理

对于 Go 语言，一般来说并没有使多个语言版本并存的需求。Go 语言现在还没有经历过类似 Python 2.x 到 3.x
或者 Ruby 1.x 到 2.x 这样破坏性的版本升级。旧的代码在新的语言版本当中一般是能够正确运行的。
不过若遇到非要并存多个版本的时候，[gvm](https://github.com/moovweb/gvm)就是一个不错的选择。

gvm 的使用跟 [rvm](https://rvm.io/) 比较类似。

```bash
gvm install go1 # 安装 go1 版本
gvm use go1     # 修改环境变量使用 go1 版本的 Go
```

##总结

是否有必要使用多个 Workspace 仍然具有争议，譬如这个 StackOverflow
上的[相关问答](http://stackoverflow.com/questions/20722502/whats-a-good-best-practice-with-go-workspaces)中，
就有人提出只使用一个 Workspace 就可以应付大多数情况了。

在研究相关问题的时候，我发现很多 Go 语言的用户都还带着原来编程语言的思维，
这点从上面介绍的多个工具的特点当中就可以很容易看出来：`gvp`和`gpm`就是典型的 Python 的包管理模式，
`gvp`对应着`virtualenv`，`gpm`对应着`pip`；如果你之前是 Node.js 和 NPM 的用户，
那么`GoDeps`肯定会让你有种熟悉的感觉；更不用说最后介绍的`gom`了，它从名称到文件格式都在模仿 Ruby Gem。

不同编程背景的开发者来到 Go 语言之后各自带来了自己的依赖包管理方式，而且形成了各自的社区。
这种现象虽然使得各自圈子的开发者免去了选择恐惧症，但是造成的解决方案分裂和互不兼容的情况也需要正视。
这时我们不禁要问，Go 自己的解决方式应该是什么样的？Go 语言为何没有一个官方标准的解决方案呢？

从[Go FAQ](http://golang.org/doc/faq#get_version)的一段文字当中我们可以得到部分答案：

> Versioning is a source of significant complexity, especially in large code bases, 
> and we are unaware of any approach that works well at scale in a large enough variety
> of situations to be appropriate to force on all Go users.
> (依赖包的版本管理是一个非常复杂的问题，特别是在代码量比较大的时候。
> 我们一直没有找到任何一种方式能够在各种情形下都能良好工作，
> 因此也没有一种方式足够好到应该强迫所有的 Go 用户使用它)

因此现阶段来看，对于 Go 语言的包管理解决方案，我们也就只能“仁者见仁，智者见智”了。

最后，对于想要了解 Go 语言的包管理以及更多可用的工具的读者，这里再推荐两篇相关的文章：
[Go Package Management](http://nathany.com/go-packages/) 和
[A Journey in Golang Package Manager](http://dev.af83.com/2013/09/14/a-journey-in-golang-package-manager.html)

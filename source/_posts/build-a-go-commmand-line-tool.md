---
title: '用 Go 写个小工具：wu 的炼成'
date: 2016-08-14 18:02:47
tags: [golang, programming, wu, filesystem, watch, utility, os, cli]
---

最近使用 Golang 编写完成了一个命令行下的小工具: [wu](https://github.com/shanzi/wu)，
这个小工具的主要用途是监视文件改动并执行指定的命令。尽管有点重新发明轮子的嫌疑，
但是设计和实现它的过程中我还是有不少收获的。

<!-- more -->

我很早就对 Golang 有兴趣了，之前在没有经过系统学习的情况下跌跌撞撞地完成了一个很小的应用，
并最终总结成 [自己动手写 Git HTTP Server](https://io-meter.com/2014/07/09/simple-git-http-server/) 这篇文章，
之后又稍微研究和总结了一下现有的 [Go 语言的包依赖管理](https://io-meter.com/2014/07/30/go's-package-management/)。

自那以后，Go 语言又有了一些新的进展，比如从 1.6 版本引入的 `vendor` 的概念， 
算是在解决包管理问题上的出现新趋势，也催生出 [govendor](https://github.com/kardianos/govendor)
这样新的工具出来。

不过包管理并不是这篇文章想要主要讨论的问题。此前，通过 [The Go Programming Language](http://www.gopl.io/)
这本书，我系统地对 Go 语言各个方面的使用以及部分设计和实践有了更全面的了解，
因此这次以更加“正确”的方法实现了 wu 这个小工具，在此将从构思到实现各方面的思考记录一些下来，
也算是分享一点经验。在这篇文章中，我们会谈到`os/exec`、`flag`、`time`、`encoding/json`、`os/signal`等库的使用。

## 构思和准备

在开始着手实现之前，对于要写什么和怎么实现等方面都进行了一些思考。简单来说，
我首先确定了想要写一个通过监听文件系统修改，从而可以自动重启命令的工具。
写这样一个工具的原因是部分 Light weight 的 web framework 并没有内置自动重新加载的功能。
这样每次修改完代码就需要手动的结束原来的 Server 并重新启动。写一个小工具来自动化这一过程看起来是个不错的主意。

在进一步的构思之前，我简单的进行了一些检索，参考了一些同类的工具（主要是 NodeJS 社区内的一些解决方案），
确定了一些实现的目标：

1. 这个工具应该非常简单和轻量，它所完成的事情就应该是简单的监视文件和执行命令，
   最多添加一些简单的配置，不应需要繁复的设置
2. 这个工具应该具有最少的依赖，相比于 Gulp 和 Grunt 等 NodeJS 的解决方案，
   这一工具应该可以说以便携式可执行文件的方式分发，这也是选择 Go 来实现的一个优势
3. 可配置，应该有一个简单的配置文件，使得用户可以记录下来执行的选项，从而不需要每次敲打复杂的命令

以上三点决定了 wu 的大部分设计，除此之外，在实现之前我也预先考虑了一下可能遇到的问题：

1. 进程通讯的问题：因为 wu 本质上是需要通过启动子进程来运行命令的，因此就需要考虑如何与子进程进行通讯，
如何获得子进程运行的状态，如何强制结束子进程等等问题都需要事先进行一定的研究
2. 并发和并行的问题：除了需要启动和维护子进程，我们的工具还要侦听文件系统的改变，
随着用户的操作很多事件会并发的产生，如何正确地处理这些并发也是一个重要的问题。
幸好我们是在使用 Golang 解决这一问题，在很多地方 Go 语言确实为并发控制提供了很棒的解决方案。
3. 多个文件同时写入问题：这个问题和并发问题比较类似，因为用户可能同时写入多个文件，
这时激发多次进程重启绝不是我们想要的结果。把一段时间内接受到的写入事件合并成一个来处理是一种可能的解决方案。

我们的应用的主要流程大体上可以用下图表示：

![Main Loop](/img/posts/wu-main-loop.png)

从图中可以看出应用的主循环是不断的 Start 和 Terminate 子进程的过程。其中只有红色的`Wait and gather changed files`
步骤会处理文件系统传来的信号，这一步骤也是阻塞的，也就是说如果如果没有信号出现，应用会一直停留在这一步。
我们在这里可以做一个等待，也就是说可以给这一步设定一个最短运行事件，让程序将这一段时间出现的所有文件处理事件都捕捉下来，
从而避免一次保存多个文件导致的重复执行。反过来，在这个阶段之后接收到的文件修改事件将会被留到下一个循环当中处理。
Go 语言的 channel 设计可以让我们方便地做到这一点。

### 文件系统侦听

除了预先考虑一些可能会遇到的问题，我们还要对实现要用到的工具做一个预先的调查。
其中最重要的当然是文件系统改变如何进行侦听的问题。在这里我们使用了一个提供了 Go 语言接口库
[fsnotify](https://github.com/fsnotify/fsnotify)，使用之前我们需要使用`go get`命令先将对应包下载到本地环境中。

```
go get github.com/fsnotify/fsnotify
```

这个工具提供了获得文件修改事件的一个较为底层的访问接口，当指定侦听的文件或文件夹之后，
我们可以获得一个 channel 用来接收事件。fsnotify 的一点不足在于侦听文件夹并不会递归进行，
也就是当使用它侦听了某一文件夹时，这个文件夹子目录下的修改并不会被捕捉到，因此我们必须手动完成这一工作。

### 运行子进程

解决的监视文件系统修改的问题，就要考虑如何该进行子进程的启动、守护和结束控制的方法了。
Go 语言的标准库`os/exec`完全提供了我们所需的接口。我们使用:

```
cmd := exec.Command(name, args...) 
```

可以获得一个 Command 对象，通过调用`cmd.Start()`方法就可以启动子进程。我们有两种方法结束子进程，
一个是通过`cmd.Process.Signal`方法为进程发送一个终止信号，比如`SIGINT`或`SIGQUIT`，
但是有的程序可能会忽略这些信号继续运行，因此一个必要的逻辑就是在一个 Timeout 之后应用还没有结束，
我们需要通过发送`SIGTERM`或`SIGKILL`信号强制终止进程。在最初的实现中，我使用了`cmd.Process.Kill()`
方法来结束进程，却发现了一些意外的问题，在后文中会详细的介绍遇到的问题及其解决方案。

### 命令行参数解析

接下来就是用户交互的方式了。尽管我们的 App 是一个极简的设计，但是还是要给用户一些命令行的接口，
Go 语言的标准库提供了`flag`库方便我们完成这一任务。`flag`的功能非常简单，他有两种的使用方式，
一种是在`main`包的包层级声明指针变量作为接受参数值的位置:

```
var value = flag.String("option", "default value", "help text")
```

这种方法的内部原理是 flag 在内部声明了一个变量并返回了他的指针。同样，
`flag`包也会在全局的一个单例对象上保存一个对这一变量的引用。当`flag.Parse()`被执行时，
保存在单例对象上的所有变量会被赋予解析出来的值。因为我们得到的是变量的一个指针，
因此当我们使用`*value`来获得变量值的时候，将会获得`flag`内部解析出来的值。

如果觉得每次都需要用取指针值操作`*value`来获得变量值的话，我们可以通过在`main`包中的
`init`方法中获取值的方法来实现同样的功能。比如说:

```
var value string
func init() {
    flag.StringVar(&value, "option", "default value", "help text")    
}
```

可见，我们传给`StringVar`的仍然是`value`变量的指针，`StringVar`在内部实际进行的是跟以前类似的操作，
最终当我们执行`flag.Parse()`后，`value`的值也会被变成解析出来的值。这里我们可以看到，
指针的存在使得一些原来可能很难表现的操作变得简单，但是同时也产生了很多的副作用。
比如说，在并发程序中，因为一个变量的引用被传递到多个未知的地方，一段简单的函数的运行过程当中，
变量的值可能会发生意想不到的改变，因而造成了预期之外的结果。在我们实现这一工具的过程当中也会遇到类似的问题需要解决。

好在`flag`操作本身比较简单，也不会并发和重复，这样使用还是安全的。另外，上述情况中我们都是在`main`
函数执行之前进行`flag`参数的声明工作，这是为了确保`main`函数一开始执行`flag.Parse`的时候，所有参数已经声明完毕。
我们自然也可以在函数里再进行这些工作，只要能确保执行的顺序即可。此外，如果你在同一个包的多个文件里声明了多个`init`函数，
这些函数虽然都会被执行，但是执行的顺序是未定义的行为，这也是需要注意的地方。

### 时间

接下来是`time`库的一点介绍，我们使用`time`库来实现`sleep`和`wait`的功能。简单来说，如果想要阻塞一个 goroutine 一段时间，
`time.Sleep(duration)`和`<-time.After(duration)`两种方法都可以使用。请注意第二种方法中的`<-`，
`time.After`方法实际上返回一个`<-chan struct{}`，这个通道会在给定事件之后接受到一个消息，因此当你一开始使用`<-`
操作符试图从通道中取消息时，通道会被阻塞，从而获得与`Sleep`一样的效果。这个方法实际上是非常有用的——尤其是跟
`select`语句配合使用的。当然`time`库还提供了`time.Tick`等多种方法，可供我们获得各种各样基于时间信号的通道。

### JSON 读写

为了引进配置文件功能，我们还引入了`encoding/json`库，在这一工具当中，我们只使用了非常简单的`Decode`和`Encode`功能。
要`Decode`一个文件当中包含的 JSON，使用下面的方法:

```
func readJSON(filename string) SomeType {
    file, err := os.Open(filename)
    defer file.Close()

    if err == nil {
    var obj SomeType
    if err := json.NewDecoder(file).Decode(&obj); err != nil {
        // Fatal
    }
    return obj
    }
    // Fatal
}
```

由于我们希望输出的 Config 文件的 JSON 是有缩进的，在写入时我们使用`json.MarshalIntent`方法将 JSON
输出到`[]byte`中，再直接用`file`的`Write`方法写入文件。

```
func writeJson(filename string, obj SomeType) {
    file, err := os.Create(filename)
    defer file.Close()
    
    if err != nil {
    // Fatal
    }
    if bytes, err := json.MarsalIntent(conf, "", "  "); err == nil {
    file.Write(bytes)
    } else {
    // Fatal
    }
}

```

其中，`SomeType`是一个为接受 JSON 数据而定义的`strunt{}`，值得注意的是，只有 Public 的元素才能被`encoding/json`
库读出和写入。

## POSIX Signal 处理

最后一个值得注意的地方就是，如果用户通过`CTRL-C`向我们发送了终止进程的信号的话，我们如何才能优雅地结束程序。
不做任何操作的情况下程序可能会立刻停止，从而导致我们启动的子进程仍然在持续运行，从而形成了无人监管的幽灵进程。
因此我们有必要捕捉应用程序接受到的 Interrupt 信号，从而可以在此后执行一定的清理操作并终止子进程。

这一点可以通过`os/signal`库完成，下面的代码给出了一个简单的例子:

```
func main() {
    ch := make(chan os.Signal)
    signal.Notify(ch, os.Interrupt)

    for sig := range ch {
    // Signal received!
    }
}
```

## 实现与调试

经过多方研究做好了充分的准备之后，我们终于可以着手编写我们的主程序了。在下面的实现当中，
我们将会大量使用通道(channel)作为 goroutine 之间通讯和同步的工具。同时我们也用到了一些 Go 
语言的使用模式和最佳实践。

### Runner 的实现

首先的需求就是，我们希望以面向对象的方式将我们的主运行循环包装成对象，这样当我们通过`os/signal`
捕捉到用户传来的信号时，就可以通过一个方法来执行退出循环的方法。同样我们也需要一个结构体来保存进行执行的状态，
比如说保留一个对执行 Command 的引用等。

在这里我们要使用到一个简单的模式，那就是如何模仿一般面向对象当中的构造函数模式。总所周知，
由于 Go 语言面向对象的实现模式不同，我们没法强制用户在新建对象和结构体的时候一定要执行我们指定的某一函数。
比如说用户总可以通过`SomeType{"some", "params"}`字面量的形式来生成新的`struct{}`对象。
这在需要对结构体字段正确性进行验证或对某些字段进行自动初始化的时候很不方便。

然而如果更换一个思路，我们其实可以保证用户新建对象时一定要通过构造函数进行。最简单的方法就是通过定义子包来进行访问控制。
我们知道，在一个包中小写字母开通的类型、函数和变量外部都不可以访问，因此通过如下步骤我们就可以模仿传统的构造函数模式了:

1. 定义一个子包，比如`github.com/shanzi/wu/runner`
2. 在子包中定义一个公共的接口，比如`Runner`
3. 在子包中定义一个私有的结构体类型，比如`runner`
4. 为公共的接口`Runner`声明一个构造函数，这个构造函数返回私有的结构体类型`runner`

由于`Runner`是一个接口(`interface`)，它为外界提供了一个类似鸭子类型的方法提示。在 Go 语言的设计当中，
一个类型服从一个接口并不需要显式地声明出来——只要类型提供了接口所声明的所有方法即可。
这一有趣的设定使得外界可以通过接口的定义在一定成都上窥探出所接受到的对象的内部结构而不需要知道对象具体的类型，
因此如果我们在`Runner`的构造函数中放回一个`runner`对象的时候，外界就将`runner`当作`Runner`所定义的那样使用，
从而实现了暴露`runner`所提供的方法的目的。反过来，因此`runner`是私有的结构体，外界也不能直接访问和构造出它的对象来。

由于 Go 语言的接口只能定义方法，如果外界想要获得结构体的属性，就必须通过`Getter`和`Setter`方法。
在我们的设计中，希望将应用侦听的目录路径、侦听的文件匹配模式和命令暴露出来，因此定义了如下的接口和结构体:

```
type Runner interface {
    Path() string
    Patterns() []string
    Command() command.Command
    Start()
    Exit()
}

type runner struct {
    path     string
    patterns []string
    command  command.Command

    abort chan struct{}
}

func New(path string, patterns []string, command command.Command) Runner {
    return &runner{
        path:     path,
        patterns: patterns,
        command:  command,
    }
} 
```

注意到，我们在`runner`结构体当中添加了一个没有暴露的`abort`通道，这个通道配合`select`
将会为我们提供一个优雅地结束 goroutine 的方法: 由于我们并不能从外部强制结束另一运行当中的 goroutine，
因此我们需要通过通道传递信号来通知 goroutine 结束运行。在介绍这个以前，我们先来看 `Start` 方法的实现:

```
func (r *runner) Start() {
    r.abort = make(chan struct{})
    changed, err := watch(r.path, r.abort)
    if err != nil {
        log.Fatal("Failed to initialize watcher:", err)
    }
    matched := match(changed, r.patterns)
    log.Println("Start watching...")

    // Run the command once at initially
    r.command.Start(200 * time.Millisecond)
    for fp := range matched {
        files := gather(fp, matched, 500*time.Millisecond)

        // Terminate previous running command
        r.command.Terminate(2 * time.Second)

        log.Println("File changed:", strings.Join(files, ", "))

        // Run new command
        r.command.Start(200 * time.Millisecond)
    }
}
```

可以看到，`Start`函数其实包含了我们应用主循环的全部内容，它首先构造一个新的`abort`通道，
传递进入`watch`函数调用`fsnotify`开始监听工作，然后通过`range`开始我们的主循环。
在这里`changed`和`matched`都是新生成的通道，`changed`输出对当前目录下所有文件监听获得的事件，
`matched`输出将`changed`当中事件以文件模式匹配过滤后的机构。在这里我们对通道的使用非常像 Python
当中的生成器。不但如此，我们通过将通道串联起来，还可以进行逐级过滤，从而在最后只获得我们关心的内容。
在这里通道不但可以被看作生成器，也可以被看作有些编程语言提供的 Lazy Sequence (惰性求值列表)。
实际上，很多 Gopher 直接将 Channel 当成一个高效且线程安全的队列使用。这种生成器/过滤器也是一种常用的模式。

回到我们的`watch`函数，我们将从这一函数的实现中看到`abort`通道是如何发挥作用的。

```
func watch(path string, abort <-chan struct{}) (<-chan string, error) {
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        return nil, err
    }

    for p := range list(path) {
        err = watcher.Add(p)
        if err != nil {
            log.Printf("Failed to watch: %s, error: %s", p, err)
        }
    }

    out := make(chan string)
    go func() {
        defer close(out)
        defer watcher.Close()
           for {
            select {
            case <-abort:
                // Abort watching
                err := watcher.Close()
                if err != nil {
                    log.Fatalln("Failed to stop watch")
                }
                return
            case fp := <-watcher.Events:
                if fp.Op == fsnotify.Create {
                    info, err := os.Stat(fp.Name)
                    if err == nil && info.IsDir() {
                        // Add newly created sub directories to watch list
                        watcher.Add(fp.Name)
                    }
                }
                out <- fp.Name
            case err := <-watcher.Errors:
                log.Println("Watch Error:", err)
            }
        }
    }()

    return out, nil
}
```

我们可以看到，在`watch`函数中，我们开启了一个新的 goroutine，在这个 goroutine 当中我们进行了以下工作:

1. 如果 abort 通道返回，结束监听，函数返回
2. 如果有文件事件产生，进行初步过滤和处理，对于新建的文件夹，要在这里显式加入侦听当中
3. 发现文件错误，通过 Log 打印出来并忽略

我们把这个`select`语句包含在一个永真`for`循环中，这样除非`abort`信号获得消息，其他的消息处理之后就会立即进入新的循环。

看过了`Start`函数，我们来看`Exit`函数的实现:

```
func (r *runner) Exit() {
        log.Println()
        log.Println("Shutting down...")

        r.abort <- struct{}{}
        close(r.abort)
        r.command.Terminate(2 * time.Second)
} 
```

可以看到，除了一些打 Log 的工作，`Exit`函数的主要工作就是向`abort`通道中传递信息并关闭它。最后结束我们的 Command。

在这里必须简单提到一点，对于这种传递的消息不包含信息量而只有消息的到达本身有含义的通道为什么我们要通过传递`struct{}`
类型而不是写起来更短的`int`或者`bool`来完成呢？这是因为在 Go 语言中只有`struct{}{}`是不占用空间的——他的`size`是`0`。
其他任何类型都不能保证除了通道内部本身的空间使用之外不添加新的空间占用。因此无论是这种通道，还是我们希望使用`map`
模拟集合，都应该用`struct{}{}`做为值的类型。

## Command 的实现

为了方便我们对子进程的管理，wu 的实现当中，我们还将 Command 相关的操作封装到了`github.com/shanzi/wu`包当中，
下面给出了 Command 包定义的接口和构造函数的定义是:

```
type Command interface {
        String() string
        Start(delay time.Duration)
        Terminate(wait time.Duration)
}

type command struct {
        name   string
        args   []string
        cmd    *exec.Cmd
        mutex  *sync.Mutex
        exited chan struct{}
}

func New(cmdstring []string) Command {
        if len(cmdstring) == 0 {
                return Empty()
        }

        name := cmdstring[0]
        args := cmdstring[1:]

        return &command{
                name,
                args,
                nil,
                &sync.Mutex{},
                nil,
        }
}
```

可以看到，在结构体当中，我们除了保存一些传进来的参数，还用`cmd`字段保存了当前运行的 Command 的引用，
此外，因为多个方法可能会并发地修改结构体中元素的值，我们使用`sync`类提供的`Mutex`锁来实现对象的互斥访问。
最后还保留了一个 Channel 用来在进程结束之后获得通知。

Command 的`Start`方法是 wu 当中最复杂的一部分，它首先会强制`Sleep`一段时间，以免子进程在很短的时间里被重复启动，
此后通过`mutex`加锁获得对`command`结构体对象修改的权限，随后构造和启动子进程，并在新的 goroutine 里通过`cmd.Wait()`
来等待进程结束。当进程结束之后将会打印 Log 并通过`exited`通道发布结束消息。

```
func (c *command) Start(delay time.Duration) {
        time.Sleep(delay) // delay for a while to avoid start too frequently

        c.mutex.Lock()
        defer c.mutex.Unlock()

        if c.cmd != nil && !c.cmd.ProcessState.Exited() {
                log.Fatalln("Failed to start command: previous command hasn't exit.")
        }

        cmd := exec.Command(c.name, c.args...)

        cmd.Stdin = os.Stdin
        cmd.Stdout = os.Stdout
        cmd.Stderr = os.Stdout // Redirect stderr of sub process to stdout of parent

        // Make process group id available for the command to run
        cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}

        log.Println("- Running command:", c.String())

        err := cmd.Start()
        exited := make(chan struct{})

        if err != nil {
                log.Println("Failed:", err)
        } else {
                c.cmd = cmd
                c.exited = exited

                go func() {
                        defer func() {
                                exited <- struct{}{}
                                close(exited)
                        }()

                        cmd.Wait()
                        if cmd.ProcessState.Success() {
                                log.Println("- Done.")
                        } else {
                                log.Println("- Terminated.")
                        }
                }()
        }
}
```


Command 的`Terminate`方法也利用到了`select`语句，它的主要逻辑是，先给子进程发送`SIGINT`信号促使子进程自然退出，
此后用`select`同时侦听`exited`通道和`time.After(wait)`通道，以便在`SIGINT`失效的情况下设法强制退出。
前面提到过，`time.After(wait)`会返回一个在给定时间后发送消息的通道，这里使用`select`从两个通道当中选择先得到的消息，
因此当`wait`时间过后`exited`还没有消息传来，就会进入强制退出的分支。这就是一般在 Go 语言中实现 Timeout
的模式或者说方法。

目前为止，已经将 wu 的主要代码逻辑介绍完了，在之后的调试当中，主要发现和修正了两个比较严重且有代表性的问题，
那就是空命令的问题和结束命令的问题。

### Empty Command

第一个问题在于，如果用户没有给定运行的 Command 程序应该如何处理的问题。在 wu 里，我选择了什么也不做。
在这里，我们并没有通过分支语句来在函数中进行判断。得益于 Go 语言接口类型的设计，我们并不一定要在`Command
构造函数里返回`command`结构体——任何服从`Runner`接口的类型皆可。为此，我们可以使用最简单的方式定义一个空的 Command 类。

```
// An empty command is a command that do nothing
type empty string

func Empty() Command {
        return empty("Empty command")
}

func (c empty) String() string {
        return string(c)
}

func (c empty) Start(delay time.Duration) {
        // Start an empty command just do nothing but delay for given duration
        <-time.After(delay)
}

func (c empty) Terminate(wait time.Duration) {
        // Terminate empty command just return immediately without any error
}
```

### Kill Command

之前提到过，`os/exec`包中的`Command`类其实可以通过`cmd.Process.Kill()`方法来结束。在一般的执行当中都取得了成功。
但是我却发现当使用`wu go run main.go`启动一个 Web Server 时，在文件修改后旧的子进程总是不能被正确地结束。
显示发送`SIGINT`没有效果，之后执行了`Kill`函数之后，尽管`go run`命令退出，但是 Web Server 仍然在运行，
因此导致了端口占用的问题，使得新的 Command 执行失败。

经过检索后发现，这是因为`go run`命令实际上相当于`build`和执行两条命令，它本身也是通过子进程来运行编译好的新进程，
因此当信号发送给`go run`时，它运行的子进程本身没有收到`SIGINT`因此并不会退出。`go run`也因为一直等待子进程而保持运行。
最后当执行`Kill`函数之后，只有`go run`命令被结束，而他的子进程仍然在执行当中。

知道原因之后就可以提出解决方案了，首先在执行 Command 之前，我们要强制新的 Command 和他的子进程可以获得 Group PID。

```
// Make process group id available for the command to run
cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
```

此后，我们需要自己实现一个`kill`函数，为整个子进程组都发送同样的信号:

```
func (c *command) kill(sig syscall.Signal) error {
        cmd := c.cmd
        pgid, err := syscall.Getpgid(cmd.Process.Pid)
        if err == nil {
                return syscall.Kill(-pgid, sig)
        }
        return err
}
```

由此，我们模拟了用户在按下`CTRL-C`后 Shell 的行: 为整个进程组发送结束信号。这样，我们运行的 Command
就可以保证被正确结束了。当然，这一套操作只在 *NIX 操作系统上可用。在 Windows 上并没有这样的信号机制——
还好，wu 并不需要支持 Windows。

## 总结

![Screen Shot](/img/posts/wu-screenshot.png)

wu 是我完成的又一个有点规模的 Go 语言的应用。由于这次对于 Channel 和 Go 的一些理念有了更深的认识，
编写代码也更加顺畅，也更能体会出一歇 Go 设计上的优点了。

Go 语言在很多方面的设计上确实有独到之处，比如使用通道作为并发同步的工具，而 Channel 的作用又不仅限于此，
它还可以用来模拟队列、生成器、惰性列表，用来实现多级过滤模式等等。Go 语言的接口和面向对象的设计在很多时候也非常灵活。

然而，没有范型使得容器类难以实现，没有异常捕捉使得很多函数调用有点啰嗦等问题也确实为代码的编写引入了一些麻烦。
虽然引入了 Vendor ，支持`internal`包的概念，但是总体来说 Go 语言在包管理上仍然有很大的提升空间。

我个人使用 Go 语言的体验还是不错的，接下来一段时间仍将在这门语言上在做一些研究。


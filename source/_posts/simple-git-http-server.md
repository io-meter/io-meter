title: '自己动手写 Git HTTP Server'
date: 2014-07-09 21:26:54
tags: [Git, golang, go, web, server, rpc]
category: web
---

在 Github 上可以使用 HTTP 协议 fetch 和 push 仓库中的代码，
其实想要写一个简单的 Git HTTP Server 是相当容易的。
这篇文章总结了使用 Go 语言实现这样一个 Server 的过程和相关知识。

<!-- more -->

# 基本原理

先介绍一下 Git HTTP Server 的实现原理。本地的 Git 在使用 HTTP 协议访问远程的 Git 仓库时，
会分别发起如下三种类型的请求：

1. `GET /:working_path/*` 直接 Serve 远程仓库的静态文件到客户端，这里就是本地的仓库从上游获得数据的地方
2. `GET /:working_path/info/refs` 用于访问远程仓库的 refs 数据，比如都有哪些 Branch 和 Tags 等等
3. `POST /:working_path/git-<command>` 用于在远程仓库执行指令，进行数据交流。Git 的 push
和 fetch 都要依赖这个请求来完成

在开始之前，我们首先定义一个 gitRoot 路径，所有的远程仓库在服务端都存放在这下面。譬如：

```go
var gitRoot = path.Join(os.TempDir(), "git_repo")
```

此外，为了方便路由，我选用了一个相当轻量级的 Go Web 库 [Goji](http://goji.io/)。
用下面的命令安装这个依赖:

```shell
go get github.com/zenazn/goji
```

做完准备工作就要开始实现 Server 了。接下来由从最简单的第一类请求讲起：

# 处理静态文件请求

静态文件请求，也就是前面所说的第一类请求，Git 会直接通过这类请求来访问远程仓库的数据文件，
譬如`HEAD`、`refs`等。客户端的 Git 主要通过这种方式来获取一些 Git 自己产生的文件的详细内容，
一般来说并不会直接请求仓库里的代码等文件。类似`git clone`和`git fetch`这样的操作会交给第二类请求。

在 Go 中使用`http`库实现处理静态文件请求是非常方便的。我们定义一个名为`generic`的函数用来处理相关请求：
```go
func generic(c web.C, w http.ResponseWriter, r *http.Request) {
    reponame := c.URLParams["reponame"]
    repopath := path.Join(gitRoot, reponame)
    filepath := path.Join(gitRoot, r.URL.String())
    if strings.HasPrefix(filepath, repopath) {
        http.ServeFile(w, r, filepath)
    } else {
        w.WriteHeader(404)
    }
}
```
这里使用了`filepath := path.Join(gitRoot, r.URL.String())`这行代码来计算远程仓库在文件系统上的真实位置，
这行代码也可以改写成任何其他策略产生的路径。只要这个路径指向的位置是一个通过`git init --bare`方式创建的仓库。
在`main`函数中用下列方法绑定路由并开启服务器。

```go
func main() {
    // access file contents
    goji.Get("/:reponame/*", generic)
    goji.Head("/:reponame/*", generic)

    // start serving
    goji.Serve()
}
```

这样，对于所有指向`/:reponame/`下任意路径的请求都会用`generic`函数处理，这里允许请求方法为`GET`和`HEAD`。
返回静态文件则使用`http`包中的`ServeFile`函数直接完成。

这样，第一类请求就可以被轻松处理了。

# info/refs

接下来是稍微复杂一点的`info/refs`请求。本地的 Git 一般会在`git fetch`的时候进行这样的请求。
这个请求主要用来返回仓库的 References 数据，包括 Branch、Tags 等。跟上面讲过的第一类请求不同，
这个请求需要服务端将相关数据总结整理成特定的格式返回给客户端，从而节省请求次数。

根据用户操作类型的不同，Git 还会附加一个 URL 参数`service`。`service`一般的取值包括
`git-upload-pack`、`git-receive-pack`等。分别是客户端`fetch`和`push`时的状态。

我们并不需要了解这些请求的细节，只需要按照格式返回数据即可。那么该怎么返回正确的数据呢？
如果是对 Git 非常熟悉的同学可能会知道`upload-pack`和`receive-pack`本身就是 Git 可以使用的命令。
这两个命令都可以接受一个`--advertise-refs`参数。

根据 Git 的[文档](http://git-scm.com/book/en/Git-Internals-Transfer-Protocols)
`--advertise-refs`可以使 Git 列出仓库所拥有的 References 以及那些客户端希望抓取到的 References。
而这些数据就被称为 Advertise Refs。

此外我们还可以使用`--stateless-rpc`参数。这个参数使强制 Git 使用无状态的
[RPC](http://zh.wikipedia.org/wiki/%E9%81%A0%E7%A8%8B%E9%81%8E%E7%A8%8B%E8%AA%BF%E7%94%A8)。
简单来说，可以让 Git 返回的 References 列表中，代表每个 Reference 的字段长度恰好和一个数据包的长度相同，
这样每个 Reference 都可以被放在单独一个数据包上传输，不依赖于前面的数据包(也就是实现了所谓的“无状态”)。
有些时候，在这里获得到了某个 Reference 还需要再去请求更详细的信息，这种方式下提高了并行性。

返回的数据的 MIME type 必须是 `application/x-<command>-advertisement`
其中`<command>`就是调用时用的命令。此外在`git receive-pack`和`git upload-pack`返回的数据的基础上，
我们还需要给数据添加一个头信息。其格式如下图所示:

![](/img/posts/git-info-refs-head.png)

把以上这些写成函数就是下面这样。请重点注意其中有关`serverAdvert`的操作。
这里还使用了 Go 语言标准库中的`exec`包。
```go
func inforefs(c web.C, w http.ResponseWriter, r *http.Request) {
    reponame := c.URLParams["reponame"]
    repopath := path.Join(gitRoot, reponame)
    service := r.FormValue("service")
    if len(service) > 0 {
        w.Header().Add("Content-type", fmt.Sprintf("application/x-%s-advertisement", service))
        gitLocalCmd := exec.Command(
            "git",
            string(service[4:]),
            "--stateless-rpc",
            "--advertise-refs",
            repopath)
        out, err := gitLocalCmd.CombinedOutput()
        if err != nil {
            w.WriteHeader(500)
            fmt.Fprintln(w, "Internal Server Error")
            w.Write(out)
        } else {
            serverAdvert := fmt.Sprintf("# service=%s", service)
            length := len(serverAdvert) + 4
            fmt.Fprintf(w, "%04x%s0000", length, serverAdvert)
            w.Write(out)
        }
    } else {
        fmt.Fprintln(w, "Invalid request")
        w.WriteHeader(400)
    }
}
```

# RPC 远程调用

第三类请求是数据调用请求。简单来说就是由客户端 Git 指定命令在服务端对应仓库执行，
执行的过程中，客户端 Git 将命令需要的输入当做 HTTP 的 POST 数据发送到服务端，
服务端需要将这些输入转发给本地 Git 命令的标准输入流(stdin)，反过来还要将命令的标准输出(stdout)
当做 HTTP 请求的返回数据转发给客户端。

这个流程说起来很绕，但其实还蛮简单的。其通讯的模型可以用下图来表示：
![](/img/posts/git-server-rpc-model.png)

为了在 Go 中完成这样的转发，我们首先要设法从 Request 对象中读出 Body 的内容来。
为了节约传输大量数据时消耗的内存，把所有发送来的数据都读取出来再写入是不能接受的，
因此我们要以串流的方式来从 Body 中读取数据并同时写入 stdin。

为了获取所执行的命令的输入输出流，我们需要调用`exec.Command`类提供的`StdinPipe()`和`StdoutPipe()`方法。
串流转发的功能只需要方便地使用`io.Copy`方法即可。`io.Copy`方法默认将会以每次 32KB 的块大小读出数据然后写入到目标文件中。

众所周知，基本的 HTTP 是一种半双工的通讯协议，数据的发送和接收是不能交错进行的。所以我们也只能先把 HTTP 
请求的内容串流给 Git 命令的 stdin，完成后再从 stdout 里读取数据发送回客户端。

最后执行完`receive-pack`命令，还要执行`git update-server-info`命令来更新服务端仓库的信息。
Response 的 MIME 是`application/x-git-<command>-result`。

具体实现如下:

```go
func rpc(c web.C, w http.ResponseWriter, r *http.Request) {
    reponame := c.URLParams["reponame"]
    repopath := path.Join(gitRoot, reponame)
    command := c.URLParams["command"]
    if len(command) > 0 {

        w.Header().Add("Content-type", fmt.Sprintf("application/x-git-%s-result", command))
        w.WriteHeader(200)

        gitCmd := exec.Command("git", command, "--stateless-rpc", repopath)

        cmdIn, _ := gitCmd.StdinPipe()
        cmdOut, _ := gitCmd.StdoutPipe()
        body := r.Body

        gitCmd.Start()

        io.Copy(cmdIn, body)
        io.Copy(w, cmdOut)

        if command == "receive-pack" {
            updateCmd := exec.Command("git", "--git-dir", repopath, "update-server-info")
            updateCmd.Start()
        }
    } else {
        w.WriteHeader(400)
        fmt.Fprintln(w, "Invalid Request")
    }
}
```

# 路由的安排

为了使路由按照正确的顺序匹配，应该要注意绑定请求时候的顺序。第一类请求虽然最简单，
但是他的匹配模式可以完全包含后两种，因此正确的顺序应该是：

1. 优先匹配 info/refs 请求
2. 其次匹配 RPC 请求
3. 以上两个都没有匹配，则匹配静态文件请求

实现为:

```go
    // get repo info/refs
    goji.Get("/:reponame/info/refs", inforefs)
    goji.Head("/:reponame/info/refs", inforefs)

    // RPC request on repo
    goji.Post(regexp.MustCompile("^/(?P<reponame>[^/]+)/git-(?P<command>[^/]+)$"), rpc)

    // access file contents
    goji.Get("/:reponame/*", generic)
    goji.Head("/:reponame/*", generic)
```

值得注意的是，只有 RPC 请求因需要写入文件而允许使用 POST 方法，另外两种请求都是 GET 或 HEAD 方法。

# Security

为了限制对仓库的访问，我们还可以使用 HTTP 的 [Basic Auth](http://en.wikipedia.org/wiki/Basic_access_authentication)
协议对进行用户认证。

首先，对于未认证的请求，需要返回 Code 为 `401 Unauthorized` 的 HTTP Response，
并在 Response 的 Header 上加上如下格式的字段。

```plain
WWW-Authenticate: Basic realm="[your realm]"
```

其中`realm`用来指定服务器的名称或者 UID，Git 会根据 realm 的值的不同，
使用本地缓存的不同用户名/密码组合。如果没有对应密码，Git 就会请求用户输入。

如果用户输入了密码，在 HTTP 请求的 Header 中将会出现下面格式的一个字段，其中`[username:password]`
的内容使用`base64`编码，需先解码：

```plain
Authenticate: Basic [username:password]
```

可以将上述 Authenticate 的逻辑通过 Goji 提供的 Middleware 机制实现，这样可以使对所有请求都会要求认证。
实现细节不再赘述，可以参见[我实现的版本](https://gist.github.com/shanzi/1aa571f8f3b8f4608d60#file-gittp-go-L173)。

# 总结

为了方便使用，我还为 Server 添加了创建和删除远程仓库的代码，具体实现已经全部放在
[Gist](https://gist.github.com/shanzi/1aa571f8f3b8f4608d60)上。当服务器开始运行之后，可以简单的使用 curl 
命令在服务器上创建空白仓库以及删除已有仓库。
```bash
# create remote repo
curl -u username:password -X PUT http://localhost:8000/new_repo
# delete remote repo
curl -u username:password -X DELETE http://localhost:8000/exists_repo 
```


我们目前实现的版本得益于 Git 本身和 Go 语言的强大，总行数还不超过 250 行，
请[查看完整代码](https://gist.github.com/shanzi/1aa571f8f3b8f4608d60)。

不过我们还没有考虑到 Git 对 GZip的支持。尤其对于第一类对静态文件的请求，
支持 GZip 将能够显著提高传输效率。这个需求使用 Go 语言自带的`compass/gzip`包可以很方便的实现。

最后来看一下成果吧！

![](/img/posts/git-simple-server-result.png)

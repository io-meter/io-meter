---
title: Play with Intel NUC
date: 2016-05-29 12:50:22
tags: [NUC, Linux, Docker, Dokku, Splunk, Ubuntu, Intel, IoT, Transmission]
---

话说最近入手了一台 Intel NUC 用来做下载机，买回来之后为了更加充分的利用这台小主机的性能，那么呢，我就做了三件小事：
第一件事就是给 NUC 装上了 Ubuntu 14.04 系统，并配置了 Docker 和 Dokku 的环境；第二件事呢，部署了 Transmission 下载服务和NFS；
第三件事，就是给 NUC 安装了 Splunk 做 Monitor。如果说还有什么其他事情的话，
那就是写了这篇博客，供大家娱乐娱乐，这也是很大的，但是关键还是前面那三件小事儿。很惭愧，只做了一点微小的工作。

<!-- more -->

# Intel NUC 简介

所谓的 [Intel NUC](http://www.intel.com/content/www/us/en/nuc/overview.html)，
指的就是下图当中由 Intel 原厂提供的小主机。NUC 迄今为止已经出到第六代了,
其配置从低端的 i3 版本到高端的 i7 版本都有。Intel NUC 原装配置是不含内存和硬盘的，但是一般的淘宝店家都会提供套餐。
最新的 6 代 NUC 分为薄厚两种，厚版除了一块插在主板上的 SSD 外，还可以另配一块笔记本硬盘或 SSD。
尽管无论哪个版本的 NUC，其 CPU 等都是配置的笔记本级别的低电压硬件，不能和台式机相比。
但是作为 x86 架构的微型主机，NUC 的性能还是相当令人满意的。除了可以流畅运行包括 Windows 在内的各种操作系统，
最新的第六代还可以支持 4K 视频显示。

![Intel NUC](/img/posts/intel-nuc.png)

要说我买 Intel NUC 这件事，那可以说也是历史的行程。之前在学校的时候，我一直使用实验室的主机做下载机，
现在毕业了，原来的主机自然不好继续使用，但是我还是有固定的下载需求的。如果能在内网做部署一台下载机，
我就可以让它离线挂机下载，下载到的视频文件还是直接在线串流播放，避免了再下载到笔记本上的繁琐操作。

单就下载这个功能来说，其实并不需要太好的性能。现在有的路由器和开发版都可以处理得了。
在我面前的选择主要是树莓派、Intel NUC 和 Gen 8 三种。这里要提一下 HP 的 Gen 8，它是专门开发来家用的 NAS，
可以说最为符合我的需求。Gen 8 具有良好的扩展性，可以挂载多块大容量硬盘，CPU 也可以更换。
最终选择 NUC 的原因在于，它相对于 ARM 架构的树莓派更加具有适用性，性能也比较强。而 Gen 8 原装的 CPU
略微不能满足的需求，升级 CPU 的成本也比较高。NUC 还有一个优点是内置硬件和接口非常齐全，视频上支持 HDMI 和 DP 两种接口，
自带千兆以太网卡、蓝牙和 WIFI，除此之外还有红外接口等。

在综合以上这些考虑之后，我就念了两句诗，最终购买了配备 i3 CPU 的 NUC6i3SYH 版本。淘宝套餐当中还包括 8G 内存、
128G SSD 和遥控器。这样以后如果有屏幕或电视的话，NUC 也可以用来做家用的媒体中心。

# 安装 Ubuntu 14.04 和 Dokku

Ubuntu 14.04 并不是 NUC 的首选。事实上，Intel 官方的 Linux 显卡驱动只兼容到 15.10 ，
这导致在 14.04 下的显示性能会比较差。当时我最终还是选用它的原因在于而我希望部署的 Dokku
目前仍然被 Lock 在作为 LTS 版本的 14.04 上。

为什么一定要考虑 [Dokku](https://github.com/dokku/dokku) 呢？Dokku 是一个极精简的 PaaS 平台，它可以在你的 VPS 和主机上模拟 Heroku
那样的使用体验——只需要将你写好的代码仓库 push 到主机上，Dokku 就会替你完成一系列部署工作，
将你的 Web 服务运行在一个 Docker container 当中并配置好 Nginx，这使得你可以方便地自定义域名。
再加上 Dokku 提供一系列方便的插件用来管理各种 Service ，比如各种关系型数据库、NoSQL、memcached 等等，
这些服务都是跑在各自的 Docker 容器内，具有很好的隔离性。Dokku 甚至有一个插件帮你加入和配置 [Let's Encrypt](https://letsencrypt.org/) 证书。
这一系列方便的特性使得想要充分利用 NUC 性能部署更多服务的我来说是具有极大吸引力的。

NUC 到货的时候系统是预装的 Windows，想要在没有屏幕的情况下换装成 Ubuntu 还是花费了我一些心思。
后来还遇到 NUC 在家里的 WIFI 环境中丢包严重的问题，总之颇费了一些周折。在系统准备好了之后，
就正式开始 Dokku 的安装过程了。

在一般情况下，我们只需要在终端当中执行下述命令就可以成功安装 Dokku 了。

```
wget https://raw.githubusercontent.com/dokku/dokku/v0.5.7/bootstrap.sh
sudo DOKKU_TAG=v0.5.7 bash bootstrap.sh
```

这一命令会依次完成检查环境、安装必备依赖安装并设置 Dokku 的工作。然而这一招在国内的网络上并不好使，
究其原因，还是因为部分资源收到了 GFW 的干扰，导致安装缓慢以至于无法安装。如果有条件的话，建议先配置好代理或 VPN
等翻墙方案再安装。除此之外，我在安装的最后一步还遇到了 `gnu_utils` 爆出的 HTTPS 认证错误。
导致依赖安装完成之后无法继续。这一方面可能是国内网络的原因，另一方面可能跟 Dokku 使用的一个第三方镜像源托管站
[Package Cloud](https://packagecloud.io) 自身的问题有关。解决这一问题的方法是我们先手动把缺失的最后几个包使用 Wget
工具下载下来，再使用 `dpkg` 命令手动安装。

```
wget https://packagecloud.io/dokku/dokku/packages/ubuntu/trusty/gliderlabs-sigil_0.4.0_amd64.deb
wget https://packagecloud.io/dokku/dokku/packages/ubuntu/trusty/sshcommand_0.4.0_amd64.deb
wget https://packagecloud.io/dokku/dokku/packages/ubuntu/trusty/sigil_0.4.0_amd64.deb
wget https://packagecloud.io/dokku/dokku/packages/ubuntu/trusty/dokku_0.5.7_amd64.deb
wget https://packagecloud.io/dokku/dokku/packages/ubuntu/trusty/herokuish_0.3.13_amd64.deb

dpkg -i gliderlabs-sigil_0.4.0_amd64.deb sshcommand_0.4.0_amd64.deb sigil_0.4.0_amd64.deb dokku_0.5.7_amd64.deb herokuish_0.3.13_amd64.deb
```

注意，在这里最好将 `herokuish` 这个包最后一个安装，这个包实际上是 Dokku 整个功能的核心，它会给 Docker import
一个叫做 [Herokuish](https://github.com/gliderlabs/herokuish) 的镜像，这个镜像则是另一个类似 PaaS 工具 [Deis](https://deis.io/) 提供的。
由于众所周知的原因，从 DockerHub 当中导入镜像的速度十分缓慢，因此在这一步很容易出错。建议一定要使用 Tmux 或 Screen
这样的工具保证进程可以在后台执行。在 `heorkuish` 安装的最后一步会在对应的 Docker Image 之中执行 `herokuish install`
命令，这一过程很遗憾也是会因为网络原因失败的。如果你使用的是代理方式翻墙，此时在 Docker 
当中运行的命令可能无法获得你的代理配置，因此在这一步最好在 VPN 环境下进行操作，或者直接使用下面的命令进入 Docker
容器手动执行。

```
docker run -it gliderlabs/herokuish /bin/bash

# Now in docker environment
export http_proxy=[your proxy config]
export https_proxy=$http_proxy

herokuish install
ln /bin/herokuish /start
ln /bin/herokuish /exec
ln /bin/herokuish /build
exit # Exit from docker

# Out of Docker
docker ps -a # Find containter id of recent changes
docker commit [container id] gliderlabs/herokuish:latest
```

这样，我们就完成了基本的设置。最后，我们只需要在主机上执行下述命令就可以完成 Dokku 的初始化设置，
在这一步当中，Dokku 将会自动为系统添加一个名叫 dokku 的新用户，并提供一些交互式的选项以供选择。

```
dokku plugins-install-dependencies
```

Dokku 的实现原理是什么呢？其实 Dokku 本身只是一系列脚本和插件组成的插件集合，这些脚本的做用是方便用户管理和修改配置。
模拟 Heroku 的工作主要是 Herokuish 提供的，这个包会模仿 Heroku 的行为来检查你所要部署的代码类型、
自动安装依赖并 build 成 Docker Images。这一工具提供的功能几乎和 Heroku 完全一样，甚至大多数为 Heroku 编写的 buildpack
都可以无缝拿来使用。唯一的区别是 Dokku 使用 Docker 作为容器而 Heroku 则实用的是自己开发的容器系统。


`sshcommand` 也是一个重要的命令，当你使用 `git push` 将代码 Push 到 Dokku 这里时，Dokku 需要使用你的 Public Key
来验证你的身份。在这一步当中，使用平常使用 Public Key 登录的方法将 `id_rsa.pub` 的内容加到 `.ssh/authorized_keys`
里是不管用的——`git`使用不同的方法来验证。因此需要使用 sshcommand 来添加公钥。例如:

```
cat .ssh/id_rsa.pub|sudo sshcommand acl-add dokku chase@chase-nuc
```

只需要将你的 `id_rsa.pub` 的内容作为 `sshcommand` 的标准输入即可，这一步可以通过 `ssh` 来做，
从而也可以把本地机器的公钥加入。

接下来就可以像使用 Heroku 那样使用 Dokku 了，更详细的 Deploy 教程可以查看 Dokku
的[官方文档](http://dokku.viewdocs.io/dokku~v0.5.7/application-deployment/)。


# Transmission & NFS

配置完 Dokku，要做的第一件事当然就是将 Transmission 部署进去了。[Transmission](https://www.transmissionbt.com)
是一款知名的 BT 客户端，他的主要特点就是提供了 Web 访问接口，从而很适合用来安装在下载机上进行远程控制。
把 Transmission 部署在 Docker 里其实不算是罕见的需求了，Github 上已经有一批现成的 Dockerfile。
参考其中两个，我修改出来一个适合部署在 Dokku 中的版本。因为 Dokku 实际上是允许使用 Dockerfile 进行 Build，
所以要注意的问题主要只有一下几点：

1. Transmission 默认需要开放 `51413` 端口的 TCP 和 UDP 访问作为 Peer 连入的端口，但是如果直接在 Dockerfile
里 EXPOSE 出来，Dokku 会自动使用 Nginx 为这些端口设置代理，而 Nginx 是不支持 UDP 代理的，
所以我们并不在 Dockerfile 中指定 Expose 端口
2. Dokku 会自动为 Build 出来的 Docker Image 添加一个 `PORT` 环境变量，默认会使用 Nginx 将对应域名的 `80`
端口 Proxy 到 Docker 容器的这个端口，因此我们希望将 Transmission 的 Web Interface 暴露到这里
3. 很自然地，我们需要为 Transmission 所在的 Docker 容器挂载一些外部文件系统。
其中下载目的位置希望能够映射到主机的固定位置。但是 Dockerfile 本身并不支持直接指定主机挂载的目录。
4. Dokku 的 zero down time deploy 功能会在重新部署的时候检查新的容器是否启动成功，
只有在启动成功之后才会关闭老的容器。这一功能在部署一般的 Web 服务时非常方便，
但是在部署 Transmission 时，由于 Docker 会绑定 PEERPORT，因此会导致新的进程无法启动成功。
因此需要 disable 这个功能，同样在每次部署之前也要手动 Shutdown 老的容器。

为了解决这些问题，最后只能手动地完成一些设置工作。我将修改好的 Dockerfile 和运行脚本放在了 GitHub 上的
[dokku-transmission](https://github.com/shanzi/dokku-transmission) 仓库上了。如果要使用它，只需要先 clone 下来，
然后在 Dokku 所在机器上执行下述命令新建 App :

```
docker apps:create [your_app_name]
```

预先为 App 设置一些环境变量，譬如 Web Interface 的用户名和密码等:

```
docker config:set [your_app_name] USERNAME=[username] PASSWORD=[password] PEERPORT=51413
```

在之后，最重要的一步就是使用 Dokku 为运行 App 添加一些 Docker Options，这些 Docker Options
将会在 Dokku 每次启动 Docker 容器时追加作为参数，因此我们可以在这里指定主机相关的设置:

```
export $PEERPORT=51413
dokku docker-options:add [your_app_name] deploy,run "-v [PATH_ON_HOST_MACHINE]:/root/Downloads -p $PEERPORT/udp -p $PEERPORT/tcp"
```

我们可以用如下命令禁用掉 Dokku 对我们的 Transmission app 进行 zero down time 检查:

```
dokku checks:disable [your_app_name]
```

最后，只需将仓库 Push 到 Dokku 这里，Transmission 进程就可以跑起来了。

尽管使用上麻烦一点，但是好在我们并不需要频繁地更新我们的 App。Dokku
会帮我们做好进程的守护工作——在应用挂掉或者系统启动时都会自动帮我们启动进程。

下图就是 Transmission Web Interface 部署成功后的样子了，终于可以开心的下载 BT 资源了！

![Transmission Web](/img/posts/transmission.png)

把东西下载到了 NUC 上，还要想办法把这些东西通过网络暴露出来访问。除了传统的 HTTP server
和 FTP 等方法，我选择了使用 NFS。这样在其他机器上就可以将 NUC 上对应的文件夹通过网络挂载起来了。

安装 NFS 的方法，来自 Digital Ocean 的[这篇文章](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-14-04)
讲的很好了。简单来说我们只需要通过 `sudo apt-get install nfs-kernel-server` 命令来安装 NFS Server，
之后再在 `/etc/exports` 文件中配置目录设定即可。需要注意的是，如果想要 Mac 可以挂载 NUC 暴露出来的 NFS，
必须要添加 `insecure` 选项，否则会连接失败(尽管可以通过在终端中输入命令成功挂载)。我的配置文件内容如下:

```
/var/nfs 192.168.0.0/24(rw,sync,no_subtree_check,insecure)
```

接下来就可以在 Mac 上通过 Finder 的 Go > Connect to Server 功能直接挂载 NFS 了。

![NFS](/img/posts/nfs.png)

顺便说一下，NFS 除了这种使用，还可以代替传统的 Volume 挂载方法，作为向 Docker 容器提供储存的一种解决方案。
此外，大规模分布式使用中，还有 [Flocker](https://github.com/ClusterHQ/flocker) 这种工具。
不过杀鸡焉用牛刀，在 NUC 上我就选择了简单直接的解决方案。

# Splunk for Monitor

利益相关，这里就稍微介绍一下 [Splunk](http://splunk.com) 吧。一般原始的方式处理 Log 当中的数据，我们都是直接写出简单的脚本来扫描 Log
文件进行处理。这种方法对于偶尔的需求还是可以处理的，但是一来需要做很多重复操作，二来不适合数据量大以及分布式的应用。
传统的脚本处理方法也并不怎么用户友好。那么一种同时面向技术和非技术类用户的产品就应运而生。
这样的工具完成了以下几个工作:

1. Log 的收集: 从单机或分布式节点当中收集 Log 数据，发送到处理节点
2. Log 的索引: 对收集到的 Log 数据建立索引，方便日后查询
3. Log 的分析: 允许使用一定的接口从索引当中检索结果和数据并拿来可视化

Splunk 是目前较为成熟的企业 Log 数据处理工具。与之相似的比较成熟的解决方案就是 [ELK](https://www.elastic.co/products) 了。
ELK 分别代表 Elasticsearch、Logstash 和 Kibana，这三个组件分别完成了索引、Log 收集和可视化三部分的工作，
是目前炙手可热的开源解决方案。互联网企业很多都选择了这种开源的技术，可以说对 Splunk 的产品造成了很大的威胁。

回到 Splunk，尽管它的免费版本一天只能处理 500M 的数据量，但对于 NUC 上的使用已经足够了。Splunk 作为企业应用，
总体来说使用体验和部署的友好程度还是不错的。从[这里](https://www.splunk.com/en_us/download/splunk-enterprise.html)
下载 Splunk 对应 Linux 的安装包，安装或解压到 `/opt/splunk` 即可。安装结束之后我们需要第一次运行一下
`/opt/splunk/bin/splunk` 程序， 这一步 Splunk 会要求我们接受 License 等，我们用下面一条命令全部跳过:

```
/opt/splunk/bin/splunk enable boot-start --accept-license --answer-yes --no-prompt
```

这一步同时也会安装一些 `init` 脚本到系统当中，Splunk 服务此时就已经安装成功了，我们可以使用下述命令启动它:

```
service splunk start
```

Splunk 进行可视化的原理是什么呢？其实它定义了一门叫 SPL 的语言专门用来进行查询，查询的结果都可以以表格的形式显示出来。
而 Splunk 的可视化图表也可以以 SPL 的搜索结果作为输入，因此就可以以一种流水管线的形式将数据送入可视化组件进行表示。
Splunk 还支持 Realtime 的数据展示，这正是我们所需要的。

我的计划是将 Splunk 作为系统的 Monitor。让他可以显示当前 CPU、内存和硬盘等的使用情况。原理很简单，
既然 Splunk 是一个 Log 处理工具，那我们以一定的时间间隔将 CPU 占用、内存占用等情况以 Log 的形式打印出来，
这样 Splunk 不久可以处理了么？事实上 Splunk 官方就有一个 [Splunk App for \*NIX](https://splunkbase.splunk.com/app/273/)
为我们提供了一个(很丑的) Web 界面来配置这些东西。值得注意的是，在使用之前必须通过 `sudo apt-get install sysstat`
安装必须的命令。

下图展示了使用 SPL 进行检索的示意图，使用 Log 检索工具，我们就不用每次编写脚本来查询我们在 Log 当中感兴趣的点，
更可以瞬间就得到统计以及变换后的结果并将结果输入到可视化模块进行可视化。Splunk 以及 SPL 的使用是一个大主题，
而且考虑到写的越来越像软文了，我还是就此打住吧。

![SPL Search](/img/posts/spl-search.png)

将每种类型的数据输出成可视化图表之后还可以将其作为 Dashboard 的一个 Panel 保存下来，
这样每次访问的时候就可以从主页上看到一个概览了。除了系统的各项指标，我还配置了 Nginx 的访问次数统计和
ssh 次数的每日热度图。最终结果如下:

![Splunk Dashboard](/img/posts/splunk-dashboard.png)

最后我还将 Splunk 配置在了 Nginx 之后使用 VHOST 访问，由于 Splunk 默认是在 `0.0.0.0` 上绑定自己的 Web
服务的，因此外界还可以直接访问，这是我不希望的。因此通过修改 `/opt/splunk/etc/system/local/web.conf`
配置，填入一下内容就可以改变 Splunk Web 服务绑定的 Host 地址了。

```
[settings]
enableSplunkWebSSL = 0
server.socket_host = 127.0.0.1 
```

# 总结

我的 Intel NUC 买回来两周，终于有时间好好把玩一下了。目前部署的服务对机器性能的利用率仍然很低，
未来还要好好想想有什么有趣的应用才行。总体看来，Intel x86 架构的 Mini PC 虽然价格较贵，
但是从性能和适用性上来讲还是比 ARM 开发版和路由器等更强一点，简单来说就是像我这样不怎么有时间折腾的人的选择。
当然诸如树莓派这种开发板添置各种有趣的传感器更为方便，很适合作为 Internet of Things 的一个处理节点，
以后如果有时间也应该研究一下。

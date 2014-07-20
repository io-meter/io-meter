title: Web 设计新趋势: 使用 SVG 代替 Web Icon Font
date: 2014-07-20 16:20:01
category: web
tags: [svg, font, web, html, design, drawing]
---

如果你还在使用 Icon Font 作为网页中显示图标的实现方案，那么你可能有点 Out 了。
由于使用 Icon Font 显示图标存在一些缺陷，开发者们一直在致力于探索使用 SVG 作为替代的方法。
这篇文章列举了目前 SVG 比较常见的使用方法。

<!-- more -->

关于使用 Icon Font 的缺陷，这篇来自 CSS Trick 的 [《Inline SVG vs Icon Font》](http://css-tricks.com/icon-fonts-vs-svg/)
可谓是总结的相当全面了。在我看来，Icon Font 的主要缺陷有以下几条：

1. 浏览器将其视为文字进行抗锯齿优化，有时得到的效果并没有想象中那么锐利。
尤其是在不同系统下对文字进行抗锯齿的算法不同，可能会导致显示效果不同。
2. Icon Font 作为一种字体，Icon 显示的大小和位置可能要受到`font-size`、`line-height`、`word-spacing`等等 CSS 属性的影响。
Icon 所在容器的 CSS 样式可能对 Icon 的位置产生影响，调整起来很不方便。
3. 使用上存在不便。首先，加载一个包含数百图标的 Icon Font，却只使用其中几个图标，非常浪费加载时间。
自己制作 Icon Font 以及把多个 Icon Font 中用到的图标整合成一个 Font 也非常不方便。
4. 为了实现最大程度的浏览器支持，可能要提供至少四种不同类型的字体文件。包括`TTF`、`WOFF`、`EOT`
以及一个使用 SVG 格式定义的字体。


开发者们想出了多种使用 SVG 的技巧来解决/缓解上述问题，下面我们来逐个盘点目前常见的使用方法。

# img/object 标签

使用 img 和 object 标签直接引用 SVG 是早期常见的使用方法。
这种方法的缺点主要在于要求每个图标都单独保存成一个 SVG 文件，使用时也是单独请求的。
如果在页面中使用的多个图标，每个都是单独请求的话会产生很多请求数，增加服务端的负载和拖慢页面加载速度，
因此现在很少使用了。

不过，在 IE 中可以使用 object 标签实现最后讨论的 SVG Defs/Symbols 的效果。

# Inline SVG

所谓 Inline SVG，就是直接把 SVG 写入 HTML 中，这种方法简单直接，而且具有最强的可调性。
使用这种方法，你可以使用 CSS 的`fill`属性和`stroke`属性来控制填充颜色和边线的颜色，
如果 SVG 图标包含多个部分，你甚至可以设置每个部分的样式。同时，使用 JavaScript 修改 SVG 和生成动画效果都可以实现。

Inline SVG 作为 HTML 文档的一部分，不需要单独请求。临时需要修改某个图标的形状也比较方便。
但是 Inline SVG 使用上比较繁琐，需要在页面中插入一大块 SVG 代码因此不适合手写，图标复用起来也比较麻烦。

好在我们大部分的页面都是由某种模板渲染出来的，无论是使用 PHP、Jinja2 还是 ERuby 模板语言，
都可可以定义一个函数来帮我们 include 这些 SVG。因此在很多情况下是很好的解决方案，
其不适合的主要使用场景就是纯静态页面或者前后端分离客户端页面。

# Data URIs

Data URIs 是一种不怎么常见的技巧。之前我们在 CSS 里定义元素的背景图片时，常使用像下面这种方式

```css
.icon {
  backgound-image: url(icons/a.png)
  /* ... */
}
```

而现在，`url`当中可以放置的可以不仅仅是指向资源的 URL 链接，而可以是数据本身。使用 Data URIs，无论是图片还是 SVG，
你都可以将其编码为 base64 并直接写入 CSS。譬如

```css
.icon{ 
  background: url(data:text/svg+xml;base64,<base64 encoded data>)
}
```

关于 base64 编码，请参考[wiki](http://zh.wikipedia.org/wiki/Base64)，Data URI 的格式定义如下

```plain
data:[&lt;mime type&gt;][;charset=&lt;charset&gt;][;base64],&lt;encoded data&gt;
```

使用这种方法，SVG Icon 使用起来和 Icon Font 一样只需要为元素添加 CSS 即可，所有的资源都可以整合在一个 CSS 文件中，
不需要额外引用 SVG 文件。
如果你在使用 [Gulp](http://gulpjs.com/) 或者 [Grunt](http://gruntjs.com/) 这样的 Build 工具，那么将 SVG 整合到一个 CSS
当中是可以非常方便地自动化完成的。这个任务只有简单的字符串和编码处理，基本不需要依赖非 JavaScript 的库和资源。

使用这种方法的缺点是不能方便地使用 CSS 修改 Icon 的颜色和边线属性。

# SVG Sprite

目前，一些提供制作 Icon Font 功能的网站(如[icomoon](http://icomoon.io))已经提供输出 SVG Sprites 功能了。
SVG Sprites 可以看做上述 Data URIs 方法和之前使用位图的 Sprite 方法的组合。

在 Icon Font 还没普及、图标还主要依靠位图显示的时候，前端工程师都会使用 Sprite 来减少图片请求的次数。
其原理很简单：将所有的图标以一定的间隔排列起来组成一整张大图片，使用时对于某个 Icon，编写如下所示的 CSS。

```css
.icon-a {
  background-image: url(/path/to/pic/contains/all/icons.png);
  background-position: 0 120px !important;
  width: 24px;
  height: 24px;
}
```

上述 CSS 通过设定`background-position`调整大图片在背景中的位移，只将某个单个 Icon 暴露出来，其他部分都切掉。
对于所有的 Icon 都写成这样的 CSS 即可使用了。基础的 SVG Sprite 其实只是将原来的位图改成了SVG。

SVG Sprite 相比于原来的位图 Sprite 的一个优点就是可以通过 CSS 调整 Icon 显示的大小。
使用时还可以 Fallback 到位图的 Sprite，因此有极好的浏览器兼容性。
不过和 Data URIs 方法一样它同样存在不能方便调整颜色样式的问题。

目前辅助生成 SVG Sprites 的工具有 [grunt-iconizr](https://github.com/jkphl/grunt-iconizr)、
[gulp-svg-sprites](https://github.com/shakyShane/gulp-svg-sprites) 等。
使用这两个工具，只需将用到的 SVG 放到某个文件夹中就可以自动被拼合成 Sprite 并输出对应 CSS。
两个工具都支持生成 PNG 格式的位图作为 Fallback，缺点是生成位图要依赖[phantomjs](http://phantomjs.org/)
这个重量级 JS 库。

# SVG Defs/Symbols

SVG Defs 和 Symbols 的原理类似，这里着重介绍一下 SVG Symbols 的使用，
SVG Defs/Symbols 本质上是对 Sprite 的进一步优化。之前，我们需要使用相对位置来控制哪个图标被显示出来，
但是其实 SVG 本身的定义允许你以某一种方式直接引用 SVG 中的某一部分。

将多个图标整合成一个 SVG 中的多个 Symbols 之后是下面这样的

```svg
<svg xmlns="http://www.w3.org/2000/svg" style="display: none;">

    <symbol id="circle-cross" viewBox="0 0 32 32">
      <title>circle-cross icon</title>
      <path d="M16 1.333q2.99 0 5.703 1.161t4.677 3.125 3.125 4.677 1.161 5.703-1.161 5.703-3.125 4.677-4.677 3.125-5.703 1.161-5.703-1.161-4.677-3.125-3.125-4.677-1.161-5.703 1.161-5.703 3.125-4.677 4.677-3.125 5.703-1.161zm0 2.667q-2.438 0-4.661.953t-3.828 2.557-2.557 3.828-.953 4.661.953 4.661 2.557 3.828 3.828 2.557 4.661.953 4.661-.953 3.828-2.557 2.557-3.828.953-4.661-.953-4.661-2.557-3.828-3.828-2.557-4.661-.953zm3.771 6.885q.552 0 .948.391t.396.943-.396.948l-2.833 2.833 2.833 2.823q.396.396.396.938 0 .552-.396.943t-.948.391-.938-.385l-2.833-2.823-2.823 2.823q-.385.385-.948.385-.552 0-.943-.385t-.391-.938q0-.563.385-.948l2.833-2.823-2.833-2.833q-.385-.385-.385-.938t.391-.948.943-.396.948.396l2.823 2.833 2.833-2.833q.396-.396.938-.396z"/>
    </symbol>

    <symbol id="circle-check" viewBox="0 0 32 32">
      <title>circle-check icon</title>
      <path d="M16 1.333q2.99 0 5.703 1.161t4.677 3.125 3.125 4.677 1.161 5.703-1.161 5.703-3.125 4.677-4.677 3.125-5.703 1.161-5.703-1.161-4.677-3.125-3.125-4.677-1.161-5.703 1.161-5.703 3.125-4.677 4.677-3.125 5.703-1.161zm0 2.667q-2.438 0-4.661.953t-3.828 2.557-2.557 3.828-.953 4.661.953 4.661 2.557 3.828 3.828 2.557 4.661.953 4.661-.953 3.828-2.557 2.557-3.828.953-4.661-.953-4.661-2.557-3.828-3.828-2.557-4.661-.953zm4.49 7.99q.552 0 .943.391t.391.943-.396.948l-5.656 5.656q-.385.385-.938.385-.563 0-.948-.385l-2.833-2.823q-.385-.385-.385-.948 0-.552.391-.943t.943-.391.948.396l1.885 1.885 4.708-4.719q.396-.396.948-.396z"/>
    </symbol>

    <!-- .... -->
</svg>

```

每个 Symbol 设置一个 id 作为引用时的名字。使用 id 引用这个 SVG 中的 Icon 有两种方式。

第一种，将上述 SVG 作为 body 的第一个元素插入在 HTML 中(Chrome 存在一个
[bug](https://code.google.com/p/chromium/issues/detail?id=349175) 导致不在这里显示不出图像)，
此后，在需要显示某个 Icon 的地方插入下面的代码即可：

```svg
<svg class="icon">
  <use xlink:href="#circle-cross"></use>
</svg>
```

这里的`use`标签直接使用`#circle-cross`这个 id 引用了 SVG 中的图标。这种方式的浏览器兼容性较好。

我更喜欢的是第二种方式，这种方式不需要在 body 开始的地方插入 SVG，而是使用完整路径引用 Icon。
也就是：

```svg
<svg class="icon">
  <use xlink:href="/img/posts/svg-icons.svg#circle-check"></use>
</svg>
<svg class="icon">
  <use xlink:href="/img/posts/svg-icons.svg#circle-cross"></use>
</svg>
```

显示出来的效果就是下面这个样子(可以使用浏览器的 Debug 工具来检视下面的代码)。

<div style="border:1px solid #eee;background:#fff;padding:10px">
<svg style="fill:black;width:64px;height:64px" xmlns="http://www.w3.org/2000/svg">
  <use xlink:href="/img/posts/svg-icons.svg#circle-check"></use>
</svg>
<svg style="fill:black;width:64px;height:64px" xmlns="http://www.w3.org/2000/svg">
  <use xlink:href="/img/posts/svg-icons.svg#circle-cross"></use>
</svg>
</div>

这种方式使用上跟`img`标签没有什么太大的差别了。好处在于所有的图标都在一个文件中，因此只会请求一次。
这种不需要像 Sprite 那样繁琐的设置图片的位移。使用 id 命名图标并使用时直接使用 id 引用既直观又简单。
其灵活性和 Inline SVG 几乎一样——你可以设置颜色、边线样式、大小等等。
视浏览器的不同，有时你需要使用作为 SVG 标签的开始。

```svg
<svg xmlns="http://www.w3.org/2000/svg">
```

对于 IE 则需要使用`object`标签代替`<svg><use>`。关于兼容性讨论详见
[这篇文章](http://css-tricks.com/svg-sprites-use-better-icon-fonts/)。

除了前面提到过的几个 Gulp 和 Grunt 的插件都已经支持 SVG Defs/Symbols 了之外，这里再推荐一个更轻量的 Gulp 插件
[gulp-svg-symbols](https://github.com/Hiswe/gulp-svg-symbols)。如果只使用 SVG Symbols 而无需 Sprite 支持，
那么使用 [gulp-svg-symbols](https://github.com/Hiswe/gulp-svg-symbols) 可以免去对 [phantomjs](http://phantomjs.org/)
的依赖。

# 结语

SVG Icons 作为一种 Icon Font 的替代，使用上具有灵活性好、显示效果好、可控性强等诸多优点。
尤其是最后介绍的 SVG Defs/Symbols 这种方式，已经让 SVG Icon 变成了一种实用的解决方案。

浏览器兼容性是前端领域永恒的话题，目前来看对 SVG 的支持情况确实没有 Web Font 那么好，
其中，主流移动浏览器(Safari, Chrome, IE for Windows Phone)都已经基本兼容 SVG，
但是桌面领域中仍然要面对 IE8 以下的浏览器，此外对 SVG Defs/Symbols 的支持也还存在差异。

现阶段，如果对浏览器兼容性要求比较严苛(主要是支持 IE6-8)，则可以权衡考虑 Icon Font 和带 Fallback 的 SVG Sprite。
否则的话，推荐使用 SVG Defs/Symbols 的方式代替 Icon Font。

个人认为 SVG Icons 已经具备普及的条件了。

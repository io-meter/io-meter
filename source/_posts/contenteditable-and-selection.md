title: 在线文本编辑器的基本实现原理
date: 2014-09-01 19:56:26
category: web
tags: [contenteditable, selection, html5, editor, online editor, completion]
---

最近研究了一下在浏览器中实现的 WYSIWYG 文本编辑器的原理，
在了解基本原理并浏览了 [zenpen](http://zenpen.io) 这个相对简单的在线编辑器的源码后，
在这方面有种豁然开朗的感觉。

<!-- more -->

说来让人惊讶，最初在浏览器中使之变为可能的浏览器是 IE5。在那个时代，
IE 的确也算是非常先进的浏览器了，现在广为使用的 AJAX 技术，不也是 IE5 最早提供的么？
不过这里就不再讨论当初 IE 那套陈旧的 API 了，而主要来讨论 HTML5 之后被各个浏览器广泛支持的一些技术方法。

进行 WYSIWYG 的文本编辑，需要的几个基础是

1. 使得某一部分 DOM 可以被编辑
2. 可以获取和操作用户选中的区域
3. 可以在编辑的同时对所编辑的部分 DOM 进行修改，实现如添加样式等功能

这些功能都非常方便实现。

## 使 DOM 可编辑

使一部分 HTML 的 DOM 作为容器进入编辑状态，只需要为这个 DOM 添加一个 `contentEditable` 属性。
一般来讲，是使用`div`或者`article`元素作为这样的容器。

被添加 `contentEditable` 属性的容器元素的子元素都就可以由用户修改了，
如果想在这个容器下面嵌套一个不可修改的子元素，需要显式地在这个子元素中添加
`contentEditable='false'`这样的声明。

## 获取和操作用户选择

操作和获取用户选择是一个非常有用的功能，它不但可以用来实现这里提出的编辑器的功能，
还可以用来实现在光标位置显示提示选单等多种功能，在后面对所编辑部分进行样式修改的时候也常用到。

想要获取一个 Selection 对象非常简单:

```javascript
var selection = window.getSelection();
```

Selection 对象有`anchorNode`和`focusNode`两个属性，可以用来获得选中部分的开始和结束元素，
不过实用不多(一般实用 Range 代替)。此外还有一个`isCollapsed`
属性值得注意，当其为`true`时，代表选择区域开始和结束在相同的点，也就是没有选中内容时光标闪烁的模式。

selection 对象还有诸多方法可以用来修改选择范围，主要就是对 Range 对象的编辑。

首先可以通过
```javascript
var range = selection.getRangeAt(0);
```
来获取被选中的第一个区块，以此类推还可以获得第二个、第三个。简单起见我们只讨论选中一个区块的情况。
获得了 Range 对象，我们就可以方便地进行获得选中区域内容了。
Range 对象有一对属性分别用来获得选择区域的开始和结束点。
```javascript
range.startContainer // 开始点所在的容器(元素)
range.endOffset // 开始点在其容器内的偏移
range.endContainer // 结束点所在的容器(元素)
range.endOffset // 结束点在其容器内的偏移
```
除了这两对属性，还有一个非常有用的属性，那就是

```javascript
range.commonAncestorContainer // 选择范围的共同父元素
```

上面这个属性常用于检测选择范围的类型/样式，比如检测到选中范围的公共父元素是一个 `h1` 元素，
那么可以在工具栏中将代表 `h1` 元素的按钮设为激活状态。

需要注意的是，返回的 Range 对象是一类可变对象。简单来说，如果用户的选区改变了，
那么 Range 对象的内容也会改变。因此要记录某一时刻的 Range ，就要记录上面提到了的两对属性。
此外，Range 对象的属性都是只读的，需要通过对应的函数方法来修改。

Range 还定义了各种获取内容和修改内容的函数，详细参数和方法可以参见[文档](https://developer.mozilla.org/en-US/docs/Web/API/range)，
这里对几个常见的 Use Case 说明一下。

### 获得选取的坐标范围

我们可以获得光标在网页上的精确位置，对于选区还能得到其矩形边框的几何表示，
这为我们显示提示菜单提供了方便。获得光标的位置或者选区的矩形边框可以使用

```javascript
var rect = range.getBoundingClientRect();
```
`rect`对象包括的属性包括矩形的`top`、`left`、`right`、`bottom`坐标。
如果选取是`collapsed`的话，这四个属性就可以用来计算光标的位置了。

### 保存选区并恢复

如果对选择区域内的元素进行了修改，比如添加新元素、改变元素类型等等，
那么原来的选区会失效。因此一个比较有用的技巧就是在修改元素之前，先保存选区 Range，
待修改完成后再恢复。

一个完整的例子如下：

```javascript
var selection = window.getSelection();
var range = selection.getRangeAt(0);

// 保存所有 Range 的属性
var startContainer = range.startContainer;
var startOffset = range.startOffset;
var endContainer = range.endContainer;
var endOffset = range.endOffset;

// 进行元素修改操作
// ......

// 构造新的 Range
var newRange = document.createRange(); // 注意，此处必须创建一个新的选区，在原来的 range 上修改无效
newRange.setStart(startContainer, startOffset);
newRange.setEnd(endContainer, endOffset);

// 恢复选区
selection.removeAllRanges();
selection.addRange(newRange);
```

需要注意的是，有些操作可能会自动修改选区，那么使用上面方法就不能达到恢复选区的目的了。
一个常用的技巧是为恢复选区添加一个延迟，也就是在上面将`addRange`调用放入`setTimeout`当中。

```javascript
setTimeout(function(){
  selection.addRange(newRange);
}, 50);
```

## 编辑 DOM，修改样式

文本编辑器的一个很重要的功能就是修改内容的样式，比如将文字加粗、倾斜、加下划线等。
还包括将段落修改为标题、块引用等。一个比较直观的方法是按照上述介绍的保存和恢复选区的方法，
按照需求修改元素添加样式即可。但是这种方法其实细想起来比较复杂，尤其是段落中，
存在混杂多种样式，以至于存在样式可以嵌套的情况(比如一段即是加粗又是倾斜的文字)，
维护节点关系和清理空白节点会很繁琐。

好在浏览器为我们提供了一个方便的接口来实现这样的功能。那就是`document.execCommand`，
这个接口将各种操作抽象成命令的形式。下面展示了实现一些基本功能的方法。

```javascript
document.execCommand('bold'); // 加粗所选
document.execCommand('italic'); // 倾斜所选
document.execCommand('underline'); // 下划线所选

document.execCommand('createLink', true, 'http://io-meter.com'); // 为所选文字加链接
document.execCommand('unlink'); // 取消链接

document.execCommand('formatBlock', true, 'h1'); // 将光标所在段落修改为一级标题
document.execCommand('formatBlock', true, 'p'); // 将光标所在块修改为段落
```

除此之外，浏览器还提供了一些编辑命令，如`copy`、`cut`和`paste`等。
完整的命令列表可以参见[这个文档](https://developer.mozilla.org/en-US/docs/Web/API/document.execCommand)

需要注意的是，各种浏览器对这些命令的支持也有些不同，因此需要格外注意。
了解了这些命令，就具备实现编辑器中修改样式等功能的基本知识了。

## 侦听修改

在此还需要说一点题外话，在以往要侦听用户对文本的修改，一般是绑定`keydown`事件，
此外考虑到用户还可能选取并拖拽来改变内容，可能还要添加`mouseup`事件，
这种方法是低效且繁琐的，还对元素样式的修改无能为力。

还好，现代浏览器提供了一种新的机制用来检测内容的修改，那就是 Mutation Observer
机制。关于这方面，有一篇很好的文章值得阅读：
[Detect, Undo And Redo DOM Changes With Mutation Observers](http://addyosmani.com/blog/mutation-observers/)。

## 总结

OK，了解了这些知识，实现一个简单的 Web 文本编辑器是不是也显得不那么难了呢？
虽然在这些 API 上不同浏览器还有些差异，但是已经被广泛应用在实现在线文本编辑功能了。

其实如果只专注于现代浏览器，那么实现如同 Medium 那样漂亮又实用的编辑和写作工具也不是非常困难的事！

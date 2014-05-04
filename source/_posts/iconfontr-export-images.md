title: 写个 Icon Font Viewer: 终
date: 2014-05-04 15:49:49
tags: [Bezier Path, SVG, image, export, icon fontr, osx, xcode]
category: [Iconfontr] 
---

终于到了 Icon Fontr 最重要的一部分了。这一次要把已经读入为`NSBezierPath`
的 Icon 导出成 SVG 和图片，以便于在桌面和移动应用或者 UI 设计软件当中使用。

<!-- more -->

# NSBezierPath To SVG Path

首先是将`NSBezierPath`转换为 SVG 中的 Path 的方法。这一部分实际上是相当容易的。

我们先来了解一下 SVG Path 的基础知识，根据 MDN 上的[介绍](https://developer.mozilla.org/en/docs/Web/SVG/Tutorial/Paths)
对于一个`<path></path>`标签，由名为`d`的属性来指定了遗传
一串简写的图形绘制命令。其可用参数可以总结如下:

* `M`: Move To，移动光标到某一位置，接受一组 x y 参数
* `L`: Line To，从光标位置起始绘制一条直线到目标点，接受一组 x y 参数
* `H`, `V`: `L` 的变形，在水平和竖直方向绘制，只接受一个参数，分别是 x, y 
* `Z`: Close Path，闭合曲线。只有将曲线闭合之后才能对 Path 进行颜色填充
* `C`: Cubic Curve To，绘制三阶 Bezier 曲线，接受三组 x y 参数，分别是两个控制点和一个结束点的位置
* `Q`: Quadratic Curve To，绘制二阶 Bezier 曲线，接受两组 x y 参数，分别是一个控制点和一个结束点的位置
* `A`: Arc To, 绘制圆弧，这个命令的参数比较复杂，这里就不赘述了。在一些绘制 SVG 的实现中使用 A 可以
绘制出比较完美的圆弧。但是大多绘制 API （包括 Cocoa Drawing）都是使用 Bezier 曲线来拟合圆弧的

除了以上的绘制命令，SVG 还提供两个用来续接 SVG 曲线的命令，`S`和`T`，分别对应的是`C`和`Q`。
接受比`C`和`Q`少一组 x y 的参数。这两个命令用于在原来的 SVG 上续接 G2 平滑的曲线
(在衔接点一阶导数连续)。

对于所有这些大写字母组成的命令，其参数都是相对于画布的绝对值。此外还分别有一套与之对应的小写字母组成的命令。
接受相同数量的参数，区别在于所有参数的意义都是增量值。此外需要注意的是，
这些命令都省略了第一个控制点。比如说，对于一条三阶 Bezier 曲线，是需要 4 个控制点才能定义出来的，
而命令`C`只有三组参数，省略的第一个控制点就是画布光标的当前位置(也就是`M`命令指定的点)。

一个 SVG 绘制曲线的例子如下：
```xml
<?xml version="1.0" standalone="no"?>
<svg width="190px" height="160px" version="1.1" xmlns="http://www.w3.org/2000/svg">
  <path d="M10 80 C 40 10, 65 10, 95 80 S 150 150, 180 80" stroke="black" fill="transparent"/>
</svg>
```
直接使用内联 SVG 显示出来就是下面的效果(需支持 SVG 的浏览器支持才能显示)。
<svg style="border:#eee 1px solid;margin:1em 0;" width="100%" height="160px" version="1.1" xmlns="http://www.w3.org/2000/svg">
  <path d="M10 80 C 40 10, 65 10, 95 80 S 150 150, 180 80" stroke="black" fill="transparent"/>
</svg>

接下来回到`NSBezierPath`这边，阅读
`NSBezierPath`的[文档](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ApplicationKit/Classes/NSBezierPath_Class/Reference/Reference.html#//apple_ref/doc/c_ref/NSBezierPathElement)
可以发现，其绘制的命令与 SVG 基本上是对应的。
比如说它包含如下一组函数:
```objectivec
– (void)moveToPoint:(NSPoint)point;
– (void)lineToPoint:(NSPoint)point
– (void)curveToPoint:(NSPoint)point controlPoint1:(NSPoint)point1 controlPoint2:(NSPoint)point2
– (void)closePath
```

基本上就对应了 SVG 中的`M`、`L`、`C`和`Z`了。因此，只要能够将`NSBezierPath`当中储存的信息，
按照这种命令方式取出一个序列，我们就可以将其转化为 SVG 的绘制命令了。

这当然是一件可以实现的事情咯，我们需要做的就是使用下面两个`NSBezierPath`的对象方法:

```objectivec
- (NSInteger)elementCount;
- (NSBezierPathElement)elementAtIndex:(NSInteger)index associatedPoints:(NSPointArray)points
```

其中第一个方法可以得到`NSBezierPath`对象包含的元素个数，而第二个方法提供了取出元素及其相关的控制点的方法。
`NSBezierPathElement`的定义是：

```objectivec
typedef enum {
   NSMoveToBezierPathElement,
   NSLineToBezierPathElement,
   NSCurveToBezierPathElement,
   NSClosePathBezierPathElement
} NSBezierPathElement;
```

看到这里就应该明白了，`NSBezierPath`包含的元素就是控制的命令，而且这些控制命令是 SVG 的控制命令的子集。
因此我们能很方便的将元素分别映射到`M`、`L`、`C`和`Z`上。

这里需要注意两个地方:

其一，`NSPointArray`其实是一个指向`NSPoint`的指针，在`NSBezierPath`里，一个元素的控制点最多有三个。
因此可以`malloc`三倍`NSPoint`的长度的空间，并将指针传入`associatedPoints:`中，
最后的控制点数据将会借由指针传出。

其二，SVG 的坐标系统和`NSBezierPath`的坐标系统是不一样的，对于 SVG 来说，原点在左上角，Y 轴朝下。因此需要进行坐标变换。
关于坐标系统的问题[上一篇文章](http://io-meter.com/2014/04/24/iconfontr-make-your-nscontrol/)曾经讨论过。

OK，有了这些基础知识，将`NSBezierPath`曲线转换为 SVG 也就不在话下了。
我的实现代码在[这里](https://github.com/shanzi/iconfontr/blob/master/iconfontr/NSBezierPath%2BSVGPathString.m#L21)。
其实有了这些知识，将 SVG 转换为`NSBezierPath`也基本足够了。要点就在于要把`NSBezierPath`不支持的一些控制命令，
如`Q`、`H`等转换为原来的`C`和`L`等。这部分就等以后有机会再详细说明吧。

# NSBezierPath To PNG

将`NSBezierPath`转为 PNG 位图输出也很容易，之前在[这里](http://io-meter.com/2014/04/12/iconfontr-1/)
里给出了在 View 中绘制图标的代码。得益于`Cocoa Drawing`框架的良好设计，我们可以直接复用这些代码来绘制到图片甚至PDF。
绘制到图片的方法有很多。主要包括：

* 先绘制到 View，再从 View 中获得绘制出的位图图像
* 绘制到一个 Off Screen 的 GraphicContext 再从 GraphicContext 中抽出图像，好处是不需要在屏幕中显示出来
* 使用`NSBitmapImageRep`创建一个`NSGraphicContext`，再在这个 Context 上进行绘制。
和上一种方法的区别是绘制完成后不需要再手动抽取图像。因为图像已经被直接写入对应`NSBitmapImageRep`了

我们使用最后一种方法来绘制，创建一个`NSImageRep`的函数参数比较复杂，如下所示：

```objectivec
NSBitmapImageRep *rep = [[NSBitmapImageRep alloc] initWithBitmapDataPlanes:NULL
                                                                  pixelsWide:width
                                                                  pixelsHigh:height
                                                               bitsPerSample:8
                                                             samplesPerPixel:4
                                                                    hasAlpha:YES
                                                                    isPlanar:NO
                                                              colorSpaceName:NSCalibratedRGBColorSpace
                                                                 bytesPerRow:0
                                                                bitsPerPixel:32];
```

接下来详细介绍下这些参数：

* `bitmapDataPlanes`：指定图片的色彩通道，这个参数可以理解为事先指定一部分空间用来存储产生的像素数据。
如果指定为`NULL`，函数会通过其他参数来估计空间以分配内存。
* `pixelsWide`, `pixelsHigh`: 这两个参数指定位图的像素宽和高
* `bitsPerSample`: 每个`Sample`的位宽，这里的`Sample`可以翻译为位图色彩的采样或者分量，对于我们通常用的`RGB`
色彩空间，红色分量`R`就是一个`Sample`。常用的 RGB 颜色值，每个分量的大小最高是`255`，
这就代表这样一个`Sample`是一个 8 bits 的数，因此这里我们把位宽设为 8。 `NSBitmapImageRep`最高支持 16 bits 的位宽
* `hasAlpha`: 是否支持透明度
* `isPlanar`: Plane 这个概念和 PhotoShop 中的通道类似。如果这个参数为`YES`，那么色彩值将会分通道储存。
如果参数为`NO`，那么同一个像素的颜色分量将会紧挨着存储在一起。
* `colorSpaceName`: 颜色空间的名称，这里使用的是 RGB 色彩空间，可选的还有 CMYK 色彩空间、灰度色彩空间等。
* `bytesPerRow`: 一行像素需要的空间，这个值可以依据`bitsPerSample`和`pixelsWide`来计算出来，
但是如果在实际使用中分配的空间不够，那么超出的部分将会被截断。这里我取了零，让程序自己确定。
* `bitsPerPixel`: 这个参数其实也可以不指定，它指的是一个像素的位宽，我们前面指定了每个颜色分量的位宽是8，
每个像素有 4 个分量，那么这个值就应该是32

总体来说，初始化一个`NSBitmapImageRep`虽然参数很多，但是其中大多是为了保证安全和某些特殊情况而要求指定的。
大概了解这些参数的含义，在必要的时候指定正确的值即可。

创建了`NSBitmapImageRep`对象之后，可以再从这个对象创建一个`NSGraphicContext`，并把后者指定为`currentContext`。
此后就可以像在 View 里一样绘制图形了。

```objectivec
[NSGraphicsContext saveGraphicsState];
NSGraphicsContext *g = [NSGraphicsContext graphicsContextWithBitmapImageRep:rep];
[NSGraphicsContext setCurrentContext:g];
// ...
// drawing code here
// ...
[NSGraphicsContext restoreGraphicsState];
```

如何将绘制完成的`NSBitmapImageRep`转换为 PNG 文件的格式呢？只需要调用`Rep`对象的入下方法即可。
这个函数返回的是一个可以直接写入文件的`NSData`类型的对象。
Cocoa 支持的文件格式，除了 PNG 以外，还有 TIFF、JPG 等等。
```objectivec
[rep representationUsingType:NSPNGFileType properties:nil]
```
至此，图像的输出功能的核心就完成了。

# 自食狗粮，为 Icon Fontr 设计一个图标

所谓自食狗粮，就是要自己用自己开发的东西来帮助自己开发。具体到这里，
就是要用 IconFontr 导出来的图标为它自己设计一个图标。我从 [ionicons](http://ionicons.com)
中选取了四个图标，稍加处理，最后得到了下面的样子

![Icon](/img/iconfontr/iconfontr256.png)

# 结语

至此，一个完整功能的 Icon Viewer 就完成啦。其实 IconFontr 还有很多可以改进的地方，
比如说可以有一个批量输出 Icon 的功能，下图是我实现出来的样子:

![Multiple Resolutions](/img/iconfontr/resolutions.png)

我还提供了几种预设的分辨率，比如 iOS 的 Tabbar 图标的尺寸等等。
这样就比较方便在 Desktop 和 iOS 应用里使用，也可以用来画原型图等等。

有一些文章([1](http://css-tricks.com/icon-fonts-vs-svg/),[2](http://ianfeather.co.uk/ten-reasons-we-switched-from-an-icon-font-to-svg/)
认为，我们应该尽量使用内联 SVG 而不是 Icon Font 在网页中显示图标。Icon Fontr 也为这种需求提供了方便。
我为其添加了一个复制 SVG 的功能:

![Copy SVG](/img/iconfontr/copysvg.png)

这个系列的文章今天就告一段落了，不过在接下来的时间里，我还会继续维护 Icon Fontr，为其添加各种好用的功能，
各位看官如果有什么需求，欢迎在 Comment 里告诉我。
也欢迎点击[这里](http://cl.ly/2N3i0U0G402k)下载一份目前的测试版本试用，并反馈 Bug。
IconFontr的代码托管在 Github 上，别忘了来[这里](https://github.com/shanzi/iconfontr/) Star 一下。

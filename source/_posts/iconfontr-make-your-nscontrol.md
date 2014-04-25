title: 写个 Icon Font Viewer : 叁
date: 2014-04-24 18:48:48
tags: [iconfontr, NSControl, cocoa, drawing, osx]
category: iconfontr
---

紧接着[上一篇文章](http://io-meter.com/2014/04/18/seperate-codes-and-zoom-icon/)，
这次来实战一下自定义 NSControl 实现一个选择颜色组合的功能。
此外还会更详细的讲解一下 NSView 的绘制方法。

<!-- more -->

首先新建一个 UIControl 子类吧，这里起名叫`IFColorPicker`，在`IFColorPicker.h`
里，我们先作如下定义：

```objectivec
@interface IFColorPicker : NSControl

@property(nonatomic, readonly) NSColor *foregroundColor;
@property(nonatomic, readonly) NSColor *backgroundColor;
@property(nonatomic) NSInteger pickedIndex;

@end
```

注意我们定义了两个只读的属性，`foregroundColor`和`backgroundColor`，
当用户在选色器上点击一种颜色组合的时候，在视图里就通过访问这两个属性来获取当前的前景和背景色。
`pickedIndex`则是一个可读可写的属性，允许从外部写入选择的颜色 Index。

所谓颜色的 Index，是指我们事先指定了一部分颜色组合，按顺序给分配一个 Index，选色是只能从这些组合里选，
Index 其实也是到色彩组合在 Control 中出现顺序。

## 选取颜色组合

怎么样能方便的构造这些颜色组合呢？
对于 NSColor，我们有一些方便的类函数可以快速的构造出想要的颜色。比如`[NSColor blackColor]`
或`[NSColor controlColor]`等。`NSColor`大概定义了数十个这样的函数，但是在此之外的函数就要通过下面的方法来生成了。

```objectivec
+ (NSColor *)colorWithDeviceRed:(CGFloat)red green:(CGFloat)green blue:(CGFloat)blue alpha:(CGFloat)alpha;
```

的确有一些选色软件(比如我所用的 [Color Maker](http://colormaker.cescobaz.com/))是支持直接生成这个代码，
但是仍然过于冗长了，而且调出好看的颜色也不容易，还好我们有一些`NSColor`的第三方扩展可以解决这个问题:

* [FPBrandColors](https://github.com/magtory/FPBrandColors) 提供了返回各种品牌的主颜色的函数，
比如导入头文件之后可以用`[NSColor amazon]`调用返回亚马逊的 Logo 的色调。
* [NSColor-Crayola](https://github.com/CaptainRedmuff/NSColor-Crayola)
和 [NSColor-Pantone](https://github.com/CaptainRedmuff/NSColor-Pantone)
是两套色彩集合，使用和上面基本一致，都是为`NSColor`添加了诸多类函数用来方便构造颜色。

这里我们从 [NSColor-Crayola](https://github.com/CaptainRedmuff/NSColor-Crayola) 来选择几种好看的颜色作为背景色。
在`IFColorPicker.m`里添加下面的函数。

```objectivec
- (NSColor *)backgroundColorAtIndex:(NSInteger)index
{
  switch (index) {
    case 1:
      return [NSColor crayolaCeruleanColor];
    case 2:
      return [NSColor crayolaAquamarineColor];
    case 3:
      return [NSColor crayolaBananaColor];
    case 4:
      return [NSColor crayolaBittersweetColor];
    case 5:
      return [NSColor crayolaBurntOrangeColor];
    case 6:
      return [NSColor crayolaFernColor];
    case 7:
      return [NSColor crayolaInchwormColor];
    case 8:
      return [NSColor crayolaRedColor];
    default:
      return [NSColor whiteColor];
  }
}
```

对于前景色，只选择黑白两种，因此函数如下所示：

```objectivec
- (NSColor *)foregroundColorAtIndex:(NSInteger)index
{
  switch (index) {
    case 0:
      return [NSColor blackColor];
    default:
      return [NSColor whiteColor];
  }
}
```

## 绘制控件

这里更详细的介绍一下在 Cocoa 中的图形绘制技术。首先我们知道`NSControl`其实是`NSView`
的子类，所以绘制`NSControl`本质上就是绘制`NSView`。要自定义`NSControl`的绘制操作，需要重载`drawRect:(NSRect)dirtyRect`
方法，这里的`dirtyRect`不一定是 View 的 Frame，而是系统认为需要重绘的区域。
应该避免完全在这个区域外的绘制工作以提高性能。

绘制所用的 API 其实有两套，一套是 Objective-C 风格的 Cocoa API，一套是 C 风格的 CoreGraphic API。
通常来说，后者的性能更高一点，更底层点。在`drawRect:`中两种 API 都可以使用。
此外`NSView`还有另外一种基于`CALayer`的绘制模式，这种绘制模式是基于`Core Animation`的。
使用 Layer 进行绘制可以充分利用 `Core Animation` 和 `Quatz` 
封装好的`OpenGL`的性能，特别适合包含动画或模糊等特殊效果或需要频繁重绘的 View。

想要使一个 View 使用`CALayer`绘制需要先执行`[view setWantsLayer:YES]`，之后重载下面的函数。
可以看出，这个函数传入了一个`CGContextRef`指针，绘制就需要针对这个`CGContextRef`执行，
这同时也意味着绘制`CALayer`只能使用 C 风格的 CoreGraphic API。重载这个函数后，`drawRect:`就不会再被调用了。

```objectivec
- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;
```

因为我们设计的控件并不需要频繁的重绘，也不包含动画，因此采取较为简单的 Cocoa API 来完成。
`CoreGraphic`风格的 API 以后有机会再谈。

### Drawing Context

在使用 Cocoa API 进行绘图时，我们要明确一个概念：绘图实际上一定是针对于一个 Context 进行的，
在 Cocoa API 中是`NSGraphicContext`，CoreGraphic API 中是`CGContextRef`。
本质上来说，Cocoa API 是对 CoreGraphic 的封装(这也是它稍慢的原因)，但是在使用 Cocoa API 
的时候，我们一般不需要获取到这个 Context 对象的指针，原因是 Cocoa 会为我们设定好当前的上下文信息。

具体来讲，在每次绘制时会有一个全局的变量指代当前的图形 Context，可以使用`[NSGraphicContext currentContext]`
函数获取到这个对象。这个对象由 Cocoa 自动指定，开发者不需要了解它具体是什么对象(它有可能来自屏幕上可显示的 View，
也有可能是一份 PDF 文档或者位图图像等等)。于此同时，Cocoa API 的各个部分也会使用这个 Context 进行各种操作。
比如对于`NSColor`来说，下面的代码就把当前 Context 的填充颜色设定为黑色：

```objectivec
NSColor *black = [NSColor black];
[black setFill];
```

`NSBezierPath`的绘制操作，也是自动应用于当前的 Context 的。譬如，在上面的代码执行后，
执行下面的代码就会填充出一个黑色的矩形：

```objectivec
NSBezierPath *box = [NSBezierPath bezierPathWithRect:NSMakeRect(0, 0, 100, 200)];
[box fill];
```

这种实现方式看起来比较优雅，但是同时也存在一个问题，那就是绘制必须运行在程序的主线程(主 RunLoop)上，
因为在多线程的状态下这个全局的 Context 可能在执行过程中被改变，从而导致绘制混乱。
在辅助线程中进行绘制是可能的，但是必须非常小心，而且不适合使用 Cocoa API。

### Axis and Transform

接下来需要注意的绘制图形的坐标系统。如果有 Windows 或者 Qt 等图形库的使用经验，
可能知道对于他们来说，窗口和视图的坐标系统是以左上角为原点，Y 轴的指向朝下。
Cocoa 与之不同，他的坐标系是和一般的几何坐标系一致的：原点在左下角，Y 轴正向朝上(其实 Cocoa
的坐标系统也存在 API 不统一的问题，比如在 iOS 开发中 UIKit 的坐标就是是原点左上
还好 OSX 在最新的 API 下已经统一为左下了)。

我们之前提到过，在绘制的时候存在一个全局的 currentContext。同样，使用`NSAffineTransform`
进行坐标变换的时候也是针对这个 currentContext 进行的。需要注意的是，`NSAffineTransform` 
的坐标变换操作需要通过调用`concat`方法才会作用于 Context，而且这种作用是累加的。
且看下面的代码：

```objectivec
NSAffineTransform *transform = [NSAffineTransform transform];
[transform translateXBy: 10.0 YBy: 0.0];

[transform concat] // the first time
[transform concat] // the second time
```

上文中，Context 的坐标系被向右移动了两次，每次 10.0 个单位，所以最后坐标系被移动了 20.0 个单位。
另外需要注意的是，在移动坐标系的时候原来已经被绘制在 Context 上的图形并不会被影响。，

如果在绘制中需要使用坐标系变换，为了保险起见应该在变换之前保存 Context 的变换矩阵状态，
在绘制完之后再恢复。如下面的示例代码所示：

```objectivec
[NSGraphicsContext saveGraphicsState];
// transforming and drawing code
// ...
[NSGraphicsContext restoreGraphicsState];
```

了解完以上的知识，绘制 UIControl 就不成问题了。在我的实现中，只生成了一个 BezierPath 对象，
然后通过不断的 Transform 和改变 Context 的当前填充颜色来绘制出多个选色点出来。具体的代码见
[这里](https://github.com/shanzi/iconfontr/blob/master/iconfontr/IFColorPicker.m#L44)。

## 鼠标点击的处理

接下来要处理鼠标点击，这里的需求是鼠标左键应用前景和背景色、右键点击互换前景色和背景色。

之前的文章已经提到过，处理鼠标点击可以重载`mouseUp:`和`rightMouseUp:`函数。
接下来还有一项重要的任务就是确定用户点击的到底是哪个颜色组合。

简单来说，就是要通过用户点击的位置来计算用户选择的 Index。计算 Index 并不复杂，
因为在绘制的时候我们实际上是计算过一次选色点所在的位置，按照相同的原理计算一下偏移位置就可以了。
这里要强调的是鼠标点击位置的获取。

`mouseUp:`和`rightMouseUp:`方法都有一个`NSEvent`类型的参数，对于鼠标事件来讲，
我们可以通过下面的函数获得鼠标相对于所在 View 的坐标。

```objectivec
NSPoint locationInWindow = theEvent.locationInWindow;
NSPoint locationInView = [theView convertPoint:locationInWindow fromView:nil];
```

`locationInWindow`属性顾名思义，就是鼠标点击在窗口中的位置，
可以用用`NSView`的`convertPoint: fromView:`方法把它坐标变换自己的坐标空间中，
`NSView`其实有一系列类似的方法，不但可以变换点的坐标，还可以变换矩形。
同时也存在`fromView:`和`toView:`两套方法。这两套方法中，如果 From 或 To
的 View 是 nil 的话，计算的是相对 Window 的变换。

坐标变换之后就可以计算出用户点击的是哪种颜色组合了。

## 发送 Action

一个好的 NSControl 还要能够发送事件，最好还能够用 Interface Builder 来绑定事件。
发送一个事件的调用是：

```objectivec
// IFColorPicker.m
[self sendAction:self.action to:self.target];
```
`self.action`和`self.target`都是`NSControl`定义的属性，IB 在绑定 Action 的时候也会设定这些值。
而我们可以用`IBAction`定义一个可以在 IB 里连接的回调接口，在`IFDocumentWindowController.h`
中加入下面的定义
(函数实现见[这里](https://github.com/shanzi/iconfontr/blob/master/iconfontr/IFDocumentWindowController.m#L53))：

```objectivec
- (IBAction)changeColor:(id)sender;
```

这时在 Interface Builder 中绑定好后运行，却会发现并没有起到效果。
其实是因为`NSControl`默认认为自己内部有一个`NSActionCell`的，它的`action`以及`target`
属性也是映射到自己的`NSActionCell`上的。这里我们并没有添加`NSActionCell`
因此这两个属性将总是`nil`。

我们的控件没有必要定义自己的 ActionCell，为了能让代码正常工作，只好复写这两个属性，
自己定义两个变量来保存他们。

## 总结

OK，我们的 ColorPicker 控件就这么写好了，修改下图标绘制的代码我们就可以改变图标的预览颜色了。
鼠标左键和右键效果就如下所示，还是蛮不错的吧。

![Left Mouse, Light Green](/img/iconfontr/leftmouse-lightgreen.png)

![Right Mouse, Dark Green](/img/iconfontr/rightmouse-darkgreen.png)

下一篇文章，就来到我们整个项目的最重点了，我们要实现 Icon 导出图片和 SVG 的功能，
最后还要“自食狗粮”，用自己导出的素材为 IconFontr 制作一个图标，可不要错过哦。

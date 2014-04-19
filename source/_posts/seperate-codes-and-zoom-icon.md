title: 写个 Icon Font Viewer : 贰
date: 2014-04-18 23:21:46
tags: [OSX, Objective-C, icon font, iconfontr, NSControl, Cocoa, Interface Builder, Xcode]
category: iconfontr
---

这次我们先来做一些代码拆分，将窗口管理的代码从`IFDocument`当中剥离出来。随后将会实现
Icon 的缩放功能。
完整代码可以在[这里](https://github.com/shanzi/iconfontr/tree/R02_scale_and_color)找到。

<!-- more -->

这次的技术要点有：

* 在`NSDocument`中切换为使用`WindowController`来管理窗体
* `NSScrollView` 的 ZOOM 功能的使用，以及`JNWCollectionView`缩放问题的处理
* 键盘事件的侦听

## 切换到 WindowController

对于初学者来说，理解 Xcode 当中的UI设计器与程序代码的交互可能需要一点时间。Xcode 4
之前，现在的窗口设计器部分其实是独立于 Xcode 的一个应用，名叫
Interface Builder([Wiki](wikipedia.org/wiki/Interface_Builder)，简称 IB)。
现在说起来可能有点不可思议，不过 IB 最初是的确是开发出来写 Lisp 的。

[上一篇文章](io-meter.com/2014/04/12/iconfontr-1/)介绍了使用`NSDocument`来实现简单的
Icon Font 预览，当时窗口管理的代码是写在`IFDocument.m`当中的。 这样做有一个坏处，
如果窗口控制代码比较复杂，在`IFDocument.m`类当中的代码就会变得不好维护。
所以我们需要一个独立的`WindowController`来控制窗口。

在`File>new>Files..`菜单中选择新建一个 Objective-C Class，设置如下图所示

![IFDocumentWindowController](/img/iconfontr/IFDocumentWindowController.png)

这里如果勾选 Also create XIB file for user interface 的话，Xcode 会自动帮你新建并设置好 XIB 文件。
我们想要复用原来的`IFDocument.xib`，因此取消勾选这个选项。

接下来重新调整 XIB 文件和代码关系，`IFDocument.m`中存在这么一个函数:

```objectivec
- (NSString *)windowNibName
{
  return @"IFDocument";
}
```

这个函数指定了当前的`IFDocument`类对应的 UI 设计文件(也就是`IFDocument.xib`)。
我们把这个函数删除掉以切断`IFDocument.m`与`IFDocument.xib`的关系。

将`IFDocument.xib`重命名为`IFDocumentWindow.xib`。点击进入设计器之后，我们先把这个 XIB
文件分配给刚刚新建立的`IFDocumentWindowController`。如下图所示，在左边一栏点选`File's Owner`
后右边的参数面板中作如下调整:

![Set File's Owner](/img/iconfontr/set-files-owner.png)

我们知道，在右边参数面板中的`Custom Class`选项，就是指定 IB 中的对象与代码中定义的类的对应关系的。
我们在这里改变了`File's Owner`对应的类，就相当于在这个 XIB 文件和之前建立的
`IFDocumentWindowController`类之间建立了一个关系。

右键单击`File's Owner`弹出的浮动窗口里有一栏叫做 outlets。这些 outlets 就是在代码中定义的用以和 XIB 中对象对接的接口。
在此需确保`Window`这一栏已经和设计器中的窗口连接在一起了
(如果没有，从右边的圆点处，拖动到窗口上释放以绑定连接)。如下图所示。

![Outlets](/img/iconfontr/outlets.png)

建立了连接之后，我们在`IFDocumentWindowController.m`当中就可以用`self.window`来获得 XIB 当中的窗口了，
这个接口是在`IFDocumentWindowController`的父类`NSWindowController`当中定义的。
我们在`IFDocumentWindowController.h`中用下面代码来添加一个新的接口。

```objectivec
@interface IFDocumentWindowController : NSWindowController

@property(assign) IBOutlet JNWCollectionView *collectionView;

@end
```
注意，在对象的类型前有一个`IBOutlet`关键字，这个关键字就是声明这个指针可以绑定到 IB 中的一个对象
(IBOutlet 中的 IB 当然代表 Interface Builder 咯)。在代码中声明了 Outlet 之后，我们可以在 IB 中把 `collectionView` 
这个接口和 XIB 中的 collectionView 对象连接起来了。

最后我们还要让 WindowController 知道自己对应的 XIB 文件是谁，以便于在
初始化的时候加载进来。在`IFDocumentWindowController.m` 中添加如下一个函数:

```objectivec
- (NSString *)windowNibName
{
  return @"IFDocumentWindow";
}
```

接下来清理一下原来的`IFDocument.m`并把窗口控制相关的代码迁移到`IFDocumentWindowController`当中。

怎么样把原来的`IFDocument`和新的 `IFDocumentWindowController` 联系起来呢？在`makeWindowControllers`
函数中创建`IFDocumentWindowController`的一个实例并调用`addWindowController:`函数就可以了。

```objectivec
- (void) makeWindowControllers
{
  IFDocumentWindowController *windowController = [[IFDocumentWindowController alloc] init];
  [(IFDocumentWindowController *)windowController setGlyphPathes:_glyphPathes];
  [self addWindowController:windowController];
}
```

通过上面的步骤，可以看出了 XIB 和代码配合的的几个要点：

* File's Owner 的类将 XIB 和一个 Controller 联系起来
* 在代码中可以用 IBOutlet 来定义一个接口，在运行时这个接口将会指向 XIB 中的一个对象
* 在 Files's Owner 对应的 Controller 可以通过一个函数来指定对应 XIB 文件的名字
* 之后的文章还会介绍使用 IBAction 关键字来接收一个事件回调

## 实现缩放功能

老实说，缩放功能的实现比预计的要麻烦很多。本来以为作为`NSScroller`子类的`JNWCollectionView`
能够很方便的开启缩放功能，结果却因在缩放过程中其会对子视图进行排版而导致各种 Bug，
经过多次尝试，终于还是找到了一个解决方案。

在`JNWCollectionView`上实现缩放的要点主要有：

* 在缩放进行时，必须要禁止`JNWCollectionViewLayout`更新 layout 布局，否则将会出现卡顿
* 缩放结束后要调整`JNWCollectionViewLayout`的`documentView`大小，否则将无法横向滚动
* 如果要显示横向滚动条，需要修改`JNWCollectionViewLayout`的`ScrollDirection`

首先创建几个子类：

![New Subclasses](/img/iconfontr/new-subclasses.png)

其中`IFMagnifyCollectionView`是`JNWCollectionView`的子类，`IFCollectionGridLayout`
是`JNWCollectionGridLayout`的子类。为了在缩放过程中禁止`IFCollectionGridLayout`刷新排版，
给他添加一个新的属性并重载 `shouldInvalidateLayoutForBoundsChange`方法，
此外还要复写`ScrollDirection`方法来显示横向滚动条：

```objectivec
// JNWCollectionGridLayout.h

#import "JNWCollectionViewGridLayout.h"

@interface IFCollectionGridLayout : JNWCollectionViewGridLayout
@property(nonatomic) BOOL allowsLiveLayout;
@end


// JNWCollectionGridLayout.m

@implementation IFCollectionGridLayout

- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds
{
  // Forbid layout update when zooming
  // _allowsLiveLayout will be set to NO when zooming
  return _allowsLiveLayout;
}

- (JNWCollectionViewScrollDirection)scrollDirection
{
  // scroll in both vertical and horizontal direction
  return JNWCollectionViewScrollDirectionBoth;
}

@end
```

`IFMagnifyCollectionView`的父类`NSScrollView`其实已经提供了缩放的支持，
所以并不需要自己捕捉多点触摸的事件。开启缩放功能使用下面的函数就可以了。最关键的一句就是
`self.allowsMagnification=YES;`，代表着允许使用缩放。

```objectivec
- (void)invokeMagnification
{
  if (!self.allowsMagnification) {
    self.allowsMagnification = YES;  // turn on magnification
    self.magnification = 1.0;
    self.maxMagnification = 3;
    self.minMagnification = 1.0;
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(liveMagnifyWillStart:)
                                                 name:NSScrollViewWillStartLiveMagnifyNotification
                                               object:self];
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(liveMagnifyDidEnd:)
                                                 name:NSScrollViewDidEndLiveMagnifyNotification
                                               object:self];
  }
}
```

在函数的最后用`NSNotificationCenter`添加了两个消息侦听，这两个消息分别在缩放开始和结束时发出。
侦听这两个消息的目的就是为了在缩放开始和结束的时候设定`IFCollectionGridLayout`的`allowsLiveLayout`
属性，以实现在缩放时禁止重新布局的需求。

`liveMagnifyWillStart:`和`liveMagnifyDidEnd:`这两个回调函数的内容这里就略去不表了，
具体实现看
[IFMagnifyCollectionView](https://github.com/shanzi/iconfontr/blob/R02_scale_and_color/iconfontr/IFMagnifyCollectionView.m)。

接下来我们解决缩放后的滚动问题，不同于 iOS 上的`UIScrollView`，`NSScrollView`没有`contentSize`属性。
实际上，`NSScrollView`包含一个子视图`documentView`，所有`NSScrollView`显示的 Subviews
都放置在`documentView`中。而`documentView`的`frame`属性指定了想要显示的内容的*完整*大小，
`NSScrollView`的`documentVisbleRect`属性指定了想要内容*实际显示*的范围。

想要实现正确的滚动效果，就要正确的设置`documentView.frame`和`documentViewVisibleRect`两个属性。
在 Zoom 的过程中，`documentView`的大小其实是不变的，之所以我们可以看到内容被放大了，是因为对
`documentView`发送了`scaleUnitSquareToSize:`消息，而`documentViewVisibleRect`也同时被缩小了。

现在的问题是，缩放结束之后`JNWCollectionView`会根据可视范围的大小重新调整`documentView`的大小，
`documetView`的宽度将总是和`documventVisibleRect`一样，这样横向滚动就失效了。
一个最简单的解决方案是重载`documentVisibleRect`函数，不让它的宽度随着放大而相对变小。

```objectivec
- (NSRect)documentVisibleRect
{
  NSRect visibleRect = [super documentVisibleRect];
  visibleRect.origin.x = 0;
  visibleRect.size.width *= self.magnification;
  return visibleRect;
}
```

这样做牺牲了一定的性能，因为`JNWCollectionView`是根据`documentVisibleRect`来决定一个子视图是否可见，
据此移除看不见的 Subviews 并节约资源。最终还是是要修改`JNWCollectionView`的源码才能达到最好的效果，
这里也不详细介绍了。

为了方便使用，还可以添加键盘的事件侦听，只要设定`IFCollectionView`的`magnification`属性的大小就行了:

```objectivec
#define kEqualsKeyCode 24
#define kMinusKeyCode 27

- (void)keyDown:(NSEvent *)theEvent
{
  // cancel the sound played after keydown
}

- (void)keyUp:(NSEvent *)theEvent
{
  if ([theEvent keyCode]==kEqualsKeyCode) {
    self.magnification += 0.2;
  }
  else if([theEvent keyCode]==kMinusKeyCode) {
    self.magnification -= 0.2;
  }
}
```

想要捕捉键盘事件，只要重载`keyUp`、`keyDown`这样的函数就行了。包括鼠标、键盘和触摸事件在内的回调函数最初都定义在
[NSResponder](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ApplicationKit/Classes/NSResponder_Class/Reference/Reference.html)
中，在所有的`NSResponder`的子类中都可以使用。

对于键盘事件，`[theEvent keyCode]`返回的是一个设备无关的虚拟键值，符号`-`的值是`27`，符号`=`(也就是`+`所在的键)
的值是`24`。这里需要复写一下`keyDown`函数，避免按键时出现提示音。

现在就可以在触摸板上用 Spin 手势和键盘上的加减符号实现放大缩小了，效果如下图所示：

![Zoom Result](/img/iconfontr/zoom-result.png)

## 一点总结和预告

这次的文章主要记录了拆分代码和实现缩放的功能，其中详细介绍了 XIB 和代码之间的关系以及配合的问题，
感觉讲解的还不是很清楚，如果各位看官有什么问题，欢迎直接询问我。

在上面的结果图上可以看到其实已经实现了选择预览颜色的功能，不过限于篇幅原因还是决定留到下一篇文章再讲。
下次将会详细介绍一下如何编写一个自定义的 NSControl，附带图形绘制更详细的介绍以及用`IBAction`
来实现事件的回调的方法。

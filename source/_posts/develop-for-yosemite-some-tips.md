title: Develop for Yosemite: 几个小技巧
date: 2014-10-24 15:02:02
tags: [OSX, mac, cocoa, swift, develop, desktop]
category: osx
---

Apple 终于发布了 Xcode 6.1，带来了 Swift for OSX 等多个更新，
这几天我简单研究了下在 Yosemite 下实现一些新的小需求的方法，
这里使用 Swift 语言描述总结一下。

<!-- more -->

# Unified Toolbar

在 Yosemite 中，包括 Safari、系统偏好设置等多个应用都采取了 Unified Toolbar
设计。Toolbar 和左上角控制窗口关闭、最小化和全屏的三个按钮在同一行。如下所示

![Unified Toolbar](/img/posts/unified-toolbar.png)

实现这样的功能很简单，只要把 Window 的 Title 隐藏掉就好了。在 Window 对应的 WindowController
中复写`windowDidLoad`函数，将`titleVisibility`设置为`.Hidden`即可。
```
override func windowDidLoad() {
  super.windowDidLoad()
  self.window?.titleVisibility = .Hidden // NSWindowTitleHidden in ObjC
}
```

# Translucent View

Yosemite 视觉设计上的一大特点就是引入了大量模糊透明的效果，
特别是工具栏、菜单和侧边栏的模糊效果非常明显。这些模糊效果都是`NSVisualEffectView`提供的。
这里演示一下将整个窗口的 ContentView 都加上透明效果的方法。

首先在 Interface Builder 中选中窗口的 ContentView，并将其 Class 类型设置为 `NSVisualEffectView`。

![ContentView Class](/img/posts/content-view-class.png)

此后，为此 View 添加一个 ViewController，并在其`viewDidLoad`函数中，设置view的`blendingMode`属性。

```
override func viewDidLoad() {
  super.viewDidLoad()
  if let view = self.view as? NSVisualEffectView {
      // Make view translucent
      view.blendingMode = .BehindWindow
  }
}
```

得到的效果如下：

![Translucent View](/img/posts/translucent-view.png)

这里的`blendingMode`有两种选择。`.BehindWindow`是将窗口后面的内容模糊显示，
此外还有一个`.WithinWindow`选项，用来实现同一窗口不同View下，类似于Toolbar那样的模糊效果。

# 检查 Dark Mode

Yosemite 新增的 Dark Mode 是我的最爱，检查 Dark Mode 的状态也有很多应用。
一个最直接的应用就是在右上角显示的状态栏图标，需要使这些图标能根据黑白模式的不同而改变配色。

直接获得系统 Dark Mode 属性的方法我们稍后介绍，这里先介绍一下 Yosemite 为解决图标颜色问题给出的官方解决方案。

Yosemite 推荐的方法是使用`NSImage`新增的`setTemplate`方法。简单来说，`setTemplate`
就是将其代表透明度的`alpha`通道拿出来作为其形状，这样系统就可以自动地根据黑白的 Mode 来将其填充为黑色或者白色。

为了演示这一特性，我实现选择了两个图片作为图标。因为 Template 下需要用到图片的透明度，因此 PNG 格式是很好的选择。
如下图所示，这两个图标分别用在状态栏的图标选中和未选中的模式中。

![StatusItem Icons](/img/posts/statusitem-icons.png)

我在主窗口的`WindowController`下使用下面的代码来设置 StatusItem，注意其中两行`NSImage`对象`setTemplate`方法的调用。
```
class AppWindowController: NSWindowController {
  
  var statusItem: NSStatusItem!
  var statusItemMenu: NSMenu!
  
  override func windowDidLoad() {
    statusItemMenu = NSMenu()
    statusItemMenu.addItemWithTitle("Testing Menu", action: nil, keyEquivalent: "")
    
    let statusBar = NSStatusBar.systemStatusBar()
    self.statusItem = statusBar.statusItemWithLength(statusBar.thickness)
    
    statusItem.image = NSImage(named: "light-bubble")
    statusItem.alternateImage = NSImage(named: "dark-bubble")
    
    statusItem.image?.setTemplate(true) // set image to template
    statusItem.alternateImage?.setTemplate(true) // set image to template

    statusItem.menu = statusItemMenu
  }
```

最后，在 Dark Mode 下看到的图标效果为：

![Dark Mode Icons](/img/posts/darkmode-images.png)

值得注意的是，在 Dark 模式下被标记为`Template`的图标，其本身的颜色会被忽略，但是在普通模式下，其颜色还是会生效，
因此对于`StatusItem`的`alternateImage`，直接使用白色图标是一个很好的选择(这样在普通模式下显示为蓝底白图标)。

上面给出了一般地使用`NSImage`的`setTemplate`函数让系统自动为图标改变颜色的方法。
但是有些时候我们仍不得不切实地检查系统是不是真的在 Dark Mode 下。比如说

1. 你的 StatusItem 是使用 View 显示，而非直接使用图片
2. 你希望根据系统是否在 Dark Mode，而调整你整个 App 的 UI 色调

这时就必须使用一些侧面的方法来实现了。目前提出来的比较有效的方法是直接读取系统偏好设置。写成函数如下

```
func isDarkMode() -> Bool {
  let systemUserDefaults:Dictionary = NSUserDefaults.standardUserDefaults().persistentDomainForName(NSGlobalDomain)!
  let style:AnyObject? = systemUserDefaults["AppleInterfaceStyle"]?
  if let styleString = style as? String {
    return styleString.lowercaseString == "dark"
  }
  return false
}
```

监听 Dark Mode 的改变，则通过监听`AppleInterfaceThemeChangedNotification`这个系统 Notification 来实现。

```
NSDistributedNotificationCenter.defaultCenter().addObserver(
      self, selector: "darkModeChanged:", name: "AppleInterfaceThemeChangedNotification", object: nil)
```

需要注意的是，这些方法对于运行在沙盒中的 OSX 应用可能不可用。

# Storyboard

在 Xcode 6.1 中新增的一个大特性就是为 OSX 应用开发提供了 Storyboard 支持。

iOS 开发者对 Storyboard 应该很熟悉了，那么 OSX 的 Storyboard 的使用起来如何呢？
首先就是 TabView 使用起来比原来方便多了，而弹出 Sheet 和 Popover 也可以直接在 Interface Builder
中完成，不需要再写一些冗余的代码。

这里稍微介绍一下在 Storyboard 里弹出 Popover 的方法，其基本思路和 iOS 上很相似。

如下图，首先在主 View 中添加一个 Pushup Button 和一个新的 ViewController (右边)。

![Add Button and View](/img/posts/add-button-and-view.png)

然后将 Button 的 Action 拖到新的 View 上，并选择 Popover 连接起来。如图

![Binding Popover](/img/posts/binding-popover.png)

OK，Popover 就完成了。此时从我们的主 View 会有一条线连接到我们 Popover 的 View 上，
这个就是新生成的 Segue，选中它还可以进一步设置 Popover 的效果，也可以选择其他的弹出方式。

![Segue](/img/posts/segue-osx.png)

# 总结

OK，这几天使用 Swift 编写 OSX 应用时探索出来东西大概就是这么多了。

作为一个曾经的 Objective-C 非专业写者，我算是满心欢喜地投入了 Swift 的怀抱，
而同时 Apple 在这次 Yosemite 当中为 OSX 提供的新特性也没有让人失望。

当然 Xcode 6 和 Yosemite 提供的新玩意还有很多，也还要继续慢慢挖掘才好！




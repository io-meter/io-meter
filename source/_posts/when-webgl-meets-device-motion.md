title: When WebGL Meets Device Motion
date: 2014-10-03 14:33:38
tags: [webgl, web, accelerate meter, mobile, iPad, iOS, iOS 8, device motion]
category: web
---

众所周知，iOS 8 中的一大亮点功能就是为 Safari 添加了 WebGL 的支持并默认开启，
这可以说是 WebGL 的发展历程中的一个重要的里程碑。而我们也终于可以在 iOS 上使用加速度感应器来控制
Web 页面上的 3D 元素的交互。 作为一个小实验，我为之前写的一个 WebGL 小页面添加了重力感应的控制的交互。

<!-- more -->

升级了 iOS 8 的 iOS 设备可以访问 [sjtug.org](http://sjtug.org) 浏览完成后的页面效果。
关于这个页面 WebGL 方面的实现，之前已经在 [Let Rocket Fly](http://io-meter.com/2014/04/05/let-rocket-fly/)
里详细介绍了，这里就不再赘述。本文就主要来讨论一下 Device Motion 相关的内容

# 配置 Mobile Debug 环境

首先，我们要想办法方便地调试页面。因为需要利用加速度感应器，因此最好还是使用真实设备进行调试，
需要先想办法将 iOS 设备上的 Safari 和 OSX 上 Safari 的调试器连接起来。这一功能在 iOS6、
和 OSX 10.8 的时候就已经得到了支持。

首先，在 iOS 8 的 Safari 偏好设置中 Advanced 子菜单里将 Web Inspector 子选项设为开启状态，如下图所示。

![Enable Web Inspector](/img/posts/enable-web-inspector.png)

此后，将你的 iOS 设备使用 USB 连接到 Mac 上，再开启桌面版 Safari，就可以在 Developer 菜单下找到对应的设备了。
有时需要重启 Safari 才能看到菜单项。如果找不到 Safari 的 Developer 菜单，你需要首先在偏好设置里激活它。

![Devices Menu](/img/posts/devices-menu.png)

OK，现在我们就可以在桌面版的 Safari 上使用调试器调试 iOS 设备上打开的网页了。不光可以捕捉到网页中 Console 输出的
log，甚至还可以完成查看元素、控制页面刷新等等任务，使用上和调试桌面网页没有什么区别。

![Debug Panel](/img/posts/debug-panel.png)

# 添加 Device Motion 支持

为了给页面中的小火箭添加 Device Motion 交互支持，我们首先需要判断浏览器是否支持这一特性，
最简单的方法就是使用下面的代码检测`DeviceMotionEvent`是否存在。

```javascript
if (window.DeviceMotionEvent) {
  // the browser support device motion
}
```

可以设定`window.ondevicemotion`或使用`addEventListener`的方法侦听加速度的变化。

```javascript
window.addEventListener('devicemotion', function(e) {
  // do someting
});
```

这里得到的`Event`对象是`DeviceMotionEvent`对象，可以获取的主要参数为：

1. `acceleration`: 过滤掉重力之后的加速度，有`x`、`y`、`z`三个分量
2. `accelerationIncludingGravity`: 带有重力的加速度，有`x`、`y`、`z`三个分量
3. `rotationRate`: 设备在三个方向旋转的角速度，有`alpha`、`beta`、`gamma`三个分量

有了这三个属性，我们就可以方便的设计基于设备运动的页面交互了。在本文中，只使用了`accelerationIncludingGravity`这个属性。
需要注意的是，`DeviceMotionEvent`所提供的运动参量的坐标系不会随着屏幕窗口方向的改变而改变，也就是说，上述`x`、`y`、`z`
等分量，总是相对于设备本身而言的。具体来说，`x`方向为垂直于设备的竖直方向，`y`为平行于设备的竖直方向，`z`垂直于屏幕向外。

由于这个特性，我们还需要通过检测屏幕的不同 Orientation 方向，来保证我们交互效果的一致性，
不能因横屏或竖屏浏览网页而产生不一样的交互。获取屏幕的 Orientation 可以使用
`window.orientation`属性。

在 Safari 中，`window.orientation`是使用数字表示方向的。值和屏幕方向的对应关系为

1. 值为0，屏幕竖直向上
2. 值为-90，屏幕顺时针旋转90度
3. 值为90，屏幕逆时针旋转90度
4. 值为180，屏幕旋转180度

了解了上述基础知识后，为我们的小火箭添加运动控制就非常简单了。代码如下:

```javascript main.js https://github.com/sjtug/sjtug.github.io/blob/master/js/main.js#L57 view on Github
      if (window.DeviceMotionEvent) {
        window.ondevicemotion = function(e) {
          var ax = event.accelerationIncludingGravity.x;
          var ay = event.accelerationIncludingGravity.y;
          var orient = window.orientation;
          if (orient==0) {
            mx = -ax * 80;
            my = -ay * 80;
          }
          else if (orient==90) {
            mx = ay * 80;
            my = -ax * 80;
          }
          else if (orient==-90) {
            mx = -ay * 80;
            my = -ax * 80;
          }
          else if (orient==180) {
            mx = -ax * 80;
            my = ay * 80;
          }
        }
      }
```

# The End

iOS 8 支持 WebGL 是一件令人振奋的事情。自此，我们不但可以在 iOS 这一主流移动平台上使用 WebGL 进行开发，
也可以利用移动设备特有的一些 API 实现更为有趣的交互。这也许意味着 WebGL 将迎来一次大普及。

尽管在移动设备上运行复杂的 WebGL 页面并不现实，但是像本文这样通过添加简单的 3D 图形，
使原来朴素的页面变得有趣起来，也不啻为一种很好的应用。

title: Let Rocket Fly
date: 2014-04-05 14:53:12
tags: [webGL, blender, Three.js]
category: web
---

最近完成了我第一个相对完整的 WebGL 页面 [(点这里)](http://sjtug.github.io)，
主要内容是一只可以旋转交互的小火箭。这篇文章就主要记录一下这个小项目的制作
过程。

<!-- more -->

## Logo 设计

这个项目是作为我们新近成立的一个同好会类型的组织 SJTUG 的第一个页面。在
开始制作时，对于作为页面主题的组织 Logo 其实还没有什么想法。偶然的情况下
发现小火箭这种形象很适合作为 Logo，适当调整一下，做一个交互的东西也比较方便，
因此就果断的朝这个方向构思了。

首先是在矢量绘图软件中大概的完成 Logo 的平面形象，主要是从 Google Image 上搜索
出来一些已有的图案作为参考。用 CorelDRAW 绘制的 Logo 成图如下。

![rocket](/img/posts/Rocket480.png "Rocket Logo")

在这里，记得要将 Logo 输出成不同的分辨率，以便于适配 Retina 的屏幕分辨率，
对于移动设备和不支持 WebGL 的浏览器，应该要能 fallback 到静态的图片。
支持 Retina 显示图片的话，可以用 [retina.js](http://retinajs.com)，也可以像
我这里一样使用 CSS 。一个可以支持大多数设备的 CSS Media 选择代码如下所示：

```css
@media	only screen and (-webkit-min-device-pixel-ratio: 1.3),
only screen and (-o-min-device-pixel-ratio: 13/10),
only screen and (min-resolution: 120dpi)
{
    /* Things for retina display */
    #logo {
        background-image: url(/img/Rocket960.png);
    }
}

``` 

## Blender 建模与导出

得益于 Three.js 提供了一个方便的 Blender 导出脚本， 制作基于 Web 的 3D 交互也方便很多。
在这里使用的 [Blender](http://blender.org) 虽然不及总所周知的 MAYA 或者 3Ds MAX 那么出名，
但是作为一个开源的软件，其功能、易用性都是可圈可点的，自 2.50 版本其UI变得非常成熟了。

这个小火箭的形体是非常简单的，建模并不需要花费很多时间，在建模的时候简单的绘制一个参考
图作为参考会方便很多。最后得到的模型截图如下：

![blender model](/img/posts/blender-rocket.png "blender model")

在图中显示的橘色条纹并不是贴图的效果。实际上，在这个例子中，既没有必要，也应该尽量避免
使用贴图。因为贴图的文件大小本身除了会拖慢页面的加载速度，在制作过程中为了得到比较好的
效果，还要模型进行 UV 展开等操作。此外使用贴图的话，条纹的边缘很有可能变的略显模糊。

那么该如何制作这个条纹呢？其实很简单，就是在飞船主体的表面用 knife 工具切出条纹所在的
表面，然后选择起来作为一个组赋予一个颜色。着这个过程中难免会出现非四边形的表面，
之前的 Blender 是不支持超过四个边的多边形的，那时必须要非常小心的增加不少边才能达到目的，
而且线框显示出来也不好看，还好较新的版本已经基本解决这个问题了。

虽然如此，超过四条边的多边形还是不建议出现的，不过由于我们的材质是纯色的，
而且并不需要移动节点位置，因此显示出来不会有很大的问题。条纹的基本情况就是下图所示了。

![stripe on rocket](/img/posts/blender-rocket-faces-knife.png "stripe on rocket")

接下来是导出的工作，虽然 Three.js 提供的导出工具是支持导出颜色和材质的，但是在使用中
发现，也许是因为色彩系统的不一致，输出的模型在显示出来颜色并不正确，因此需要再手动
编辑一下输出的文件来调整颜色。

主要需要修改的是 `materials` 字段下每个材质的 `color` 属性
这个属性实质上相当于整形值，比如白色的十六进制表示 `0xffffff` 转化过来应该是 
`255^3 + 255^2 +255 = 16777215`。此外，在 `embeds` 字段当中内嵌的 `material` 
字段也是需要修改 `colorDiffuse` 属性的， 只不过此时颜色是由 RGB 三个颜色值组成的列表，值
最大是 `1.0`，因此要把整形的颜色分量值除以 `255`。

因为这里使用的是纯色的材质，不需要 Lambert 光照模型，因此我们在输出的 json 中把所有的
`lambert` 和 `MeshLambertMaterial` 都分别改成 `basic` 和 `MeshBasicMaterial`。

## 使用 Three.js 导入模型

Three.js 提供了将模型导入的类和函数，将其导入的代码很简单:

```javascript
var loader = new THREE.SceneLoader();
loader.load("/js/rocket.json", function(s){
  scene = s.scene;
  // do something to the scene
})
```

回调函数中的 `s.scene` 参数就是读出来的场景对象。为了能对火焰和飞船主体分别制作动画，我将他们
分成两个对象放置在场景中，然后在这里要分别取出来。在上面代码 do something 注释的地方
将两个物体分别赋给两个全局变量以便于之后调用。顺便还要旋转调整物体，让它和静态 Logo
 的位置角度都接近一致。最后把事先建立好的摄影机放到场景中。因为是使用`BasicMaterial`
所以灯光不是必须的。

```javascript
  rocket = s.objects['Rocket'];
  fire = s.objects['Fire'];

  // rotate objects here
  rocket.rotateOnAxis(rot_axis, 1.508);
  rocket.translateOnAxis(trans_axis, -2)
  rocket.rotateOnAxis(rot_lean, -0.63);

  fire.rotateOnAxis(rot_axis, 1.508);
  fire.translateOnAxis(trans_axis, -2)
  fire.rotateOnAxis(rot_lean, -0.63);

  scene.add(camera);
```

至于将 webGL 的 `canvas` 对象放置到页面中的过程这里就不赘述了。为了和静态logo一致，放置
一个橘色的带有 `border-radis: 50%` 属性的 `div` 到 `canvas` 对象后面，正好是一个圆形背景。

## 创建交互

在上面的代码已经提到过创建动态交互的基础函数了。在较新版本的 Three.js 里，可以用`rotateOnAxis` 
来实现物体绕轴旋转的变换，而不仅仅只能绕直角坐标系的 XYZ 轴旋转了。我们使用这个函数使小火箭可以
绕自己指向的轴进行旋转。

此外还可以对摄像机进行变换。摄像机最有用一个属性是 `position`，用来设定摄像机的位置。此外还有一个
非常神奇的函数，那就是 `lookAt`。`lookAt` 函数指定摄像机指向的点，Three.js 会调整摄像机的朝向使得
摄像机恰好“看向”这个点。

在最后的交互中，不但火箭会绕着自己的轴旋转，在鼠标扫过指定链接时还需要有一种加速感觉，
也就是让小火箭产生些微的震动。这其实就是通过让摄像机 `lookAt` 的点不断随机震动实现的。

```javascript
axis_s.set(-Math.random(), -Math.random(), 0);
axis_s.multiplyScalar(0.4);
camera.lookAt(axis_s);
```

让火箭的火焰不断震动则是通过设定 `scale` 属性:

```javascript
if(firescale>=1) firescale = firel - firer*Math.random();
else firescale = firel + firer*Math.random();
fire.scale.z = firescale;
```

最后就是动画的效果了，如何让小火箭旋转的动画看起来更平滑？经过各种尝试，最后是通过限制
火箭旋转的速度来实现的。抽象来说，把页面上每个点映射到火箭的一个旋转和角度。当鼠标
移动时，并不是实时的就把火箭调整到这个位置，而是假设鼠标是一个目标，代表火箭实际状态的
点去不断追击鼠标。同时我们还考虑两点之间的距离作为追击速度的的参数，距离越近，速度越慢。
最后就形成了这种有加速的动画的感觉。

## 总结

这个小火箭算是我第一个 webGL 的小尝试，得益于 Three.js 的强大功能，最后实际手写的 JavaScript
只有 136 行，时间大约花费在两天。在这个过程中，主要的精力都放在调整模型和交互上面了，具体来说
就是需要对于模型和动画的感觉进行细致的调整，不断地尝试最后才能得到一个比较好的效果。实际上，
往往实现在脑海里觉得不错的效果，真正制作出来真正看在眼里就会觉得非常奇怪。

这个小项目的代码是静态托管在 Github pages 上的，源代码可以在 [这里](https://github.com/sjtug/sjtug.github.io)
找到。  

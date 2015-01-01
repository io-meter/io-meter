title: 使用 WebGL 实现素描效果的渲染
date: 2014-12-31 15:48:27
category: web
tags: [webGL, sketch, rendering, GLSL, shader, blender, web]
---

这次来介绍一下我最近刚完成的一个小玩意儿：通过 WebGL 在网页上显示一个素描风格的场景。

<!-- more -->

欢迎先使用支持 WebGL 的浏览器浏览一下本文对应的 [Live Demo](http://chasezhang.me/sketch-rendering/)。
本文的代码以及场景文件也以 BSD 的协议在 Github 上发布了([这里](https://github.com/shanzi/sketch-rendering))。

开始之前先说点题外话，在过去的两个月里，我还开发了一个用来管理矢量图标并生成图标字体的工具：
[MyIcons](http://io-meter.com/myicons)。这是一个 Web-based 的工具，很方便部署到 Heroku，有需要的朋友欢迎围观和体验！
我也会在之后的博文中逐步给出这个工具有关问题的解说。

## 一点点综述

这次的素描渲染 Demo 是基于 Three.js 的，跟之前的文章 [Let Rocket Fly](http://io-meter.com/2014/04/05/let-rocket-fly/)
不同，这次要涉及到 Shader 的编写，因此复杂度也要比原来高很多。

实现一个这样的渲染效果，主要的步骤包括：

1. 准备模型和场景
2. 通过 WebGL (Three.js) 导入场景
3. 实现 Shader 以表现接近素描的效果

在最重要的第3步中，我们要实现的主要有两个效果：

1. 模型边缘的描边(不同于单纯的线框)
2. 模型表面类似于素描的线条效果

为了实现这样的效果，我们实际并不能直接在单一的 3D 的空间上完成的，而需要另外准备一个二维场景用于合成。
总体的渲染与合成流程如下：

![Pipeline](/img/posts/sketch-rendering-pipeline.png)

其中的 3D 场景，就是我们想要处理成素描效果的场景。这里使用了一个小技巧，
那就是我们并非直接将 3D 场景中的渲染效果输出到屏幕，而是先将三种不同类型的渲染结果输出到位于显存中的 Buffer
(Three.js 中的`WebGLRenderTarget`)里。再在 2D 场景中合成这些输出结果。

这个 2D 场景非常简单，里面只有一个恰好和视口大小一样的矩形平面和一个非透视类型的 Camera，
将我们从 3D 场景得到的不同类型的渲染图作为矩形平面的贴图，这样我们就可以编写 Shader
来高效地处理合成效果了。最终输出的结果其实是 2D 场景的渲染结果，但是观看的人不会感觉到任何差异。

使用这样一个简单的 2D 场景进行后期合成可以说是一个非常常用的技巧，因为这样可以通过 OpenGL 充分利用显卡的渲染性能。

## 准备场景

首先要做的工作是准备用来渲染的场景，选用的建模软件当然是我最喜欢的 [Blender](http://blender.org)。我参考
BlenderNation 上刊登的一副[室内场景作品](http://www.blendernation.com/2014/08/06/image-entropy/)进行了仿制。
我仿制的场景渲染结果如下:

![Scene](/img/posts/room-output.png)

选用这个场景的主要原因是场景的主体结构都非常简单，大多数物体都可以通过简单的立方体变换和修改而成。
大量的平面也方便表现素描的效果。

建模的细节不再赘述。在这一阶段还有一个主要的工序需要完成，那就是 UV 展开和阴影明暗的烘焙(Bake)。

模型的 UV 展开实质上就是确定模型的贴图坐标与模型坐标的映射关系。一个好的 UV 映射决定了模型渲染时贴图的显示效果。
因为模型表面的素描效果实际是通过贴图实现的，因此如果没有一个好的 UV 映射，显示出来的笔触可能会出现扭曲、变形、
粗细不一等各种问题。UV 展开可以说是一个非常繁琐耗时的工序。最后为了减少工作量，我不得不删除了一些比较复杂的模型。

我将场景中的所有模型合并为一个物体，并完成 UV 展开后的结果如下：

![UV Mapping](/img/posts/room-uv-mapping.png)

完成 UV 展开之后将会进行烘焙。所谓的烘焙(Bake)就是将模型在场景环境下的明暗变化、阴影等事先渲染并映射到模型的贴图上。
这个技术常用于静态场景中。在这种静态场景里，灯光的位置和角度不会变化，只有摄像机的方向会改变。
因此实际上物体的明暗阴影都是固定的，将其固定在贴图中之后，使用 OpenGL 渲染时不再进行明暗处理和阴影生成。
这样可以节约大量的计算时间。而且使用 CPU 渲染的阴影往往可以使用更为复杂的算法以获得真实的效果。

Blender 的烘焙选项在 Render 选项卡的最下方，这里选择 Full Render 来将一切光源产生的明暗阴影都固定下来。

![Bake Panel](/img/posts/bake-panel.png)

对照之前的 UV 展开，我烘焙出来的光影贴图如下：

![Room Baked](/img/posts/room-baked.png)

最后，使用 Three.js 提供的[输出插件](https://github.com/mrdoob/three.js/tree/master/utils/exporters/blender)，将我们的场景输出成 Three.js 可以识别的`.json`文件。
我输出的模型文件和相关贴图都已经上传到 GitHub 的[仓库](https://github.com/shanzi/sketch-rendering)里。

这里再为有兴趣的同学推荐一个来自台湾同胞的 Blender 基础教程([YouTube](https://www.youtube.com/playlist?list=PLE885296A496C3D38))。
个人感觉是 Blender 的中文视频教程中比较好的一个，虽然时间录制早了些，但是讲解很清晰。
而且本文制作时使用的建模、UV 展开、贴图和烘焙技巧都有介绍。

## 编写 Shader

终于到了这篇文章的重中之重了，Shader 是通过 GPU 实现图形渲染的核心，通过 OpenGL
实现的任何 2D 或 3D 效果都离不开它。

### 一点点基础知识

众所周知， WebGL 使用的 Shader 语言其实是 OpenGL 的一个嵌入式版本
OpenGL ES 所定义的，这一 Shader 语言使用了类似 C 语言的语法，但是有下面几个区别：

1. Shader 语言没有动态分配内存的机制，所有内存(变量)的空间都是静态分配的
2. Shader 语言是强类型的，不同类型的数不能隐式转换(比如整形不能隐式转换为浮点型)
3. Shader 语言提供的一些数据结构，如向量类型`vec2`、`vec3`、`vec4`
和矩阵类型`mat2`、`mat2`、`mat4`是直接可以使用加减乘除运算符进行操作的。

在 WebGL 中，我们可以自己编写的 Shader 有两种类型

1. Vertex Shader: 模型的每个顶点上调用
2. Fragment Shader: 模型三个顶点组成的面上显示出来的每个像素上执行

在渲染时，GPU 会先在每个顶点上执行 Vertex Shader，再在每个像素上执行 Fragment Shader。
Vertex Shader 主要用来计算每个定点投影在视平面上的位置，但是也可以用来进行一些颜色的计算并将结果传送给 Fragment Shader。
Fragment Shader 则决定了最终显示出来的每个像素的颜色。

接下来介绍 Shader 的变量修饰词。Shader 的变量修饰词可以分为5种:

1. (无): 默认的变量修饰符，作用域只限本地
2. `const`: 只读常量
3. `attribute`: 用来将每个节点的数据和 Vertex Shader 联系起来的变量，简单来说就是在某一个顶点上执行
Vertex Shader 时，变量的值就是这个顶点对应的值。这种对应关系是在初始化 WebGL 的程序时手动指定的。
不过幸好 Three.js 已经为我们完成这一任务了。
4. `uniform`: 这种类型的变量也是运行在 CPU 的主程序向 Shader 传递数据的一个途径，主要用于与所处理的 Vertex 和 Fragment
无关的值，比如摄像机的位置、灯光光源的位置方向等，这些参数在每一帧的渲染时都不变，因此使用`uniform`传递进来。
5. `varying`: 用来从 Vertex Shader 向 Fragment Shader 传递数据的变量。在 Vertex Shader 和 Fragment Shader
上定义相同变量名的`varying`变量，在运行时 Fragment Shader 中变量的值将会是组成这个面的三个顶点所提供的值的线性插值。

Three.js 已经为我们预设了必要的`attribute`和`uniform`，
预设变量列表可以参见[文档](http://threejs.org/docs/#Reference/Renderers.WebGL/WebGLProgram)。

两种 Shader 都有一个`main`函数，不过执行的参数并非通过`main`函数的参数传入程序，
输出结果也不是通过`main`函数的返回值返回的。实际上，OpenGL 已经固定了每种 Shader 的默认输入变量和输出变量的名称与类型，
程序可以直接访问和设置这些变量。当然，外部程序也可以通过`attribute`和`uniform`机制来指定额外的输入。

一个典型的 Vertex Shader 如下面的代码所示：

```glsl
void main(void) {
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

其中，`position`、`projectionMatrix`、`modelViewMatrix` 这些变量都是 Three.js 默认设置好并传递进 Shader 的。
`position`是`attribute`类型，它代表了每个 Vertex 在 3D 空间中的坐标，另外两个变量是`uniform`，是 Three.js
根据场景的属性而设定的。`gl_Position` 就是 OpenGL 指定的 Vertex Shader 的输出值。

一个典型的 Vertex Shader 是通过给出的顶点`position`，以及相关的一些变换投影矩阵，
计算出这个顶点做透视投影后显示在屏幕中的 2D 坐标。因此在这里也可以实现各种透视效果，
如常见的投影透视(近大远小)、平视透视(远近一样大)，甚至超现实的反投影透视(近小远大)等。

Fragment Shader 的主要用处是确定某个像素的颜色，其已经指定的输出值为`gl_FragColor`，这是一个`vec4`类型的变量，
代表了 RGBA 类型的颜色表示，为每一个表面输出白色的 Fragment Shader 如下:

```glsl
void main(void) {
  gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);
}
```

除了直接计算颜色，还可以通过贴图(texture)来确定某个 Fragment 的颜色。在 WebGL 中，贴图是通过`uniform`的方式传递进
Shader 里的，其类型是`sample2D`。随后，我们可以使用`texture2D(texture, uv)`函数获得某一个像素的颜色，这里的`uv`
是一个二维向量，可以通过 Vertex Shader 获得。

在 Three.js 实现访问贴图的一个简单的例子是：

```glsl
// Vertex Shader
varying vUv;

void main(void) {
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  vUv = uv;
}

// Fragment Shader
uniform sample2D aTexture;
varying vUv;

void main(void) {
  gl_FragColor = texture2D(aTexture, vUv);
}
```

在 Vertex Shader 中使用的`uv`变量，也是 Three.js 中已经提供好的`attribute`。接下来就是在 Three.js 中使用 Shader 的方法了。

### 在 Three.js 中使用 Shader 

Three.js 提供了`ShaderMaterial`用于实现自定义 Shader 的
Material。下面是一个来自其[官方文档](http://threejs.org/docs/#Reference/Materials/ShaderMaterial)的例子。

```javascript
var material = new THREE.ShaderMaterial( {
  uniforms: {
    time: { type: "f", value: 1.0 },
    resolution: { type: "v2", value: new THREE.Vector2() }
  },
  attributes: {
    vertexOpacity: { type: 'f', value: [] }
  },
  vertexShader: document.getElementById( 'vertexShader' ).textContent,
  fragmentShader: document.getElementById( 'fragmentShader' ).textContent
});
```

你可以通过设置`uniforms`和`attributes`等参数向 Shader 传递数据，传递的格式文档中都有介绍。
我们也是在这里将 Shader 需要用到的 Texture 通过`uniforms`传递进去的。Texture 写在 unifroms 里的`type`是`t`，
`value`可以是一个 Three.js 的`Texture`对象，也可以是`WebGLRenderTarget`。

这里只是将值传递了进去，你还是要在 Shader 源码里自己声明这些变量才能访问他们，
在 Shader 里定义的名称应该与你在 JavaScript 中给出的键名相同。

## 显示模型的 Outline

模型的 Outline 就是在卡通风格的图画中围绕在物体边缘的线，因为卡通风格中物体的总体色调都比较平面化，
所以需要这样的线来强调物体与物体之间的区分。

实现这种 Outline 有两种简单直观的方法：

1. 使用深度作为特征，将深度变化大的地方标记出来
2. 使用表面法线的方向作为特征，将发现变化大的地方标记出来

这两种方法都各自有自己的缺点。比如深度特征时，很容易将一个与观察方向夹角比较小的面全部标记为黑色；
而法线特征时，又无法将前后两个法线相近但是距离较远的表面区分开。这里参考另一篇相关内容的英文博客
[Sketch Rendering](http://floored.com/blog/2014/sketch-rendering.html) 的方法来实现。

这种方法结合了深度和法线，假设有两个点 A 和 B，通过计算 A 的空间位置到 B 的法线所构成的平面的距离作为衡量，
判断是否应该标记为 Outline。A 和 B 的空间位置则需要通过 A 和 B 的深度来计算出来。
因此，我们需要先将我们的 3D 场景的深度和法线渲染图输出出来。

Three.js 已经提供了`MeshDepthMaterial`和`MeshNormalMaterial`分别用来输出深度和法线渲染图。
我们直接使用这两个类就好了。假设我们已经初始化了一个`depthMaterial`和一个`normalMaterial`，
那么将整个场景里的物体都用某一个 Material 进行渲染的话，我们可以使用

```javascript
objectScene.overrideMaterial = depthMaterial; // 或 normalMaterial
```

这样的方法实现。

此外，我们不希望渲染结果直接输出到屏幕，因此我们需要先新建一个 `WebGLRenderTarget` 作为一个 FrameBuffer 来存放结果。
此后这个`WebGLRenderTarget`可以直接作为贴图传入用于合成的 2D 场景。

```javascript
var pars = {
  minFilter: THREE.LinearFilter,
  magFilter: THREE.LinearFilter,
  format: THREE.RGBFormat,
  stencilBuffer: false
}

var depthTexture = new THREE.WebGLRenderTarget(width, height, pars)
var normalTexture = new THREE.WebGLRenderTarget(width, height, pars)
```

使用下面的代码，将渲染结果输出到 FrameBuffer 里:

```javascript
// render depth
objectScene.overrideMaterial = depthMaterial;
renderer.setClearColor('#000000');
renderer.clearTarget(depthTexture, true, true);
renderer.render(objectScene, objectCamera, depthTexture);

// render normal
objectScene.overrideMaterial = normalMaterial;
renderer.setClearColor('#000000');
renderer.clearTarget(normalTexture, true, true);
renderer.render(objectScene, objectCamera, normalTexture);
```

在输出之前，别忘记使用`renderer`的`clearTarget`函数将 Buffer 清空。
如果将我们在这一步生成的贴图显示出来的话，大概是下面的样子：

![Depth & Normal Texture](/img/posts/sketch-depth-and-normal.png)

## 生成素描笔触

接下来就是在物体的表面生成绘制的素描线条效果了。这个方面其实比想象中更简单一点，
我们的素描效果是使用的是如下一系列贴图组成的:

![Hatching Maps](/img/posts/hatch-maps.png)

接下来的问题就是找一种方法将这种不同密度的贴图融合在一起，这种问题被称为 Hatching。
这里使用的 Hatching 方法是 MicroSoft Research
在 2001 年发表的一篇[论文](http://research.microsoft.com/en-us/um/people/hoppe/proj/hatching/)中给出的。

不同于原文中使用 6 张贴图合成的方法，这里采用了使用 3 张贴图合成，然后将贴图旋转90度再合成一次，
从而获得交叉的笔划。

```glsl
void main() {
  vec2 uv = vUv * 15.0;
  vec2 uv2 = vUv.yx * 10.0;
  float shading = texture2D(bakedshadow, vUv).r + 0.1;
  float crossedShading = shade(shading, uv) * shade(shading, uv2) * 0.6 + 0.4;
  gl_FragColor = vec4(vec3(crossedShading), 1.0);
}
```

`shade`函数就是用合成多个贴图的函数，具体代码可以参见 GitHub
上的[这个文件](https://github.com/shanzi/sketch-rendering/blob/master/coffee/hatch_material.coffee)。

## 最后的合成

最后就是要在我们的二维场景里进行最后的合成了。构造这样一个二维场景的代码很简单:

```javascript
var composeCamera = new THREE.OrthographicCamera(-width / 2, width / 2, height / 2, -height / 2, -10, 10);
var composePlaneGeometry = new THREE.PlaneBufferGeometry(width, height);
composePlaneMesh = new THREE.Mesh(composePlaneGeometry, composeMaterial);
composeScene.add(composePlaneMesh);
```

场景的主要构造就是一个和视口一样大小的矩形几何体，摄像机则是一个`OrthographicCamera`，这种摄像机没有透视效果，
正合适用于我们这种合成的需求。

将前几步输出到 FrameBuffer (也就是`WebGLRenderTarget`)的结果作为这个矩形表面的贴图，
然后我们编写一个 Shader 来进行合成。

这一次，我们不再需要输出到 Buffer 上，而是直接输出到屏幕。而 Outline 的生成也是在这一步完成的。
用来计算 Outline 的函数是:

```glsl
float planeDistance(const in vec3 positionA, const in vec3 normalA, 
                    const in vec3 positionB, const in vec3 normalB) {
  vec3 positionDelta = positionB-positionA;
  float planeDistanceDelta = max(abs(dot(positionDelta, normalA)), abs(dot(positionDelta, normalB)));
  return planeDistanceDelta;
}
```

在当前坐标周围取一个十字形的采样，对于上下和左右取出的点分别执行上面的函数，
最后使用`smoothstep`来获得 Outline 的颜色:

```glsl
vec2 planeDist = vec2(
    planeDistance(leftpos, leftnor, rightpos, rightnor),
    planeDistance(uppos, upnor, downpos, downnor));
float planeEdge = 2.5 * length(planeDist);
planeEdge = 1.0 - 0.5 * smoothstep(0.0, depthCenter, planeEdge);
```

在最后实现的版本里，我还尝试了再混入法线方式生成的边缘线的效果。最终生成的 Outline 效果如下:

![Outline](/img/posts/sketch-outline.png)

最后，将 Hatching 过程输出的结果混合进来:

```glsl
vec4 hatch = texture2D(hatchtexture, vUv);
gl_FragColor = vec4(vec3(hatch * edge), 1.0);
```

完整的实现可以参见我放在 GitHub
上的[源码](https://github.com/shanzi/sketch-rendering/blob/master/coffee/compose_material.coffee)。

大功告成！最后的合成效果如图:

![Final Result](/img/posts/sketch-result.png)

各位可以访问我使用简单添加了一点交互之后得到的 [Live Demo](http://chasezhang.me/sketch-rendering/)
(请使用支持 WebGL 的现代浏览器进行访问，加载模型和全部贴图可能需要一小会，请耐心等待)。

我实现的所有代码以及模型都已经以 BSD 协议发布到 GitHub
上了([这里](https://github.com/shanzi/sketch-rendering))。

## 总结一下

虽然是作为我在学校一门课程的 Final Project 的一部分完成的项目，
但是在这个过程中我总算是对于 Shader 的编写方面有所入门。此外，这次进行 Blender
进行建模也感觉比以前顺利了许多。

虽然对 Blender 和 WebGL 的爱好现在看起来还没有什么现实价值，但是能够自己完成一个有趣的 Project
还是很有成就感的！

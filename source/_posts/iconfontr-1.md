title: 写个 Icon Font Viewer : 初
date: 2014-04-12 21:03:28
tags: [OSX, Objective-C, icon font, icon, font, iconfontr, fontawesome, zocial, ionicons, bezier path]
category: [iconfontr]
---

得益于现代浏览器的普及，使用以 Web Font 作为基础的 Icon Font 来显示图标越来越流行了。
Icon Font 本身作为一种矢量素材，具有很多优点，比如可以在不同缩放和 Retina 显示屏上获得很好的显示效果，
可以用 CSS 指定颜色等。

<!-- more -->

现在比较流行的 Icon Fonts 包括 [Fontawesome](http://fontawesome.io)、
[Foundation Icon](http://zurb.com/playground/foundation-icon-fonts-3)、
社交类的 [Zocial](http://zocial.smcllns.com/)等。
当然还有我最喜欢的无论数量还是质量都是上上乘的 [ionicons](http://ionicons.com/)。

在线制作 Icon Font 的网站，比如 [Fontastic](http://fontastic.me)、[IconMoon](http://icomoon.io/)
可算是层出不穷。将 Icon Font 用于 iOS 开发的框架 [FontAwesomeKit](https://github.com/PrideChung/FontAwesomeKit)
也值得关注。

总结了这么多资源，接下来进入正题。为了方便将已有 Icon Font 当做参考来设计风格统一的自定义图标，
自然需要一个方便的将 Icon Font 中的 Glyphs 导出成图片或者 SVG 的工具。
这个系列的文章就来记录一下如何编写一个 OSX 下的 Icon Font 浏览器和导出器。

第一篇的内容是：浏览功能的实现。项目代码托管在[这里](https://github.com/shanzi/iconfontr/)。

## 建立项目

OK，在 Xcode 里建立一个 cocoa 应用先。在新建项目的时候选中 Document-Based Application 的选项，
以方便实现打开文件的功能。然后把 Document Extension 设为 ttf。如下图所示，我给这个 App 起名叫 Iconfontr。

![create project](/img/iconfontr/document-base.png)

在开始编写代码之前先对项目做一个设置：在项目 Target 的 Info 标签页下的 Document Type
设置为 Viewer (因为我们只需要浏览而不需要编辑 Icon Font)。

## 读入 Font

接下来是把 TTF 文件读入的工作，因为 `NSFont` 类并没有提供读取任意路径字体的方法(只能读取已安装字体)，我们需要先将字体
读入为 `CGFontRef` 类型，再经过`CTFontRef`转换回`NSFont`。

在 IFDocument.h 中为 IFDocument 类添加一个属性。

```objectivec
// IFDocument.h
@property(nonatomic, readonly) NSFont *font;
```

重载 IFDocument 读入文件的函数：

```objectivec
- (BOOL)readFromURL:(NSURL *)url ofType:(NSString *)typeName error:(NSError *__autoreleasing *)outError
{
  // read font from url
  CFURLRef theCFURL = (__bridge CFURLRef)url;
  CGDataProviderRef dataProvider = CGDataProviderCreateWithURL(theCFURL);
  CGFontRef theCGFont = CGFontCreateWithDataProvider(dataProvider);

  if (theCGFont == NULL) return NO;

  // cast CGFont back to NSFont
  CTFontRef theCTFont = CTFontCreateWithGraphicsFont(theCGFont, 16, NULL, NULL);
  _font = (__bridge NSFont *) theCTFont;
  
  return YES;
}
```

怎么样获取某个字体文件包含的所有 Glyphs 呢？`NSFont`没有提供简单直接的函数，因此要自己想办法解决这个问题。
首先可以通过 `[font numberOfGlyphs]` 获得到字体包含的 Glyphs 总数，而`NSGlyph`本身是整数类型，因此我们尝试使 
`glyph`的值从`1`开始向后遍历，直到取出所有的 Glyphs 为止。

同时我们还可以用函数`[font boundingRectForGlyph:glyph]`获得到 Glyph 的 `boundingRect`，因此可以通过检测一个
Glyph 的`boundingRect`是不是空来检查这个 Glyphs 是否存在。在上面的函数中`return YES`之前的地方加入如下代码。
其中`_glyphPathArray`是一个`NSMutableArray`。 

```objectivec

  // get available glyphs
  NSUInteger glyphCount = [_font numberOfGlyphs];
  _glyphPathArray = [[NSMutableArray alloc] initWithCapacity:glyphCount];
 
  for (NSUInteger i=1; i<=glyphCount; i++) {
    NSRect boundingRect = [_font boundingRectForGlyph:(NSGlyph)i];
    
    if (!NSIsEmptyRect(boundingRect)) {

      // convert glyph into bezier path
      NSBezierPath *path = [[NSBezierPath alloc] init];
      [path moveToPoint:NSMakePoint(-NSMidX(boundingRect), -NSMidY(boundingRect))];
      [path appendBezierPathWithGlyph:(NSGlyph)i inFont:_font];
      [_glyphPathArray addObject:path];
    }
  }
```

为什么这里不从`0`开始遍历呢？因为`NSGlyph`有两个保留的值，分别是`NSControlGlyph = 0x00FFFFFF`
和`NSNullGlyph = 0x0`。因此值为`0`的 glyph 是不存在的，强制读取反而会产生乱码。

在上面的代码中值得注意的还有`NSBezierPath`的`appendBezierPathWithGlyph: inFont:`方法，
就是这个方法使得我们有可能将某个字符的外形抽取出来，这也是整个 Viewer 的核心所在。

## 显示 Glyphs

一般来说，是要用`NSCollectionView`来显示 Glyphs 的，不过习惯了 iOS 开发的开发者可能会觉得
`NSCollectionView`的使用方法没有`UICollectionView`那么直观。`NSCollectionView`主要依赖 Bindings
来显示内容，在某些时候可以节省很多代码，但是在需要细致定制的时候却有些麻烦。

这里采用一个模仿`UICollectionView`的第三方库 [JNWCollectionView](https://github.com/jwilling/JNWCollectionView)，
用 CocoaPods 安装这个依赖。运行`pod install` 之后打开 `iconfontr.xcworkspace` 进行下一步的操作。

在`IFDocument.xib` 里加入并设置好`JNWCollectionView`，这里不再赘述。记得要将`JNWCollectionView`的
`dataSource` 指定为 `File's Owner`。

显示 Glyph 的要点在于`IFGlyphView`(这个类是`JNWCollectionViewCell`的子类)。
这个 View 重载了以下方法来将 Bezier 曲线绘制出来。

```objectivec

- (void)drawRect:(NSRect)dirtyRect
{
  [super drawRect:dirtyRect];
  [NSGraphicsContext saveGraphicsState];
  
  
  // selection
  if (self.selected) {
    NSBezierPath *border = [NSBezierPath bezierPathWithRect:self.bounds];
    [[NSColor colorWithCalibratedRed:0.3 green:0.5 blue:1.0 alpha:1.0] setStroke];
    [border setLineWidth:4.0];
    [border stroke];
  }

  // transform coordinates
  NSAffineTransform *transform = [NSAffineTransform transform];
  [transform translateXBy:NSMidX(self.bounds) yBy:NSMidY(self.bounds)];
  [transform concat];
  
  // draw path
  [[NSColor blackColor] setFill];
  [_bezierPath fill];
  
  
  [NSGraphicsContext restoreGraphicsState];
}
```

## 初之结语

在上文介绍的方法之后再加上一点交互，iconfontr 就初具雏形了。下图为用 iconfontr 浏览
[ionicons](http://ionicons.com/) 图标的效果。

![Basic Viewer](/img/iconfontr/result1.png)

可以发现，截止目前这个小 App 其实不限于显示 Icon Font，而是可以打开任何
TTF 类型的字体。实际上，如果使用现在的 iconfontr 打开一个比较大的字体文件（比如包含上万字的中文字形）
很可能会出现性能问题，好在一般的 Icon Font 只有不到1000个字形，因此就不做进一步的优化了。

当然，如果只是一个字体浏览器，就没有必要叫做 iconfontr 了。接下来还要针对 Icon Font
的特点做一些特殊的优化，以下的功能会留到之后的文章中介绍：

* 使用`WindowController`管理 Window 以实现更细致的控制
* 设置选中的 Glyph 的前景和背景颜色以便预览效果
* 缩放 Glyph
* 导出图片和 SVG

截止于这一步的代码在项目仓库的 [R01\_BASIC\_VIEWER](https://github.com/shanzi/iconfontr/tree/R01_BASIC_VIEWER)
TAG 上。

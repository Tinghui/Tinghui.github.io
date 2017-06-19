---
layout: post
title: "Content Compression Resistance和Content Hugging"
date: 2015-01-28 13:54
published: true
comments: true
categories: [AutoLayout]
---

AutoLayout中，`Content Compression Resistance`和`Content Hugging`这两个概念，从字面上很难理解它们意思和用途，苹果官方文档中相关描述也比较少。中文翻译中大多数都把它们按字面翻译成`内容压缩阻力`和`内容吸附性`，也不是很好理解。最近搜索和查看了一些文章和资料，搞清楚了它们的作用，在此整理一下。

<!--more-->

## 固有内容尺寸（Intrinsic Content Size）

要理解内容压缩阻力和内容吸附性这两个概念，首先要理解固有内容尺寸（Intrinsic Content Size）这一概念。

每个视图都有内容压缩阻力优先级（Content Compression Resistance Priority）和内容吸附性优先级（Content Hugging Priority）。**但只有当视图定义了固有内容尺寸后，这两种优先级才会起作用；**否则如果都没有定义内容尺寸大小，又如何知道应该抗压缩或者吸附至什么大小呢。

那么，固有内容尺寸是什么，有什么作用呢？

引用自苹果官方AutoLayout指南里面对固有内容尺寸的描述：

> **Intrinsic Content Size**
>
> Leaf-level views such as buttons typically know more about what size they should be than does the code that is positioning them. This is communicated through the intrinsic content size, which tells the layout system that a view contains some content that it doesn’t natively understand, and indicates how large that content is, intrinsically.
>
> For elements such as text labels, you should typically set the element to be its intrinsic size (select Editor > Size To Fit Content). This means that the element will grow and shrink appropriately with different content for different languages.

什么意思呢？

简单来说就是，像按钮、文本标签这类视图控件，在布局的时候，它们自己内部比外部布局代码更清楚自己需要多大的尺寸来显示自己的内容。这个尺寸就是由固有内容尺寸（intrinsic content size）来传达的。这就相当于，固有内容尺寸告诉布局系统：“这个视图里面包含了一些你不能理解的内容，但是我给你指出了那些内容有多大的尺寸。”

由此可见，固有内容尺寸是为了实现视图自适应大小而准备的。

## Content Compression Resistance 与 Content Hugging

关于这两个概念，最容易理解的文档说明在UIView Class Reference文档里面：

> **`- contentCompressionResistancePriorityForAxis:`**
>
> Returns the priority with which a view resists being made smaller than its intrinsic size.
>
>
> **`- contentHuggingPriorityForAxis:`**
>
> Returns the priority with which a view resists being made larger than its intrinsic size.

通过以上两个接口的说明，其意义已经相当清楚了：内容压缩阻力优先级就是视图反压缩的优先级，优先级越大，视图就越不容易被压小；内容吸附性优先级就是视图反拉伸的优先级，优先级越大，视图就越不容易被拉大。

## 例子

下面，引用一个来自[stackoverflow](http://stackoverflow.com/questions/15850417/cocoa-autolayout-content-hugging-vs-content-compression-resistance-priority)的例子，这个例子很形象的解释了内容压缩阻力和内容吸附性优先级的作用。

假设，你有一个下面这样的按钮：

```
[		Click Me		]
```

按钮与其父视图之间的边距约束优先级是500。

那么，如果按钮的吸附性优先级（Hugging priority）大于500，按钮看起来会是这样：

```
[Click Me]
```

如果，吸附性优先级小于500，按钮会是这样：

```
[		Click Me		]
```

如果现在父视图收缩了，按钮的压缩阻力优先级（Compression Resistance priority）大于500，它看起来会是这样：

```
[Click Me]
```

否则，如果压缩阻力优先级小于500，它会是这样：

```
[Cli..]
```

如果不是这样，很可能是其他的一些约束扰乱了你的整个布局。
例如，你可能将边距约束优先级设置成了1000。或者可能是加一个优先级较高的宽度约束。遇到这种情况，可以试试“Editor > Size to Fit Content”命令。

## 总结

因此，我们可以简单总结为：

1. **Content Hugging: 反拉伸**
2. **Content Compression Resistance: 反压缩**
3. **仅当视图定义了自己的Intrinsic Content Size，那么它的Content Compression Resistance优先级和Content Hugging优先级属性才有作用。**


参考

1. [Auto Layout Guide]((https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/AutolayoutPG/AutoLayoutConcepts/AutoLayoutConcepts.html#//apple_ref/doc/uid/TP40010853-CH14-SW2))
2. [UIView Class Reference](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIView_Class/index.html#//apple_ref/occ/instm/UIView/contentHuggingPriorityForAxis:)
3. [Advanced Auto Layout Toolbox](http://www.objc.io/issue-3/advanced-auto-layout-toolbox.html)
4. [Cocoa Autolayout: content hugging vs content compression resistance priority](http://stackoverflow.com/questions/15850417/cocoa-autolayout-content-hugging-vs-content-compression-resistance-priority)

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="30%" height="30%" /></p>
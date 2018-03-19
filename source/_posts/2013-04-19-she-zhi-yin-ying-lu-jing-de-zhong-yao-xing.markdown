---
layout: post
title: "给阴影设置shadowPath的重要性"
date: 2013-04-19 10:46
comments: true
published: true
categories: iOS
tag: [iOS,性能]
---


iOS中，给视图加阴影很容易，只需要：

1. 项目中引入QuartzCore框架
2. 实现文件中import QuartzCore的头文件
3. 然后仅需一行代码`[myView.layer setShadowOpacity:0.5]`

![image](https://markpospesel.files.wordpress.com/2012/04/shadow1.png?w=584)  

然而，最简单的方法通常都不是性能最好的方法。
<!-- more -->

如果再用这个视图作动画(特别当它是UITableViewCell的一部分的时候)，你可能会注意到动画的卡顿。这是因为在计算视图阴影的时候，为了计算出怎样渲染视图的阴影，Core Animation需要做一次离屏渲染才能确定视图的形状。

为了证明这一点。现在打开模拟器，勾选上「Debug」菜单中的「Color Off-screen Rendered」选项。

![image](https://markpospesel.files.wordpress.com/2012/04/simulator-menu.png?w=584)

或者，接上iOS设备，启动Instruments(⌘I)，选择Core Animation模板，选中Core Animation栏，勾上"Color Offscreen-Rendered Yellow"选项。

![IMAGE](https://markpospesel.files.wordpress.com/2012/04/instruments.png?w=584)

之后在模拟器（或设备）上，会看到类似下面的画面：

![IMAGE](https://markpospesel.files.wordpress.com/2012/04/offscreen-rendered1.png?w=584)

这表明某些东西（在我们这个例子中就是阴影）造成了一个昂贵的离屏渲染。

### 快速修正

幸运地是，修正阴影的这个性能问题和添加阴影一样简单。只需告诉Core Animation视图是什么形状的就行了。使用view.layer的`setShadowPath:`方法：

```
[myView.layer setShadowPath:[[UIBezierPath bezierPathWithRect:myView.bounds] CGPath]];
```

(注意：因为视图不同的形状，这里代码可能有所不同。UIBezierPath有许多便捷的方法可以调用。比如你的视图有圆角，可以使用`bezierPathWithRoundedRect:cornerRadius:`方法。)

现在再测试一下，指示离屏渲染内容的黄色区块应该消失了吧。

### 注意

当视图的bounds变化后，你需要更新layer的shadowPath。同样，如果你要做一个bounds变化的动画，也需要给layer的shadowPath做动画来匹配bounds的变化。给layer的shadowPath做动画需要用到CAAnimation，因为UIView不能直接对shadowPath做动画(shadowPath是CALayer的属性)。用CAKeyframeAnimation可以很简单地对shadowPath做一个动画，只需从一个CGPath变化到另一个CGPath就可以了。

原文: [On the importance of setting shadowPath](http://markpospesel.wordpress.com/2012/04/03/on-the-importance-of-setting-shadowpath/)

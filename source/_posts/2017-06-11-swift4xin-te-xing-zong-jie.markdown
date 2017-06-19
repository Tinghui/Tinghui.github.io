---
layout: post
title: "Swift4新特性总结"
date: 2017-06-11 10:27
comments: true
published: false
categories: 
- Swift
- Objc
---

本文主要是对 [WWDC2017 - Session402 What's New in Swift](https://developer.apple.com/videos/play/wwdc2017/402/) 的简单总结，仅关注语言相关特性，不涉及编译器和工具链相关提升和优化。

#### Private 访问权限

![](/images/posts/wwdc17/402_hd_whats_new_in_swift_0001.png)

如上图所示代码，== 和 < 分别是 Equatable 和 Comparable 的协议方法。当你想将它们分离到单独的 extension 时，由于它们使用了私有变量，这在之前是不允许的，因为 private 的访问权限禁止在 extension 里面访问私有属性。

你可以把 private 换成 fileprivate 来解决这个访问权限问题。但随之而来的是，fileprivate 会将你的属性暴露给整个文件作用域，有点太暴露了。

对此，苹果在Swift4中做了调整（详见[SE-0169](https://github.com/apple/swift-evolution/blob/master/proposals/0169-improve-interaction-between-private-declarations-and-extensions.md)），现在 extention 中能访问同一个文件内的私有属性了。

![](/images/posts/wwdc17/402_hd_whats_new_in_swift_0002.png)

#### 组合使用类和协议

![](/images/posts/wwdc17/402_hd_whats_new_in_swift_0003.png)

如图代码所示，“???”三个问号处的类型，应该写UIControl呢，还是Shakeable？写UIControl，UIControl并没有shake()方法；写Shakeable，Shakeable类型也没有state属性呐。Swift3里面，没有太好的方式来表示这里的数据类型。

Swift4里面，类可以和任意数量的协议进行组合了。

![](/images/posts/wwdc17/402_hd_whats_new_in_swift_0004.png)

其实这个特性，Objc早就拥有了，现在Swift4也支持了。

![](/images/posts/wwdc17/402_hd_whats_new_in_swift_0005.png)







![](/images/posts/thx_money.png)



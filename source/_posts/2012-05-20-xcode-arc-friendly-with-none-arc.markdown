---
layout: post
title: "兼容ARC和non-ARC代码"
date: 2012-05-20 21:00
published: true
comments: true
categories: 工具
tag: [工具, Xcode]
---

Objective-C引入ARC(Automatic Reference Counting)后，我们经常会面对这样一种困境：自己的项目使用了ARC，却发现要使用的第三方库是non-ARC的；或者自己的项目是non-ARC的，但是想使用一个ARC的第三方库。

真是左右为难啊，是该让non-ARC迁就ARC，还是让ARC迁就non-ARC？

目前网上有2种做法来解决这个问题：

1. 将自己的ARC项目转换成non-ARC项目。
2. 将第三方类库编译成framework的形式。

但是都太麻烦了，其实我们只需要在Xcode中设置源代码的Compiler Flags就能让ARC和non-ARC文件共存。<!--more-->

点击Project->Targets->Build Phases标签->展开Compile Sources，双击某个.m文件的文件名，然后加上 `-fno-objc-arc` 这个标记，就可以指定此.m文件按照non-ARC方式编译。对应的如果加上 `-fobjc-arc` 标记，就可以指定.m文件按照ARC方式编译。

![](http://www.codeography.com/images/arc-compiler-flag.png)


另外还有一个很有用的技巧：在源代码中用__has_feature来判断是否是ARC或者non-ARC。

如以下代码，如果此代码的源文件不是按照ARC方式编译，就会报错。


```objc
#if ! __has_feature(objc_arc)
#error This file must be compiled with ARC. Use -fobjc-arc flag (or convert project to ARC).
#endif
```

参考：[Codeography](http://www.codeography.com/2011/10/10/making-arc-and-non-arc-play-nice.html)

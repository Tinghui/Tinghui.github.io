---
layout: post
title: "在Xcode中添加自定义代码片段"
date: 2012-05-06 20:00
published: true
comments: true
categories: 工具
tag: [工具, Xcode]
---
我们经常会定义一些retain的property，而且大概每次我们都会像这样写：

``` objc
@property (nonatomic, retain) Type *name;
```

每次都把 @property (nonatomic, retain) 挨个敲一遍，太累了。


那么能不能像Xcode自带的代码提示功能一样，每次只敲几个字符，代码提示就自动出来了，这样不是方便了许多吗？

是的，Xcode提供了代码片段管理功能。我们可以自定义自己的代码片段并有自动提示功能。下面我就以这段代码为例，展示如何在Xcode中添加自定义的代码片段。<!--more-->

1. 用Xcode随便打开或新建一个项目，然后随便打开一个.h或者.m文件。

2. 随便找个空白位置，输入“@property (nonatomic, retain) <#type#> *<#name#>;”。(不含双引号，“<#”、“#>”这两个符号的作用，你一会儿就明白了。)

3. 打开Xcode右侧的Utilities View，然后在其靠底部的位置找到并打开Code Snippets Library。

4. 选中我们刚刚输入的那段代码，把它拖到Code Snippets Library中。

5. 滚动到Code Snippets Library的最底部，找到一个花括号上面带个“User”文字的图标。

6. 单击那个图标，会弹出一个窗口。然后点击窗口底部左边的Edit按钮。

7. 在Title和Completion shortcut这两项中，输入代码片段的标题和快捷键。快捷键用于激活代码提示，标题则会显示在代码提示中。此例中，我们输入标题为“Objective-C @property retain”，快捷键为“@property ”。

8. 选择对应的platform、language和Completion scope。然后点击“Done”按钮。
	> 这个例子中，platform我们选All；language选Objective-C；Completion scope选Class Interface Methods。
	> Completion scope指定了激活代码提示的快捷键的有效的区域，比如这里我們选的Class Interface Methods就是说明这段代码的快捷键在声明类方法的区域才能激活代码提示；在其他任何区域，无论怎么敲这个快捷键，都不会出现这段代码的提示。

好了，现在删掉我们刚刚输入的代码。然后随便找个类的头文件，在定义property的区域，试试敲入我们刚刚设置的快捷键。你注意到了吗？我们仅仅才敲入“@p”这两个字符，代码提示就已经出来了。选中代码，回车，Xcode自动把代码给我们补全了，是不是方便多了？


PS.
* 现在你知道“<#”、“#>”这两个符号的作用了吧？
* 为什么例子中，我们的快捷键“@property ”后面要加一个空格？试试不加空格有什么效果？

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>
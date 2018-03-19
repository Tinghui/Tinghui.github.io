---
layout: post
title: "什么时候使用updateConstraints"
date: 2015-09-01 23:22
comments: true
published: true
categories: iOS
tag: [AutoLayout]
---

UIView中的`updateConstraints`方法一直是一个令人纠结的地方。按照[苹果官方文档](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIView_Class/#//apple_ref/occ/instm/UIView/updateConstraints)中给出的建议：

> Custom views that set up constraints themselves should do so by overriding this method. When your custom view notes that a change has been made to the view that invalidates one of its constraints, it should immediately remove that constraint, and then call setNeedsUpdateConstraints to note that constraints need to be updated. Before layout is performed, your implementation of updateConstraints will be invoked, allowing you to verify that all necessary constraints for your content are in place at a time when your custom view’s properties are not changing.

当视图的改变使得某个约束无效时，应当把该约束移除，并调用`setNeedsUpdateConstraints`以标记视图需要更新约束，然后在`updateConstraints`中更新约束。

然而，实际上却并不那么好用：

<!--more-->

1. 每次执行`updateConstraints()`方法时，视图的状态并不是完全相同的。视图可能已经有一些约束了，而且大部分情况下你可能只想更改一部分约束。这就导致`updateConstraints()`中散落着各种if语句用来判断指定的约束是否已经存在。
2. 视图也不一定完全拥有它的所有约束。视图层级中的其他视图可能会把约束加到你的视图上，所以你的代码也不能假设constraints数组中的哪个约束是干什么的。这意味着你必须用单独的属性来跟踪所有可能需要修改的约束。
3. 改变约束通常是在响应事件的时候。如果你遵循了官方文档的建议，在处理事件，改变内部状态后，就需要调用`setNeedsUpdateConstraints()`。结果就是：修改布局的代码（在updateConstraints()里面）和触发修改的代码分离在不同的地方，逻辑难以理解。

那么我们应该按照文档建议的那样使用`updateConstraints()`吗？在今年的WWDC技术讲座 [Mysteries of Auto Layout (Part 2) ](https://developer.apple.com/videos/wwdc/2015/?id=219)上，苹果给出了新的建议：

> Really, all this is is a way for views to have a chance to make changes to constraints just in time for the next layout pass, but it’s often not actually needed.

> All of your initial constraint setup should ideally happen inside Interface Builder. Or if you really find that you need to allocate your constraints programmatically, some place like viewDidLoad is much better. updateConstraints is really just for work that needs to be repeated periodically.

> Also, it’s pretty straightforward to just change constraints when you find the need to do that; whereas, if you take that logic apart from the other code that’s related to it and you move it into a separate method that gets executed at a later time, your code becomes a lot harder to follow, so it will be harder for you to maintain, it will be a lot harder for other people to understand.

> So when would you need to use updateConstraints? Well, it boils down to performance. If you find that just changing your constraints in place is too slow, then update constraints might be able to help you out. It turns out that changing a constraint inside updateConstraints is actually faster than changing a constraint at other times. The reason for that is because the engine is able to treat all the constraint changes that happen in this pass as a batch.

总结下来就是：不要将`updateConstraints()`用于视图的初始化设置。当你需要在单个布局流程（single layout pass）中添加、修改或删除大量约束的时候，用它来获得最佳性能。如果没有性能问题，直接更新约束更简单。



参考：

1. [How to Use updateConstraints](http://oleb.net/blog/2015/08/how-to-use-updateconstraints/)
2. [UIView Class Reference](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIView_Class/#//apple_ref/occ/instm/UIView/updateConstraints)
3. [WWDC session: Mysteries of Auto Layout (Part 2)](https://developer.apple.com/videos/wwdc/2015/?id=219)

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>


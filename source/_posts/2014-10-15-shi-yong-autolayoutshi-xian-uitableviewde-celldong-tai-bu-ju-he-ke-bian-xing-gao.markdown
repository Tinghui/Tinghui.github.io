---
layout: post
title: "使用Autolayout实现UITableView的Cell动态布局和动态行高"
date: 2014-10-15 18:22
comments: true
published: true
categories: [AutoLayout]
---

本文翻译自：[stackoverflow](http://stackoverflow.com/questions/18746929/using-auto-layout-in-uitableview-for-dynamic-cell-layouts-variable-row-heights)

有人在[stackoverflow](http://stackoverflow.com/questions/18746929/using-auto-layout-in-uitableview-for-dynamic-cell-layouts-variable-row-heights)上问了一个问题：

> 如何在UITableViewCell中使用Autolayout来实现Cell的内容和子视图自动计算行高，并且保持平滑的滚动？

这个问题获得了接近1000的支持和1100+的收藏，答案更是超过了1800+的支持，很详细的说明了如何在iOS7和iOS8上实现UITableView的动态行高计算。答案对实现UICollectionView的动态行高也具有参考意义，所以在这里将这个答案翻译了一下，希望对大家有所帮助。以下是答案的全文翻译：

全文略长，不喜欢阅读可以直接看示例代码：

* [iOS8的示例代码](https://github.com/smileyborg/TableViewCellWithAutoLayoutiOS8) - iOS8以上才支持
* [iOS7的示例代码](https://github.com/smileyborg/TableViewCellWithAutoLayout) - iOS7+

<!--more-->

### 概念描述

不管在哪个iOS版本上进行开发，前两步是必须的：

#### 1、设置好布局约束

在`UITableViewCell`子类中，添加约束，使子视图的边缘与**contentView**的边缘固定(pin)（最重要的是要有顶部和底部的边距约束）。**注意：不能将子视图的边缘设置成与cell的边缘固定，只能设置为与contentView的边缘固定！** 确保每个子视图在垂直方向上的内容压缩阻力(compression resistance)和内容吸附性约束(content hugging constraints)没有被你添加的更高优先级的约束覆盖，以使得这些子视图的固有内容尺寸(intrinsic content size)来推动contentView的高度。（嗯？点击[英文](http://stackoverflow.com/questions/22599069/what-is-the-content-compression-resistance-and-content-hugging-of-a-uiview)或[中文](http://codingobjc.com/blog/2015/01/28/autolayoutzhong-de-content-compression-resistancehe-content-huggingdao-di-shi-shi-yao-yi-si/)。）

记住，重点是cell的子视图与contentView要有垂直方向上的连结，让它们能够对contentView“施加压力”，使contentView扩张以适合它们的尺寸。

下面用一个带有一些子视图的cell作为示例，展示了**一些**必要的约束（没有展示全部的约束）：

![](http://i.stack.imgur.com/CTUPi.png)

可以想象，当更多的文本被添加到“Multi-line body”那个label上面后，它就需要垂直地增高以适应文本，这实际上将强迫cell增加高度。（当然，前提是你需要把约束设置正确！）

设置正确的约束是使用Autolayout实现动态行高时**最难也最重要**的部分。如果你犯了一个错误，它可能使后面一切都无法工作——所以，不要着急，慢慢来！我建议你用代码来设置约束，这样你就完全知道每个约束被加到了什么地方，出问题的时候也更容易调试。特别是如果你利用好一些优秀的开源库，使用代码设置约束可以变得和使用Interface Builder设置约束一样简单，而且更加强大。这里有一个我设计、维护和使用的开源库：[https://github.com/smileyborg/PureLayout](https://github.com/smileyborg/PureLayout)

* <del>如果你用代码来设置约束，应该在UITableViewCell子类的`updateConstraints`方法里面一次性完成。注意，`updateConstraints`可能不止被调用一次，因此要避免重复添加相同的约束。在`updateConstraints`中，可以将添加约束的代码包在一个if语句中(比如使用一个叫`didSetupConstraints`的布尔属性，运行一次添加约束的代码后就将其置为YES)，以确保不重复添加相同的约束。另外，更新已有约束的代码（比如调整约束的`constant`属性），也应该将它们放置在`updateConstraints`中，但是要在`didSetupConstraints`条件语句的外面，这样才能保证每次调用的时候都被执行。</del>

* 译注：上面这段`updateConstraints`中添加约束的描述，由于文章久远，已经不合时宜。苹果官方在 [WWDC2015 session219](https://developer.apple.com/videos/wwdc/2015/?id=219) 中已经给出了`updateConstraints`使用的[新建议](/2015/09/01/2015-09-01-he-shi-shi-yong-updateconstraints/)。

#### 2. 确立唯一的Cell重用标示符

为cell中每一组独特的约束，使用一个唯一的cell重用标示符。也就是说，如果cell有不止一种布局，每一种布局都应当有其对应的重用标示符。（当cell有多种布局包含不同数量的子视图的时候或者子视图以不同的方式布局的时候，你就需要使用一个新的重用标示符。）

例如，要在一个cell中显示一条email消息，可能会有4种不同的布局：第一种，只有主题；第二种，主题和正文；第三种，主题和图片附件；第四种，主题、正文和图片附件。每一种布局都需要完全不同的约束才能实现。因此，一旦cell被初始化并且约束被加到其中任意一种类型的cell上之后，cell应当得到一个唯一的重用标示符来指定该cell类型。这样，当你dequeue重用cell的时候，该cell类型的约束已经添加好了，拿来即用。

注意，由于固有内容尺寸的不同，具有相同布局约束的cell仍然可能具有不同的高度！不要混淆了不同的布局（不同的约束）和由于不同的内容尺寸而计算出（通过相同的约束来计算）的不同的视图frame这两个概念，它们本质上是两个完全不同的东西。(译注：本段翻译的不好，如果有疑惑，可以看看[原文](http://stackoverflow.com/questions/18746929/using-auto-layout-in-uitableview-for-dynamic-cell-layouts-variable-row-heights)。)

* 不要将拥有不同布局约束的cell丢到同一个重用池中（也就是使用相同的重用标示符），然后又在每次dequeue过后企图将旧的约束移除后从头开始重新添加约束。内部自动布局引擎并没有被设计来可以处理大规模的约束更改，你会看到大量的性能问题。

#### iOS8适用 - Self-Sizing Cells

##### 3. 启用行高估算

在iOS8上，苹果将许多在之前你比较难实现的东西都内置实现了。为了让cell实现自适应（self-sizing），必须先将tableView的`rowHeight`属性设置为常量`UITableViewAutomaticDimension`。然后，只需将tableView的`estimatedRowHeight`属性设置为一个非零值即可开启行高估算功能，例如：

```objc
self.tableView.rowHeight = UITableViewAutomaticDimension;
self.tableView.estimatedRowHeight = 44.0; // 设置为一个接近于行高“平均值”的数值
```

这样就为tableView提供了一个还没有被显示在屏幕上的cell的临时估算的行高。当cell即将滚入屏幕范围内的时候，会计算出真实的高度。为了确定每一行的实际高度，tableView会自动让每个cell基于其contentView的已知固定宽度（tableView的宽度减去其他额外的，像section index或accessoryView这些宽度）和被添加到contentView及其子视图上的布局约束来计算`contentView`的高度。真实的行高被计算出来之后，旧的估算的行高会被更新为这个真实的行高（并且其他任何需要对tableView的contentSize或contentOffset的更改都自动替你完成）。

一般来说，行高的估算值不需要太精确——它只是用来修正tableView中滚动条的尺寸的，当你在屏幕上滑动cell的时候，即使估算值不准确，tableView还是能很好地调节滚动条。将tableView的`estimatedRowHeight`属性设置成（在`viewDidLoad`或类似的方法中）一个接近于行高“平均值”的常量值即可。*仅在行高极端变化的时候（比如相差一个数量级），滚动过程中才会产生滚动条的“跳跃”现象。这个时候，你才需要考虑实现`tableView:estimatedHeightForRowAtIndexPath:`方法，为每一行返回一个更精确的估算值。*

#### iOS7支持（自己实现cell尺寸自适应功能）

##### 3. 完成一个完整的布局过程 & 获得行高

首先，实例化一个离屏(offscreen)的cell实例，为每个重用标示符实例化一个与之对应的cell实例，这些cell实例严格的仅用于高度计算。（离屏表示cell的引用被存储在view controller的一个属性或实例变量之中，并且这个cell绝对不会被用作`tableView:cellForRowAtIndexPath:`方法的返回值显示在屏幕上。）接下来，这个cell的内容（例如，文本、图片等等）还必须被配置为与显示在table view中的内容完全一样。

然后，强制cell立即更新子视图的布局，再在cell的`contentView`上调用`systemLayoutSizeFittingSize:`方法以计算出cell所需的高度。使用`UILayoutFittingCompressedSize`参数得到适合cell中所有内容所需的最小尺寸。然后将其高度作为`tableView:heightForRowAtIndexPath:`方法的返回值返回给table view。

##### 4. 使用估算的行高

如果你的table view超过几十行，你会发现在第一次加载table view的时候会卡住主线程。因为，在第一次加载的过程中，会对每一行调用`tableView:heightForRowAtIndexPath:`方法（为了计算滚动条的尺寸）。

iOS7中，你可以（也绝对应当）使用table view的`estimatedRowHeight`属性。这样会为还不在屏幕范围内的cell提供一个临时估算的行高值。然后，当这些cell即将要滚入屏幕范围内的时候，真实的行高值会被计算出来（通过`tableView:heightForRowAtIndexPath:`方法），估算的行高会被替换掉。

一般来说，行高的估算值不需要太精确——它只是用来修正tableView中滚动条的尺寸的，当你在屏幕上滑动cell的时候，即使估算值不准确，tableView还是能很好地调节滚动条。将tableView的`estimatedRowHeight`属性设置成（在`viewDidLoad`或类似的方法中）一个接近于行高“平均值”的常量值即可。*仅在行高极端变化的时候（比如相差一个数量级），滚动过程中才会产生滚动条的“跳跃”现象。这个时候，你才需要考虑实现`tableView:estimatedHeightForRowAtIndexPath:`方法，为每一行返回一个更精确的估算值。*

##### 5. 缓存行高（如果需要）

如果上面提到的你都做了，但是`tableView:heightForRowAtIndexPath:`的性能仍然慢的不可接受。非常不幸，这个时候你需要给行高做一些缓存（这是苹果的工程师们给出的改进建议）。大体的思路是，第一次计算时让自动布局引擎解析布局约束计算行高，然后将计算出来的行高缓存起来，之后所有对该cell的高度请求都返回缓存值。当然，还要保证任何导致cell高度变化的情况发生时都要清除缓存的行高——这通常发生在cell的内容变化时或其他重大事件发生的时候（比如用户调节了动态类型文本大小(Dynamic Type text size)的滑动条）。

##### iOS7示例代码（包含详细的注释）

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    // 判断indexPath对应cell的重用标示符，
    // 取决于特定的布局需求（可能只有一个，也或者有多个）
    NSString *reuseIdentifier = ...;

    // 取出重用标示符对应的cell。
    // 注意，如果重用池(reuse pool)里面没有可用的cell，这个方法会初始化并返回一个全新的cell，
    // 因此无论怎样，此行代码过后，你会得到一个布局约束已经完全准备好，可以直接使用的cell。
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:reuseIdentifier];

    // 用indexPath对应的数据内容来配置cell，例如：
    // cell.textLabel.text = someTextForThisCell;
    // ...

    // 确保cell的布局约束已经被设置好，因为它可能刚刚才被创建。
    // 假设你已经在cell的updateConstraints方法中设置好了约束，使用下面两行代码：
    [cell setNeedsUpdateConstraints];
    [cell updateConstraintsIfNeeded];

    // 如果你使用了多行的UILabel，不要忘了给label设置正确的preferredMaxLayoutWidth值。
    // 如果你没有在cell的layoutSubviews方法中设置，就需要在这里设置。例如：
    // cell.multiLineLabel.preferredMaxLayoutWidth = CGRectGetWidth(tableView.bounds);

    return cell;
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    // 判断indexPath对应cell的重用标示符，
    NSString *reuseIdentifier = ...;

    // 从缓存字典中取出重用标示符对应的cell。如果没有，就创建一个新的然后存储在字典里面。
    // 警告：不要调用table view的dequeueReusableCellWithIdentifier:方法，因为这会导致cell被创建了但是又未曾被tableView:cellForRowAtIndexPath:方法返回，会造成内存泄露！
    // 译注：原文这里说的dequeueReusableCellWithIdentifier:会造成内存泄漏的说法是错误的，并不会造成内存泄漏。
    UITableViewCell *cell = [self.offscreenCells objectForKey:reuseIdentifier];
    if (!cell) {
        cell = [[YourTableViewCellClass alloc] init];
        [self.offscreenCells setObject:cell forKey:reuseIdentifier];
    }

    // 用indexPath对应的数据内容来配置cell，例如：
    // cell.textLabel.text = someTextForThisCell;
    // ...

    // 确保cell的布局约束已经被设置好，因为它可能刚刚才被创建。
    // 假设你已经在cell的updateConstraints方法中设置好了约束，使用下面两行代码：
    [cell setNeedsUpdateConstraints];
    [cell updateConstraintsIfNeeded];

    // 将cell的宽度设置为与tableView的宽度一样。
    // 这点很重要。
    // 如果cell的高度取决于table view的宽度（例如，多行的UILabel通过单词换行等方式换行），
    // 那么这使得对于不同宽度的table view，我们都可以基于其宽度而得到cell的高度。
    // 但是，我们不需要在-[tableView:cellForRowAtIndexPath]方法中做相同的处理（设置宽度），
    // 因为，cell被用到table view中的时候，这一步是自动完成的。
    // 也要注意，某些情况下，cell的最终宽度可能不等于table view的宽度。
    // 例如当table view的右边显示了section index的时候，必须要减去这个宽度。
    cell.bounds = CGRectMake(0.0f, 0.0f, CGRectGetWidth(tableView.bounds), CGRectGetHeight(cell.bounds));

    // 触发cell的布局过程，会基于布局约束计算所有视图的frame。
    // （注意，你必须在cell的layoutSubviews方法中给多行的UILabel设置好preferredMaxLayoutWidth值；
    // 或者在下面2行代码前手动设置！）
    [cell setNeedsLayout];
    [cell layoutIfNeeded];

    // 得到cell的contentView需要的真实高度
    CGFloat height = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;

    // 为cell的分割线加上额外的1pt高度。因为分隔线是被加在cell底边与contentView底边之间的。
    height += 1.0f;

    return height;
}

// 注意：除非行高极端变化并且你已经明显的觉察到了滚动时滚动条的“跳跃”现象，你才需要实现此方法；否则，直接用tableView的estimatedRowHeight属性即可。
- (CGFloat)tableView:(UITableView *)tableView estimatedHeightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    // 以最小计算量，返回实际高度数量级之内的一个行高估算值。
    // 例如：
    //
    if ([self isTallCellAtIndexPath:indexPath]) {
        return 350.0f;
    } else {
        return 40.0f;
    }
}
```

### 示例项目

* [iOS8的示例代码](https://github.com/smileyborg/TableViewCellWithAutoLayoutiOS8) - iOS8以上才支持
* [iOS7的示例代码](https://github.com/smileyborg/TableViewCellWithAutoLayout) - iOS7+


最后，推荐两个相关的开源库：

* [PureLayout](https://github.com/smileyborg/PureLayout)：原文作者使用和开源的布局库，用代码写布局约束的时候很方便。
* [UITableView-CellHeightCalculation](https://github.com/Tinghui/UITableView-CellHeightCalculation)：根据本文思路封装的UITableView动态行高计算和行高缓存库，由本人开源和维护。

有任何问题，欢迎大家留言讨论！

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="30%" height="30%" /></p>



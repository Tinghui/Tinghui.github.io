---
layout: post
title: 打造更好的iOS文本输入体验
date: 2017-08-13 17:28:20
published: true
comments: true
categories:
- UI
- iOS
- 文本
---

做客户端开发，不可避免的需要和文本输入打交道。今天我们来聊聊，如何在iOS里面打造更好的文本输入体验。

全文包含4部分内容：

[1. 处理键盘遮挡视图问题](#1)
[2. 动态高度的输入框](#2)
[3. 记住用户选择的键盘](#3)
[4. 自定义键盘](#4)

<!-- more -->

#### <span id="1">处理键盘遮挡视图问题</span>

我们都知道，键盘有不同的尺寸，不同语言的键盘或不同设置的键盘，尺寸都会不同。比如纯英文键盘和汉字九宫格键盘高度就不一样，更不用说还有其他不同高度的第三方键盘呢。

而处理键盘遮挡视图问题，首先需要得到键盘的高度。iOS中有几个和键盘相关的通知：

```objc
static let UIKeyboardWillShow: NSNotification.Name
static let UIKeyboardDidShow: NSNotification.Name
static let UIKeyboardWillHide: NSNotification.Name
static let UIKeyboardDidHide: NSNotification.Name
static let UIKeyboardWillChangeFrame: NSNotification.Name
static let UIKeyboardDidChangeFrame: NSNotification.Name
```

这些通知的`userInfo`里面都包含一个叫`UIKeyboardFrameEndUserInfoKey`的key。我们注册这些通知，就可以从通知中获得键盘的`frame`信息：

```objc
@objc private func handleKeyboardNotification(_ notification: Notification) {
    guard let userInfo = notification.userInfo,
        let frame = userInfo[UIKeyboardFrameEndUserInfoKey] as? CGRect else {
            return
    }
```

获得`frame`后，并不能立即使用。因为键盘都是全屏的，它的`frame`总是处于屏幕坐标系当中的。我们需要将键盘的`frame`转化到我们的视图坐标系当中：

```objc
    let convertedFrame = view.convert(frame, from: UIScreen.main.coordinateSpace)
```

然后再计算出，键盘和我们的视图之间的重叠高度：

```objc
    let intersectedKeyboardHeight = view.frame.intersection(convertedFrame).height
```

之后再调节我们的布局约束，把被遮挡内容往上移，即可避免键盘遮挡我们的视图内容。例如：

```objc
    UIView.animate(withDuration: 0.3) {
        self.bottomConstraint.constant = intersectedKeyboardHeight + 60.0
        self.view.layoutIfNeeded()
    }
}
```

当然，如果视图是scrollView，也可以调整scrollView的`contentInset`和`contentOffset`，解决遮挡。

通常，在处理键盘遮挡问题时，我们都会让界面跟随着键盘动画一起移动。然而我们自己做的动画，可能会遇到和键盘动画不同步的问题。这个问题，[这篇文章](http://nonomori.farbox.com/post/ios-7-jian-pan-dong-hua)有讨论。正确的做法是：从`userInfo`中取出键盘动画的`duration`和`curve`来使用。

不过，现在通常都不需要自己处理键盘遮挡问题。原生的UITableViewController会自动调整`contentInset`，并且如果需要也会自动滚动内容，以避免键盘遮挡问题。此外，也有很多专门处理键盘交互的开源库，比如 [TPKeyboardAvoiding](https://github.com/michaeltyson/TPKeyboardAvoiding) 就是其中一个比较优秀的库，也是我经常使用的库，在此推荐给大家。它代码入侵少，不需要写任何配置或初始化代码，仅需将界面上使用的scrollView、tableView 或 collectionView 的类替换为TPKeyboardAvoiding对应类即可。剩下的“键盘遮挡”、“点击空白区域隐藏键盘”、“点击键盘Next按钮自动跳到下一个输入框”等问题就交给它处理。

#### <span id="2">动态高度的输入框</span>

动态高度的输入框，也很常见，比如微信聊天界面的输入框，高度就会随输入内容动态增长。实现这个需求，其实并不复杂。

首先，禁用textView的`scrollable`属性：

```objc
textView.isScrollEnabled = false
```

禁用`scrollable`属性之后，可以直接用textView的`sizeThatFits`方法，计算内容高度：

```objc
newSize.height = textView.sizeThatFits(CGSize(width: textView.bounds.width
  , height: CGFloat.greatestFiniteMagnitude)).height + kVertialPadding
```

如果要限制最大高度，增加一个高度约束即可：

```objc
override var intrinsicContentSize: CGSize {
    var newSize = bounds.size
       
    //disable textView's scrollable, then textView can calculate height itself.
    let oldScrollable = textView.isScrollEnabled
    textView.isScrollEnabled = false
    newSize.height = textView.sizeThatFits(CGSize(width: textView.bounds.width
      , height: CGFloat.greatestFiniteMagnitude)).height + kVertialPadding
    textView.isScrollEnabled = oldScrollable
       
    return newSize
}

override func awakeFromNib() {
   super.awakeFromNib()
   
   ...
   
   textChangeObserver = NotificationCenter.default.addObserver(forName: NSNotification.Name.UITextViewTextDidChange, object: textView, queue: OperationQueue.main) { [unowned self] (notification) in
      guard let textView = notification.object as? UITextView else {
          return
      }
      
      var height = self.intrinsicContentSize.height
      height = max(height, self.minHeight)
      height = min(height, self.maxHeight)
      
      let isMaxHeight = (height == self.maxHeight);
      textView.isScrollEnabled = isMaxHeight
      self.textViewHeightConstraint.isActive = isMaxHeight
      self.textViewHeightConstraint.constant = height - self.kVertialPadding
   }
}
```

#### <span id="3">记住用户选择的键盘</span>

在使用多语言的用户中，会遇到这样一个问题。例如，和朋友A聊天用的是中文，和朋友B聊天用的是英文，这样就会需要在两种键盘之间不停地切换。

在iOS的「信息」应用中有个小细节，就是当我和朋友A聊天时，应用知道我使用的是中文键盘；而当我和朋友B聊天时，应用知道我使用的是英文键盘。在键盘弹出时，自动弹出对应语言的键盘。

![](/images/posts/wwdc17-242-imessage-keyboard.jpg)

这是如何做到的？

原来是iOS系统提供一个被称作“文本输入上下文标示符”的标示符，可以将它和任何一个UIResponder关联起来。当系统发现我们将一个标示符和某个UIResponder关联起来后，它都会把标示符自动跟用户选择的键盘关联起来，并保存到user default中。下次，当这个UIResponder成为第一响应者(FirstResponder)的时候，对应的键盘就会被自动检索出来。

![](/images/posts/wwdc17-242-remember-user-selected-keyboard.jpg)

我们唯一要做的是找一个好一点的标示符。标示符需要具有唯一性，不要重复。比如说，用户ID、对话ID、群组ID，这些都是很好的标示符。

```objc
override var textInputContextIdentifier: String? {
   return "an unique identifier. ex. userId, "
}
```

#### <span id="4">自定义键盘</span>

创建自定义键盘，首先要继承`UIInputViewController`，然后在其提供的`inputView`上加入自定义的键盘界面控件。响应用户的输入事件，主要依靠`textDocumentProxy`属性进行。比如，可以通过`textDocumentProxy`属性向光标位置插入文本、或删除光标前面的文本。下图展示了键盘运行过程中一些重要的对象，以及它们在开发流程中的位置。

![](/images/posts/wwdc17-242-keyboard-architecture_2x.png)

我们开发的自定义键盘，如果只在自己的应用内部使用，只需将`inputView`传递给输入框或响应者对象：

```objc
self.textView.inputView = self.customInputViewController.inputView
```

如果自定义键盘，要在整个系统中使用，则需要创建一个自定义键盘扩展。
创建自定义键盘扩展非常简单，如果你已经实现了一个`UIInputViewController`子类，你都不需要修改任何代码，仅需要：

1. 在项目里面新建一个「Custom Keyboard Extension」类型的target
2. 将已经实现的`UIInputViewController`子类加入target
3. 将自定义键盘需要的图片等资源加入target
4. 将target的Info.plist中 NSExtension - NSExtensionPrincipalClass 键值中的类名改为你的自定义`UIInputViewController`子类名

然后，用户只要在设置里面开启键盘，就可以在系统的任何地方使用。

![](/images/posts/wwdc17-242-enable-custom-keyboard.png)

全文完！

参考：
[WWDC 2017 - Session 242 - The Keys to a Better Text Input Experience](https://developer.apple.com/videos/play/wwdc2017/242/)
[UIWindow Reference](https://developer.apple.com/documentation/uikit/uiwindow)
[iOS 7 键盘动画](http://nonomori.farbox.com/post/ios-7-jian-pan-dong-hua)
[App Extension Programming Guide - Custom Keyboard](https://developer.apple.com/library/content/documentation/General/Conceptual/ExtensibilityPG/CustomKeyboard.html)
[UIInputViewController Class Reference](https://developer.apple.com/documentation/uikit/uiinputviewcontroller)

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>


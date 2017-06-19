---
layout: post
title: "改变导航控制器的Push和Pop动画效果"
date: 2012-07-07 20:00
categories: [动画, iOS]
published: true
comments: true
---

UINavigationController的push和pop动画只有从右边推入和推出这一种效果。以下代码可以很灵活的改变UINavigationController的push或pop动画效果。

```objc
CATransition* transition = [CATransition animation];
transition.duration = 0.5;
transition.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
transition.type = kCATransitionPush; //4种: kCATransitionMoveIn, kCATransitionMoveIn, kCATransitionReveal, kCATransitionFade
transition.subtype = kCATransitionFromTop; //4种: kCATransitionFromLeft, kCATransitionFromRight, kCATransitionFromTop, kCATransitionFromBottom
[self.navigationController.view.layer addAnimation:transition forKey:nil];
[[self navigationController] popViewControllerAnimated:NO]; //关键地方：Animated:NO禁用navigationController自带的动画
```

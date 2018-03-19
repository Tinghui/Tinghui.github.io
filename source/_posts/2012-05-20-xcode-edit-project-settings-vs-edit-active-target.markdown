---
layout: post
title: "Target Settings和Project Settings的区别"
date: 2012-05-20 23:00
published: true
comments: true
categories: 工具
tag: [工具, Xcode]
---

![](/images/posts/Project-setting-vs-Target-setting.png)

一个XCode project包含了两种设置：Project Settings 和 Target Settings。

它们之间的主要区别在于：Project settings应用于project里面的所有target；而Target settings只对target本身有效，不影响project中的其他target。<!--more-->

* 如果一个选项在project settings中和target settings中都被设定了值(值会被显示为粗体)，那么target settings优先于project settings。
* 如果一个选项在target settings中没有被设定（不以粗体显示），那么它会继承project settings中的设置。如果它在project settings中也没有被设定（不以粗体显示），就会继承Xcode的默认设置。
* 一般情况下，建议配置target settings。如果在同一个project中，多个target需要使用相同的设定，那么配置project settings。

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>
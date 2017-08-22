---
layout: post
title: "iOS的Model-View-ViewModel模式"
date: 2014-05-17 13:21
comments: true
publish: true
categories: [翻译, 设计模式, 架构]
---

如果你已经做过一段时间的iOS开发，那么你一定听说过模型—视图—控制器或MVC模式。这是构建iOS应用的标准方式。不过最近，我越来越厌倦MVC的缺点。本文，我们先重温一下MVC是什么，它有什么缺点。然后再介绍一种构建应用的新方式：Model-View-ViewModel。

## Model-View-Controller

模型-视图-控制器模式(Model-View-Controller)是构造代码的权威范式。苹果甚至[这么说](https://developer.apple.com/library/ios/documentation/general/conceptual/devpedia-cocoacore/MVC.html)。MVC模式，所有对象都被界定为，要么是一个模型，要么是一个视图，或者要么是一个控制器。模型持有数据，视图向用户呈现一个可交互的界面，视图控制器做模型和视图之间的中介者。

![](http://teehanlax.com.s3.amazonaws.com/wordpress/wp-content/uploads/mvc1.png)

这个图表中，视图将用户交互通知给控制器。然后视图控制器根据这个状态的变化来更新模型。然后该模型通知(通常是通过KVO的方式)相关的控制器更新它们的视图。iOS应用大部分代码都采用这种机制。 
<!--more-->

模型对象通常是非常简单的对象。很多时候，都是Core Data的[managed object对象](https://developer.apple.com/library/ios/documentation/DataManagement/Devpedia-CoreData/managedObject.html#//apple_ref/doc/uid/TP40010398-CH23-SW1)。当然，如果你不喜欢用Core Data，也可以用其他流行的[模型层](https://github.com/MantleFramework/Mantle)对象。根据苹果的说法，模型对象包含了数据和对数据的操作。但实际应用中，模型对象通常非常轻量，对数据的操作都被混合进了控制器中。

视图（[典型的](http://blog.gaborcselle.com/2012/11/letterpress-deconstructed.html)）是UIKit组件或者程序员定义的UIKit组件的组合。就是那些在你的xib和Storyboard里面的控件：应用程序中可视和可交互的组件，像按钮、文字标签那些。视图不应该直接引用模型，并且只应该通过IBAction事件引用控制器。不属于视图的业务逻辑不应该存在于视图中。

剩下就是控制器了。控制器中放满了应用中的“胶水代码(glue code)”：那些调解模型和视图之间的交互的所有代码。控制器负责管理它们所拥有的视图的视图层级。它们要负责视图的加载、显示和隐藏等逻辑。我们还倾向于将那些不适合放在模型里面的或不适合放在视图里面的业务逻辑都装进控制器里面。这样就会面对使用MVC时的第一个问题。

## 臃肿的视图控制器

由于视图控制器里面放了太多的代码，它们常常变得特别的臃肿。视图控制器有上千行代码也不是闻所未闻的事情。这就使得你的应用无法保持轻量：臃肿的视图控制器难以维护（因为其庞大的规模），包含太多属性也让它们的状态难以控制，太多的协议代码和控制器的逻辑也搅合在一起。

臃肿的视图控制器还造成不管是通过手动测试还是单元测试都难以测试，因为它们拥有太多可能的状态。将代码拆解成更小的区块是非常好的主意。这又让我联想到最近的一个[故事](http://mikehadlow.blogspot.co.uk/2013/12/are-your-programmers-working-hard-or.html)。

## 迷失的网络逻辑

MVC（苹果推荐的那个MVC）的定义被描述为：所有的对象都可以被划分为模型、视图或控制器。那么，负责网络通信的代码放在哪里呢？使用这些接口的代码又放在什么地方呢？

你可以试着聪明地将它放进模型对象里面，不过那样会变得非常令人费解。因为网络请求应该是异步的，所以一个网络请求如果比拥有它的模型对象存活的更久，那么，真的太复杂了。你肯定也不应该将网络通信的代码放到视图里面，所以就只剩下控制器了。但这同样也不是个好主意，因为它只会将视图控制器变得更臃肿。

那么，到底放什么地方呢? MVC完全没有任何一个地方可以放那些不适合放在这三个地方的代码。

## 糟糕的可测试性

MVC的另一个大问题是，它使得开发人员很难编写单元测试的代码。由于视图控制器中混合了视图管理逻辑的代码和业务逻辑的代码，要将这些部分拆分出来做单元测试变成了一项艰巨的任务。最后只能演变为不测试......

## 模糊的“管理”定义

前面提到过，视图控制器管理了视图的层次体系；他有一个"view"属性，并且通过IBOutlet的方式可以访问这个视图的任何一个子视图。当有了很多outlet后就不好扩展，并且某些时候，你可能会用子视图控制器（child view controller）来帮助管理子视图。

这究竟会引向何方呢？什么时候将这些拆分开来会变得更有利？验证用户输入的业务逻辑是属于控制器呢，还是属于模型？

这里有许多模糊的界定，没有人能对其达成一致的意见。好像无论你怎么画这些线条，视图和对应的控制器都会如此紧密地耦合在一起，那你还不如把它们当作一个组成部分。

嘿嘿！说到这儿，就有了一个想法......

## Model-View-ViewModel

在理想的世界中，MVC可能工作的很不错。然而，我们生活在现实世界中，而事实也并非如此。我们已经详细介绍了MVC典型用法中的不足之处，让我们来看看另一种模式：模型-视图-视图模型(Model-View-ViewModel)。

MVVM[来自微软](http://msdn.microsoft.com/en-us/library/hh848246.aspx)， 但是不要因此就反对它。MVVM和MVC非常相似。它正式承认了视图和控制器的紧耦合性质，然后引入了一个新组件。

![](http://teehanlax.com.s3.amazonaws.com/wordpress/wp-content/uploads/mvvm1.png)

MVVM模式下，视图和视图控制器正式的连接在一起，我们把它们当作同一个组件。视图仍然不拥有对模型的引用，所以控制器同样也不引用模型。相反，它们都引用视图模型。

视图模型是验证用户输入、处理视图展示逻辑、发起网络请求以及放置其他杂项代码的一个好地方。视图模型不引用任何视图。它里面的逻辑如果适用于OS X，那么同样也应该适用于iOS（换句话说，不在视图模型里面#import UIKit.h就不会错）。

由于展示逻辑被划归到视图模型中，视图控制器本身就变得非常非常轻量级。最棒的是，当你刚刚开始使用MVVM的时候，你可以只在你的视图模型里面放少量的逻辑，到你对这个模式更加熟悉后再对它们进行迁移。


使用MVVM编写的iOS应用具有高度的可测试性。因为视图模型包含了所有的展示逻辑并且不引用视图，所以完全可以编程测试它。尽管[众多](http://programming.oreilly.com/2013/05/upward-mobility-unit-testing-core-data.html)的[黑客](http://www.cimgf.com/2012/05/15/unit-testing-with-core-data/)都[参与到](http://stackoverflow.com/questions/1876568/ocmock-with-core-data-dynamic-properties-problem)了Core Data模型的测试中，但用MVVM编写的应用程序完全可以用单元测试。

以我的经验，使用MVVM会略微增加一些代码量，但代码的复杂度会有整体性的降低。是非常有价值的折衷结果。

如果你再看看MVVM那个示意图，你会发现我使用了模糊的形容词——"通知(notify)"和"更新(update)"，但是并没有指出要怎么做。你可以用KVO，像在MVC里面一样，但是这很快就会变得难以控制。实践中，使用[ReactiveCocoa](http://www.teehanlax.com/blog/getting-started-with-reactivecocoa/)将各个分散的区块粘合在一起是一个很好的方式。

关于如何联合使用MVVM和ReactiveCocoa，请阅读Colin Wheeler的这篇[优秀文章](http://cocoasamurai.blogspot.ca/2013/03/basic-mvvm-with-reactivecocoa.html)，或查看我写的一个[开源应用](https://github.com/AshFurrow/C-41)。也可以阅读我写的关于ReactiveCocoa和MVVM的[书](https://leanpub.com/iosfrp)。

译自：[Model-View-ViewModel for iOS](http://www.teehanlax.com/blog/model-view-viewmodel-for-ios/)

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>
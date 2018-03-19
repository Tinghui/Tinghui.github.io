---
layout: post
title: "iOS MVC设计模式最佳实践要点"
date: 2013-05-20 11:04
comments: true
categories: 编程思想
tag: [编程思想,设计模式]
---

一些关于MVC模式的实践要点，摘自《iOS5 Programming Pushing The Limits》。

### 模型(Model)

* **一个好的模型类仅以独立的描述性的方式封装数据。**

	比如一个Person类，可能有name、address、brithdate、image这些描述性的数据。但是不应该有显示或修改这些功能。
	
* **模型类应当只引用其他模型类，永远不要引用视图(View)类或控制器(Controller)类**

	模型类可以定义一个协议，用delegate的形式和它们交互，这样就不需要引用一个特定类型的视图类或者控制器类了。
<!--more-->	

### 视图(View)

* **视图类负责展示数据**

* **视图类负责接收用户输入事件，但是不负责处理它们**

	当用户点击一个视图后，视图可以以delegate的方式发出通知“我被点击了”，但是不应该直接进入处理逻辑或者修改其他视图什么的。
	
	比如，UITableViewCell上，点击一个删除按钮后，视图应该通知delegate删除按钮被点击了，而不是直接告诉模型类删除数据或者告诉tableView移除屏幕上显示的内容，这些都是控制器的责任。


* **视图类不要引用控制器类**
	
	像模型类一样，可以用delegate的方式和控制器类交互。
	
* **除了superView或subView，视图类不要引用其他任何视图类**


* **视图类可以引用模型类**

	但它引用的模型类通常就是它要显示的那个模型类。
	
	没有引用任何特殊模型类的视图重用性更高。比如UITableViewCell，它只用于显示字符串和图片，没有引用什么特殊的模型类，所以重用性高。
	
	有时，你需要根据实际情况在重用性和易用性上做取舍。通常，对于应用开发人员来说，定制的视图类通常更容易使用；但对于框架开发人员(比如UIKit团队)来说，就会更侧重于重用性。
	
### 控制器(Controller)

* **控制器类负责协调模型和视图，它实现了应用程序的大部分逻辑**

	控制器类通常是程序中最不能重用的部分，这就是为什么要严格禁止模型类或视图类直接引用控制器类的原因。
	
	这里所说的直接引用是指引用一个特定类型的类。比如UITableView引用一个实现了<UITableViewDelegate>协议的类是OK的，都是不应该引用一个特定的，比如叫MyTableViewController的类。
	
* **另一常见的错误是，使用appDelegate保存全局数据**

	如下例子，经常这么写的话，会造成很多对象直接引用appDelegate类，这样代码会很难重用，因为这些代码都要依赖于特殊的appDelegate类。如果要访问全局数据的化，用单例模式好些。

	``` objc
	//不要这么干
	MyAppDelegate *appDelegate = (MyAppDelegate *)[[UIApplication sharedApplication] delegate];
	Something *something = [appDelegate something];
	//不要这么干
	```
	

	





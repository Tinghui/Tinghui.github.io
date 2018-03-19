---
layout: post
title: "高效的ARC代码和陷阱"
date: 2012-12-05 10:00
published: true
comments: true
categories: Objc
tag: [Objc,翻译,ARC]
---

Objective-C是一个非常酷的编程语言。[TIOBE](http://www.tiobe.com/index.php/content/paperinfo/tpci/index.html)已经公布了十一月的编程语言排行榜，Objective-C很可能会再次成为年度编程语言。Objective-C的流行不只是因为iOS和Mac OS X平台的原因，还得益于它在移动设备上的高性能。在Objective-C中，手动内存管理代替了垃圾回收机制。当然，现在Objective-C也不需要纯手动管理内存了，苹果引入了自动引用计数(Automatic Reference Counting, 简称ARC)机制，在编译时自动加入内存管理代码。大部分情况下，这非常棒。然而，当ARC和Core Foundation对象混用的时候，总是感觉很混乱。今天，我们聊聊ARC的要点和陷阱，尤其是用toll-free bridging将Objc对象和CF对象相互转换时的一些注意事项。<!--more-->

### 简介

本文先简单的介绍一下ARC的两个主要的使用场景：Objective-C的property 和 ARC修饰符。然后再重点关注一下ARC与Core Foundation框架的内存管理问题。

### ARC 和 Objective-C的property

首先感谢一下，没有[@amattn](https://twitter.com/amattn)写的《ARC最佳实践》，就很难有这篇文章。点击这个链接查看[ARC Best Practices](http://amattn.com/2011/12/07/arc_best_practices.html)的原文。下面是我总结的文章中的重点：

* 需要retain的实例变量对象，用 `strong`。

	```objc
	@property (nonatomic, strong) id childObject;
	```

* 防止引用循环(reference cycle)，要用 `weak`。

	```objc
	@property (nonatomic, weak) id delegate;
	```

* 基本数据类型，用 `assign`。

	```objc
	@property (nonatomic, assign) CGFloat width;
	@property (nonatomic, assign) CGFloat height;
	```

* 不可变的容器类型以及字符串和block，用 `copy`。要避免使用可变容器类型(比如：NSMutableArray这些...)作为property，如果要用可变的容器类型，就用 `strong`。

	```objc
	@property (nonatomic, copy) NSString* name;
	@property (nonatomic, copy) NSArray* components;
	@property (nonatomic, copy) (void (^)(void)) job;
	@property (nonatomic, strong) NSMutableArray* badPatterns;
	```

* dealloc中
	* 移除观察者。
	* 注销通知。
	* ~~停掉timer。~~(<strong>译注</strong>：因为系统会retain传递给timer的target对象，在timer停掉前，target对象是不会被dealloc的。 所以原文里的这点是错误的，<font color="red">别把停掉timer的操作放到dealloc中。</font>在确定不再需要当前这对象的时候，要主动停掉timer。)

* IBOutlets一般用 `weak`，除非它是文件的所有者(File’s Owner)在nib文件中的顶层对象。如果将它设置为strong了，就需要在`viewDidUnload`方法中将它设置为nil。

	(<strong>译注</strong>：`viewDidUnload`在iOS6.0以后没有了，要移到`didReciveMemoryWorning`中。详细请阅读苹果的《[View Controller Programming Guide for iOS](https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/DefiningYourSubclass.html#//apple_ref/doc/uid/TP40007457-CH7-SW1)》)

### ARC类型修饰符

类型修饰符的使用规则，可以看看我[以前的一篇文章](http://www.idryman.org/blog/2012/10/29/type-qualifiers/)。ARC相关的类型修饰符，有4个：

* `__strong`表示到一个对象的强引用。只要有任意一个强引用指向对象，对象就不会被销毁。这个是ARC中所有对象的默认属性。
* `__weak`指定到一个对象的弱引用。弱引用不会使它指向的对象保持一直存在，当没有任何强引用指向这个对象后，对象会被释放，然后弱引用会指向nil。
* `__unsafe_unretained`指定到一个对象的弱引用，当它指向的对象被销毁后，它不会指向nil，就变成一个野指针了。
* `__autoreleasing`用来表示参数按引用传递，在函数返回后，这个参数就被自动释放了。

注意，ARC类型修饰符只能用在指针上。也就是说，必须把修饰符放到星号的右边。

```objc
MyClass * __weak w_self = self; // 正确
MyClass __weak * w_self = self; // 错误! 可能导致很严重的bug!
__weak MyClass * w_self = self; // 错误!

__weak typeof(self) w_self = self;
// 正确，将会被展开为 __weak (MyClass *) w_self = self;
// (参见gcc的typeof说明)

typeof(self) __weak w_self = self; // 正确, 这个安全些
```

等等，难道网上这么多写法都是错误的？哦，原来因为苹果官方文档里面提到：“你应该正确修饰变量。当修饰符用在对象变量上的时候，正确的格式是 `ClassName * qualifier variableName;` ”
> You should decorate variables correctly. When using qualifiers in an object variable declaration, the correct format is: `ClassName * qualifier variableName;`

### ARC 和 Toll-free绑定

ARC和Core Foundation混用的时候坑最多。以下是一些重要的原则：

* 传递Objc对象给一个CF引用时，要retain它。
* 传递一个CF引用给一个Objc对象时，要release它。
* 传递时如果不明确的指定对象的所有权，会很危险。有时Clang编译器会帮你修正，但不是总是会，这样就会造成很严重的问题。
* Core Foundation里面没有autorelease。你必须遵守它的内存管理规则：
	* 名字中带有`Create`或`Copy`的函数返回的对象，你会持有它，因此你要负责释放它。
	* 如果函数名字中是带的`get`，你不会持有它返回的对象，不需要你释放它。

ARC中有两种方法来retain一个CF对象：用类型转换符`(__bridge_retained)` 或者 C函数`CFBridgingRetain`。我更喜欢后者，因为代码简洁清晰。

```objc
CFArrayRef arr = CFBridgingRetain( @[@"abc", @"def", @3.14] );
// or CFArrayRef arr = (__bridge_retained CFArrayRef)@[...];
// do stuffs..
CFRelease(arr);
```

当用Core Foundation中名字中带有Create或者Copy的方法获得一个对象的时候，用`(__bridge_transfor)`或者`CFBridgingRealease`来将对象的所有权转移给ARC，这样就能让ARC负责对象的释放。

```objc
- (void)logFirstNameOfPerson:(ABRecordRef)person {
	NSString *name = (NSString *)CFBridgingRelease(ABRecordCopyValue(person, kABPersonFirstNameProperty));
	NSLog(@"Person's first name: %@", name);
}
```

### Toll-free绑定的注意事项

要当心下面这种代码：

```objc
- (CGColorRef)foo {
	UIColor* color = [UIColor redColor];
	return [color CGColor];
}
```

它任何时候都可能崩溃。由于我们没有保持对UIColor的引用，它可能创建后就立即被释放掉了！CGColor也跟着释放掉了，就会引起崩溃。

有3种方法来修正这个问题：

* 使用`__autorelease`修饰符。这样UIColor就被延迟到当前runloop结束时才被释放，就不会崩溃了。应该是[@amattn](https://twitter.com/amattn)第一个发现了这种解决方法。

	```objc
	- (CGColorRef)getFooColor {
		UIColor * __autoreleasing color = [UIColor redColor];
		return [color CGColor];
	}
	```

* 使用Core Foundation的命名规则来改变返回值的所有权。

	```objc
	- (CGColorRef)fooColorCopy {
		UIColor* color = [UIColor redColor];
		CGColorRef c = CFRetain([color CGColor]);
		return c;
	}

	CGColorRef c = [obj fooColorCopy];
	// do stuffs
	CFRelease(c);
	```

* 用self来持有CF对象。但如果self被dealloc，仍然会引起崩溃。

	```objc
	- (CGColorRef)getFooColor {
		CGColorRef c = self.myColor.CGColor;
		return c;
	}
	```

### ARC中使用Block的注意事项

如果用block做实例变量，block会隐式的retain self，这样就会造成引用循环。

```objc
@interface MyClass {
	id child;
}
@property (nonatomic, strong) (void(^)(void)) job;
@end


@implementation MyClass

- (void)foo {
	self.job = ^{
		[child work];
		// will expand to [self->child work]
	};
}
```

解决这个问题的方法是使用self的弱引用，然后在block内部临时将弱引用转换为强引用。在block内部使用强引用的原因，是因为我们想确保在使用self的时候它不会被其它人释放掉。

```objc
- (void)foo {
	MyClass * __weak w_self = self;
	self.block = ^{
		MyClass *s_self = w_self; // self retained, but only in this scope!
		if (s_self) {
			[s_self->child work];
			// do other stuffs
		}
	};
}
```

### NSError的注意事项

实现带NSError参数方法的时候，要注意正确地使用类型修饰符！

```objc
- (void)doStuffWithError:(NSError * __autoreleasing *)error; // correct
- (void)doStuffWithError:(__autoreleasing NSError **)error; // wrong!
```

创建一个NSError对象的时候，通常最好把它定义成一个自动释放对象：

```objc
NSError * __autoreleasing error = nil; // 正确
__autoreleasing NSError *error = nil; // 错误
NSError *error = nil; // clang编译器下正确。(clang编译器会帮我们做优化)
```

### 总结

ARC很方便，但是刚开始使用时可能不容易搞清楚用法。当面对复杂的对象所有权关系时，可以写一些测试代码，看看对象的retain count是怎样的。下面是我用来测试retain count的代码：

```objc
- (void)testCGColorRetainCount1 {
	CGColorRef s_ref;
	@autoreleasepool {
		UIColor * __autoreleasing shadowColor = [UIColor colorWithRed:0.12 green:0.12 blue:0.12 alpha:1.0];
		s_ref = shadowColor.CGColor;
		CFRetain(s_ref);
	}
	STAssertEquals(CFGetRetainCount(s_ref), 1L, @"retain count owned by us");

	CGColorRef(^strangeBlock)(void) = ^{
		return CGColorCreateCopy(s_ref);
	};

	CGColorRef myCopy = strangeBlock();
	STAssertEquals(CFGetRetainCount(s_ref), 2L, @"retain count owned by block and us");
	CFRelease(s_ref);
	STAssertEquals(CFGetRetainCount(s_ref), 1L, @"retain count owned by block");
	STAssertEquals(CFGetRetainCount(myCopy), 1L, @"retain count owned by us");
	CFRelease(myCopy);
}
```

希望这篇文章对你有帮助！欢迎讨论和分享！

原文：[Effective Obj-C ARC and Pitfalls in It](http://www.idryman.org/blog/2012/11/22/arc-best-practices-and-pitfalls/)

译注：如果还想了解ARC更详细的知识，可以看看官方文档《[Transitioning to ARC Release Notes](http://developer.apple.com/library/ios/#releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)》或这篇《[iOS5 ARC完全指南](http://www.cocoachina.com/bbs/read.php?tid=92507)》。

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>
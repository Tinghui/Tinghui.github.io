---
layout: post
title: "一起来构建@synchronized"
date: 2015-06-27 12:03
comments: true
published: false
categories: [翻译, Objc, Swift]
---

本期的「一起来构建」，我将探索实现Objective-C的@synchronized语法。我会用Swfit来实现它，Objective-C版本的实现也什么差别。

##回顾

@synchronized是Objective-C中的一种控制构造（control construct）。 它需要一个对象的指针作为参数，然后后面接着一个代码块。对象指针作为锁，任何时候只允许一个线程访问该代码块。

这是多线程编程使用锁的一个简单方法。例如，你可以用一个NSLock来保护对一个NSMutableArray的访问：

```objc
NSMutableArray *array;
NSLock *arrayLock;

[arrayLock lock];
[array addObject:obj];
[arrayLock unlock];
```

或者你可以使用@synchronized使用数组本身作为锁：

```objc
@synchronized(array) {
	[array addObject:obj];
}
```

我个人更喜欢使用一个明确的锁，既说得清楚是怎么回事，也因为一些原因@synchronized执行效率不是很好，我们接下来会看到。但是，它很方便，并且构建一个@synchronized也很有意思。
<!--more-->

##实现原理
Swift版本的@synchronized是一个函数，它接受一个对象和一个闭包，持有一个锁并调用闭包：

```objc
func synchronized(obj: AnyObject, f: Void -> Void) {
	...
}
```

现在问题来了，如何将任意一个对象转换成一个锁？

理想世界中（从实现这个函数的角度），每个对象都应该有一些额外的空间预留一个锁。然后synchronized函数就可以在那些额外空间上适当地使用锁定和解锁的功能。然而并米有额外的空间存在，这可能是幸运的，因为它会使得系统上的所有对象因为一个大部分对象永远不会用到的功能而造成内存膨胀。

另一种方法是，使用一个表将一个对象映射到一个锁。然后synchronized函数可以查找表中的锁，并锁定和解锁。这种方法的问题在于，表本身需要是线程安全的，这需要它自己的锁或某种充满想象力的无锁数据结构。使用一个单独的锁更简单。

为了防止不断地创建锁，表需要追踪锁的使用情况，在不再需要时销毁或重用锁。

##实现

将对象映射到锁的表，用NSMapTable非常适合。它可以配置为使用原始对象地址作为其键值，并且它可以持有键和值的弱引用，这允许系统自动回收不用的锁。这样进行设置:

```objc
let locksTable = NSMapTable.weakToWeakObjectsMapTable()
```

锁对象是NSRecursiveLock类的实例。由于它是一个类，它可以很好地在NSMapTable中使用，而不像pthread_mutex_t。并且NSRecursiveLock也与@synchronized一样提供了递归语义。

表本身也需要一个锁。自旋锁（spinlock）在这里效果很好，因为访问表将是短暂的：

```objc
var locksTableLock = OS_SPINLOCK_INIT
```

表已准备好，我们可以实现函数了：

```objc
func synchronized(obj: AnyObject, f: Void -> Void) {
```

它做的第一件事是在locksTable表查找与obj对应的锁。这必须在locksTableLock锁住的情况下进行：

```objc
OSSpinLockLock(&locksTableLock)
var lock = locksTable.objectForKey(obj) as! NSRecursiveLock?
```

如果没在表中找到对应的锁，就创建并设置一个新的锁：

```objc
if lock == nil {
	lock = NSRecursiveLock()
	locksTable.setObject(lock!, forKey:obj)
}
```

拿到锁后，主表锁可以被释放。为了避免死锁，必须在调用闭包f之前释放：

```objc
OSSpinLockUnlock(&locksTableLock)
```

现在我们可以调用闭包f，在调用之前和之后用lock进行加锁和解锁：

```objc
lock!.lock()
f()
lock!unlock()
```

##对比苹果的实现

苹果的@synchronized实现被作为Objective-C运行时的一部分代码开源了。源代码可以在这里找到：

[http://www.opensource.apple.com/source/objc4/objc4-646/runtime/objc-sync.mm](http://www.opensource.apple.com/source/objc4/objc4-646/runtime/objc-sync.mm)

这份实现更注重速度而不是上面简单的玩具式实现能比的。然而，看看什么相同什么不同也挺有趣。

基本的概念是相同的。有映射对象指针和锁的全局表，@synchronized代码块前后有加锁和解锁操作。

对于底层的锁对象，苹果的版本使用的是配置为递归锁的pthread_mutex_t。由于NSRecursiveLock很可能是用pthread_mutex_t实现的，这减少了中间环节，避免了在运行时对Foundation的依赖。

表本身被实现为一个链表，而不是哈希表。由于通常在任何给定时间只有少量的锁存在，所以这仍将表现良好，而且可能执行效率比哈希表更好，因为哈希表的性能优势在处理大量数据集的时候才能显现出来。另外用一个per-thread缓存保存了当前线程上最近查找过的锁，这进一步提升了性能。

相比于单个全局表，苹果的实现中用一个数组保存了16个表。根据不同对象的地址，它们被映射到不同的表。这减少了不同对象之间操作@synchronized代码块时不必要的竞争，因为它们很可能会使用不同的全局表。

相比于使用弱指针产生大量的额外开销，苹果的实现中为每个锁保存了一个内部引用计数。当引用计数到达0，锁就变成可被新对象重用。虽然不用的锁没有被销毁，但重用意味着在任何给定时间，锁的总数被限制为最大可激活锁的数量，而不是根据新对象的使用而无限制的增长。

苹果的实现很出色很快速，但相比于使用一个单独的、明确的锁，它仍然会带来一些不可避免的额外开销，特别是：

1. 如果碰巧被分配到相同的全局表中，不相关的对象仍然可能产生竞争。
2. 普通情况下在查找锁的时候，如果自旋锁不存在于per-thread缓存中，就必须要创建和释放自旋锁。
3. 在全局表中为对象查找锁要做额外的工作。
4. 每个锁定/解锁周期都会带来递归语义的开销，即使不是必需的时候。

然而，这些问题或多或少是由于@synchronized的目的而与生俱来的，不能怪责到实现头上。这是一段伟大的代码，值得一读。

##总结

@synchronized是一个有趣的具有一定实现挑战的语言构造。从根本上说，它提供了线程安全，但实现中它自身也需要安全的同步性。背后使用一个全局锁来保护对表的访问解决了这个难题。苹果的实现使用了聪明的技巧使得其相当快。

本文译自：[https://www.mikeash.com/pyblog/friday-qa-2015-02-20-lets-build-synchronized.html](https://www.mikeash.com/pyblog/friday-qa-2015-02-20-lets-build-synchronized.html)











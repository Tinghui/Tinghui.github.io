---
layout: post
title: "在init和dealloc中使用accessor"
date: 2013-04-24 16:23
comments: true
published: true
categories: Objc
tag: [Objc,翻译]
---

过去几年Cocoa社区已经发生了很大变化。曾经，在init/dealloc中使用accessor方法是很令人厌恶的，这种做法彻底地被认为是错误的。他们的说法是：最好直接操作实例变量，而不使用accessor方法。并且建议像这样写代码：

```objc
- (id)initWithWhatever:(id)whatever
{
    if((self = [self init]))
    {
        _whatever = [whatever retain];
    }
    return self;
}

- (void)dealloc
{
    [_whatever release];

    [super dealloc];
}
```
<!--more-->


与之对应，使用accessor的版本，是这样的：

```objc
- (id)initWithWhatever:(id)whatever
{
    if((self = [self init]))
    {
        [self setWhatever:whatever];
    }
    return self;
}

- (void)dealloc
{
    [self setWhatever: nil];

    [super dealloc];
}
```


### accessor的优点

与在其他任何地方使用accessor一样，在init/dealloc中使用accessor有这样一些优点：它们使你的代码与实现解耦，并且还可以减轻你在内存管理方面的工作。

回忆一下，多少次你意外的把代码写成这样了：

```objc
- (id)init
{
    _ivar = [NSArray arrayWithObjects:...];
    return self;
}
```

我想答案可能各异(我已经很久没有犯过这种错误了)，但是使用accessor可以确保你不会犯这种错误：

```objc
- (id)init
{
    [self setIvar: [NSArray arrayWithObjects:...]];
    return self;
}
```

此外，你也许有一些附属状态(像缓存，统计等等这些)需要在对象的值变化时进行设置或者关闭。正确的使用accessor可以保证不需要过多重复的代码，所有的这些都能够在对象被创建时或被销毁时正常的工作。(译注类似KVO这种情形)


### accessor的缺点

使用accessor的缺点可以简单的总结成一句话：有副作用。有时候这些副作用会在init/dealloc中产生不良影响。

在写一个setter方法的时候，如果你打算在init/dealloc里面调用它，要确保它的行为正确。这意味着要处理对象没有完全被创建时的情况。

更糟糕的情况是，如果你一旦要在子类中重写setter方法，你需要将它写成能够处理父类用它来初始化或销毁实例变量的情况。比如，这段看上去很正常的代码却有潜在的危险：

```objc
- (void)setSomeObj:(id)obj
{
    [anotherObj notifySomething];
    [super setSomeObj:obj];
}

- (void)dealloc
{
    [anotherObj release];
    [super dealloc];
}
```

如果父类用setter方法来销毁someObj，就会在dealloc中`[anotherObj release]`已经执行后再执行子类重写的那个版本的setter方法。重载的setter方法就会去操作一个野指针，很可能就会来一个漂亮的崩溃。

要优雅地修正这个问题不是很难。在dealloc中释放后将anotherObj指向nil就行了。一般情况下，要使你的重载行为保持正常并不是很困难，但是如果你要这样用accessor，那么一定要记住这一点。

### 键-值观察者(KVO)
这类讨论中经常提到的一个话题就是键-值观察者，因为accessor的副作用通常都是KVO引起的。当对象没有被完全创建时或者已经被销毁后，KVO的调用就会带来不幸的事情。但是我个人认为这是一个不靠谱的说法。

因为99%的情况下都是，只有一个对象被完全创建后KVO才能被设置到对象上，并且会在对象被销毁前撤销掉。实际情况下，KVO不太可能会在对象创建前就被激活或者对象销毁后才被终止。

在对象被完全初始化前，外部代码不可能对对象做任何操作。除非你的初始化方法自己要把指针到处传递给外部对象。同样，当你的对象正在执行dealloc方法的时候，外部对象也不可能继续保持对你的对象的KVO引用。父类的代码确实会在这个时候执行，但是父类的代码不太可能会导致外部对象可以观察一个它自己甚至都没有的属性。

### 结论

现在优点和缺点都知道了，你会在init和dealloc中使用accessor吗？在我看来，都可以。用，也没太大的优势；不用，也没什么缺点。大多数的实例变量我都没用accessor方法。但是在某些特定情况下，在对变量做了一些特殊处理的时候，这些情况下使用accessor会带来显著好处的时候，我会毫不犹豫地使用accessor，少些烦恼。


原文：[Friday Q&A 2009-11-27: Using Accessors in Init and Dealloc](http://www.mikeash.com/pyblog/friday-qa-2009-11-27-using-accessors-in-init-and-dealloc.html)

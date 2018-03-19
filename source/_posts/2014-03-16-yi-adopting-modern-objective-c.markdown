---
layout: post
title: "Adopting Modern Objective-C"
date: 2014-03-16 11:46
comments: true
categories: Objc
tag: [Objc,翻译]
---

苹果2014年03月10日发布了一个新文档，介绍了Objective-C的几个新技巧，包括：

* [用instancetype代替id](#instancetype)
* [用@property代替实例变量](#property)
* [用NS_ENUM或NS_OPTIONS代替enum](#enum)
* [采用ARC](#arc)

文档名字叫《[Adopting Modern Objective-C](https://developer.apple.com/library/ios/releasenotes/ObjectiveC/ModernizationObjC/RevisionHistory.html#//apple_ref/doc/uid/TP40014150-CH99-SW1)》，将它翻译成中文了，以下是正文。
<!--more-->

# 采用现代的Objective-C

历经多年，Objective-C语言已经得到了许多增长和演变。虽然核心概念和做法保持一致，但这个语言的部分已经发生显著的变化和改进。这些现代化的改进增强了Objective-C的类型安全、内存管理、性能和一些其他方面，使你可以更轻松地编写正确的代码。在你现有的和将来的代码中采用这些改进可以使你的代码变得更一致，可读性更强，更灵活。

XCode提供了一个工具来帮你完成这些结构上的更改。但是在你开始使用这个工具之前，你应该想了解一下它会给你的代码带来什么样的改变，以及为什么会带来这样的改变。本文档重点介绍了一些你可以在你的代码中应用的最显著的和最有用的新特性。


## <span id="instancetype">**instancetype**</span>

在返回类实例对象的方法中用`instancetype`关键词作为方法的返回值类型。这包括在`alloc`、`init`和类工厂方法等方法中。

在适当的地方用`instancetype`代替`id`，可以提高你的Objective-C代码的类型安全。例如，考虑下面的代码：

```objc
@interface MyObject : NSObject
+ (instancetype)factoryMethodA;
+ (id)factoryMethodB;
@end

@implementation MyObject
+ (instancetype)factoryMethodA { return [[[self class] alloc] init]; }
+ (id)factoryMethodB { return [[[self class] alloc] init]; }
@end

void doSomething() {
    NSUInteger x, y;
    x = [[MyObject factoryMethodA] count]; // Return type of +factoryMethodA is taken to be "MyObject *"
    y = [[MyObject factoryMethodB] count]; // Return type of +factoryMethodB is "id"
}
```

因为`+factoryMethodA`的返回值类型为`instancetype`，该消息表达式的类型为`MyObject*`。由于MyObject没有`count`方法，编译器给出了一个关于x行的警告：

```objc
main.m: ’MyObject’ may not respond to ‘count’
```

然而，因为`+factoryMethodB`的返回值类型为`id`，编译器无法给出关于y行的警告。由于id类型的对象可以是任何类，并且由于一个叫`count`的方法可能存在于某个类的某个地方，对于编译器来说，它就认为`+factoryMethodB`方法返回的对象可能实现了该方法。

为了确保instancetype工厂方法有正确的子类化行为，在alloc类内存的时候一定要用`[self class]`，而不要直接引用类名。遵循这一约定可以确保编译器能够正确的推断出子类类型。例如，考虑MyObject子类的情形：

```objc
@interface MyObjectSubclass : MyObject

@end

void doSomethingElse() {
    NSString *aString = [MyObjectSubclass factoryMethodA];
}
```

编译器给出以下警告：

```objc
main.m: Incompatible pointer types initializing ’NSString *’ with an expression of type ’MyObjectSubclass *’
```

这个示例中，+factoryMethodA消息返回一个MyObjectSubclass类型的对象，这是消息接收者(receiver)类型的对象。编译器会相应地推断出+factoryMethodA的返回值类型应该是子类MyObjectSubclass，而不是定义了该工厂方法的父类。

### 如何应用

在你的代码中，在适当的地方用`instancetype`做返回值，替换掉`id`。通常是在`init`方法和类工厂方法的情形中。尽管编译器会自动将返回值类型为id的并且开头为"alloc"、"init"或"new"的方法的返回值类型转换成instancetype类型，但它不会去转换其他的方法。Objective-C的约定是，要为所有有需要的方法明确地写上instancetype。

要注意的是，只有在id作为返回值类型的地方才能用instancetype代替它，其他地方不行。与id不同的是，instancetype关键字只能在方法声明中被用作返回值类型。

例如：

```objc
@interface MyObject
- (id)myFactoryMethod;
@end
```

应该变成：

```objc
@interface MyObject
- (instancetype)myFactoryMethod;
@end
```

或者，你也可以用Xcode中的modern Objective-C转换器来自动转换你的代码。欲了解更多信息，请参阅“[使用Xcode重构你的代码](#end)”。

## <span id="property">**Properties**</span>

Objective-C的属性(property)是指用`@property`语法定义的公有的或私有的方法。

```objc
@property (readonly, getter=isBlue) BOOL blue;
```

属性保存了对象的状态。它们反映了对象的固有属性和对象与其他对象之间的关系。属性提供了一种安全的、便捷的方式来与这些属性进行交互，而无需编写一套自定义的访问(accessor)方法(虽然，如果有需要的话，属性确实允许自定义getter和setter方法)。

尽量在尽可能多的地方使用属性来代替实例变量，会有许多好处：

* **自动生成getter和setter方法。** 当你定义了一个属性，默认会自动为你创建对应的getter和setter方法。

* **更好的定义一组方法。** 因为访问方法的命名约定，它会让getter和setter方法的用途更明确。

* **属性关键词可以表达出对应行为的额外信息。** 属性提供了`assign(相对于 copy)`、`weak`、`atomic(相对于 nonatomic)`等等特性。

属性方法遵循一个简单的命名约定。getter方法的名字和属性名字相同(例如，date)，setter方法的名字是属性名字带一个"set"前缀并采用驼峰命名规则(例如，setDate)。布尔类型的属性还可以定义一个以"is"开头的getter方法：

```objc
@property (readonly, getter=isBlue) BOOL blue;
```

其结果是，以下都是有效的：

```objc
if (color.blue) { }
if (color.isBlue) { }
if ([color isBlue]) { }
```

当决定哪些可以作为一个属性的时候，请记住，以下这些不属于属性：

* `init`方法
* `copy`和`mutableCopy`方法
* 类工厂方法
* 开启某项操作并返回一个BOOL结果的方法
* 明确的改变了一个getter的内部状态的副作用方法。

另外，在你的代码中标示属性特性的时候请考虑以下规则：

* 一个可读写(read/write)的属性有两个访问方法。setter方法接受一个参数并且没有返回值，getter方法不接受任何参数并返回一个值。如果将这组方法转换成一个属性，就可以用`readwrite`关键字来标记它。
* 一个只读(read-only)的属性只有一个访问方法。即getter方法，它不接受任何参数，并且返回一个值。如果将这个方法转换成一个属性，就可以用`readonly`关键字标记它。
* getter方法应当是幂等的(idempotent，如果一个getter方法被调用两次，那么第二次调用时返回的结果应该和第一调用时返回的结果相同)。然而，如果一个getter方法每次调用时，是被用于计算结果，这是可以接受的。

### 如何应用

识别出一组可以被转换成一个属性的方法，如这些方法：

```objc
- (NSColor *)backgroundColor;
- (void)setBackgroundColor:(NSColor *)color;
```

用`@property`语法和适当的关键字将它们定义成一个属性：

```objc
@property (copy) NSColor *backgroundColor;
```

有关属性关键词和其他注意事项，请参阅《[Encapsulating Data](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html)》。

或者，你也可以用Xcode中的modern Objective-C转换器来自动转换你的代码。欲了解更多信息，请参阅“[使用Xcode重构你的代码](#end)”。

## <span id="enum">**枚举宏**</span>

`NS_ENUM`和`NS_OPTIONS`宏提供了一种在基于C的语言中定义枚举的简洁而又简单的方法。这两个宏增强了XCode中相关的代码补全功能，并且明确地指定了枚举和可选项(options)的类型和大小。此外，该语法定义的枚举既可以使得旧的编译器能够正确地计算枚举值，也可以使新的编译器能够解析出枚举值的类型信息。

使用`NS_ENUM`宏来定义枚举，即一组互斥的值：

```objc
typedef NS_ENUM(NSInteger, UITableViewCellStyle) {
    UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
};
```

`NS_ENUM`宏同时定义了枚举的名称和类型，这个示例中名称是UITableViewCellStyle，类型是NSInteger。枚举的类型应该尽量是NSInteger的。

使用`NS_OPTIONS`宏来定义可选项(options)，即一组可以相互结合的按位掩码(bitmasked)的值：

```objc
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
    UIViewAutoresizingNone                 = 0,
    UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
    UIViewAutoresizingFlexibleWidth        = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
    UIViewAutoresizingFlexibleHeight       = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5
};
```
像枚举一样，`NS_OPTIONS`宏也同时定义了名称和类型。但是，可选项的类型通常应该是NSUInteger的。

### 如何应用

更换你的枚举声明，比如这个：

```objc
enum {
    UITableViewCellStyleDefault,
	UITableViewCellStyleValue1,
	UITableViewCellStyleValue2,
	UITableViewCellStyleSubtitle
};
typedef NSInteger UITableViewCellStyle;
```

更换成NS_ENUM语法：

```objc
typedef NS_ENUM(NSInteger, UITableViewCellStyle) {
    UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
};
```

但当你用enum来定义一个位掩码的时候，比如这里：

```objc
enum {
    UIViewAutoresizingNone                  = 0,
    UIViewAutoresizingFlexibleLeftMargin    = 1 << 0,
    UIViewAutoresizingFlexibleWidth         = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin   = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin     = 1 << 3,
    UIViewAutoresizingFlexibleHeight        = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin  = 1 << 5
};
typedef NSUInteger UIViewAutoresizing;
```
要换成使用NS_OPTIONS宏：

```objc
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
    UIViewAutoresizingNone                 = 0,
    UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
    UIViewAutoresizingFlexibleWidth        = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
    UIViewAutoresizingFlexibleHeight       = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5
};
```

或者，你也可以用Xcode中的modern Objective-C转换器来自动转换你的代码。欲了解更多信息，请参阅“[使用Xcode重构你的代码](#end)”。

## <span id="arc">**自动引用计数(ARC)**</span>

自动引用计数(Automatic Reference Counting)是一个编译器特性，它提供了Objective-C对象的自动内存管理功能。而你不必再需要记住什么时候使用`retain`、`release`和`autorelease`了，ARC会评估你的对象的生命周期，并在编译期自动帮你插入适当的内存管理调用。编译器也会为你生成相应的`dealloc`方法。

### 如何应用

XCode提供了一个工具，可以自动帮你完成ARC转换过程中的手动操作部分(比如移除对`retain`和`release`的调用)，也可以帮助你解决迁移工具不能自动处理的问题。要使用ARC迁移工具，在XCode菜单中选择“Edit > Refactor > Convert to Objective-C ARC”，迁移工具会将项目中的所有文件转换至使用ARC。

欲了解更多信息，请参阅《[Transitioning to ARC Release Notes](https://developer.apple.com/library/ios/releasenotes/objectivec/rn-transitioningtoarc/introduction/introduction.html)》。

## <span id="end">**使用XCode重构你的代码**</span>

XCode提供了一个现代的Objective-C转换工具，可以协助你完成转换工作。虽然该转换工具可以在某些潜在的地方帮助你识别和应用这些现代的特性，但它不解释你代码的语义。例如，它不能检测出你的`-toggle`方法是一个可以影响你的对象的状态的方法，它会错误地提议将这个方法转换成一个更现代化的属性。请务必手动检查和确认转换工具给你的代码提供的任何修改。

综上所述的现代化的特性，这个转换工具可以提供以下这些转换功能：

* 在适当的地方用instancetype替换id
* 将enum转换至NS_ENUM或NS_OPTIONS
* 将部分方法更新成@property语法

除了这些，这个转换工具还会建议你对代码做一些其他的修改，包括：

* 转换至字面值语法(literals)，因此像`[NSNumber numberWithInt:3]`这样的语句会变成`@3`。
* 使用下标语法，因此像`[dictionary setObject:@3 forKey:key]`这样的语句会变成`dictionary[key] = @3`。

要使用这个现代的Objective-C转换工具，在Xcode菜单中选择“Edit > Refactor > Convert to Modern Objective-C Syntax”。

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>
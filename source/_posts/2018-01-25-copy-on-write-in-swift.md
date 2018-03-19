---
title: Swift的写时复制机制
date: 2018-01-25 20:18:09
tags: [Swift,内存,写时复制]
---

在Swift中，我们有引用类型（比如：类）和值类型（比如：结构体、元组、枚举）。值类型具有复制语义。这意味着如果你将一个值类型分配给一个变量，该值的底层数据将会被复制。你会得到两个具有相同内容的值，但分配在不同的内存地址中。

如下例所示，把`str1`赋值给`str2`后，`str2`的内容和`str1`的内容相同，都是字符串“Hello”，但两个值的内存地址不同。

```swift
var str1 = "Hello"  //Hello
var str2 = str1     //Hello，str1的内容被复制给str2，str2的内容和str1的内容一样

address(of: str1)   //0x604000059670
address(of: str2)   //0x604000059cd0
```

然而，把包含大量信息的值类型作为参数分配或传递给函数时，对于性能来说，复制是非常昂贵的操作。为了尽量减少这个问题，Swift标准库为一些值类型（比如数组、字典）实现了一套叫做**[写时复制](https://zh.wikipedia.org/wiki/寫入時複製)**的机制。

其核心思想是，如果有多个调用者（callers）同时要求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。

所以，如果只是将数组分配给一个变量或将其传递给一个函数并不会复制它，直到对它进行修改时才可能会进行复制，这样就提高了性能。

<!--more-->

如下例所示，将`arr1`赋值给`arr2`时，并没有产生复制行为，两个数组的内存地址是相同的。当给`arr2`中加入一个新元素`7`之后，`arr2`发生改变，可以看到两个数组的内存地址不同。

```swift
var arr1 = [1, 3, 5]    //[1, 3, 5]
var arr2 = arr1         //[1, 3, 5]

address(of: arr1)       //0x600000075020
address(of: arr2)       //0x600000075020

arr2.append(7)          //[1, 3, 5, 7]

address(of: arr1)       //0x600000075020
address(of: arr2)       //0x600000082640

func address(of object: UnsafeRawPointer) -> String {
    let addr = Int(bitPattern: object)
    return String(format: "%p", addr)
}
```

## 实现写时复制

写时复制不是值类型的默认行为，在Swift标准库中，也只有少数几个值类型实现了写时复制，比如：数组、字典、集合。也就是说并非所有的标准库中的值类型都有这个行为。如果我们有一个包含大量信息的结构体，也需要自己实现写时复制来提高性能以避免无用的复制。

为了实现高效的写时复制特性，我们需要知道一个对象是否是唯一的。如果它是唯一的，我们可以直接原地修改对象，否则我们就需要在修改前创建对象的一个副本。为此我们需要用到Swift中的`isKnownUniquelyReferenced`函数来检查引用是否只有一个持有者。

```swift
final class Ref<T> {
  var val : T
  init(_ v : T) {val = v}
}

struct Box<T> {
    var ref : Ref<T>
    init(_ x : T) { ref = Ref(x) }

    var value: T {
        get { return ref.val }
        set {
          if (!isKnownUniquelyReferenced(&ref)) {
            ref = Ref(newValue)
            return
          }
          ref.val = newValue
        }
    }
}
```

这个例子来自Swift官方的[性能优化技巧](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#advice-use-copy-on-write-semantics-for-large-values)，它展示了如何使用引用类型为一个泛型值类型实现写时复制。本质上，它只是一个管理引用类型的包装器，如果该值未被唯一引用，则只返回新实例；否则，只改变引用类型的值。

## 总结

写时复制是一种非常巧妙的用于优化值类型的复制性能的方法。这是一个在Swift中大量被使用的机制，尽管大多数时候我们都没有明确地看到它，但标准库中某些常用类型已经实现了这个机制。我们自定义的值类型，如果需要实现相同的机制，使用`isKnownUniquelyReferenced`函数也能很容易的实现。

</br>
参考：
1. [Swift进阶](https://www.objccn.io/products/advanced-swift/) 一书中的「结构体和类」章节
2. [Value and Reference Types](https://developer.apple.com/swift/blog/?id=10)
3. [Understanding Swift Copy-on-Write mechanisms](https://medium.com/@lucianoalmeida1/understanding-swift-copy-on-write-mechanisms-52ac31d68f2f)

</br>
<p style="text-align:center"><img src="/images/posts/thx_money.png" width="50%" height="50%" /></p>


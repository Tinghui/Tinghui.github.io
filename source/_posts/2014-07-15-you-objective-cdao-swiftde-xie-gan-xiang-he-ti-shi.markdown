---
layout: post
title: "Objective-C到Swift：一些想法和建议"
date: 2014-07-15 09:25
comments: true
categories: [Objc, Swift, 编程思想]
---

本文分享一些在由Objective-C转换至Swift的过程中产生的思考，并尝试总结了一些技巧建议和一些误区陷阱。本文尽量通过对比来展示这两种语言在同一个问题上的不同处理方法。开始吧！

## 单文件 vs. 接口-实现文件

第一个重大的变化是：`interface.h/implementation.m`的文件结构被舍弃了。

我是这种文件结构的坚定支持者。只通过接口文件来获取或共享类相关的信息，不仅安全而且很方便。

在Swift中，接口和实现并没有分离。我们只能实现自己的类（并且在写代码的时候甚至不能添加可见性修饰符）。
<!--more-->

如果真的无法忍受这个变化，可以使用以下这些方法，但有些要慎用：

第一个是显而易见的：**要使用普遍的用法**

良好的文档可以轻易地增加类的可读性。例如，我们可以将我们想要“公开（public）”的元素都移到文件的顶部，可以使用类扩展（extension）来隔离公有的和私有的区域。

另一个真正常见的做法是：所有私有的方法和变量的命名都以下划线“_”开头。

这里有个混合了这两种方法的简短例子：

```objc
// Public
extension DataReader {
    var data { }
    func readData(){
        var data = _webserviceInteraction()
    }
}
 
// Private implementation
class DataReader: NSObject {
    
    let _wsURL = NSURL(string: "http://theurl.com")
     
    func _webserviceInteraction()->String{
        // ...
    }
}
```

虽然我们不能修改类元素的可见性，但是我们可以尝试让它们变得“更难”被直接访问到。

有种奇特的解决方法是：使用内嵌的类来部分隐藏私有区域(至少可以对代码自动补全功能达到隐藏)。下面是一个例子：

```objc
import UIKit
 
class DataReader: NSObject {
     
    // Public ***********************
    var data:String?{
        get{return private.internalData}
    }
     
    init(){
        private = DataReaderPrivate()
    }
 
    func publicFunction(){
        private.privateFunc()
    }
 
     
    // Private **********************
    var private:DataReaderPrivate
     
    class DataReaderPrivate {
        var internalData:String?
         
        init(){
            internalData = "Private data!"
        }
         
        func privateFunc (){}
    }
}
```

我们将私有实现放入了一个名叫**private**的常量实例，然后用“正规的”类实现来作为公共的接口。这些私有的元素并没有被真正的隐藏，但要访问它们，就不得不通过"private"常量。

```objc
let reader = DataReader()
reader.private.privateFunc()
```

**问题是：**要达到部分隐藏私有元素的目的而使用这种怪异的模式是否值得？

我的建议是再等等，再等等可见性修饰符相关的更新(<font color="orange">译注：在翻译这篇文章的时候，苹果已经在Swift中引入[访问权限](https://developer.apple.com/swift/blog/?id=5)了</font>)。在此期间，不管是否使用类扩展的方式，都要写好文档。

## 常量和变量

在Objective-C中，我真的很少使用**const**关键词，甚至即便我知道某些数据永远不会被改变的时候我也没有用const（好吧...我羞愧）。在Swift中，苹果建议开发者多考虑选择使用常量(**let**)而不是使用变量(**var**)。因此，多留意一下，试着找到变量的最佳角色。最终，你会比你预想多得多的使用常量。

## 只写必须要写的

看看下面这两行代码，找出有什么不同：

```objc
let wsURL:NSURL = NSURL(string:"http://wsurl.com");
vs
let wsURL = NSURL(string:"http://wsurl.com") 
```

在开始转向Swift的前两周，我强迫自己不要在每行代码后面写**分号**。现在好多了（但是我现在经常忘了在Objective-C中写分号 ![](http://www.thinkandbuild.it/wp-includes/images/smilies/icon_neutral.gif) ）。

**类型推断**是一种直接从变量的定义推断出它的类型的能力。这又是一个带来方便的地方。但是如果是从一种冗长(verbose)的语言(像Objective-C)而来时，会觉得有点难以接受。

我们应当试着使用一些命名约定来对方法名进行命名，否则如果选择了一种非常不幸的命名方式，那么另外的开发者(也包括你自己)会很难猜出类型推断出的是什么类型：

```objc
let a = something() 
```

一个更合理的名字可以帮助我们理解：

```objc
let a = anInt() 
```


另一个重要的变化是圆括号的使用：它们不再是必需的了。

```objc
if (a > b){}
     vs   
if a > b {}
```

但是请记住，我们写在圆括号当中的东西是被当作一个表达式的，并不总是可以不写括号。例如，在变量绑定（variable binding）中，我们不能使用圆括号：

```objc
if (let x = data){} // Error! 
if let x = data {} // OK!
```

我们没有被强制要求必须要采用类型推断或者移除掉分号和圆括号，但是我们可以认为这些是Swift推荐的代码写法。最终提高了可读性，并且减少了一些键盘打字量。

## 可选类型(OPTIONALS)

有多少次，你都在和要么返回“一个值”要么返回“空”的函数打交道，你有没有想过什么是定义“空”的最好方式？我用过NSNotFound、-1、0和其他一些自定义的返回值......

感谢可选类型(Optionals)，现在我们拥有了“空或有值(nothing-value)”的完整定义，只需要在数据类型后面加一个问号即可。

我们可以这样写：

```objc
class Person{
    let name:String
    let car:Car? // Optional value
     
    init(name:String){
        self.name = name
    }
}
 
// ACCESSING THE OPTIONAL VALUE ***********
 
var Mark = Person(name:"mark")
 
// use optional binding 
if let car = Mark.car {
    car.accelerate()
}
 
// unwrap the value
Mark.car?.accelerate()
```

这个例子中，"人有一辆车"这个关系被定义成了一个可选类型。这表示car这个属性可以是nil，一个人可以没有车。

然后我们使用了可选类型绑定(optional binding)(if let car =)或可选类型解析(unwrap)(car?)来访问这个值。

如果我们没有将一个属性定义为一个可选类型，那么我们需要为它设置一个值，否则编译器很快就会抱怨。

定义一个非可选类型属性值的最后机会是在初始化方法里面。因此，我们需要确定类的属性会如何和类的其他部分进行交互以及在类实例中它们会有何种行为。

这些改进彻底改变了我们构思类的方法。


## 可选类型解析(OPTIONALS UNWRAPPING)

如果你发现可选类型用起来有些困难，那是因为你还没理解为什么编译器要求你在使用一个值之前要先将它给解包(unwrap)...

```objc
Mark.car?
```

...我建议你将可选类型想象成一个结构体（它是一个结构体，所以不会太难 ![](http://www.thinkandbuild.it/wp-includes/images/smilies/icon_razz.gif)），它没有直接包含你的值，而是给它包(wrap)了一层。如果内部这个值定义了，你把外面包的一层给解开(unwarp)后，就得到值了；否则就得到nil。BOOM，想通了吧！

使用“!”这个符号是强行解包的意思。解包的时候不会关心内部是什么值。你就在“冒着风险”使用里面的值。如果值为nil，应用程序就会挂掉。

## 代理模式

在用Objective-C和Cocoa写了多年程序之后，我们都对代理模式上瘾了。

不要怕！我们仍然采用和过去一样的方式来使用这种模式。下面有个超级简单的使用代理模式的例子：

```objc
@objc protocol DataReaderDelegate{
    @optional func DataWillRead()
    func DataDidRead()
}
 
class DataReader: NSObject {
    
    var delegate:DataReaderDelegate?
    var data:NSData?
 
    func buildData(){
         
        delegate?.DataWillRead?() // Optional method check
        data = _createData()
        delegate?.DataDidRead()       // Required method check
    }
}
```

这里使用了一个优雅的**可选链(optional chaining)**来代替**respondToSelector**进行代理是否存在的检查。

```objc
delegate?.DataWillRead?()
```

注意，由于我们使用了**@optional**，所以需要在协议前面加**@obj**。如果我们忘了，编译器也会给我们一个明确的警告信息。

为了实现这个代理，我们用另外一个类实现了这个协议，然后用它进行赋值，就和在Objective-C中的做法一样：

```objc
class ViewController: UIViewController, DataReaderDelegate {
                             
    override func viewDidLoad() {
        super.viewDidLoad()
         
        let reader = DataReader()
        reader.delegate = self
    }
 
    func DataWillRead() {...}
     
    func DataDidRead() {...}
}
```

## 目标—动作模式(TARGET ACTION PATTERN)

另一个我们在Swift中仍然在用的模式就是**目标-动作模式(target-action)**，并且这次甚至是以在Objective-C中一模一样的方式来使用它。

```objc
class ViewController: UIViewController {
     
    @IBOutlet var button:UIButton
     
    override func viewDidLoad() {
        super.viewDidLoad()
         
        button.addTarget(self, action: "buttonPressed:", forControlEvents: UIControlEvents.TouchUpInside)
    }
 
    func buttonPressed(sender:UIButton){...}
}
```

只有一个真正不同的地方，就是定义**selector**的方式。我们只需将方法原型写成一个字符串，它就可以自动被转换成类似下面这样的东西：

```objc
Selector("buttonPressed:") 
```

## 单例模式

爱或不爱，单例模式仍然是被使用的最多的模式。

我们可以用GCD的dispatch_once来实现它，或者可以直接依靠天生线程安全的**let**关键词来实现。

```objc
class DataReader: NSObject {
     
    class var sharedReader:DataReader {
         
        struct Static{
            static let _instance = DataReader()
        }
         
        return Static._instance
    }
...
}
```

让我们快速浏览一下这段代码。

1. sharedReader是一个静态的复合（compound）属性（我们也可以用一个函数来代替这个实现）。
2. 静态（非复合）属性现在还不允许出现在类实现中。因此要感谢**内嵌类型**，我们在类里面添加了一个内嵌的结构体。

 结构体支持静态属性，所以我们只需把静态属性加在那儿就可以了。

3. _instance属性是一个常量。它不能被修改为其他值，并且是线程安全的。

 
我们可以像下面这样引用DataReader的单例实例：

```objc
DataReader.sharedReader
```

## 结构体和枚举

Swift中，结构体和枚举有许多你很难在其他语言中能够找到的特性。

它们支持方法：

```objc
struct User{
    // Struct properties
    let name:String
    let ID:Int
     
    // Method!!!
    func sayHello(){
        println("I'm " + self.name + " my ID is: \(self.ID)")
    }
}
 
let pamela = User(name: "Pamela", ID: 123456)
pamela.sayHello()
```

你可以看到，这个结构体使用了一个由Swift自动创建的初始化方法（我们也可以添加其他自定义的实现）。

枚举的语法和我们以前使用的语法有点不同。

它使用关键词**case**来定义枚举值：

```objc
enum Fruit { 
  case orange
  case apple
}
```

枚举也不仅限于整型值：

```objc
enum Fruit:String { 
  case .orange = "Orange"
  case .apple = "Apple"
}
```

也可以构造行为更复杂的枚举类型：

```objc
enum Fruit{
     
    // Available Fruits
    case orange
    case apple
     
    // Nested type
    struct Vitamin{
        var name:String
    }
     
    // Compound property
    var mainVitamin:Vitamin {
     
    switch self{
    case .orange:
        return Vitamin(name: "C")
         
    case .apple:
        return Vitamin(name: "B")
        }
    }
} 
 
let Apple = Fruit.apple
var Vitamin = Apple.mainVitamin
```

在前面的代码中，我们添加了一个内嵌类型（Vitamin）和一个复合类型（mainVitamin）。这两个类型都依靠枚举值来初始化元素值。令人兴奋，对吧？

## 可变和不可变

在Objective-C中，我们习惯了像NSArray和NSDictionary这类的有可变和不可变版本的类。

在Swift中，我们不再需要不同的数据类型，只需利用常量或变量定义即可。

变量数组是可以被修改的。但常量数组，我们就无法修改其存储的值。因此，只要记住“let = 不可变， var = 可变”这个规则就好（但注意：在Swift的Beta 3之前，你是可以修改一个let数组的）。

## Block和Closure

<font color="orange">译注：由于个人偏好问题，没有将“Block”翻译成“块代码”，“Closure”翻译成“闭包”，这两个词都保留了原文，需要的请自行脑补它们的翻译。 </font>

我爱死Block的语法了，它们真是太清晰、太好记了！

```objc
</好了，反话模式结束>
```

但，顺便还是要说一句：用Cocoa做开发这么多年后，我们已经非常习惯于使用这个语法。有的时候，我甚至更喜欢用它来代替代理模式来完成一些简单的任务。它们灵巧、使用方便，完全是有用的。

Swift中，类似于Block的东西是Closure。Closure极其强大，并且苹果也做了大量的工作来简化我们编写它们的方式。

Swift官方文档上的例子让我惊呆了。

文档上，是以这样的一个定义开始的：

```objc
reversed = sort(names, { (s1: String, s2: String) -> Bool in
    return s1 > s2
})
```

然后它被重构成了这样：

```objc
reversed = sort(names, >) 
```

因此，感谢类型推断，我们有不同的方式来实现一个Closure，可以使用速记参数列表(shorthand arguments)（$0, %1）和直接用运算符函数(operator functions)（>）。

这篇文章中，我不打算过多的讨论Closure的语法，但是我想花些时间谈谈Closure是如何捕获(capture)值的。

在Objective-C中，如果我们想在block中修改一个变量的值，我们需要将它定义成**__block**。但是使用Closure，就用不着这样了。

我们可以随意访问和修改周围作用域中的任意一个值。Closure已经足够聪明，能够自己捕获要使用的外部元素。一个元素可以被以一个“**副本**”或一个“**引用**”的方式捕获。如果Closure要修改元素的值，它就会创建一个引用；如果不修改，它就创建一个副本。

我们来看看下面这个例子：

```objc
class Person{
     
    var age:Int = 0
     
    @lazy var agePotion: (Int) -> Void = {
        (agex:Int)->Void in
            self.age += agex
    }
     
    func modifyAge(agex:Int, modifier:(Int)->Void){
        modifier(agex)
    }   
}
 
var Mark:Person? = Person()
Mark!.modifyAge(50, Mark!.agePotion)
Mark = nil // Memory Leak
```

agePotion这个closure使用了self，持有了一个指向当前实例的强引用。与此同时，那个实例又持有了一个指向这个closure的引用...BOOM...产生强引用循环了！

为了避免这种问题，我们要使用捕获列表（**Capture List**）。这个列表关联了一个指向我们要在closure中使用的实例的弱引用或无主（unowned）引用。其对应的语法也相当简单，只需在closure的定义前面加上**[unowned/strong self]**，然后就可以得到一个无主引用或弱引用。

```objc
@lazy var agePotion: (Int) -> Void = {
     [unowned self](agex:Int)->Void in
         self.age += agex
}
```

## 无主引用和弱引用

我们已经知道弱引用是如何在Objective-C中工作的。它在Swift中的工作方式也一样，没有什么不同的。

那么，无主引用呢？我真的很感激引入了这个关键词，因为它可以对类与类之间的关系起到一个很好的暗示作用。

我们来表述一下一个人与他的银行账户之间的简单关系：

1. 一个人可以有一个银行账户（可选的）
2. 一个银行账户应该属于某一个人（必需的）

我们可以用如下的代码来描述这种关系：

```objc
We can describe this relation with code: 
class Person{
    let name:String
    let account:BankAccount!
     
    init(name:String){
        self.name = name
        self.account = BankAccount(owner: self)
    }
}
 
class BankAccount{
    let owner:Person
     
    init(owner:Person){
        self.owner = owner
    }
}
```

这些关系将会产生一个引用循环。第一种解决方法是，在“BankAccount.owner”属性上加一个弱引用。然而，如果使用**无主**引用，我们可以定义另一种有用的约束条件：属性必须有一个值，它不能为nil（这种方式下，前面列出的2个条件我们都满足了）。

关于无主引用实在没有什么可以多说的。它和弱引用一样，不会增加它指向的对象的引用计数，但是保证了它引用的不是一个空值（nil）。

## 最后的思考

我不得不承认：我有时还是会遇到编译错误，并最终沉默地看着它们，心想：“WAT?”

越用得多，它就变得越清晰。它值得上我花的每一个小时来实验和学习它。它带来了许多不同于Objective-C的有趣的变化和一些以前不存在的东西。这些都让我想多实践一下，去适应它。

它是iOS和OSX开发中的一股值得欢迎的新鲜空气，我肯定你们也会喜欢它的！

Ciao

[Follow @bitwaker](https://twitter.com/bitwaker)

译自：[FROM OBJECTIVE-C TO SWIFT: THOUGHTS AND HINTS](http://www.thinkandbuild.it/from-objective-c-to-swift/)

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="30%" height="30%" /></p>
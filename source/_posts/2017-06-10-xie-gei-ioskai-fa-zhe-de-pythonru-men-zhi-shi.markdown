---
layout: post
title: "写给iOS开发者的Python入门介绍"
date: 2017-06-10 14:00
comments: true
published: false
categories: Python
tag: [Python]
---

最近对Python来了兴趣，在[codecademy.com](https://www.codecademy.com)上断断续续花了几天，学完了Python的入门教程，这里做个简单的总结。

## 基本语法

Python是一门简单易学的编程语言，它的Hello Word是这样的：

```
print "Welcome to Python!"
```

##### 变量

Python定义变量时不指定类型，必须有初始值，初始值是什么类型，变量就是什么类型。内建的基本数据类型有：int、float、boolean。boolean的取值为`True`或`False`。

```
my_int = 7
my_float = 1.23
my_bool = True
```

Python在语法意义上没有常量的概念，只在[PEP 8 – Style Guide for Python Code](http://www.python.org/dev/peps/pep-0008/)中约定常量名应为大写加下划线。

##### 数学运算

加、减、乘、除、求模，这些数学运算都是很直观的，不过多说明：

```
addition = 72 + 23
subtraction = 108 - 204
multiplication = 108 * 0.5
division = 108 / 9
spam = 51 % 5
```

对比Objc和Swift，Python多了一个操作符`**`，表示指数运算：

```
eight = 2 ** 3 # 2的3次方
eighty_one = 3 ** 4 # 3的4次方
```

##### 注释

单行注释用`#`：

```
# This is a comment line
mysterious_variable = 42
```

多行注释用三引号包起来：

```
"""comment line1
comment line2
comment line3
"""
mysterious_variable = 42
```

##### 代码结构

Python使用空格缩进来组织代码结构，如果缩进没对，代码结构就是错误的，最终会导致程序报错或执行错误。不过好在大部分IDE都有缩进指示.

例如以下代码（`def`关键词用于定义函数，稍后有介绍），在第二行会报错：
> IndentationError: expected an indented block

```
def spam():
eggs = 12
return eggs
        
print spam()
```

## 字符串

Python的字符串类型是`str`，再次说一句，定义变量不需要指定类型，给个初始值就行了：

```
brian = "Hello life!"
```

Python里面单引号也能表示字符串，但字符串里面又包含单引号时，需要加转义字符`\`：

```
'This isn\'t flying, this is falling with style!'
```

使用双引号，就不需要转义字符了：

```
"This isn't flying, this is falling with style!"
```

##### 下标访问

Python中，可以通过下标访问字符串中的字符：

```
c = "cats"[0]
n = "Ryan"[3]
```

##### 字符串常用方法

字符串类型常用的4个方法：`len()`、`lower()`、`upper()`、`str()`。

`len()` - 求字符串长度

```
parrot = "Norwegian Blue"
print len(parrot) # 打印出 14
```

`lower()` - 转换成小写

```
parrot = "Norwegian Blue"

print parrot.lower() # 打印出 norwegian blue
```

`upper()` - 转换成大写

```
parrot = "norwegian blue"

print parrot.upper() # 打印出 NORWEGIAN BLUE
```

`str()` - 将非字符串类型转化成字符串

```
str(2) # 将2转化成"2"
str(3.14) #将3.14转化成"3.14"
```

其中len和str是通用函数，可以支持其他数据类型。
lower和upper是字符串类型的成员函数，需要通过字符串实例对象访问。

##### 格式化

字符串支持连接：

```
print "Spam " + "and " + "eggs"
print "The value of pi is around " + str(3.14)
```

格式化由`%`表示，注意字符串和变量之间也是用`%`连接：

```
string_1 = "Camelot"
string_2 = "place"

print "Let's not go to %s. 'Tis a silly %s." % (string_1, string_2)
```

## 日期和时间

Python中使用日期和时间，需要import datetime:

```
from datetime import datetime

now = datetime.now()
print now # 打印出当前日期和时间，2017-06-10 07:53:49.077099类似格式
```

获取年月日等单元信息很方便，相比起来NSDate就要麻烦很多：

```
from datetime import datetime

now = datetime.now()
print now
print now.year
print now.month
print now.day
```

## 条件和流程控制

Python支持的比较操作：==、!=、<、<=、>、>=，和Objc/Swift没什么差别。

布尔操作符并没有：&& 、|| 和 !，与之对应的是：and、or、not。优先级 not > and > or：

##### if语句

if/else if/else，在Python中对应的是：if/elif/else。

if语句，条件表达式没有小括号，以冒号结尾，语句体没有大括号，依靠缩进来表示：

```
def the_flying_circus():
    if 1 < 10 and 1 > 0:
        return True
    elif 1 > 1:
        return False
    else:
        return False
```








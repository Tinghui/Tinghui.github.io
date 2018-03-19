---
layout: post
title: "使用objc的AssociatedObject函数给现有类添加新property"
date: 2013-05-16 11:49
comments: true
categories: Objc
tag: [Objc,运行时]
---

Objective-C的Category(类别)是一个很强大的特性。使用Category我们可以给现有类增加一些新的方法。但是也仅限于方法，不能直接增加新的property。但通过运行时中的`objc_setAssociatedObject`/`objc_getAssociatedObject`函数，就可以很方便的为现有类增加新property。

如下面代码所示：

```objc
//
//  NSObject+Test.h
//  Test
//
//  Created by ZhangTinghui on 13-5-16.
//  Copyright (c) 2013年 ZhangTinghui. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface NSObject (Test)

@property (nonatomic, copy) NSString *testString;

@end
```
<!--more-->

``` objc
//
//  NSObject+Test.m
//  Test
//
//  Created by ZhangTinghui on 13-5-16.
//  Copyright (c) 2013年 ZhangTinghui. All rights reserved.
//

#import "NSObject+Test.h"
#import <objc/runtime.h>

@implementation NSObject (Test)

- (void)setTestString:(NSString *)testString
{
    objc_setAssociatedObject(self, @selector(testString), testString, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)testString
{
    return objc_getAssociatedObject(self, @selector(testString));
}

@end
```
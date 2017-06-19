---
layout: post
title: "使用NSSecureCoding协议进行编解码"
date: 2014-04-15 14:08
comments: true
categories: [翻译, NSCoding, runtime]
---

在iOS和Mac OS上，`NSCoding`是一种简单方便的数据存储方法。它可以直接将你的数据模型对象写入一个文件，之后又可以直接将它们读入内存而不需要编写任何文件解析和序列化的逻辑。将一个对象（假设它已经实现了NSCoding协议）保存至一个文件，只需要这样做：

```objc
Foo *someFoo = [[Foo alloc] init];
[NSKeyedArchiver archiveRootObject:someFoo toFile:someFile];
```

之后要加载时，只需要这样：

```objc
Foo *someFoo = [NSKeyedUnarchiver unarchiveObjectWithFile:someFile];
```
<!--more-->

那些编译到应用里面的资源采用这种方式挺好（比如nib文件，它其实就是用的NSCoding方式），但是使用NSCoding读写用户数据文件的问题在于，由于你是将整个类编码进一个文件，因此在你的应用中就隐式的赋予了该文件实例化类对象的权限。

虽然你不能在一个被NSCoding的文件中存储可执行代码（至少在iOS上不行），但黑客可能会用一个特制的文件欺骗你的应用，来实例化一个你意想不到的类对象，或者在一个你想象不到的环境下实例化类对象。虽然这样做很难带来任何真正的伤害，但这肯定会导致应用崩溃或用户数据丢失。

iOS6中，苹果引入了一个基于NSCoding的新协议，叫做`NSSecureCoding`。NSSecureCoding和NSCoding几乎完全相同，除了解码的时候你需要指定要解码的对象的key和类，并且如果指定的类和从文件解码到的对象的类不匹配的时候，NSCoder会抛出一个异常来告诉你该数据已经被篡改。

大多数支持NSCoding的系统对象都已经升级到支持NSSecureCoding，所以给你的NSKeyedUnarchiver设置要求使用安全编码(secure coding)功能，就可以确保你加载到的数据文件是安全的。如下所示：

```objc
// Set up NSKeyedUnarchiver to use secure coding
NSData *data = [NSData dataWithContentsOfFile:someFile];
NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
[unarchiver setRequiresSecureCoding:YES];
 
// Decode object
Foo *someFoo = [unarchiver decodeObjectForKey:NSKeyedArchiveRootObjectKey];
```

注意，如果使用NSKeyedUnarchiver的安全编码功能，那么存储在这个文件中的所有对象都必须遵循NSSecureCoding协议，否则你会得到一个异常。要让你的类支持NSSecureCoding协议，需要在`initWithCoder:`方法里面实现新的解码逻辑，并且要让`supportsSecureCoding`方法返回YES。`encodeWithCoder:`方法不需要修改，因为安全问题在读取过程中才可能出现，不会出现在保存的过程中。

```objc
@interface Foo : NSObject 
 
@property (nonatomic, strong) NSNumber *property1;
@property (nonatomic, copy) NSArray *property2;
@property (nonatomic, copy) NSString *property3;
 
@end
 
@implementation Foo
 
+ (BOOL)supportsSecureCoding
{
  return YES;
}
 
- (id)initWithCoder:(NSCoder *)coder
{
  if ((self = [super init]))
  {
    // Decode the property values by key, specifying the expected class
    _property1 = [coder decodeObjectOfClass:[NSNumber class] forKey:@"property1"];
    _property2 = [coder decodeObjectOfClass:[NSArray class] forKey:@"property2"];
    _property3 = [coder decodeObjectOfClass:[NSString class] forKey:@"property3"];
  }
  return self;
}
 
- (void)encodeWithCoder:(NSCoder *)coder
{
  // Encode our ivars using string keys as normal
  [coder encodeObject:_property1 forKey:@"property1"];
  [coder encodeObject:_property2 forKey:@"property2"];
  [coder encodeObject:_property3 forKey:@"property3"];
}
 
@end
```

几周前，我写了「[如何在运行时通过内省(introspection)来检测类的属性，以实现自动NSCoding](http://iosdevelopertips.com/cocoa/nscoding-without-boilerplate.html)」。

这是一个非常棒的方法，它能一下子让你的所有的模型对象都支持NSCoding，而无需重复的编写initWithCoder:和encodeWithCoder:方法，从而也减少了出错的几率。但是我们使用的这个方法不支持NSSecureCoding，因为我们无法对正在加载的对象进行类型验证。

那么，如何增强我们的自动NSCoding系统以支持NSSecureCoding呢？

如果你还记得，原有的实现是使用了`class_copyPropertyList()`和`property_getName()`这两个运行时函数来生成了一组属性名字，然后我们将它们存入了一个数组：

```objc
//引入Objective-C运行时的头文件
#import <objc/runtime.h> 
 
- (NSArray *)propertyNames
{    
  //获得属性列表
  unsigned int propertyCount;
  objc_property_t *properties = class_copyPropertyList([self class], 
    &propertyCount);
  NSMutableArray *array = [NSMutableArray arrayWithCapacity:propertyCount];
  for (int i = 0; i < propertyCount; i++)
  {
    //获得属性名字
    objc_property_t property = properties[i];
    const char *propertyName = property_getName(property);
    NSString *key = @(propertyName);
 
    //加入数组中
    [array addObject:key];
  }
 
  //记得释放属性列表，因为ARC不会替我们释放它
  free(properties);
 
  return array;
}
```

通过使用KVC,我们就能够通过名字来设置和获取一个对象的所有属性，并在一个NSCoder对象中将它们编码或解码。

实现自动NSSecureCoding时，我们也遵循相同的原则。但，除了获取属性的名字之外，我们还需要获取到其对应的类型。幸运的是，Objective-C的运行时存储了类属性的类型信息，所以获取名字的同时，获取类型数据也很容易。

类的属性可以是基本数据类型（如整型、布尔型和结构体），也可以是对象（如NSString、NSArray等等）。KVC的`valueForKey:`方法和`setValue:forKey:`方法实现了对基本数据类型的自动化“装箱（boxing）”操作。就是说，它们会将整型、布尔型和结构体这些基本数据类型转换成NSNumber或NSValue对象。这样，对我们来说就简单多了，因为我们只需要处理装箱后的对象就可以了。因此我们就能够将我们所有的属性类型按照类来处理，而不必为了不同的属性类型调用不同的解码方法。

虽然，运行时不会把每个属性装箱后对应的类名给我们，但它会给我们对应的类型编码信息——一个包含了类型信息的特殊格式的C字符串（与`@encode(var);`语法返回的字符串格式相同）。由于没有能够自动获取基本数据类型的等价类的方法，所以我们需要解析这个字符串，然后自己指定相应的类。

苹果官方描述类型编码字符串格式的文档在[这里](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)。

第一个字符就代表了对应的基本数据类型。Objective-C为每个支持的基本类型都定义了一个唯一的字符，比如‘i’表示一个整数，‘f’表示浮点数，‘d’表示double，等等。对象被表示为‘@’(后接类名)，还有另外一些生僻的类型，如‘:’表示selector，或‘#’表示类。

花括弧{...}包含起来的表达式代表struct和union类型。仅有一部分struct和union类型被KVC支持。KVC支持的这部分类型会被装箱成NSValue对象，因此我们可以将任何以‘{’开头的值都做同样的处理。

使用switch，基于字符串的第一个字符，我们就能够处理所有的已知类型：

```objc
Class propertyClass = nil;
char *typeEncoding = property_copyAttributeValue(property, "T");
switch (typeEncoding[0])
{
  case 'c': // Numeric types
  case 'i':
  case 's':
  case 'l':
  case 'q':
  case 'C':
  case 'I':
  case 'S':
  case 'L':
  case 'Q':
  case 'f':
  case 'd':
  case 'B':
  {
    propertyClass = [NSNumber class];
    break;
  }
  case '*': // C-String
  {
    propertyClass = [NSString class];
    break;
  }
  case '@': // Object
  {
    //TODO: get class name
    break;
  }
  case '{': // Struct
  {
    propertyClass = [NSValue class];
    break;
  }
  case '[': // C-Array
  case '(': // Enum
  case '#': // Class
  case ':': // Selector
  case '^': // Pointer
  case 'b': // Bitfield
  case '?': // Unknown type
  default:
  {
    propertyClass = nil; // Not supported by KVC
    break;
  }
}
free(typeEncoding);
```

要处理‘@’类型，我们还需要获得类名。类名可能包含了协议名，因此我们要把字符串进行分割，只提取出类名，然后使用NSClassFromString函数来获得对应的类：

```objc
case '@':
{
  //类的objcType只少3个字符长度
  if (strlen(typeEncoding) >= 3)
  {
    //拷贝得到C字符串形式的类名
    char *cName = strndup(typeEncoding + 2, strlen(typeEncoding) - 3);
 
    //转换为一个NSString，以便后续操作的处理
    NSString *name = @(cName);
 
    //剔除类名后面的协议名字
    NSRange range = [name rangeOfString:@"<"];
    if (range.location != NSNotFound)
    {
      name = [name substringToIndex:range.location];
    }
 
    //根据类名获取对应的类，如果没有对应的类则默认为NSObject
    propertyClass = NSClassFromString(name) ?: [NSObject class];
    free(cName);
  }
  break;
}
```

最后，我们可以将这种解析逻辑与前面的实现中的propertyNames方法的逻辑组合起来，以属性名字作为key，创建一个返回包含属性对应类的字典的方法。下面是完整的实现：

```objc
- (NSDictionary *)propertyClassesByName
{
  // Check for a cached value (we use _cmd as the cache key, 
  // which represents @selector(propertyNames))
  NSMutableDictionary *dictionary = objc_getAssociatedObject([self class], _cmd);
  if (dictionary)
  {
      return dictionary;
  }
 
  // Loop through our superclasses until we hit NSObject
  dictionary = [NSMutableDictionary dictionary];
  Class subclass = [self class];
  while (subclass != [NSObject class])
  {
    unsigned int propertyCount;
    objc_property_t *properties = class_copyPropertyList(subclass, 
      &propertyCount);
    for (int i = 0; i < propertyCount; i++)
    {
      // Get property name
      objc_property_t property = properties[i];
      const char *propertyName = property_getName(property);
      NSString *key = @(propertyName);
 
      // Check if there is a backing ivar
      char *ivar = property_copyAttributeValue(property, "V");
      if (ivar)
      {
        // Check if ivar has KVC-compliant name
        NSString *ivarName = @(ivar);
        if ([ivarName isEqualToString:key] || 
          [ivarName isEqualToString:[@"_" stringByAppendingString:key]])
        {
          // Get type
          Class propertyClass = nil;
          char *typeEncoding = property_copyAttributeValue(property, "T");
          switch (typeEncoding[0])
          {
            case 'c': // Numeric types
            case 'i':
            case 's':
            case 'l':
            case 'q':
            case 'C':
            case 'I':
            case 'S':
            case 'L':
            case 'Q':
            case 'f':
            case 'd':
            case 'B':
            {
              propertyClass = [NSNumber class];
              break;
            }
            case '*': // C-String
            {
              propertyClass = [NSString class];
              break;
            }
            case '@': // Object
            {
              //TODO: get class name
              break;
            }
            case '{': // Struct
            {
              propertyClass = [NSValue class];
              break;
            }
            case '[': // C-Array
            case '(': // Enum
            case '#': // Class
            case ':': // Selector
            case '^': // Pointer
            case 'b': // Bitfield
            case '?': // Unknown type
            default:
            {
              propertyClass = nil; // Not supported by KVC
              break;
            }
          }
          free(typeEncoding);
 
          // If known type, add to dictionary
          if (propertyClass) dictionary[propertyName] = propertyClass;
        }
        free(ivar);
      }
    }
    free(properties);
    subclass = [subclass superclass];
  }
 
  // Cache and return dictionary
  objc_setAssociatedObject([self class], _cmd, dictionary, 
    OBJC_ASSOCIATION_RETAIN_NONATOMIC);
  return dictionary;
}
```

最困难的部分已经完成。现在，要实现NSSecureCoding，我们只需要修改前面实现自动化逻辑的代码中的initWithCoder:方法，接收属性对应类以进行解析操作。同样，我们还需要让supportsSecureCoding方法返回YES：

```objc
+ (BOOL)supportsSecureCoding
{
  return YES;
}
 
- (id)initWithCoder:(NSCoder *)coder
{
  if ((self = [super init]))
  {
    // Decode the property values by key, specifying the expected class
    [[self propertyClassesByName] enumerateKeysAndObjectsUsingBlock:(void (^)(NSString *key, Class propertyClass, BOOL *stop)) {
      id object = [aDecoder decodeObjectOfClass:propertyClass forKey:key];
      if (object) [self setValue:object forKey:key];
    }];
  }
  return self;
}
 
- (void)encodeWithCoder:(NSCoder *)aCoder
{
  for (NSString *key in [self propertyClassesByName])
  {
    id object = [self valueForKey:key];
    if (object) [aCoder encodeObject:object forKey:key];
  }
}
```

好了，现在你的模型类已经拥有了一个可以打开即用的支持NSSecureCoding的简单基类。另外，你也可以直接使用我写的一个叫`AutoCoding`的分类，它正是使用了上述方法来为任何一个没有实现NSCoding和NSSecureCoding协议的对象加入自动NSCoding和NSSecureCoding能力。

译自：[Object Encoding and Decoding with NSSecureCoding Protocol](http://iosdevelopertips.com/general/object-encoding-and-decoding-with-nssecurecoding.html)

原文作者为[Nick Lockwood](https://twitter.com/nicklockwood)，他是[《iOS Core Animation: Advanced Techniques》](http://www.informit.com/store/ios-core-animation-advanced-techniques-9780133440751)一书的作者，也是iCarousel、iRate等开源项目的作者。

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="30%" height="30%" /></p>
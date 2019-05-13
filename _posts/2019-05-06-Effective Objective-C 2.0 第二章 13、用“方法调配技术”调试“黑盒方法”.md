---
layout:     post
title:      Effective Objective-C 2.0 第二章 13、用“方法调配技术”调试“黑盒方法”
subtitle:   黑魔法
date:       2019-05-06
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

之前在我的这篇文章
[iOS runtime Method Swizzling](https://www.jianshu.com/p/8adedb05a8b2)已经详细分析过，本篇介绍书中内容。

Objective-C 对象收到消息之后，究竟会调用何种方法需要在运行期才能解析出来。那你也许会问：与给定的选择子名称相对应的方法是不是也可以在运行期改变呢？没错，就是这样。若能善用此特性，则可发挥出巨大优势，因为我们既不需要源代码，也不需要通过继承子类来覆写方法就能改变这个类本身的功能。这样一来，新功能将在本类的所有实例中生效，而不是仅限于覆写了相关方法的那些子类实例。此方案经常称为“方法调配”（method swizzling）。

类的方法列表会把选择子的名称映射到相关的方法实现之上，使得“动态消息派发系统”能够据此找到应该调用的方法。这些方法均以函数指针的形式来表示，这种指针叫做IMP，其原型如下：

```
typedef id (*IMP)(id, SEL, ...); 
```

NSString类可以响应lowercaseString、uppercaseString、capitalizedString等选择子。这张映射表中的每个选择子都映射到了不同的IMP之上，如图:

![](https://upload-images.jianshu.io/upload_images/1776587-765b85b8e5098201.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Objective-C 运行期系统提供的几个方法都能够用来操作这张表。开发者可以向其中新增选择子，也可以改变某选择子所对应的方法实现，还可以交换两个选择子所映射到的指针。经过几次操作之后，类的方法表就会变成图样子：

![image.jpg](https://upload-images.jianshu.io/upload_images/1776587-09b0c6af0da82282.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在新的映射表中，多了一个名为newSelector的选择子，capitalizedString的实现也变了，而lowercaseString与uppercaseString的实现则互换了。上述修改均无须编写子类，只要修改了“方法表”的布局，就会反映到程序中所有的NSString实例之上。这下大家见识到此特性的强大之处了吧？

本条将会谈到如何互换两个方法实现。通过此操作，可为已有方法添加新功能。不过在讲解怎样添加新功能之前，我们先来看看怎样互换两个已经写好的方法实现。想交换方法实现，可用下列函数：

```
oid method_exchangeImplementations(Method m1, Method m2)
```

此函数的两个参数表示待交换的两个方法实现，而方法实现则可通过下列函数获得：

```
 Method class_getInstanceMethod(Class cls, SEL name)
```

此函数根据给定的选择从类中取出与之相关的方法。执行下列代码，即可交换前面提到的lowercaseString与uppercaseString方法实现：

```
Method originalMethod =  
    class_getInstanceMethod([NSStringclass],  
                            @selector(lowercaseString));  
Method swappedMethod =  
    class_getInstanceMethod([NSStringclass],  
                            @selector(uppercaseString));  
method_exchangeImplementations(originalMethod, swappedMethod); 
```

然而在实际应用中，像这样直接交换两个方法实现的，意义并不大。因为lowercaseString与uppercaseString这两个方法已经各自实现得很好了，没必要再交换了。但是，可以通过这一手段来为既有的方法实现增添新功能。比方说，想要在调用lowercaseString时记录某些信息，这时就可以通过交换方法实现来达成此目标。我们新编写一个方法，在此方法中实现所需的附加功能，并调用原有实现。

新方法可以添加至NSString的一个“分类”（category）中：

```
@interface NSString (EOCMyAdditions)  
- (NSString*)eoc_myLowercaseString;  
@end 
```

上述新方法将与原有的lowercaseString方法互换，交换之后的方法表如图:

![](https://upload-images.jianshu.io/upload_images/1776587-1f9f7eb4db0e53d3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

新方法的实现代码可以这样写：

```
@implementation NSString (EOCMyAdditions)  
- (NSString*)eoc_myLowercaseString {  
    NSString *lowercase = [self eoc_myLowercaseString];  
    NSLog(@"%@ => %@", self, lowercase);  
    return lowercase;  
}  
@end 
```

这段代码看上去好像会陷入递归调用的死循环，不过大家要记住，此方法是准备和lowercaseString方法互换的。所以，在运行期，eoc_myLowercaseString选择子实际上对应于原有的lowercaseString方法实现。最后，通过下列代码来交换这两个方法实现：

```
Method originalMethod =  
    class_getInstanceMethod([NSString class],  
                            @selector(lowercaseString));  
Method swappedMethod =  
    class_getInstanceMethod([NSString class],  
                            @selector(eoc_myLowercaseString));  
method_exchangeImplementations(originalMethod, swappedMethod); 
```

执行完上述代码之后，只要在NSString实例上调用lowercaseString方法，就会输出一行记录消息：

```
NSString *string = @"ThIs iS tHe StRiNg";  
NSString *lowercaseString = [string lowercaseString];  
// Output: ThIs iS tHe StRiNg => this is the string 
```

通过此方案，开发者可以为那些“完全不知道其具体实现的”（completely opaque，“完全不透明的”）黑盒方法增加日志记录功能，这非常有助于程序调试。然而，此做法只在调试程序时有用。很少有人在调试程序之外的场合用上述“方法调配技术”来永久改动某个类的功能。不能仅仅因为Objective-C语言里有这个特性就一定要用它。若是滥用，反而会令代码变得不易读懂且难于维护。

## 总结

在运行期，可以向类中新增或替换选择子所对应的方法实现。

使用另一份实现来替换原有的方法实现，这道工序叫做“Method Swizzling”，通常称为黑魔法，开发者常用此技术向原有实现中添加新功能。

一般来说，只有调试程序的时候才需要在运行期修改方法实现，这种做法不宜滥用。
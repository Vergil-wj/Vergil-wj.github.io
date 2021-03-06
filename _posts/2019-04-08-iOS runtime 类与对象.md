---
layout:     post
title:      iOS runtime 类与对象
subtitle:   runtime
date:       2019-04-08
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - runtime
    - Objective-C
---

Objective-C runtime 是一个运行时库，它提供支持 Objective-C 语言的动态属性，其优势在于程序正在运行时，我们可以把消息转发给我们想要的对象，或者随意交换一个方法的实现等；

Runtime 库主要做下面几件事：

- 封装：在这个库中，对象可以用 C 语言中的结构体表示，而方法可以用 C 函数来实现，另外再加上了一些额外的特性。这些结构体和函数被 runtime 函数封装后，我们就可以在程序运行时创建，检查，修改类、对象和它们的方法了。

- 找出方法的最终执行代码：当程序执行 [object doSomething] 时，会向消息接收者 (object) 发送一条消息 (doSomething)，runtime 会根据消息接收者是否能响应该消息而做出不同的反应，我们会在后面系列详细介绍。

要了解 runtime 的基本工作原理，，我们先来了解一下类与对象，这是面向对象的基础，我们看看在Runtime中，类和对象是如何实现的。

## Class

Objective-C 类是由 Class 类型表示，它是一个指向 objc_class 结构体的指针。它的定义如下：

```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
```

objc_class 结构体定义如下：


```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */

```

我们来看下其中每个字段的含义：

- isa: 在 OC 中，任何数据结构只要在恰当的位置具有一个指针 isa 指向一个 Class，都可以被认为是一个对象。从这里可以看到类自身也是一个对象，我们称为类对象，这个 isa 指针指向 metaClass(元类)，我们会在后面介绍它。

- super\_class: 指向该类的父类，如果该类已经是最顶层的根类(如 NSObject 或 NSProxy)，则 super_class 为 NULL。
- name: 类名
- version：类的版本信息
- info：类信息，供运行期使用的一些位标识。
- instance_size：该类实例变量大小
- ivars：该类的成员变量链表
- methodLists：方法定义的链表，实例方法从这里查找，而类方法从这个类的 meta-class 的方法列表中查找。
- cache：方法缓存。一个接收者对象接收到一个消息时，它会根据isa指针去查找能够响应这个消息的对象。在实际使用中，这个对象只有一部分方法是常用的，很多方法其实很少用或者根本用不上。这种情况下，如果每次消息来时，我们都是methodLists中遍历一遍，性能势必很差。这时，cache就派上用场了。在我们每次调用过一个方法后，这个方法就会被缓存到cache列表中，下次调用的时候runtime就会优先去cache中查找，如果cache没有，才去methodLists中查找方法。这样，对于那些经常用到的方法的调用，但提高了调用的效率。
- protocols：协议链表

## id

id 是指向 objc_object 的指针，定义如下：

```
/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

objc_object 结构体定义如下：


```
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

可以看到，这个结构体只有一个字体，即指向其类的 isa 指针。这样，当我们向一个 Objective-C 对象发送消息时，运行时库会根据实例对象的 isa 指针找到这个实例对象所属的类。Runtime 库会在类的方法列表及父类的方法列表中去寻找与消息对应的 selector 指向的方法，找到后即运行这个方法。

## Meta Class

meta-class 就是一个类对象的类。举个例子：

```
NSArray *array = [NSArray array];
```

其中 NSArray 是类对象，根据 objc\_class 定义有一个指针 isa 指向一个 Class，所以 NSArray 这个类本身又是一个对象，我们且称呼为类对象。那么
 +array 消息发送给了 NSArray，为了调用 +array 方法，这个类的 isa 指针必须指向一个包含这些类方法的一个 objc_class 结构体，这个结构体就是 meta-class。
 
当我们向一个对象发送消息时，runtime 会在这个对象所属的这个类的方法列表中查找方法；而向一个类发送消息时，会在这个类的 meta-class 的方法列表中查找。

那么问题来了，meta-class 也是一个类，它的 isa 又指向什么呢？

> 答：NSObject 类元对象。

那么 NSObject 类元对象的 isa 又指向什么呢？

> 答：NSObject类元对象自身。

即，任何 NSObject 继承体系下的 meta-class 都使用 NSObject 的 meta-class 作为自己的所属类，而基类的 meta-class 的 isa 指针是指向它自己。这样就形成了一个完美的闭环。如下图所示：

![](https://ws3.sinaimg.cn/large/006tNc79ly1g1x9mdre69j31go0sgjuq.jpg)

写个例子来验证一下：

```
Class class = [CustomObject class];//类对象

Class metaClass = object_getClass(class);//类元对象
Class metaOfMetaClass = object_getClass(metaClass);//NSObject类元对象
Class rootMataClass = object_getClass(metaOfMetaClass);//NSObject类元对象的类元对象

NSLog(@"CustomObject类对象是:%p",class);
NSLog(@"CustomObject类元对象是:%p",metaClass);
NSLog(@"metaClass类元对象:%p",metaOfMetaClass);
NSLog(@"metaOfMetaClass的类元对象的是:%p",rootMataClass);
NSLog(@"NSObject类元对象:%p",object_getClass([NSObject class]));
```

控制台输出：

```
CustomObject类对象是:0x10248aed0
CustomObject类元对象是:0x10248aea8
metaClass类元对象:0x102ce5198
metaOfMetaClass的类元对象的是:0x102ce5198
NSObject类元对象:0x102ce5198
```

## SuperClass

在面相对象中，我们知道，子类调用一个方法，如果子类没有实现，会查找基类。OC 作为一种面相对象的语言，当然支持这些。 
我们依然写一段示例代码:

```
Class class = [CustomObject class];//类对象
Class superClass = class_getSuperclass(class);//基类对象NSObject
Class superOfNSObject = class_getSuperclass(superClass);//NSObject类元对象

NSLog(@"CustomObject类对象是:%p,%@",class,class);
NSLog(@"CustomObject类superClass是:%p,%@",superClass,superClass);
NSLog(@"NSObject的superClass是:%p,%@",superOfNSObject, superOfNSObject);
```

控制台输出：

```
CustomObject类对象是:0x101792e88, CustomObject
CustomObject类superClass是:0x101fec170,NSObject
NSObject的superClass是:0x0,(null)
```

由此，我们可以绘制继承关系图:

![](https://ws1.sinaimg.cn/large/006tNc79ly1g1xb7j8extj30bg0iq3zb.jpg)


综合上述两个例子，我们绘制出完整的 isa 和 superClass 图：

![](https://ws2.sinaimg.cn/large/006tNc79ly1g1xb8tt8qlj30jg0jdabh.jpg)

## 类与对象操作函数

runtime 提供了大量的函数来操作类与对象。类的操作方法大部分是以 class_ 为前缀的，而对象的操作方法大部分是以 objc_ 或o bject_ 为前缀。

我们可以回过头去看看 objc_class 的定义，runtime提供的操作类的方法主要就是针对这个结构体中的各个字段的。

相关 API 使用请看这里：[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)

## 小结

本章主要介绍了 Runtime 中类和对象的相关结构，对 Object-C 底层面向对象实现有更深一步的理解。

###### 参考资料：

[runtime 源码 ](https://github.com/opensource-apple)

[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)

[Objective-C Runtime Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)

[南峰子技术博客 http://southpeak.github.io/categories/objectivec/](http://southpeak.github.io/categories/objectivec/)

[https://blog.csdn.net/Hello_Hwc/article/details/49687543](https://blog.csdn.net/Hello_Hwc/article/details/49687543)
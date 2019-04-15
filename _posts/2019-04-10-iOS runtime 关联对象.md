---
layout:     post
title:      iOS runtime 关联对象
subtitle:   Associated Object
date:       2019-04-10
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - runtime
    - Objective-C
---

## 关联对象 Associated Object

顾名思义，就是把一个对象关联到另外一个对象身上。

关于关联对象 runtime 提供了3个 API:

```
//设置关联对象
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)

//获取关联的对象
id objc_getAssociatedObject(id object, const void *key)

//移除关联的对象
void objc_removeAssociatedObjects(id object)

```

其中参数：

- id object: 被关联的对象。
- const void *key: 关联的key，要求唯一。
- id value: 关联的对象。
- objc_AssociationPolicy policy: 内存管理策略，是一个枚举类型，如下所示：

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,
    OBJC_ASSOCIATION_RETAIN = 01401,
    OBJC_ASSOCIATION_COPY = 01403
};
```

## 应用场景

常用于为类别（category）添加属性。

有时候我们想要为对象添加一些额外的信息，我们一般做法是在这个对象所属的类中继承一个子类，在子类中增加额外的属性，然后使用这个子类。那另一种做法就是使用 Associated Object，减少对代码的入侵。

在类别（category）中，我们一般通过它来扩展方法。但要扩展属性的话，我们可以使用 Associated Object。

下面举一个简单例子：

对 UILable 增加了一个类别，在 UILable+vjLb.h 中：


```
@interface UILabel (vjLb)

@property(nonatomic,copy)NSString *name;

@end
```

UILabel+vjLb.m 中：

```
@implementation UILabel (vjLb)

-(NSString *)name{
   return objc_getAssociatedObject(self, _cmd);
}

-(void)setName:(NSString *)name{
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

@end
```

这样就为 UILabel 动态增加了一个新的 name 属性。

那问题来了，第二个参数明明是`const void *key`类型，而我们写的是`_cmd`与`@selector(name)`，他们返回的是`SEL` 类型，为什么还能使用呢？且看我下面慢慢解释。

### 上面例子中 _cmd 与 @selector(xxx) 解释：

#### 1、_cmd

官方的解释很简单，就下面一句：[原文链接](https://developer.apple.com/documentation/objectivec/nsobject/1418637-doesnotrecognizeselector?language=objc)

> The _cmd variable is a hidden argument passed to every method that is the current selector;

其实说的就是 _cmd 在 Objective-C 方法中表示当前方法的 selector，正如同 self 表示当前方法调用的对象实例一样。

比如，我们要打印当前要调用的方法，可以这样来写：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSLog(@"current method = %@",NSStringFromSelector(_cmd));
}

```

输出结果如下：

```
current method = viewDidLoad 
```

所以在上面的例子中，name 的 getter 方法在下面两种写法是一样的：

```
-(NSString *)name{
   return objc_getAssociatedObject(self, _cmd);
}

-(NSString *)name{
   return objc_getAssociatedObject(self, @selector(name));
}
```

#### 2、@selector(xxx)

selector 可以叫做选择器，其实指的就是对象的方法，也可以理解为C语言里面的函数指针。

@selector(xxx) 的作用是找到名字为xxx的方法。一般用于 `[a performSelector:@selector(b)]` 就是说去调用 a 对象的 b 方法，和 `[a b]` 的意思一样，但是这样更加动态一些。

@selector(xxx) 返回的类型是 SEL，我们看一下 SEL 的定义：

```
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```

runtime 没有提供 objc\_selector 的定义，具体这 objc_selector 结构体是什么取决与使用 GNU 的还是 Apple 的运行时， 在 Mac OS X 中 SEL 其实被映射为一个 C 字符串，可以看作是方法的名字，它并不一个指向具体方法实现（IMP 类型才是）。对于所有的类，只要方法名是相同的，产生的selector都是一样的。

既然 SEL 被映射为一个 C 字符串，那么在设置关联对象的这个 API 中第二个参数就可以解释了：

```
//设置关联对象
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```

其中第二个参数`const void *key`就是一个 是一个无类型的常量，所以使用 `@selector(xxx)`( *被映射为 C 字符串常量*)写法是没问题的。上面的例子就有以下两种写法了：

第一种，就是上面的例子：

```
@implementation UILabel (vjLb)

-(NSString *)name{
   return objc_getAssociatedObject(self, _cmd);
   
   //或者使用下面这一句： 
   //return objc_getAssociatedObject(self, @selector(name));
}

-(void)setName:(NSString *)name{
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

@end

```

第二种：

```
const void *keyName = "keyName";

@implementation UILabel (vjLb)

-(NSString *)name{
   return objc_getAssociatedObject(self, keyName);
}

-(void)setName:(NSString *)name{
    objc_setAssociatedObject(self, keyName, name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}
```

## 小结

本片主要讲了 runtime 中关联对象 Associated Object 的用法，及其中的小知识点`_cmd`与`@selector()`。

关联对象最常用的场景就是就是为分类添加“属性”，其它的还有为 UI 控件关联事件 Block 体，为了不重复获得某种数据等。具体的实现可以参考下这篇文章：[https://www.jianshu.com/p/916aef6f7ab1](https://www.jianshu.com/p/916aef6f7ab1)

###### 参考资料：

[runtime 源码 ](https://github.com/opensource-apple)

[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)

[http://blog.sina.com.cn/s/blog_735065f90100yopd.html](http://blog.sina.com.cn/s/blog_735065f90100yopd.html)

[https://www.jianshu.com/p/916aef6f7ab1](https://www.jianshu.com/p/916aef6f7ab1)
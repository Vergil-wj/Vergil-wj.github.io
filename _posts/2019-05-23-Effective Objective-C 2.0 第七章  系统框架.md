---
layout:     post
title:      Effective Objective-C 2.0 第七章 系统框架
subtitle:   系统框架
date:       2019-05-23
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 第 47 条 熟悉系统框架

我之前这篇文章[iOS 框架](https://www.jianshu.com/p/f1942262542d)有介绍过，下面简单介绍下书中内容。

>将一系列代码封装为动态库，并在其中放入描述其接口的头文件，这样做出来的东西就叫框架。

在 iOS 上的框架为 Cocaa Touch 框架，其实其本身并不是框架，但是里面集成了一批创建应用程序时经常会用到的框架。

我们开发者碰到的主要框架就是 [Foundation](https://www.jianshu.com/p/f1942262542d)和[UIKit](https://www.jianshu.com/p/f1942262542d)两大框架。其中 Foundation 框架是 Objective-C 应用程序的“基础”。

## 第 48 条 多用块枚举，少用 for 循环

遍历 collection 有四种方式。最基本的方法是for循环，其次是 NSEnumerator 遍历法及快速遍历法，最新最先进的的方式则是“block枚举法”。

### 1、for 循环

```
//遍历数组
NSArray *anArray = /*...*/;
for (int i = 0; i < anArray.count; i++)
{
    id object = anArray[i];
    //Do something with 'object'
}

//遍历字典
NSDictionary *aDictionary = /*...*/;
NSArray *keys = [aDictionary allKeys];
for (int i = 0; i < keys.count; i++)
{
    id key = keys[i];
    id value = aDictionary[key];
    //Do something with 'key' and 'value'
}

//遍历 set
NSSet *aSet = /*...*/;
NSArray *objects = [aSet allObjects];
for (int i = 0; i < objects.count; i++)
{
    id object = objects[i];
    //Do something with 'object'
}

```

在遍历字典或者 Set 时，有点稍复杂，需要先获取字典中所有的键或 Set 中所有的对象，会增加额外的开销。

### 2、NSEnumerator

```
//数组
NSArray *anArray = /* ... */;
NSEnumerator *enumerator = [anArray objectEnumerator];
id object;
while ((object = [enumerator nextObject]) != nil)
{
    // Do something with 'object'
}

//字典
NSDictionary *aDictionary = /* ... */;
NSEnumerator *enumerator = [aDictionary keyEnumerator];
id key;
while ((key = [enumerator nextObject]) != nil)
{
    id value = aDictionary[key];
    // Do something with 'key' and 'value'
}

//Set
// Set
NSSet *aSet = /* ... */;
NSEnumerator *enumerator = [aSet objectEnumerator];
id object;
while ((object = [enumerator nextObject]) != nil)
{
    // Do something with 'object'
}

//反向遍历数组
NSArray *anArray = /* ... */;
NSEnumerator *enumerator = [anArray reverseObjectEnumerator];
id object;
while ((object = [enumerator nextObject]) != nil)
{
    // Do something with 'object'
}
```

与 for 循环相似，但在遍历数组时无法拿到下标，而我们一般开发时都会需要用到下标。

### 3、快速遍历法

```
//数组
NSArray *anArray = /* ... */;
for (id object in anArray)
{
    // Do something with 'object'
}

//字典
NSDictionary *aDictionary = /* ... */;
for (id key in aDictionary)
{
    id value = aDictionary[key];
    // Do something with 'key' and 'value'
}

// Set
NSSet *aSet = /* ... */;
for (id object in aSet)
{
    // Do something with 'object'
}
```

这种办法是简单且效率高的，然而如果在遍历字典时需要同时获取键与值，那么会多出来一步。而且，与传统for循环不同，这种遍历方式无法轻松获取当前遍历操作所针对的下标。遍历时通常会用到这个下标，比如很多算法都需要它。

### 4、基于 block 遍历方式

```
//数组
NSArray *anArray = /* ... */;
[anArray enumerateObjectsUsingBlock:^(id object, NSUInteger idx, BOOL *stop)
{
    // Do something with 'object'
    if (shouldStop)
    {
        *stop = YES;
    }
}];

//字典
NSDictionary *aDictionary = /* ... */;
[aDictionary enumerateKeysAndObjectsUsingBlock:^(id key, id object, BOOL *stop)
{
    // Do something with 'key' and 'object'
    if (shouldStop)
    {
        *stop = YES;
     }
}];

//Set
NSSet *aSet = /* ... */;
[aSet enumerateObjectsUsingBlock:^(id object, BOOL *stop)
{
    // Do something with 'object'
    if (shouldStop)
    {
    	*stop = YES;
    }
}];
```

遍历时可以直接从block里获取更多信息。在遍历数组时，可以知道当前所针对的下标。遍历有序set（NSOrderedSet）时也一样。而在遍历字典时，无须额外编码，即可同时获取键与值，因而省去了根据给定键来获取对应值这一步。用这种方式遍历字典，可以同事得知键与值，这很可能比其他方式快很多，因为在字典内部的数据结构中，键与值本来就是存储在一起的。

## 第 49 条 对自定义其内存管理语义的 collection 使用无缝桥接

通过无缝桥接技术，可以在Foundation框架中的OC对象与Core Foundation框架中的C语言数据之间来回转换。

```
NSArray *arr = @[@"1",@"2",@"3"];
CFArrayRef  aCFArray = (__bridge CFArrayRef)arr;
```

进行转换操作的修饰符共有3个：

```
OC对象 -> CF
__bridge // ARC不交出对象的所有权
__bridge_retained // ARC交出对象的所有权，手动管理内存，需要调用 CFRelease() 进行释放。
 
CF -> OC对象
__bridge_transfer // ARC获得对象的所有权，自动管理内存
```

## 第 50 条 构建缓存时选用 NSCache 而非 NSDictionary

NSCache胜过NSDictionary的之处在于：

- 当系统资源将要耗尽时，它可以自动删减缓存。
- NSCache还会先行删减“最久未使用的”(lease recently used)对象。
- NSCache 并不会“拷贝”键，而是会“保留”它。NSCache对象不拷贝键的原因在于：很多时候，键都是有不支持拷贝操作的对象来充当的。因此，NSCache 不会自动拷贝键，所以说，在健不支持拷贝操作的情况下，该类用起来比字典更方便。
- NSCache是线程安全的。而NSDictionary则绝不具备此优势，意思就是：在开发者自己不编写加锁代码的前提下，多个线程便可以同时访问NSCache.

## 第 51 条 精简 initialize 与 load 的实现代码

### +(void)load

1、对于加入运行期系统的类及分类，必定会调用此方法，且仅调用一次。

2、iOS会在应用程序启动的时候调用load方法，在main函数之前调用。

3、执行子类的load方法前，会先执行所有超类的load方法，顺序为父类->子类->分类，则先调用类里面的，再调用分类里面的。

4、在load方法中使用其他类是不安全的，因为会调用其他类的load方法，而如果关系复杂的话，就无法判断出各个类的载入顺序，类只有初始化完成后，类实例才能进行正常使用。

5、load 方法不遵从继承规则，如果类本身没有实现load方法，那么系统就不会调用，不管父类有没有实现（跟下文的initialize有明显区别）。

6、尽可能的精简load方法，因为整个应用程序在执行load方法时会阻塞，即，程序会阻塞直到所有类的load方法执行完毕，才会继续。

7.、oad 方法中最常用的就是方法交换method swizzling。

### +(void)initialize

1、对于每个类来说,在首次使用该类之前，且仅调用一次.它是由运行期系统来调用的，绝不应该通过代码直接调用。

2、惰性调用，只有当程序使用相关类(该类或子类)时，才会调用,因此，如果某个类一直都没有使用，那么initialize方法就一直不会运行。

3、运行期系统会确保initialize方法是在线程安全的环境中执行，即，只有执行initialize的那个线程可以操作类或类实例。其他线程都要先阻塞，等待initialize执行完。

4、如果类未实现initialize方法，而其超类实现了，那么会运行超类的实现代码，而且会运行两次。

5、initialize方法也需要尽量精简，只应该用来设置内部数据：
比如，某个全局状态无法在编译期初始化，可以放在initialize里面。

6、对于分类中的initialize方法，会覆盖该类的initialize方法。

## 第 52 条 别忘了 NSTime 会保留其目标对象

NSTimer 对象会保留其目标，若要释放其目标需要计时器本身失效，失效有两种方式：

 - 我们手动调用 invalidate 方法可伶计时器失效；
 - 一次性的计时器在触发完任务之后也会失效。

使用计时器反复执行任务时，通常会造成循环引用。如果我们对代码逻辑非常清楚，则手动调用 invalidate 方法就可以了，但还有更好的方法使我们我无须关注目标对象释放问题。

我们通常的写法：

```
@interface EOCClass : NSObject
- (void)startPolling;
- (void)stopPolling;
@end
 
@implementation EOCClass{
    NSTimer *_poliTimer;
}
 
- (id) init{
    return [super init];
}
 
- (void)dealloc{
    [_poliTimer invalidate];
}
 
- (void)stopPolling{
    [_poliTimer invalidate];
    _poliTimer = nil;
}
 
- (void)startPolling{
    _poliTimer = [NSTimer scheduledTimerWithTimeInterval:5.0 target:self selector:@selector(p_doPoll) userInfo:nil repeats:YES];
}
 
- (void)p_doPoll{
    // code

}
```

调用stopPolling方法或令系统将实例回收（会自动调用dealloc方法）可以使计时器失效，从而打破循环，但无法确保startPolling方法一定调用，而由于计时器保存着实例，实例永远不会被系统回收。当EOCClass实例的最后一个外部引用移走之后，实例仍然存活，而计时器对象也就不可能被系统回收，除了计时器外没有别的引用再指向这个实例，实例就永远丢失了，造成内存泄漏。

优化后的方法：

```
@interface NSTimer (EOCBlocksSupport)
+ (NSTimer*)eoc_scheduledTimerWithTimeInterval:(NSTimeInterval)interval block:(void(^)())block repeats:(BOOL)repeats;
@end
 
@implementation NSTimer( EOCBlocksSupport)
 
+ (NSTimer*)eoc_scheduledTimerWithTimeInterval:(NSTimeInterval)interval block:(void (^)())block repeats:(BOOL)repeats{
    return [self scheduledTimerWithTimeInterval:interval target:self selector:@selector(eoc_blockInvoke:) userInfo:[block copy] repeats:repeats];
}
 
+ (void)eoc_blockInvoke:(NSTimer*)timer{
    void (^block)() = timer.userInfo;
    if (block) {
        block();
    }
}
```

再修改 startPolling 方法：

```
- (void)startPolling{
    __weak EOCClass *weakSelf = self;
    _poliTimer = [NSTimer eoc_scheduledTimerWithTimeInterval:5.0 block:^{
        EOCClass *strongSelf = weakSelf;
        [strongSelf p_doPoll];
    } repeats:YES];
}
```

这段代码先定义了一个弱引用指向self，然后用块捕获这个引用，这样self就不会被计时器所保留，当块开始执行时，立刻生成strong引用，保证实例在执行器继续存活。

采用这种写法之后，如果外界指向 EOCClass 实例的最后一个引用将其释放，则该实例就可为系统所回收了。回收过程中还会调用计时器的 invalidate 方法，这样的话，计时器就不会再执行任务了。此处使用 weak 引用还能令程序更加安全，因为有时开发者可能在编写 dealloc 时忘了调用计时器的 invalidate 方法，从而导致计时器再次运行，若发生此类情况，则块里的 weakSelf 会变成 nil。

### 要点：

- NSTimer 对象会保留其目标，直到计时器本身失效为止，调用 invalidate 方法可令计时器失效，另外，一次性的计时器在触发完任务之后也会失效。

- 反复执行任务的计时器（repeating timer），很容易引入保留环，如果这种计时器的目标对象又保留了计时器本身，那肯定会导致保留环。这种环状保留关系，可能是直接发生的，也可能是通过对象图里的其他对象间接发生的。

- 可以扩充 NSTimer 的功能，用 “块”来打破保留环。不过，除非 NSTimer 将来在公共接口里提供此功能，否则必须创建分类，将相关实现代码加入其中。

---
layout:     post
title:      Effective Objective-C 2.0 第六章 块与大中枢派发(二)
subtitle:   大中枢派发
date:       2019-05-22
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 第 41 条 多用派发队列，少用同步锁

多个线程执行同一份代码的情况下，我们需要加锁，有以下三种方法：

### 1、使用同步块

```
- (void)synchronizedMethod
{
    @synchronized (self) {
    //Safe
   }
}
```

滥用@synchronized (self)则会降低代码效率，因为共用同一个锁的那些同步块，都必须按顺序执行。若是在self对象上频繁加锁，那么程序可能要等另一段与此无关的代码执行完毕，才能继续执行当前代码，这样做其实并没有必要。

在极端情况下，同步块会导致死锁，另外，其效率也不见得很高，而如果直接使用锁对象的话，一旦遇到死锁，就会非常麻烦。

### 2、使用 NSLock 对象

```
_lock = [[NSLock alloc]init];

- (void)synchronizedMethod{
	[_lock lock];
 	//Safe
	[_lock unlock];
}
```

这么做虽然能提供某种程度的“线程安全”（thread safety），但却无法保证访问该对象时绝对是线程安全的，当然访问属性的操作确实是“原子的”。在两次访问操作之间，其他线程可能会写入新的属性值，造成了在同一个线程上多次调用获取方法（getter），每次获取到的结果未必相同。

### 3、使用 GCD

```
_syncqueue = dispatch_queue_create("com.test.concurrent", DISPATCH_QUEUE_CONCURRENT);

- (NSString *)something{
    __block NSString *localSomething = nil;
    dispatch_sync(_syncqueue, ^{
        localSomething = _something;
    });
    return localSomething;
}

- (void)setSomething:(NSString *)something{
    dispatch_barrier_async(_syncqueue, ^{
        _something = something;
    });
}
```

看图理解下：

![](https://upload-images.jianshu.io/upload_images/1776587-72d708b61d18dcea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个并发队列中，读取操作使用普通的块来实现，而写入操作则是用栅栏块来实现的。读取操作可以并行，但写入操作必须单独执行，因为他是栅栏块。

### 要点

- 派发队列可用来表述同步语义（synchronization semantic），这种做法要比使用@synchronized块或NSLock对象更简单。
- 将同步与异步派发结合起来，可以实现与普通加锁机制一样的同步行为，而这么做却不会阻塞执行异步派发的线程。
- 使用同步队列及栅栏块，可令同步行为更加高效。


## 第 42 条 多用 GCD，少用 performSelector 系列方法

NSObject定义了几个方法，令开发者可以随意调用任何方法。这几个方法可以推迟执行方法调用，也可以指定运行方法所用的线程。这些功能原来很有用，但是在出现了大中枢派发及块这样的新技术之后，就显得不那么必要了。虽说有些代码还是会经常用到它们，但笔者劝你还是避开为妙。

这其中最简单的是`performSelector：`。该方法与直接调用选择子等效。所以下面两行代码的执行效果相同：

```
[self performSelector:@selector(selectorName)];
[self selectorName];
```
performSelector还有如下几个版本，可以再发消息时顺便传递参数:

```
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
```

可以看出`performSelector`方法太过局限，最多支持两个函数。

```
SEL selector;
if (/* some condition */) {
    selector = @selector(foo);
}else if (/* some other condition */){
    selector = @selector(bar);
}else{
    selector = @selector(baz);
}
[object performSelector:selector];
```

这样写编译器汇报内存泄漏警告。原因在于，编译器并不知道将要调用的选择子是什么，因此也就不了解其方法签名及返回值，甚至连是否有返回值都不清楚。而且，由于编译器不知道方法名，所以就没办法运用ARC的内存管理规则来判定返回值是不是应该释放，鉴于此，ARC采用了比较谨慎的做法，就是不添加释放操作。然而这么做可能导致内存泄漏，因为方法在返回对象时 可能已经将其保留了。

### 替代方案

最主要的替代方案就是使用块（参见第37条）。而且，performSelector系列方法所提供的线程功能，都可以通过在大中枢派发机制中使用块来实现。延后执行可以用dispatch_after来实现，在另一个线程上执行任务则可通过dispatch_sync及dispatch_async来实现。

例如延后执行：

```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5.0 * NSEC_PER_SEC));

dispatch_after(time, dispatch_get_main_queue(), ^{
    [self doSomething];
});
```

想把任务放在主线程上执行：

```
dispatch_async(dispatch_get_main_queue(), ^{
    [self doSomething];
});
```

### 要点

- `performSelector`系列方法在内存管理方面容易有疏失。它无法确定将要执行的选择子具体是什么，因而ARC编译器也就无法插入适当的内存管理方法。
- `performSelector`系列方法所能处理的选择子太过局限了，选择子的返回值类型及发送给方法的参数个数都受到限制。
- 如果想把任务放在另一个线程上执行，那么最好不要用`performSelector`系列方法，而是应该把任务封装到块里，然后调用大中枢派发机制的相关方法来实现。


## 第 43 条 掌握 GCD 及操作队列的使用时机

GCD技术确实很棒，不过有时候采用标准系统库的组件，效果会更好。一定要了解每项技巧的使用时机。

很少有其它技术能与GCD的同步机制（参见第41条）相媲美。对于那些只需执行一次的代码来说，也是如此，使用GCD的dispatch_once（参见第45条）最为方便。然而，在执行后台任务时，GCD并不一定是最佳方式。还有一种技术叫做NSOperationQueue，它虽然与GCD不同，但却与之类似，开发者可以把操作以NSOperation子类的形式放在队列中，而这些操作也能够并发执行。其与GCD派发队列有相似之处，这并非巧合。“操作队列”（operation queue）在GCD之前就有了，其中有些设计原理因操作队列而流行，GCD就是基于这些原理构建的。实际上，从iOS4与MAC OSX 10.6开始，操作队列在底层是用GCD来实现的。

在两者的诸多差别中，首先要注意：GCD是纯C的API，而操作队列则是Objective-C的对象。在GCD中，任务用块来表示，而块是个轻量级数据结构（参见第37条）。与之相反，“操作”（operation）则是个更为重量级的Objective—C对象。

用NSOperationQueue类的“addOperationWithBlock:”方法搭配NSBlockOperation类来使用操作队列其语法与纯GCD方式非常类似。是用NSOperation及NSOperationQueue的好处如下：

- 取消某个操作。如果使用操作队列，那么想要取消操作队列是很容易的。运行任务之前，可以在NSOperation对象上调用cancel方法，该方法会设置对象内的标志位，用以表明此任务不需执行，不过，已经启动的任务无法取消。若是不使用操作队列，而是把块安排到GCD队列，那就无法取消了。开发者可以在应用程序层自己来实现取消功能，不过这样做需要编写很多代码，而那些代码其实已经由操作队列实现好了。
- 指定操作间的依赖关系。一个操作可以依赖其他多个操作。开发者能够制定操作之间的依赖体系，使特定的操作必须在另外一个操作顺利执行完毕后方可执行。
- 通过键值观测机制监控NSOperation对象的属性。NSOperation对象有许多属性都适合通过键值观测机制（简称KVO）来监听。比如可以通过isCancelled属性来判断任务是否已取消，又比如可以通过isFinished属性来判断任务是否已完成。
- 指定操作的优先级。操作的优先级表示此操作与队列中其他操作之间的优先级关系。优先级高的操作先执行，优先级低的后执行。操作队列的调度算法（scheduling algorithm）虽“不透明”（opaque），但必然是经过一番深思熟虑才写成的。反之，GCD则没有直接实现此功能的办法。GCD的队列确实有优先级，不过那是针对整个队列来说的，而不是针对每个块来说的。NSOperation对象也有“线程优先级”（thread priority），这决定了运行此操作的线程处在何种优先级上。用GCD也可实现此功能，然而采用操作队列更简单，只需设置一个属性。
- 重用NSOperation对象。系统内置了一些NSOperation的子类（比如NSBlockOperation）以供开发者调用，要是不想用这些固有子类的话，那就得自己来创建了。这些类就是普通的Objective-C对象，能够存放任何信息。对象在执行时可以充分利用存于其中的信息，而且还可以随意调用定义在类中的方法。这就比派发队列中那些简单的块要强大许多。这些NSOperation类可以在代码中多次使用，他们符合软件开发中的“不重复”（Do’t Repeat Yourself，DRY）原则。

有一个API选用了操作队列而非派发队列，这就是NSNotificationCenter，这个方法接受的参数是块，而不是选择子：

```
- (id <NSObject>)addObserverForName:(nullable NSString *)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block NS_AVAILABLE(10_6, 4_0);
```

本来这个方法也可以不使用操作队列，而是把处理通知事件所用的块安排在派发队列里。但实际上并没有这样做，其设计者显然使用了高层的Objective-C API。在这种情况下，两套方案的运行效率没多大差距。设计这个方法的人可能不想使用派发队列，因为那样做将依赖于GCD，而这种依赖没有必要，前面说过，块本身和GCD无关。

经常会有人说：应该尽可能选用高层API，只在确有必要时才求助与底层。笔者也同意这个说法，但我并不盲从。某些功能确实可以用高层的Objective-C方法来做，但这并不等于说它就一定比底层实现方案好。要想确定哪种方案更佳，最好还是测试一下性能。

## 第 44 条 通过 Dispatch Group 机制，根据系统资源状况来执行任务

dispatch group是GCD的一项特性，能够把任务分组。调用者可以等待这组任务执行完毕，也可以在提供回调函数之后继续往下执行，这组任务完成时，调用者会得到通知。这个功能有许多用途，其中最重要、最值得注意的用法，就是把将要并发执行的多个任务合为一组，于是调用者就可以知道这些任务何时才能全部执行完毕。

### 创建 group:

```
dispatch_group_t group = dispatch_group_create();
```

### 给任务编组

两种方法：

```
//第一种方法：
void dispatch_group_async(dispatch_group_t group,
                          dispatch_queue_t queue,
                          dispatch_block_t block);

//第二种方法
dispatch_group_enter(dispatch_group_t group)
dispatch_group_leave(dispatch_group_t group)
```

第二种方法中，调用了dispatch_group_enter以后，必须又与之对应的dispatch_group_leave才行。这与引用计数（参见第29条）相似，要使用引用计数。就必须令保留操作与释放操作彼此对应，以防内存泄漏。

### 任务组执行完毕

```
void dispatch_group_wait(dispatch_group_t group, dispatch_time_t timeout)
```

此函数接受两个参数，一个是要等待的group，另一个是代表等待时间的timeout值。timeout参数表示函数在等待dispatch group执行完毕时，应该阻塞多久。如果执行dispatch group所需的时间小于timeout，则返回0，否则返回非0值。此参数也可以取常量DISPATCH_TIME_FOREVER，这表示函数会一直等待dispatch group执行完，而不会超时（time out）。

也可以使用：

```
void dispatch_group_notify(dispatch_group_t group,
                           dispatch_queue_t queue,
                           dispatch_block_t block);
```

与wait函数略有不同的是：开发者可以向此函数传入块，等dispatch group执行完毕之后，块会在特定的线程上执行。加入当前此案成不应阻塞，而开发者又想在那些任务全部完成时得到通知，那么此做法就很有必要了。

例如：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

dispatch_group_t dispatchGroup = dispatch_group_create();

NSArray * collection;
for (id object in collection) {
    dispatch_group_async(dispatchGroup, queue, ^{
        [object description];
    });
}

//第一种，有超时
dispatch_group_wait(dispatchGroup, DISPATCH_TIME_FOREVER);
//Continue processing after completing tasks

//或者使用第二种，没有超时
dispatch_group_notify(dispatchGroup, dispatch_get_main_queue(), ^{
    //Continue processing after completing tasks
});

```

### 要点

- 一系列任务可归入一个dispatch group之中。开发者可以在这组任务执行完毕时获得通知。
- 通过dispatch group，可以在并发式派发队列里同时执行多项任务。此时，GCD会根据系统资源状况来调度这些并发执行的任务。开发者若自己来实现此功能，则需编写大量代码。



## 第 45 条 使用 dispatch_once 来执行只需运行一次的线程安全代码

原来创建单例：

```
@implementation EOCClass

+ (instancetype)sharedInstance
{
    static EOCClass *sharedInstance = nil;
    @synchronized (self) {
        if (!sharedInstance) {
            sharedInstance = [[self alloc] init];
        }
    }
    return sharedInstance;
}
@end
```

笔者发现单例模式容易引起激烈争论，Objective-C的单例尤其如此。线程安全是大家争论的主要问题。为保证线程安全，上述代码将创建单例实例的代码包裹在同步块里。

使用 dispatch_once 创建单例：

```
+ (instancetype)sharedInstance
{
    static EOCClass *sharedInstance = nil;
    static dispatch_once_t onceToken;

    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });

    return sharedInstance;
}
```

使用dispatch_once可以简化代码并且彻底保证线程安全，开发者根本无须担心加锁或同步。所有问题都有GCD在底层处理。由于每次调用时都必须使用完全相同的标记，所以标记要声明成static。把该变量定义在static作用域中，可以保证编译器在每次执行shareInstance方法时都会复用这个变量，而不会创建新变量。

此外，dispatch_once更高效。他没有使用重量级的同步机制，若是那样的话，每次运行代码前都要获取锁，相反，此函数采用“原子访问”（atomic access）来查询标记，以判断其所对应的代码原来是否已经执行过。

### 要点

- 经常需要编写“只需执行一次的线程安全代码”（thread-safe single-code execution）。通过GCD所提供的dispatch_once函数，很容易就能实现此功能。
- 标记应该声明在static或global作用域中，这样的话，在把只需执行一次的块传给dispatch_once函数时，穿进去的标记也是相同的。

## 第 46 条 不要使用 dispatch\_get\_current\_queue

```
dispatch_get_current_queue(void);
```

此函数在 iOS 6.0 版本以后，已经废弃。




---
layout:     post
title:      Effective Objective-C 2.0 第六章 块与大中枢派发(一)
subtitle:   块
date:       2019-05-16
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 第 37 条 理解“块”这一概念

Block：带有自动变量的匿名函数。

匿名函数：没有函数名的函数，一对 {} 包裹的内容是匿名函数的作用域。

自动变量：栈上声明的一个变量不是静态变量和全局变量，是不可以在这个栈内声明的匿名函数中使用的，但在 Block 中却可以。

虽然使用 Block 不用声明类，但是 Block 提供了 Objective-C 的类一样可以通过成员变量来保存作用域外变量值的方法，那些在 Block 的一对 {} 里使用到但却是在 {} 作用域以外声明的变量，就是 Block 截获的自动变量。

### Block表达式语法：

>^ 返回值类型 (参数列表) {表达式}

例如：

```
^ int (int count){
    return count + 1;
};
```

其中可以省略返回类型：

```
^(int count){
	return count + 1;
};
```

若参数为空，则可以写为：

```
^{
	return count + 1;
};
```

即最简模式语法为：

>^ {表达式}

### Block类型变量

声明Block类型变量语法:

>返回值类型 (^变量名)(参数列表) = Block表达式

例如，如下声明了一个变量名为blk的Block：

```
int (^blk)(int) = ^(int count) {
    return count + 1;
};
```

当Block类型变量作为函数的参数时，写作：

```
- (void)func:(int (^)(int))blk {
    NSLog(@"Param:%@", blk);
}
```

借助 typedef 可简写:

```
typedef int (^blk_k)(int);

- (void)func:(blk_k)blk {
    NSLog(@"Param:%@", blk);
}
```

Block 类型变量作返回值时，写作

```
- (int (^)(int))funcR {
    return ^(int count) {
        return count ++;
    };
}
```

借助 typedef 简写：

```
typedef int (^blk_k)(int);

- (blk_k)funcR {
    return ^(int count) {
        return count ++;
    };
}
```

### Block与外界变量

#### 截获自动变量值

对于 block 外的变量引用，block 默认是将其复制到其数据结构中来实现访问的。也就是说 block 的自动变量截获只针对 block 内部使用的自动变量, 不使用则不截获, 因为截获的自动变量会存储于 block 的结构体内部, 会导致 block 体积变大。特别要注意的是默认情况下 block 只能访问不能修改局部变量的值。

![](https://upload-images.jianshu.io/upload_images/1776587-82740df0ecff6ff8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
 int i = 10;
 void (^blk)(void) = ^{
     NSLog(@"In block, i = %d", i);
 };
 i = 20;//Block外修改变量i，也不影响Block内的自动变量
 blk();//i修改为20后才执行，打印: In block, i = 10
 NSLog(@"i = %d", i);//打印：i = 20
```

#### __block 修饰的外部变量

对于用 __block 修饰的外部变量引用，block 是复制其引用地址来实现访问的。block 可以修改 __block 修饰的外部变量的值。

![](https://upload-images.jianshu.io/upload_images/1776587-257e7df0f58fe62a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
__block int i = 10;//i为__block变量，可在block中重新赋值
void (^blk)(void) = ^{
    NSLog(@"In block, i = %d", i);
};
i = 20;
blk();//打印: In block, i = 20
NSLog(@"i = %d", i);//打印：i = 20
```

### 自动变量值为一个对象情况

当自动变量为一个类的对象，且没有使用__block修饰时，虽然不可以在Block内对该变量进行重新赋值，但可以修改该对象的属性。

如果该对象是个Mutable的对象，例如NSMutableArray，则还可以在Block内对NSMutableArray进行元素的增删：

```
NSMutableArray *array = [[NSMutableArray alloc] initWithObjects:@"1", @"2",nil ];
NSLog(@"Array Count:%ld", array.count);//打印Array Count:2
void (^blk)(void) = ^{
    [array removeObjectAtIndex:0];//Ok
    //array = [NSNSMutableArray new];//没有__block修饰，编译失败！
};
blk();
NSLog(@"Array Count:%ld", array.count);//打印Array Count:1
```

###  Block 存储域

block 有三种类型：

- 全局块(_NSConcreteGlobalBlock)
- 栈块(_NSConcreteStackBlock)
- 堆块(_NSConcreteMallocBlock)

全局块存在于全局内存中, 相当于单例；
栈块存在于栈内存中, 超出其作用域则马上被销毁；
堆块存在于堆内存中, 是一个带引用计数的对象, 需要自行管理其内存；

遇到一个Block，我们怎么这个Block的存储位置呢？

(1) Block 不访问外界变量（包括栈中和堆中的变量）

Block 既不在栈又不在堆中，在代码段中，ARC和MRC下都是如此。此时为全局块。

(2) Block 访问外界变量

MRC 环境下：访问外界变量的 Block 默认存储栈中。
ARC 环境下：访问外界变量的 Block 默认存储在堆中（实际是放在栈区，然后ARC情况下自动又拷贝到堆区），自动释放。

## 第 38 条 为常用的块类型创建 typedef

上文已经提到过，见下面代码：

```
typedef int (^blk_k)(int);

- (void)func:(blk_k)blk {
    NSLog(@"Param:%@", blk);
}
```

## 第 39 条 用 handler 块降低代码分散程度

异步方法在执行完任务之后，需要以某种手段通知相关代码。实现此功能有很多办法。常用的技巧是设计一个委托协议（参见第23条），令关注此事的对象遵从该协议。对象成为delegate之后，就可以在相关事件发生时（例如某个异步任务执行完毕时）得到通知了。
如果改用块来写的话，代码会更清晰。块可以令这种API变得更紧致，同时也令开发者调用起来更加方便。

```
typedef void (^EOCNetworkFetcherCompletionHandler)(NSData *data);

@interface EOCNetworkFetcher : NSObject
- (id)initWithURL:(NSURL *)url;
- (void)startWithCompletionHandler: (EOCNetworkFetcherCompletionHandler)handler;
@end
```

```
NSURL *url = [[NSURL alloc] initWithString:@"http:www.baidu.com"];
EOCNetworkFetcher *fetcher = [[EOCNetworkFetcher alloc] initWithURL:url];
[fetcher startWithCompletionHandler:^(NSData *data) {
    _fetchedFooData = data;
}];
```

## 第 40 条 用块引用其所属对象时不要出现保留环

Block 循环引用的情况：
某个类将 block 作为自己的属性变量，然后该类在 block 的方法体里面又使用了该类本身，如下：

```
self.someBlock = ^(Type var){
    [self dosomething];
};
```
解决方法：
对Block内要使用的对象 A 使用**_weak**进行修饰，Block 对对象 A 弱引用打破循环。

1、使用__weak ClassName

```
__weak XXViewController* weakSelf = self;
self.blk = ^{
    NSLog(@"In Block : %@",weakSelf);
};
```

2、 使用__weak typeof(self)

```
__weak typeof(self) weakSelf = self;
self.blk = ^{
    NSLog(@"In Block : %@",weakSelf);
};
```

3、 对 Block 内要使用的对象 A 使用 \_\_block 进行修饰，并在代码块内，使用完 __block 变量后将其设为 nil，并且该 block 必须至少执行一次。

```
__block XXController *blkSelf = self;
self.blk = ^{
    NSLog(@"In Block : %@",blkSelf);
    blkSelf = nil;//不能省略
};
    
self.blk();//该block必须执行一次，否则还是内存泄露
```

4、 将在 Block 内要使用到的对象（一般为 self 对象），以 Block 参数的形式传入，Bloc k就不会捕获该对象，而将其作为参数使用，其生命周期系统的栈自动管理，不造成内存泄露。

```
self.blk = ^(UIViewController *vc) {
    NSLog(@"Use Property:%@", vc.name);
};
self.blk(self);
```
我们常用的还是前两条，使用 __weak 修饰。


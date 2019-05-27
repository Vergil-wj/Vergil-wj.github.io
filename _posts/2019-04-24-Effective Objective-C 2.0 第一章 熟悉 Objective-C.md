---
layout:     post
title:      Effective Objective-C 2.0 第一章 熟悉 Objective-C
subtitle:   熟悉 Object-C
date:       2019-04-24
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 一、了解 Objective-C 语言的起源

Objective-C 与 C++、Java 等类似，是一种面向对象的语言。该语言使用“消息结构”而非“函数调用”。Objective-C 语言由 Smalltalk 演化而来，后者是消息型语言的鼻祖。

Objctive-C 为 C 语言添加了面向对象的特性，是其超集。Objective-C 使用动态绑定的消息结构，也就是说，在运行时才会检查对象类型。接受一条消息之后，究竟应该执行何种代码，由运行期环境而非编译器来决定。

## 二、在类的头文件中尽量少引用其他头文件

在类的头文件 .h 中,如果需要引用其它类，尽可能少用 `#import` 改而使用 `@class`,在 实现文件 .m 中若要使用该类再使用 `#import` 。这样做是将引入头文件的时机尽量延后，只在确有需要时才引入，这样就可以减少类的使用者所需引入的头文件数量，从而减少编译时间。

### #import、#include、@class区别：

#### \#include:

>1、在C语言中，我们使用 #include 来引入头文件。
>
>2、\#include 会造成重复引用头文件。
>
>3、为了防止重复引用可采用：
>
>	#ifndef  ViewController_h
>
>	#define ViewController_h
>
>	#endif

#### \#import:

>import 是 include 的升级版，可以防止重复引入头文件这种现象的发生。
>
>import会包含这个类的所有信息，包括实体变量和方法
>
>使用 #import 头文件会自动只导入一次，不会重复导入，相当于#include和#pragma once

#### @class

>@class 用来告诉编译器，有这样一个类，使书写代码时，不报错。 但是  @class 只是使导入的类名在引用时不受影响，不能创建该类的对象，因为创建对象时也需要访问其内部方法。

在编译效率方面考虑，如果你有100个头文件都 #import 了同一个头文件，或者这些文件是依次引用的，如 A–>B, B–>C, C–>D 这样的引用关系。当最开始的那个头文件有变化的话，后面所有引用它的类都需要重新编译，如果你的类有很多的话，这将耗费大量的时间。而是用 @class 则不会。

但是也有一些情况，是不可避免要在 .h 里引用的。比如：继承某个类，必须在 .h 里 import 父类的 .h；类实现某个接口，必须在 .h 里引用接口的 .h 等等

## 三、多用字面量语法，少用与之等价的方法

字面量字符串：

```
NSString *someStr = @"someStr";
```

字面数值：

```
NSNumber *intNum = @1;
NSNumber *boolNum = @YES;
NSNumber *charNum = @'a';
```

字面量数组：

```
NSArray *animale = @[@"cat",@"dog",@"mouse"];
```

使用字面量数组的好处是，当数组元素对象中有 nil，则会抛出异常。因为字面量语法实际上是一种“语法糖”，其效果等于先是创建了一个数组，然后把方括号内的所有对象都加到这个数组中。

下面这段代码分别以两种语法创建数组：

```
NSArray *arrayA = [NSArray arrayWithObjects:obj1,obj2,obj3, nil];
NSArray *arrayB = @[obj1,obj2,obj3];
```

假如 obj1 与 obj3 都指向了有效的 Objective-C 对象，而 obj2 为 nil，则字面量语法创建的数组 arrayB 会抛出异常。而 arrayA 虽然能够创建出来，但是其中却只含有 obj1 一个对象。原因在于`arrayWithObjects `方法会依次处理各个参数，之道发现 nil 为止，由于 obj2 是 nil，所以给方法会提前结束。

所以使用字面量语法更为安全，向数组中插入 nil 通常说明程序有错，而通过异常可以更快的发现这个错误。

字面量字典：

```
NSDictionary *personData = @{
                                 @"name":@"matt",
                                 @"age":@28
                                 };
```

使用字面量语法创建出来的字符串、数组、字典对象都是不可变的，若想变成可变的版本的对象，则需复制一份：

```
NSMutableArray *mutable = @[@1,@2,@3].mutableCopy;
```

## 四、多用类型常量，少用 #define (宏)预处理指令

我们定义常量一般会使用宏 \#define：

```
#define ANIMATION_DURATION 0.3
```

这样定义出来的常量不含类型信息，编译器只是会在编译前据此查找与替换操作。即使有人重新定义了常量值，编译器也不会产生警告信息，这将导致应用程序中的常量值不一致。

在实现文件 .m 中我们应该使用类型常量来定义常量：

```
staic const NSTimeInterval kAnimationDuration = 0.3;
```

在实现文件 .m 中，定义常量名称前面一般加字母 k。若在 .h 中,即常量在类之外可见，通常以类名为前缀:

```
//.h 中声明：
extern const NSTimeInterval EOCAnimationDuration; 

//.m 中实现：
const NSTimeInterval EOCAnimationDuration = 1.0; 
```

在头文件中使用 extern 来声明全局常量，并在相关实现文件中定义其值。这种常量要出现在全局符号表中，所以其名称应该加以区隔，通常用与之相关的类名做前缀。

### \#define、const、static、extern区别

#### 1、define 宏：

-  编译时刻：宏是预编译（编译之前处理），const是编译阶段。
-  编译检查：宏不做检查，不会报编译错误，只是替换，const会编译检查，会报编译错误。
-  宏的好处：宏能定义一些函数，方法。 const不能。
-  宏的坏处：使用大量宏，容易造成编译时间久，每次都需要重新替换。

#### 2、const:

中文“常量”意思。

- const用来修饰右边的基本变量或指针变量。
- 被修饰的变量只读，不能被修改。

看下面的例子，相信你就完全理解 const 的用法：

```
int const *p   // *p只读，p变量
int * const p  // *p变量，p只读
const int * const p //p和*p都只读
int const * const p //p和*p都只读
```

#### 3、static

中文“静态”意思。

#####（1）修饰局部变量
	
保证局部变量永远只初始化一次，在程序的运行过程中永远只有一份内存，  生命周期类似全局变量了，但是作用域不变。例如：

```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    
    static int i =0;
    i++;
    NSLog(@"i = %d",i);
}
```

得到的结果是变量 i 每次都会自增。如果不使用 static 修饰，则 i 每次结果只为 1。这就是关键字 static 修饰的局部变量的作用，让局部变量永远只初始化一次，一份内存，生命周期已经跟全局变量类似了，只是作用域不变。

#####（2）修饰全局变量

使全局变量的作用域仅限于当前文件内部，即当前文件内部才能访问该全局变量。

iOS 中在一个文件声明的全局变量，工程的其他文件也是能访问的，但是我又不想让其他文件访问，这时就可以用 static 修饰它了。

#####（3）修饰函数

static 修饰函数时，被修饰的函数被称为静态函数，使得外部文件无法访问这个函数，仅本文件可以访问。这个在 oc 语言开发中几乎很少用，c 语言倒是能看到一些影子，所以不详细探讨。

#### 4、extern

中文“外部的”意思。它的作用是声明外部全局变量。这里需要特别注意extern只能声明，不能用于实现。

在开发中，我们通常会单独抽一个类来管理一些全局的变量或常量，下面来看看逼格比较高的一种做法：

我们可以在.h文件中 extern 声明一些全局的常量:

```
extern NSString * const name;
extern NSInteger const count;
```

.m 中实现

```
NSString * const name = @"王五";
NSInteger const count = 3;
```

这样，只要导入头文件，就可以全局的使用定义的变量或常量。

## 五、用枚举表示状态、选项、状态码

枚举是一种常量命名方式。

### 枚举表示状态（状态码同理）

```
//第一种写法，先定义枚举类型，再定义枚举变量
enum EOCConnetionState{
    EOCConnetionStateDisconnected,
    EOCConnetionStateConnecting,
    EOCConnetionStateConnected
};
enum EOCConnetionState state;

//第二种写法，定义枚举类型的同时定义枚举变量
enum EOCConnetionState{
    EOCConnetionStateDisconnected,
    EOCConnetionStateConnecting,
    EOCConnetionStateConnected
}state;
```

由于每种状态都用一个便于理解的值来表示，所以这样写出来的代码更易读懂。

### 枚举表示选项

一个“选项变量”的类型要能够同时表示一个或多个组合的选择，如下例子：

```
enum TTGDirection {
    TTGDirectionNone = 0,
    TTGDirectionTop = 1 << 0,    //00000001
    TTGDirectionLeft = 1 << 1,   //00000010
    TTGDirectionRight = 1 << 2,  //00000100
    TTGDirectionBottom = 1 << 3  //00001000
};
```

这里的选项是用位运算的方式定义的，这样的好处就是，我们的选项变量可以如下表示：

```
//用“或”运算同时赋值多个选项
enum TTGDirection direction = TTGDirectionTop | TTGDirectionLeft | TTGDirectionBottom;

//用“与”运算取出对应位
if (direction & TTGDirectionTop) {
    NSLog(@"top");
}
if (direction & TTGDirectionLeft) {
    NSLog(@"left");
}
if (direction & TTGDirectionRight) {
    NSLog(@"right");
}
if (direction & TTGDirectionBottom) {
    NSLog(@"bottom");
}
```

direction变量的实际内存如下：

```
0 0 0 0 1 0 1 1
```

控制台输出：

```
top
left
bottom
```

这样，用位运算，就可以同时支持多个值。

#### enum在 Objective-C 中的“升级版”

一般来说，我们不能指定枚举变量的实际类型是什么，就是说，我们不知道枚举最后是 int 型，还是其他的什么类型。但是从 C++ 11开始，我们可以为枚举指定其实际的存储类型，如下语法：

```
enum TTGState : NSInteger {/*...*/};
```

但是，我们在定义枚举的时候如何保证兼容性呢？Foundation 框架已经为我们提供了更加“统一、便捷”的枚举定义方法，如下：

```
//NS_ENUM，定义状态等普通枚举
typedef NS_ENUM(NSUInteger, TTGState) {
    TTGStateOK = 0,
    TTGStateError,
    TTGStateUnknow
};

//NS_OPTIONS，定义选项
typedef NS_OPTIONS(NSUInteger, TTGDirection) {
    TTGDirectionNone = 0,
    TTGDirectionTop = 1 << 0,
    TTGDirectionLeft = 1 << 1,
    TTGDirectionRight = 1 << 2,
    TTGDirectionBottom = 1 << 3
};
```

所以，在 iOS 开发中，枚举最好使用`NS_ENUM`和`NS_OPTIONS`定义，并指明其底层数据类型，保证统一。

### 处理枚举类型 switch 语句中不要实现 default 分支

这样的话，加入新枚举之后，编译器就会提示开发者 switch 语句并未处理所有枚举。如果写上了 default 分支，那么它就会处理这个新状态，从而导致编译器不发警告信息。


## 参考资料

1、Effective Objective-C 2.0

2、[https://juejin.im/post/5aaf6943518825556e5de48e](https://juejin.im/post/5aaf6943518825556e5de48e)

3、[http://www.cocoachina.com/ios/20161110/18035.html](http://www.cocoachina.com/ios/20161110/18035.html)

4、[https://www.cnblogs.com/feiyu-mdm/p/6392493.html](https://www.cnblogs.com/feiyu-mdm/p/6392493.html)














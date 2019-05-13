---
layout:     post
title:      Effective Objective-C 2.0 第二章 11、理解 objc_msgSend 的作用
subtitle:   objc_msgSend
date:       2019-05-04
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

我之前一篇文章
[iOS runtime 消息机制及消息转发](https://www.jianshu.com/p/a7e365f197fb)对此有详细介绍。本篇简单介绍下书中内容。

在对象上调用方法，又叫 “传递消息”。

由于Objective-C是C的超集，所以最好先理解C语言的函数调用方式。C语言使用“静态绑定”（static binding），也就是说，在编译期就能决定运行时所应调用的函数。以下列代码为例：

```
#import <stdio.h> 
 
void printHello() {  
    printf("Hello, world!\n");  
}  
void printGoodbye() {  
    printf("Goodbye, world!\n");  
}  
 
void doTheThing(int type) {  
    if (type == 0) {  
        printHello();  
    } else {  
        printGoodbye();  
    }  
    return 0;  
} 
```

如果不考虑“内联”（inline），那么编译器在编译代码的时候就已经知道程序中有printHello与printGoodbye这两个函数了，于是会直接生成调用这些函数的指令。而函数地址实际上是硬编码在指令之中的。若是将刚才那段代码写成下面这样，会如何呢？

```
#import <stdio.h> 
 
void printHello() {  
    printf("Hello, world!\n");  
}  
void printGoodbye() {  
    printf("Goodbye, world!\n");  
}  
 
void doTheThing(int type) {  
    void (*fnc)();  
    if (type == 0) {  
        fnc = printHello;  
    } else {  
        fnc = printGoodbye;  
    }  
    fnc();  
    return 0;  
}
```

这时就得使用“动态绑定”（dynamic binding）了，因为所要调用的函数直到运行期才能确定。编译器在这种情况下生成的指令与刚才那个例子不同，在第一个例子中，if与else语句里都有函数调用指令。而在第二个例子中，只有一个函数调用指令，不过待调用的函数地址无法硬编码在指令之中，而是要在运行期读取出来。

在 Objective-C 中，如果向某对象传递消息，那就会使用动态绑定机制来决定需要调用的方法。在底层，所有方法都是普通的 C 语言函数，然而对象收到消息之后，究竟该调用哪个方法则完全于运行期决定，甚至可以在程序运行时改变，这些特性使得 Objective-C 成为一门真正的动态语言。

给对象发送消息可以这样来写：

```
id returnValue = [someObject messageName:parameter]; 
```

在本例中，someObject叫做“接收者”（receiver），messageName叫做“选择子”（selector）。选择子与参数合起来称为“消息”（message）。编译器看到此消息后，将其转换为一条标准的C语言函数调用，所调用的函数乃是消息传递机制中的核心函数，叫做objc_msgSend，其“原型”（prototype）如下：

```
id objc_msgSend(id self, SEL op, ...);
```

这是个“参数个数可变的函数”（variadic function），能接受两个或两个以上的参数。第一个参数代表接收者，第二个参数代表选择子（SEL是选择子的类型），后续参数就是消息中的那些参数，其顺序不变。选择子指的就是方法的名字。“选择子”与“方法”这两个词经常交替使用。编译器会把刚才那个例子中的消息转换为如下函数：

```
id returnValue = objc_msgSend(someObject,  
                              @selector(messageName:),  
                              parameter); 
```

objc_msgSend函数会依据接收者与选择子的类型来调用适当的方法。为了完成此操作，该方法需要在接收者所属的类中搜寻其“方法列表”（list of methods），如果能找到与选择子名称相符的方法，就跳至其实现代码。若是找不到，那就沿着继承体系继续向上查找，等找到合适的方法之后再跳转。如果最终还是找不到相符的方法，那就执行“消息转发”（message forwarding）操作。消息转发将在第12条中详解。

## 总结


消息由接收者、选择子及参数构成。给某对象“发送消息”（invoke a message）也就相当于在该对象上“调用方法”（call a method）。

发给某对象的全部消息都要由“动态消息派发系统”（dynamic message dispatch system）来处理，该系统会查出对应的方法，并执行其代码。

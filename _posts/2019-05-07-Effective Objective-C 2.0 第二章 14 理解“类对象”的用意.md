---
layout:     post
title:      Effective Objective-C 2.0 第二章 14 理解“类对象”的用意
subtitle:   类对象
date:       2019-05-07
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

我之前这篇文章[iOS runtime 类与对象](https://www.jianshu.com/p/1f9d37b68aba)已经详细介绍过类与对象，本篇重点介绍书中内容。

Objective-C实际上是一门极其动态的语言。第11条讲解了运行期系统如何查找并调用某方法的实现代码，第12条则讲述了消息转发的原理：如果类无法立即响应某个选择子，那么就会启动消息转发流程。然而，消息的接收者究竟是何物？是对象本身吗？运行期系统如何知道某个对象的类型呢？对象类型并非在编译期就绑定好了，而是要在运行期查找。而且，还有个特殊的类型叫做id，它能指代任意的Objective-C对象类型。一般情况下，应该指明消息接收者的具体类型，这样的话，如果向其发送了无法解读的消息，那么编译器就会产生警告信息。而类型为id的对象则不然，编译器假定它能响应所有消息。

如果看过第12条，你就会明白，编译器无法确定某类型对象到底能解读多少种选择子，因为运行期还可向其中动态新增。然而，即便使用了动态新增技术，编译器也觉得应该能在某个头文件中找到方法原型的定义，据此可了解完整的“方法签名”（method signature），并生成派发消息所需的正确代码。

“在运行期检视对象类型”这一操作也叫做“类型信息查询”（introspection，“内省”），这个强大而有用的特性内置于Foundation框架的NSObject协议里，凡是由公共根类（common root class，即NSObject与NSProxy）继承而来的对象都要遵从此协议。在程序中不要直接比较对象所属的类，明智的做法是调用“类型信息查询方法”，其原因笔者稍后解释。不过在介绍类型信息查询技术之前，我们先讲一些基础知识，看看Objective-C对象的本质是什么。

每个Objective-C对象实例都是指向某块内存数据的指针。所以在声明变量时，类型后面要跟一个“*”字符：

```
NSString *pointerVariable = @"Some string";
```

编过C语言程序的人都知道这是什么意思。对于没写过C语言的程序员来说，pointerVariable可以理解成存放内存地址的变量，而NSString自身的数据就存于那个地址中。因此可以说，该变量“指向”（point to）NSString实例。所有Objective-C对象都是如此，若是想把对象所需的内存分配在栈上，编译器则会报错：

```
String stackVariable = @"Some string";  
// error: interface type cannot be statically allocated 
```

对于通用的对象类型id，由于其本身已经是指针了（id 定义：`typedef struct objc_object *id;`），所以我们能够这样写：

```
id genericTypedString = @"Some string"; 
```

上面这种定义方式与用NSString*来定义相比，其语法意义相同。唯一区别在于，如果声明时指定了具体类型，那么在该类实例上调用其所没有的方法时，编译器会探知此情况，并发出警告信息。

描述Objective-C对象所用的数据结构定义在运行期程序库的头文件里，id类型本身也在定义在这里：

```
typedef struct objc_object *id;

struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

由此可见，每个对象结构体的首个成员是Class类的变量。该变量定义了对象所属的类，通常称为“is a”指针。例如，刚才的例子中所用的对象“是一个”（is a）NSString，所以其“is a”指针就指向NSString。Class对象也定义在运行期程序库的头文件中：

```
typedef struct objc_class *Class;  
struct objc_class {  
    Class isa;  
    Class super_class;  
    const char *name;  
    long version;  
    long info;  
    long instance_size;  
    struct objc_ivar_list *ivars;  
    struct objc_method_list **methodLists;  
    struct objc_cache *cache;  
    struct objc_protocol_list *protocols;  
};
```

此结构体存放类的“元数据”（metadata），例如类的实例实现了几个方法，具备多少个实例变量等信息。此结构体的首个变量也是isa指针，这说明Class本身亦为Objective-C对象。结构体里还有个变量叫做super_class，它定义了本类的超类。类对象所属的类型（也就是isa指针所指向的类型）是另外一个类，叫做“元类”（metaclass），用来表述类对象本身所具备的元数据。“类方法”就定义于此处，因为这些方法可以理解成类对象的实例方法。每个类仅有一个“类对象”，而每个“类对象”仅有一个与之相关的“元类”。

假设有个名为SomeClass的子类从NSObject中继承而来，则其继承体系如图所示。

![](https://upload-images.jianshu.io/upload_images/1776587-5cfe05658371feff.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

super_class指针确立了继承关系，而isa指针描述了实例所属的类。通过这张布局关系图即可执行“类型信息查询”。我们可以查出对象是否能响应某个选择子，是否遵从某项协议，并且能看出此对象位于“类继承体系”（class hierarchy）的哪一部分。

在类继承体系中查询类型信息

可以用类型信息查询方法来检视类继承体系。“isMemberOfClass:”能够判断出对象是否为某个特定类的实例，而“isKindOfClass:”则能够判断出对象是否为某类或其派生类的实例。例如：

```
NSMutableDictionary *dict = [NSMutableDictionary new];  
[dict isMemberOfClass:[NSDictionary class]]; ///< NO 
[dict isMemberOfClass:[NSMutableDictionary class]]; ///< YES 
[dict isKindOfClass:[NSDictionary class]]; ///< YES 
[dict isKindOfClass:[NSArray class]]; ///< NO 
```

像这样的类型信息查询方法使用isa指针获取对象所属的类，然后通过super_class指针在继承体系中游走。由于对象是动态的，所以此特性显得极为重要。Objective-C与你可能熟悉的其他语言不同，在此语言中，必须查询类型信息，方能完全了解对象的真实类型。

由于Objective-C使用“动态类型系统”（dynamic typing），所以用于查询对象所属类的类型信息查询功能非常有用。从collection中获取对象时，通常会查询类型信息，这些对象不是“强类型的”（strongly typed），把它们从collection中取出来时，其类型通常是id。如果想知道具体类型，那就可以使用类型信息查询方法。例如，想根据数组中存储的对象生成以逗号分隔的字符串（comma-separated string），并将其存至文本文件，就可以使用下列代码：

```
- (NSString*)commaSeparatedStringFromObjects:(NSArray*)array {  
    NSMutableString *string = [NSMutableStringnew];  
    for (id object in array) {  
        if ([object isKindOfClass:[NSStringclass]]) {  
            [string appendFormat:@"%@,", object];  
        } else if ([object isKindOfClass:[NSNumberclass]]) {  
            [string appendFormat:@"%d,", [object intValue]];  
        } else if ([object isKindOfClass:[NSDataclass]]) {  
            NSString *base64Encoded = /* base64 encoded data */;  
            [string appendFormat:@"%@,", base64Encoded];  
        } else {  
            // Type not supported  
        }  
    }  
    return string;  
} 
```

也可以用比较类对象是否等同的办法来做。若是如此，那就要使用==操作符，而不要使用比较Objective-C对象时常用的“isEqual:”方法（参见第8条）。原因在于，类对象是“单例”（singleton），在应用程序范围内，每个类的Class仅有一个实例。也就是说，另外一种可以精确判断出对象是否为某类实例的办法是：

```
id object = /* ... */;  
if ([object class] == [EOCSomeClassclass]) {  
    // 'object' is an instance of EOCSomeClass  
} 
```

即便能这样做，我们也应该尽量使用类型信息查询方法，而不应该直接比较两个类对象是否等同，因为前者可以正确处理那些使用了消息传递机制（参见第12条）的对象。比方说，某个对象可能会把其收到的所有选择子都转发给另外一个对象。这样的对象叫做“代理”（proxy），此种对象均以NSProxy为根类。

通常情况下，如果在此种代理对象上调用class方法，那么返回的是代理对象本身（此类是NSProxy的子类），而非接受的代理的对象所属的类。然而，若是改用“isKindOfClass:”这样的类型信息查询方法，那么代理对象就会把这条消息转给“接受代理的对象”（proxied object）。也就是说，这条消息的返回值与直接在接受代理的对象上面查询其类型所得的结果相同。因此，这样查出来的类对象与通过class方法所返回的那个类对象不同，class方法所返回的类表示发起代理的对象，而非接受代理的对象。

## 总结

- 每个实例都有一个指向Class对象的指针，用以表明其类型，而这些Class对象则构成了类的继承体系。

- 如果对象类型无法在编译期确定，那么就应该使用类型信息查询方法来探知。

- 尽量使用类型信息查询方法来确定对象类型，而不要直接比较类对象，因为某些对象可能实现了消息转发功能。
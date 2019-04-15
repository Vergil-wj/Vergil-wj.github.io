---
layout:     post
title:      iOS runtime 消息机制
subtitle:   messages
date:       2019-04-15
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - runtime
    - Objective-C
---

## Sending Messages

在 Objective-C 中，如果向某对象传递消息，那就会使用动态绑定机制来决定需要调用的方法。在底层，所有方法都是普通的 C 语言函数，然而对象收到消息之后，究竟该调用哪个方法则完全于运行期决定，甚至可以在程序运行时改变，这些特性使得 Objective-C 成为一门真正的动态语言。

我们看下定义：

```
id objc_msgSend(id self, SEL op, ...);
```

其中参数解释：

- `self`: 消息接收者
- `op`: 消息的selector，一个C的字符串用来定位
- `...`: 方法参数数组

从定义可以看出消息由接受者，选择器及参数构成。

给对象发送消息可以这么写：

```
id returnValue = [someObject messageName:parameter]; 
```

编译器会将消息转换成如下函数：

```
id returnValue = objc_msgSend(someObject,  
                              @selector(messageName:),  
                              parameter); 
```

> objc_msgSend 函数会依据接收者与选择子的类型来调用适当的方法。为了完成此操作，该方法需要在接收者所属的类中搜寻其“方法列表”（list of methods），如果能找到与选择子名称相符的方法，就跳至其实现代码。若是找不到，那就沿着继承体系继续向上查找，等找到合适的方法之后再跳转。如果最终还是找不到相符的方法，那就执行“消息转发”（message forwarding）操作。

下面解释下上面这段话：

### objc_msgSend 如何找到该方法？

#### 1、我们先来看下对象定义：

```
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

其中 isa 指向对象所属的类，也就是上面例子中 `someObject` 由 isa 找到其所属的类。

#### 2、我们看下类的定义：

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

我们看到这一行

```
struct objc_method_list **methodLists 
```

很明显类会从这个方法列表中找到与选择器名称相同的方法。

#### 3、我们看下方法的定义：

```
struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}
```

其中参数：

- method_name：方法名字
- method_types：方法类型
- method_imp：方法具体实现

我们可以看到该结构体中包含一个 SEL 和 IMP，实际上相当于在 SEL 和 IMP 之间作了一个映射。有了 SEL，我们便可以找到对应的IMP，IMP 实际上是一个函数指针，指向方法实现的地址，其定义如下：

```
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```

在这里执行具体的代码，并返回值给调用者。

写个例子，再梳理一遍：

```
NSString * helloWorld =  [obj returnMeHelloWorld];
```

其传递消息流程：

- 编译成
`id objc_msgSend(self,@selector(returnMeHelloWorld:))`;
- 在 self 中沿着 isa 找到 CustomObject 的类对象
- 类对象查找自己的方法 list，找到对应的方法执行体 Method
- 把参数传递给 IMP 实际指向的执行代码
- 代码执行返回结果给 helloWorld

### 以上是实例方法的处理，那么类方法是如何处理的呢？

在类的定义中，可以看到第一行：

```
Class isa  OBJC_ISA_AVAILABILITY;
```

这个isa指向的一个Class类型，就是保存了类方法的地方，这个Class类型的东西就是类元对象。类方法的调用先从类元对象中找到对应的方法，后面就和上面举的例子中实例方法的调用流程相同了，这里不再赘述。

## 消息转发 message forwarding

当发送一条消息时，如果接受对象没有找到对应的方法，会沿着其继承体系继续向上找，如果最终还是没有找到，就会走消息转发。

消息转发主要涉及到的方法，在 NSObject.h 中：

```
+ (BOOL)resolveClassMethod:(SEL)sel
+ (BOOL)resolveInstanceMethod:(SEL)sel

- (id)forwardingTargetForSelector:(SEL)aSelector

- (void)forwardInvocation:(NSInvocation *)anInvocation
```

我们先来看下整体流程图：

![](https://ws1.sinaimg.cn/large/006tNc79ly1g233sf3nzuj30jg0b4q3g.jpg)

下面举例说明，在 ViewController.m 中：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self performSelector:@selector(method1)];
    
}
```

我们调用方法 method1，但并没有实现它，那么程序就会崩溃，控制台输出如下：

```
 -[ViewController method1]: unrecognized selector sent to instance 0x151d0f2b0
 *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[ViewController method1]: unrecognized selector sent to instance 0x151d0f2b0'
```

这段异常信息实际上是由 NSObject 的`doesNotRecognizeSelector`方法抛出的。不过，我们可以采取一些措施，让我们的程序执行特定的逻辑，而避免程序的崩溃。

### 第一种 动态方法解析，类自己处理，动态添加方法

对象在接收到未知的消息时，我们可以动态添加方法，首先会调用以下方法:

```
+resolveInstanceMethod:(实例方法)

//或者
+resolveClassMethod:(类方法)
```
在这个方法中，我们可以动态的为该未知消息新增一个处理方法，只需要在运行时通过 class_addMethod 函数动，动态添加到类里面就可以了。如下代码所示：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self performSelector:@selector(method1)];
    
}

void dynamicMethodIMP(id self, SEL _cmd){
    NSLog(@"implementation method1");
}

+ (BOOL) resolveInstanceMethod:(SEL)aSEL{
    
    if (aSEL == @selector(method1)){
        class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
```

控制台输出：

```
implementation method1
```

这里用到了 runtime 中 class_addMethod 方法，其定义如下：

```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
```

其中参数：

- cls: 要添加方法的类。
- name: 方法名。
- imp: 具体方法的实现。
- types: 方法参数的编码，详见[文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)

### 第二种 备用接受者，交给其它对象处理

如果在上一步无法处理消息，则Runtime会继续调以下方法，重定向给其它对象：

```
- (id)forwardingTargetForSelector:(SEL)aSelector
```

在这里我们创建一个新的类 TestObject，并实现 method1 方法：

TestObject.h 文件：

```
@interface TestObject : NSObject

-(void)method1;

@end
```

TestObject.m 文件：

```
@implementation TestObject

-(void)method1{
    NSLog(@"TestObject method1");
}

@end
```

然后回到 ViewController.m 文件,代码如下：

```
@interface ViewController (){
    TestObject *_testObj;
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _testObj = [[TestObject alloc]init];
    
    [self performSelector:@selector(method1)];
    
}

- (id)forwardingTargetForSelector:(SEL)aSelector{
    
    if (aSelector == @selector(method1)) {
        return _testObj;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

控制台输出：

```
TestObject method1
```

我们可以看到在这个方法中，我们并不能操作参数与返回值，也就说我们发一条消息带有参数和返回值，则本方法无法使用。

### 最后 完整消息转发，重定向消息。

对于完整转发，NSObject提供了以下方法来处理:

```
- (void)forwardInvocation:(NSInvocation *)anInvocation
```

当前面两步都无法处理消息时，运行时系统便会给接收者最后一个机会，将其转发给其它代理对象来处理。这主要是通过创建一个表示消息的 NSInvocation 对象并将这个对象当作参数传递给 `forwardInvocation:` 方法。我们在 `forwardInvocation:` 方法中可以选择将消息转发给其它对象。

还有重要的一点，我们必须重写以下方法：

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
```

看一下官方原文：

>Important
>
>To respond to methods that your object does not itself recognize, you must override methodSignatureForSelector: in addition to forwardInvocation:. The mechanism for forwarding messages uses information obtained from methodSignatureForSelector: to create the NSInvocation object to be forwarded. Your overriding method must provide an appropriate method signature for the given selector, either by pre formulating one or by asking another object for one.

我们必须重写`methodSignatureForSelector:`和`forwardInvocation:`。转发消息的机制使用从`methodSignatureForSelector`获得的信息来创建要转发的 NSInvocation 对象。重写方法必须为给定的选择器提供适当的方法签名，方法可以是预先构造一个方法签名，也可以是向另一个对象请求一个方法签名。

所以我们下面重写一下这两个方法：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self performSelector:@selector(method1)];
}

- (NSMethodSignature*)methodSignatureForSelector:(SEL)aSelector{
    //判断selector是否为需要转发的，如果是则手动生成方法签名并返回。
    if (aSelector == @selector(method1)){
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super forwardingTargetForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    //判断待处理的anInvocation是否为我们要处理的
    if (anInvocation.selector == @selector(method1)){
        NSLog(@"执行 method1");
    }else{
        [super forwardInvocation:anInvocation];
    }
}
```

控制台输出：

```
执行 method1
```

其中 anInvocation 保存着我们调用一个 method 的所有信息。

## 小结

本片文章主要了解下消息发送机制，通过它我们可以为程序动态增加很多行为，例如消息转发中的第二步和第三步，这两个方法都允许一个对象与其它对象建立关系，以处理某些消息，我们可以以此来模拟多重继承，让对象可以“继承”其它对象的特性来处理一些事情，当然最好不要这么做，如果可以，我们应该用常规方式解决问题。


###### 参考资料：

[runtime 源码 ](https://github.com/opensource-apple)

[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)

[http://book.51cto.com/art/201403/432144.htm](http://book.51cto.com/art/201403/432144.htm)

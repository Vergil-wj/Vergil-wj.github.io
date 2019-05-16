---
layout:     post
title:      Effective Objective-C 2.0 第四章 协议与分类
subtitle:   协议与分类
date:       2019-05-14
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 第 23 条 通过委托与数据源协议进行对象间通信

1、Objective-C 可以使用 “委托模式”（Delegate pattern）的编程设计模式来实现对象间的通信：定义一套接口，某对象若想接受另一个对象的委托，则需遵从此接口，以便成为其 “委托对象”（delegate）。Objective-C 一般利用 “协议” 机制来实现此模式。

2、定义协议：

```
@protocol EOCNetworkingFetcherDelegate
@optional
- (void)newworkingFetcher:(EOCNetworkingFetcher *)fetcher
            didRecevieData:(NSData *)data;
- (void)newworkingFetcher:(EOCNetworkingFetcher *)fetcher
         didFailWithError:(NSError *)error;
@end
@interface EOCNetworkingFetcher : NSObject
@property (nonatomic,weak) id delegate;
@end
```

- 委托协议名通常时在相关的类名加上Delegate 一词，也是采用 “驼峰法” 来命名。

- 类可以用一个属性存放其委托对象，属性要用weak 来修饰，避免产生 “保留环”（retain cycle）。

- 某类若要遵从某委托协议，可以在其接口中声明，也可以在"class-continuation 分类" 中声明，如果要向外界公布此类实现了某协议，就在接口中声明，如果这个协议是个委托协议，通常只会在这个类的内部使用，这样子就在分类中声明就好了。

3、如果要在委托对象上调用可选方法，那么必须提前使用类型信息查询方法，判断这个委托对象能否响应相关的选择子。

```
NSData *data;
if([_delegate respondsToSelector:@selector(networkFetcher:didRecevieData:)]){
    [_delegate networkFetcher:self didRecevieData:data];
}
```

4、delegate 里的方法也可以用于从委托对象中获取信息（数据源模式）。

5、在实现委托模式和数据源模式的时，协议中的方法是可选的，我们就会写出大量这种判断代码

```
if([_delegate respondsToSelector:@selector(networkFetcher:didRecevieData:)]){
    [_delegate networkFetcher:self didRecevieData:data];
}
```

- 每次调用方法都会判断一次，其实除了第一次检测的结构有用，后续的检测很有可能都是多余的，因为委托对象本身没变，不太可能会一下子不响应，一下子响应的，所以我们这里可以把这个委托对象能否响应某个协议方法记录下来，以优化程序效率。

- 将方法响应能力缓存起来的最佳途径是使用 “位段”（bitfield）数据类型。我们可以把结构体中某个字段所占用的二进制位个数设为特定的值。

>位段，C语言允许在一个结构体中以位为单位来指定其成员所占内存长度，这种以位为单位的成员称为“位段”或称“位域”( bit field) 。
>
>struct data {
>
>unsigned int filedA : 8;
>
>unsigned int filedB : 4;
>
>unsigned int filedC : 2;
>
>unsigned int filedD : 1;
>
>}
>
>filedA 位段占用8个二进制位，filedB 位段占用4个二进制位，filedC 位段占用2个二进制位，filedD位段占用1个二进制位。filedA 就可以表示0至255之间的值，而filedD 则可以表示0或1这两个值。

>我们可以像filedD 这样子，创建大小只有1的位段，这样子就可以把Boolean 值塞入这一小块数据里面，这里很适合这样子做。

以网络数据获取器为例，可以在该实例中嵌入一个含有位段的结构体作为其实例变量，而结构体中的每个位段则表示 delegate 对象是否实现了协议中的相关方法。此结构体的用法如下:

```
@interface EOCNetworkFetcher () {
    struct {
        unsigned int didReceiveData : 1;
        unsigned int didFailWithError : 1;
        unsigned int didUpdateProgressTo : 1;
    } _delegateFlags;
}

@end
```

这个结构体用来缓存委托对象是否能响应特定的选择子。实现缓存功能所用的代码可以写在 delegate 属性所对应的设置方法里:

```
- (void)setDelegate:(id<EOCNetworkFetcherDelegate>)delegate {
    _delegate = delegate;
    _delegateFlags.didReceiveData = [delegate respondsToSelector:@selector(networkFetcher:didReceiveData:)];
    _delegateFlags.didFailWithError = [delegate respondsToSelector:@selector(netWorkFetcher:didFailWithError:)];
    _delegateFlags.didUpdateProgressTo = [delegate respondsToSelector:@selector(networkFetcher:didUpdateProgressTo:)];
}
```

这样的话，每次调用 delegate 的相关方法之前，就不用检测委托对象是否能响应给定的选择子了，而是直接查询结构体里的标志：

```
if (_delegateFlags.didUpdateProgressTo) {
	[_delegate networkFetcher:self didUpdateProgressTo:currentProgress];
}
```

在相关方法要调用很多次时，值得进行这种优化。而是否需要优化，则应依照具体代码来定。这就需要分析代码性能，并找出瓶颈，若发现执行速度需要改进，则可使用此技巧。如果要频繁通过数据源协议从数据源中获取多份相互独立的数据，那么这项优化技术极有可能会提高程序效率。

### 要点
- 委托模式为对象提供了一套接口，使其可由此将相关事件告知其他对象
- 将委托对象应该支持的接口定义成协议，在协议中把可能需要处理的事件定义成方法。
- 当某个对象需要从另一个对象中获取数据时，可以使用委托模式。这种情景下，该模式亦成为“数据源协议”（data source protocal）。
- 如果有必要，可以实现含有位段的结构体，将委托对象是否能相应相关协议方法这一信息缓存其中。

## 第 24 条 将类的实现代码分散到便于管理的数个分类中

一个类经常有很多方法，尽管代码写的比较规范，这个文件还是会越来越大，定位问题以及阅读上都会造成不便。我们可以通过 “分类” 机制来把代码按逻辑划分到几个分区中。

例如：

```
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface TestObject : NSObject<NSCopying,NSMutableCopying>

@property(nonatomic,copy)NSString *name;

-(instancetype)initWithName:(NSString *)name;

//work methods
-(void)performDaysWork;
-(void)takeVacationFromWork;

//play methods
-(void)goToTheCinema;
-(void)goToSportsGame;

@end

NS_ASSUME_NONNULL_END
```

利用分类，改写成这样：

TestObject.h:

```
#import <Foundation/Foundation.h>
@interface TestObject : NSObject
@property(nonatomic,copy)NSString *name;
-(instancetype)initWithName:(NSString *)name;
@end
```

TestObject (work).h

```
#import "TestObject.h"
@interface TestObject (work)
-(void)performDaysWork;
-(void)takeVacationFromWork;
@end
```

TestObject (Play).h

```
#import "TestObject.h"
@interface TestObject (Play)
-(void)goToTheCinema;
-(void)goToSportsGame;
@end
```

### 要点

- 使用分类机制把类的实现代码划分成易于管理的小块。

- 将应该视为 “私有” 的方法归入为叫Private 的分类中，以隐藏实现细节。

## 第 25 条 总是为第三方类的分类名称加前缀

向第三方类中添加分类时，总应该给其名称加上你专用的前缀。

向第三方类中添加分类时，总应给其中的方法名加上你专用的前缀。

## 第 26 条：勿在分类中声明属性

可以利用运行期的关联对象机制，为分类声明属性，但是这种做法要尽量避免，因为除了 "class-continuation 分类" 之外，其他分类都无法向类中新增实例变量，因此，他们无法把实现属性所需的实例变量合成出来。

- 尽量把封装数据所用的全部属性都定义在主接口里。

- 在 “class-continuation 分类” 之外的其他分类中，可以定义存取方法，但尽量不要定义属性。

## 第 27 条：使用 ”class-continuation 分类“ 隐藏实现细节

就是在 .m 文件中声明属性和方法，隐藏实现细节：

```
#import "TestObject.h"

@interface TestObject ()

@property(nonatomic,copy)NSString *privateStr;

-(void)privateMethod;

@end

@implementation TestObject


@end
```

- 通过 “class-continuation 分类” 向类中新增实例变量。

- 如果某属性在主接口中声明为 “只读”，而类的内部又要用设置方法修改此属性，那么就在 “class-continuation 分类” 中将其扩展为 “可读写”。

- 把私有方法的原型声明在 “class-contiunation 分类” 里面。

- 若想使类所遵循的协议不为人所知，则可于 “class-contiunation 分类” 中声明。

## 第 28 条：通过协议提供匿名对象

```
@property(nonatomic,weak)id<EOCDelegate>delegate;
```

该属性是 id<EOCDelegate>，所以任何类的对象都可以充当这一属性，只要遵循 EOCDelegate 协议就行。

- 协议可在某种程度上提供匿名类型。具体的对象类型可以淡化成遵从某协议的id 类型，协议里规定了对象所应实现的方法。

- 使用匿名对象来隐藏类型名称（或类名）。

- 如果具体类型不重要，重要的是对象能够响应（定义在协议里的）特定方法，那么可使用匿名对象来表示。


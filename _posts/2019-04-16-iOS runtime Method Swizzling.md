---
layout:     post
title:      iOS runtime Method Swizzling
subtitle:   黑魔法
date:       2019-04-16
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - runtime
    - Objective-C
---

## 黑魔法 Method Swizzling

其实就是交换两个方法的实现。

我们看下主要用到的方法的定义：

```
void method_exchangeImplementations(Method m1, Method m2)
```

举个例子说明下，想跟踪在程序中每一个 view controller 展示给用户的次数：当然，我们可以在每个 view controller 的 viewDidAppear 中添加跟踪代码；但是这太过麻烦，需要在每个 view controller 中写重复的代码。创建一个子类可能是一种实现方式，但需要同时创建 UIViewController, UITableViewController, UINavigationController及其它 UIKit 中 view controller 的子类，这同样会产生许多重复的代码。

这种情况下，我们就可以使用Method Swizzling，如下代码所示：

```
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);
		
		//实例方法
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // 类方法
        // Class class = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(class, originalSelector);
        // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);

		//先尝试給源方法添加实现，这里是为了避免源方法没有实现的情况
        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
        	 //添加成功：将源方法的实现替换到交换方法的实现
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
        	//添加失败：说明源方法已经有实现，直接将两个方法的实现交换即可
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```

在这里，我们通过 method swizzling 修改了 UIViewController 的 @selector(viewWillAppear:) 对应的函数指针，使其实现指向了我们自定义的  xxx_viewWillAppear 的实现。这样，当 UIViewController 及其子类的对象调用 viewWillAppear 时，都会打印一条日志信息。

## 注意事项

### Swizzling应该总是在+load中执行

在 Objective-C 中，运行时会自动调用每个类的两个方法。+load 会在类初始加载时调用，+initialize 会在第一次调用类的类方法或实例方法之前被调用。这两个方法是可选的，且只有在实现了它们时才会被调用。由于 method swizzling 会影响到类的全局状态，因此要尽量避免在并发处理中出现竞争的情况。+load 能保证在类的初始化过程中被加载，并保证这种改变应用级别的行为的一致性。相比之下，+initialize 在其执行时不提供这种保证–事实上，如果在应用中没为给这个类发送消息，则它可能永远不会被调用。

### Swizzling应该总是在dispatch_once中执行

与上面相同，因为 swizzling 会改变全局状态，所以我们需要在运行时采取一些预防措施。原子性就是这样一种措施，它确保代码只被执行一次，不管有多少个线程。GCD 的 dispatch_once 可以确保这种行为，我们应该将其作为 method swizzling 的最佳实践。

## 小结

本片主要介绍了黑魔法 Method Swizzling 如何使用及一些注意事项，通过对 Method Swizzling 的灵活运用，将有助于我们在业务量不断增大的同时还能保持代码的低耦合度，降低维护的工作量和难度。

##### 参考资料

[https://nshipster.com/method-swizzling/](https://nshipster.com/method-swizzling/)
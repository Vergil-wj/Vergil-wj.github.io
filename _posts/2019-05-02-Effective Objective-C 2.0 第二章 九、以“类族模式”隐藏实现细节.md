---
layout:     post
title:      Effective Objective-C 2.0 第二章 九、以“类族模式”隐藏实现细节
subtitle:   以“类族模式”隐藏实现细节
date:       2019-05-02
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

类族模式也可叫做类簇模式。

在工厂模式中，有简单工厂模式，工厂方法模式，抽象工厂模式。而类族模式与简单工厂模式类似，在 Objective-C 的系统框架中普遍使用此模式，例如 UIBbutton，NSArray，NSNumber 等等。

我们想要创建一个按钮时，需要调用下面这个类方法：

```
+ (instancetype)buttonWithType:(UIButtonType)buttonType;
```

该方法所返回的对象，其类型取决于传入的按钮类型（button type）。然而，不管返回什么类型的对象，它们都继承自同一个基类：UIButton。这么做的意义在于：UIButton 类的使用者无须关心创建出来的按钮具体属于哪个子类，也不用考虑按钮的绘制方式等实现细节。使用者只需明白如何创建按钮，如何设置像“标题”（title）这样的属性，如何增加触摸动作的目标对象等问题就好。

回到开头说的那个问题上，我们可以把各种按钮的绘制逻辑都放在一个类里，并根据按钮类型来切换：

```
- (void)drawRect:(CGRect)rect {  
    if (_type == TypeA) {  
        // Draw TypeA button  
    } else if (_type == TypeB) {  
        // Draw TypeB button  
    } /* ... */  
} 
```

这样写现在看上去还算简单，然而，若是需要依按钮类型来切换的绘制方法有许多种，那么就会变得很麻烦了。优秀的程序员会将这种代码重构为多个子类，把各种按钮所用的绘制方法放到相关子类中去。不过，这么做需要用户知道各种子类才行。此时应该使用“类族模式”，该模式可以灵活应对多个类，将它们的实现细节隐藏在抽象基类后面，以保持接口简洁。用户无须自己创建子类实例，只需调用基类方法来创建即可。

## 创建类族

现在举例来演示如何创建类族。假设有一个处理雇员的类，每个雇员都有“名字”和“薪水”这两个属性，管理者可以命令其执行日常工作。但是，各种雇员的工作内容却不同。经理在带领雇员做项目时，无须关心每个人如何完成其工作，仅需指示其开工即可。

首先要定义抽象基类：

```
typedef NS_ENUM(NSUInteger, EOCEmployeeType) {  
    EOCEmployeeTypeDeveloper,  
    EOCEmployeeTypeDesigner,  
    EOCEmployeeTypeFinance,  
};  
 
@interface EOCEmployee : NSObject  
 
@property (copy) NSString *name;  
@property NSUInteger salary;  
 
// Helper for creating Employee objects  
+ (EOCEmployee*)employeeWithType:(EOCEmployeeType)type;  
 
// Make Employees do their respective day's work  
- (void)doADaysWork;  
 
@end  
 
@implementation EOCEmployee  
 
+ (EOCEmployee*)employeeWithType:(EOCEmployeeType)type {  
    switch (type) {  
        case EOCEmployeeTypeDeveloper:  
            return [EOCEmployeeDeveloper new];  
            break;  
        case EOCEmployeeTypeDesigner:  
            return [EOCEmployeeDesigner new];  
            break;  
        case EOCEmployeeTypeFinance:  
            return [EOCEmployeeFinance new];  
            break;  
    }  
}  
 
- (void)doADaysWork {  
    // Subclasses implement this.  
}  
 
@end
```

每个“实体子类”（concrete subclass）都从基类继承而来。例如：

```
@interface EOCEmployeeDeveloper : EOCEmployee  
@end  
 
@implementation EOCEmployeeDeveloper  
 
- (void)doADaysWork {  
    [self writeCode];  
}  
 
@end 
```

在本例中，基类实现了一个“类方法”，该方法根据待创建的雇员类别分配好对应的雇员类实例。这种“工厂模式”（Factory pattern）是创建类族的办法之一。

如果对象所属的类位于某个类族中，那么在查询其类型信息（introspection）时就要当心了。你可能觉得自己创建了某个类的实例，然而实际上创建的却是其子类的实例。在 Employee 这个例子中，`[employee isMemberOfClass:[EOCEmployee class]]`似乎会返回 YES，但实际上返回的却是 NO，因为 employee 并非 Employee 类的实例，而是其某个子类的实例。

## Cocoa里的类族

系统框架中有许多类族。大部分 collection 类都是类族，例如 NSArray 与其可变版本 NSMutableArray。

下面条件 if 中的代码永远不会执行：

```
id maybeAnArray = /* ... */;  
if ([maybeAnArray class] == [NSArray class]) {  
        // Will never be hit  
}
```

你要是知道 NSArray 是个类族，那就会明白上述代码错在哪里：其中的 if 语句永远不可能为真。`[maybeAnArray class]`所返回的类绝不可能是 NSArray 类本身，因为由 NSArray 的初始化方法所返回的那个实例其类型是隐藏在类族公共接口（public facade）后面的某个内部类型（internal type）。

我们可以试验下，通过

```
NSArray *arrayA = @[];
NSArray *arrayB = @[@1];
NSArray *arrayC = @[@1,@2];
    
NSLog(@"arrayA.class = %@",[arrayA class]);
NSLog(@"arrayB.class = %@",[arrayB class]);
NSLog(@"arrayC.class = %@",[arrayC class]);
```

得到：

```
arrayA.class = __NSArray0
arrayB.class = __NSSingleObjectArrayI
arrayC.class = __NSArrayI
```

可见得到的类型并不是 NSArray 类，而是其子类。

### 对象所属类的判断

上面 NSArray 例子可以看到，我们创建的对象 arrayA、arrayB 和 arrayC 它们的类并不是 NSArray 类本身。我们在用以下写法时得不到想要的结果：

```
[arrayA class] == [NSArray class] // NO

[arrayA isMemberOfClass:[NSArray class] // NO
```

我们应该改用类型信息查询方法，来判断出某个实例所属的类是否位于类族之中：

```
[arrayA isKindOfClass:[NSArray class]] // YES
```

### 类族中新增实体子类

我们经常需要向类族中新增实体子类，不过这么做的时候得留心。在 Employee 这个例子中，若是没有源代码，那就无法向其中新增雇员类别了。然而对于 Cocoa 中 NSArray 这样的类族来说，还是有办法新增子类的，但是需要遵守几条规则。这几条规则如下：

1、子类应该继承自类族中的抽象基类，如 NSArray 或者 NSMutableArray。

2、子类应该定义自己的数据存储方式。

3、子类应当覆写超类文档中指明需要覆写的方法。

## 总结

- 类族模式可以把实现细节隐藏在一套简单的公共接口后面。

- 系统框架中经常使用类族。

- 从类族的公共抽象基类中继承子类时要当心，若有开发文档，则应首先阅读。
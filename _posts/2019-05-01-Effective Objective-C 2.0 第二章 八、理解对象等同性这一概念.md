---
layout:     post
title:      Effective Objective-C 2.0 第二章 八、理解对象等同性这一概念
subtitle:   理解对象等同性这一概念
date:       2019-05-01
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

比较两个对象相等时，我们可以使用`==`操作符和`isEqual:`两个方法。不过按照`==`操作符比较出来的结果未必使我们想要的，因为该操作比较的是两个指针本身，而不是它指向的对象。

比较两个字符串相等时，我们可以用以下方法：

```
NSString *foo = @"Badger 123";  
NSString *bar = [NSStringstringWithFormat:@"Badger %i", 123];  
BOOL equalA = (foo == bar); // NO 
BOOL equalB = [foo isEqual:bar]; // YES 
BOOL equalC = [foo isEqualToString:bar]; // YES 
```

我们可以看到`==`方法得出的结果不是我们想要的，其中`isEqualToString:`是 NSString 独有的，调用该方法比调用`isEqual:`快，因为该方法知道两个比较对象的类型，而`isEqual:`不知道两个比较对象的类型。类似的还有 NSArray 和 NSDictionary 也有独自等同性判断方法：

```
isEqualToString ：NSString 类独有的等同性判断方法。

isEqualToArray：数组类独有的等同性判断方法。

isEqualToDictionary：字典类独有的等同性判断方法。
```

## isEqual 与 hash

在 NSObject 类中有两个用于判断等同性的关键方法：

```
- (BOOL)isEqual:(id)object;  
+ (NSUInteger)hash; 
```

NSObject 类对这两个方法的默认实现是：当且仅当其“指针值”（pointer value）完全相等时，这两个对象才相等。

如果我们想要覆写这两个方法，得先理解：

>如果`isEqual:`方法判定两个对象相等，那么其 hash 方法也必须返回同一个值。但是，如果两个对象的 hash 方法返回同一个值，那么`isEqual:`方法未必会认为两者相等。

### 覆写 isEqual

比如有下面这个类：

```
@interface EOCPerson : NSObject  
@property (nonatomic, copy) NSString *firstName;  
@property (nonatomic, copy) NSString *lastName;  
@property (nonatomic, assign) NSUInteger age;  
@end 
```

那么判断另一个类是否与这个类相等，需要这么写：

```
- (BOOL)isEqual:(id)object {  
    if (self == object) return YES;  
    if ([self class] != [object class]) return NO;  
 
    EOCPerson *otherPerson = (EOCPerson*)object;  
    if (![_firstName isEqualToString:otherPerson.firstName])  
        return NO;  
    if (![_lastName isEqualToString:otherPerson.lastName])  
        return NO;  
    if (_age != otherPerson.age)  
        return NO;  
    return YES;  
} 
```

其判断过程是：

1、 判断两个指针是否相等，若相等，则均指向同一个对象，所以受测对象必定相等。

2、 比较两对象所属的类。若不属于同一个类，则两对象不相等。

3、最后，检测每个属性是否相等。只要其中有不相等的属性，就判定两对象不等，否则两对象相等。

### 覆写 hash

上面提到，若两个对象相等，则它们的 hash 一定相等；但 hash 相等的两个对象，它们未必相等。即 hash 相等时两个对象相等的必要不充分条件。

如果我们覆写 hash 的话，下面这么写是完全可以的：

```
- (NSUInteger)hash {  
    return 1337;  
} 
```

都返回同一个 hash，但这么写在 collection 中会产生性能问题。因为 collection 在检索哈希表（hash table）时，会用对象的哈希码做索引。假如某个 collection 是用 set 实现的，那么 set 可能会根据哈希码把对象分装到不同的数组中。在向 set 中添加新对象时，要根据其哈希码找到与之相关的那个数组，依次检查其中各个元素，看数组中已有的对象是否和将要添加的新对象相等。如果相等，那就说明要添加的对象已经在 set 里面了。由此可知，如果令每个对象都返回相同的哈希码，那么在 set 中已有 1?000?000 个对象的情况下，若是继续向其中添加对象，则需将这 1?000?000 个对象全部扫描一遍。

hash 还可以这么实现：

```
- (NSUInteger)hash {  
    NSString *stringToHash =  
        [NSStringstringWithFormat:@"%@:%@:%i",  
            _firstName, _lastName, _age];  
    return [stringToHash hash];  
} 
```

但是这样做还需负担创建字符串的开销，所以比返回单一值要慢。把这种对象添加到 collection 中时，也会产生性能问题，因为要想添加，必须先计算其哈希码。

再来看最后一种计算哈希码的办法：

```
- (NSUInteger)hash {  
    NSUInteger firstNameHash = [_firstName hash];  
    NSUInteger lastNameHash = [_lastName hash];  
    NSUInteger ageHash = _age;  
    return firstNameHash ^ lastNameHash ^ ageHash;  
} 
```

这种做法既能保持较高效率，又能使生成的哈希码至少位于一定范围之内，而不会过于频繁地重复。编写 hash 方法时，应该用当前的对象做做实验，以便在减少碰撞频度与降低运算复杂程度之间取舍。

## 等同性判定的执行深度

创建等同性判定方法时，需要决定是根据整个对象来判断等同性，还是仅根据其中几个字段来判断。NSArray 的检测方式为先看两个数组所含对象个数是否相同，若相同，则在每个对应位置的两个对象身上调用其`isEqual:`方法。如果对应位置上的对象均相等，那么这两个数组就相等，这叫做“深度等同性判定”（deep equality）。不过有时候无须将所有数据逐个比较，只根据其中部分数据即可判明二者是否等同。

是否需要在等同性判定方法中检测全部字段取决于受测对象。只有类的编写者才可以确定两个对象实例在何种情况下应判定为相等。

比方说，我们假设EOCPerson类的实例是根据数据库里的数据创建而来，那么其中就可能会含有另外一个属性，此属性是“唯一标识符”（unique identifier），在数据库中用作“主键”（primary key）:

```
@property NSUInteger identifier; 
```

在这种情况下，我们也许只会根据标识符来判断等同性，尤其是在此属性声明为readonly 时更应该如此。因为只要两者标识符相同，就肯定表示同一个对象，因而必然相等。这样的话，无须逐个比较 EOCPerson 对象的每条数据，只要标识符相同，就说明这两个对象就是由同一个数据源所创建的，据此我们能够断定，其余数据也必然相同。

## 小结

- 若想检测对象的等同性，请提供“isEqual: ”与hash方法。

- 相同的对象必须具有相同的哈希码，但是两个哈希码相同的对象却未必相同。

- 不要盲目地逐个检测每条属性，而是应该依照具体需求来制定检测方案。

- 编写hash方法时，应该使用计算速度快而且哈希码碰撞几率低的算法。
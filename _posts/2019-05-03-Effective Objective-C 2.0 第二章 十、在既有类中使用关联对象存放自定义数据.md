---
layout:     post
title:      Effective Objective-C 2.0 第二章 十、在既有类中使用关联对象存放自定义数据
subtitle:   关联对象
date:       2019-05-03
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

我之前已经在这篇文章
[iOS runtime 关联对象](https://www.jianshu.com/p/208d281468c2)做了详细介绍。本篇只是简单介绍下。

#### 创建关联对象：

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```

- id object：对象
- const void *key：键
- id value：值
- objc_AssociationPolicy policy：内存管理策略

其中内存管理策略是一个枚举，如下：

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,
    OBJC_ASSOCIATION_RETAIN = 01401,
    OBJC_ASSOCIATION_COPY = 01403
};
```

#### 获取关联对象：

```
id objc_getAssociatedObject(id object, const void *key)
```

#### 删除关联对象：

```
void objc_removeAssociatedObjects(id object)
```

## 常用用法：给类别增加对象

给 UILable 新增一个类别 UILable(vj):

UILabel+vj.h:

```
#import <UIKit/UIKit.h>

NS_ASSUME_NONNULL_BEGIN

@interface UILabel (vj)

@property(nonatomic,copy)NSString *name;

@end

NS_ASSUME_NONNULL_END
```

UILabel+vj.m

```
#import "UILabel+vj.h"
#import "objc/runtime.h"

const void *kNameKey = @"namekey";

@implementation UILabel (vj)

-(NSString *)name{
   return objc_getAssociatedObject(self, kNameKey);
}

-(void)setName:(NSString *)name{
    objc_setAssociatedObject(self, kNameKey, name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

@end
```

这样我们就为 UILabel (vj) 新增了 name 属性。

一般我们也用不到关联对象这种写法，当我们需要在对象中存储额外信息时，一般继承一个子类，然后改用这个子类，这是最常用的写法。如非必要还是少用关联对象的写法吧，如果是由它引起的 bug 通常难以查找。
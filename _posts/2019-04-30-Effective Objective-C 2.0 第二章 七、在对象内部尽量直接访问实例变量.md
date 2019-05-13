---
layout:     post
title:      Effective Objective-C 2.0 第二章 七、在对象内部尽量直接访问实例变量
subtitle:   在对象内部尽量直接访问实例变量
date:       2019-04-30
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 第七条 在对象内部尽量直接访问实例变量

在对象内部读取数据时，应该直接通过实例变量来读，而写入数据时，则应通过属性来写。

我们来看一个例子,有一个 Person 类，并且重写它 name 属性的 set/get 方法：

```
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Person : NSObject

@property(nonatomic,copy)NSString *name;

-(void)test;

@end

NS_ASSUME_NONNULL_END
```

```
#import "Person.h"

@implementation Person

@synthesize name = _name;

-(NSString *)name{
    NSLog(@"get name");
    return _name;
}

-(void)setName:(NSString *)name{
    NSLog(@"set name");
    _name = [name copy];
}

-(void)test{
    NSLog(@"--通过属访问实例变量：");
    NSLog(@"self.name = %@",self.name);
    
    NSLog(@"--直接访问实例变量：");
    NSLog(@"_name = %@",_name);
}
@end
```

测试代码如下：

```
Person *p = [[Person alloc]init];
p.name = @"Steve";
[p test];
```

输出：

```
set name
--通过属性访问实例变量：
get name
self.name = Steve
--直接访问实例变量：
_name = Steve
```

我们可以看到，通过属性访问实例变量，调用了其 get 方法，从而有一个消息转发的过程；而直接访问实例变量则绕过了这一过程，当然这样会更快。

如果我们给 name 属性写入数据时使用了`_name = @"XXX"`，那么就会绕过其 set 方法，则我们在声明属性 name 时，为其定义的内存管理属性特质 copy 会被忽略掉，这样明显不好。

所以折中的方法是，访问实例变量时直接访问，设置实例变量值时调用属性的 setter 方法。这样既保证效率又保证了内存管理语义得到执行。

使用这种方法书写时，需要注意几个问题：

### 在初始化方法中，设置属性值应该直接访问实例变量。

如果子类重写设置方法时，可能会出现问题。

假设 Person 有一个子类 SmithPerson，该子类重写了 name 方法：

```
#import "SmithPerson.h"

@implementation SmithPerson

-(void)setName:(NSString *)name{
    if(![name hasPrefix:@"S"]){
        [NSException raise:NSInvalidArgumentException format:@"name must has S"];
    }
}

@end
```

测试代码：

```
SmithPerson *p = [[SmithPerson alloc]init];
NSLog(@"p.name = %@",p.name);
```

此时正常运行。

但是某人不小心在父类 Person 的 init 方法中使用属性访问的方式初始化 name：

```
-(instancetype)init{
    self = [super init];
    if (self) {
        self.name = @"";
    }
    return self;
}
```

此时再次运行，程序崩溃。原因是`self.name = @"";`调用了子类中覆写 name 的 setter 方法，但我们 init 方法中，初始化为空字符串不包含“S”，从而抛出异常。解决方法是在父类 init 方法中直接访问实例变量：

```
-(instancetype)init{
    self = [super init];
    if (self) {
        _name = @"";
    }
    return self;
}
```

### 懒加载时，需要通过 get 方法来访问属性

例如 Person 类中有一个 count 属性：

```
@property(nonatomic,strong)NSNumber *count;
```

```
-(NSNumber *)count{
    if (!_count) {
        _count = @0;
    }
    return _count;
}

-(void)test{
    NSLog(@"_count = %@",_count);  //直接访问
    NSLog(@"self.count = %@",self.count);  //通过 get 方法访问
}
```

测试代码：

```
Person *p = [[Person alloc]init];
[p test];
```

输出：

```
_count = (null)
self.count = 0
```

可见如果直接访问的话我们拿到的 count 属性永远是 null。所以如果使用懒加载的话必须使用 get 方法访问。

## 小结

- 在对象内部读取数据时，应该直接通过实例变量来读，而写入数据时，则应通过属性来写。
- 在初始化方法及 dealloc 方法中，应该直接通过实例变量来读写数据。
- 懒加载时，通过属性来读取数据。

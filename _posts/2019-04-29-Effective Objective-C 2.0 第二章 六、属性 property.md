---
layout:     post
title:      Effective Objective-C 2.0 第二章 六、属性 property
subtitle:   对象 消息 运行期
date:       2019-04-29
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 第六条 property

property 是 Objective-C 的一项特性，用于封装对象中的数据。

当我们声明一个属性时：

```
@property(nonatomic,copy)NSString *name;
```

编译器自动为我们生成了如下不可见代码：

```
@synthesize name = _name; //添加实例变量
-(NSString *)name;  // get 方法
-(void)setName:(NSString *)name  //set 方法
```

也就说 @property = 实例变量 + get 方法 + set 方法。

如果我们不想让编译器为我们自动生成，可以使用 @dynamic 代替 @synthesize，同时自己要手动实现相对应的 set 与 get 方法。

### property 特质（attribute）

属性特质有四类：原子性、读写权限、内存管理、方法名。

默认情况下，即我们不指定特质，那么基本数据类型的取值是：

```
atomic,readwrite,assign
```

如果是 Objective-C 对象，默认取值是：

```
atomic,readwrite,strong
```

#### 1、原子性

在默认情况下，由编译器合成的方法会通过锁定机制确保其原子性（atomicity），即在上面例子中如果我们不写`nonatomic`，则默认为`atomic`。

atomic 与 nonatomicd 的区别？

它们的主要区别就是系统自动生成的 getter/setter 方法不一样

- atomic 系统自动生成的 getter/setter 方法会进行加锁操作；
- nonatomic 系统自动生成的 getter/setter 方法不会进行加锁操作；

atomic 只是保证了 getter 和 setter 存取方法的线程安全,并不能保证整个对象是线程安全的,因此在多线程编程时,线程安全还需要开发者自己来处理.

在我们开发 iOS 程序中，我们属性都声明为 nonatomic，这样做的原因是：在 iOS 中使用同步锁开销较大，这会带来性能问题，一般情况下并不要求属性必须是 atomic 的，因为这并不能保证线程安全。若要实现线程安全，需要我们手动采用更为深层的锁定机制才行。例如，一个线程在连续多次读取某属性值的过程中有别的线程在同时改写该值，那么即便将属性声明为 atomic，也还是会读到不同的属性值。因此，开发 iOS 程序时一般都会使用 nonatomic 属性。但是在开发 Mac 时，使用 atomic 属性通常不会有性能瓶颈。

#### 2、读写权限

读写权限有两个取值，readwrite 和 readonly。声明属性时，如果不指定读写权限，那么默认是 readwrite 的。如果某个属性不想让其他人来写，那么可以设置成 readonly。

#### 3、内存管理

内存管理的取值有 assign、strong、weak、copy、unsafe_unretained。

- assign：通常用来修饰基本数据类型和结构体，如 int、float、BOOL、CGFloat 等。
- strong：强引用，定义了一种“拥有关系”，用来修饰 OC 对象。为这种属性设置新值时，设置方法会先保留新值，并释放旧值，然后再将新值设置上去。
- weak：弱引用，定义了一种“非拥有关系”,用来修饰 OC 对象。为这种属性设置新值时，设置方法既不保留新值，也不释放就值。此特质同 assign 类似，然而属性所指向的对象遭到摧毁时，属性值会置为 nil。
- unsafe_unretained：此特质语义同 assign 相同，不过是用来修饰 OC 对象。该特质表达一种“非拥有关系”。当目标对象遭到摧毁时，属性值不会自动清空，这一点与 weak 有区别。"unsafe" 从这个字段也可以看出，使用它有风险，目标对象释放后，该属性会变成野指针，再次使用该属性，非常容易崩溃。
- copy：此特质所表达的所属关系与 strong 类似，然而设置方法并不保留新值，而是将其拷贝。

#### 4、方法名

set、get 方法名。如果不想使用自动合成所生成的 setter、getter 方法，声明属性时甚至可以指定方法名。比如指定 getter 方法名：

```
@property (nonatomic, assign, getter=isPass) BOOL pass;
```

#### copy strong 区别

通常情况下，不可变对象使用 copy，可变对象使用 strong。如果使用错误的话，我们看下会发生什么情况：

##### a. 不可变对象使用了 strong

```
@property (nonatomic, strong) NSString *strongStr;
```

使用子类初始化父类：

```
NSMutableString *mutStr = [NSMutableString stringWithFormat:@"123"];
self.strongStr = mutStr; //子类初始化父类
NSLog(@"str = %@",self.strongStr);

[mutStr appendString:@"456"];
NSLog(@"str = %@",self.strongStr);
```

控制台输出：

```
self.strongStr = 123
self.strongStr = 123456
```

首先明确一点，既然类型是 NSString，那么则代表我们不希望 strongStr 被改变，否则直接使用可变对象 NSMutableString 就可以了。
我们可以看到在开发者没有感知的情况下 strongStr 意外发生了改变，然而这是我们不想看到的结果。

##### b. 可变对象使用了 copy

```
@property (nonatomic, copy) NSMutableString *mutString;
```

```
NSString *str = @"123";
self.mutString = [NSMutableString stringWithString:str];
NSLog(@"str = %p self.mutString = %p",str,self.mutString); // 两者的地址不一样
[self.mutString appendString:@"456"]; // 会崩溃，因为此时self.mutArray是NSString类型，是不可变对象
```

执行程序后,会崩溃，即 self.mutString 没有 appendString 方法。self.mutString 是 NSMutableString 类型，为何没有 appendString 方法呢？这就是使用 copy 造成的。看一下：

```
self.mutString = [NSMutableString stringWithString:str];
```

这行代码到底发生了什么。这行代码实际上完成了两件事：

```
// 首先声明一个临时变量
NSMutableString *tempString = [NSMutableString stringWithString:str];
// 将该临时变量copy，赋值给self.mutString
self.mutString = [tempString copy];
```

注意，通过 [tempString copy] 得到的 self.mutString 是一个不可变对象，不可变对象自然没有 appendString 方法，这也是为何会崩溃的原因。至于为什么是不可变，看完下面分析就清楚了。

##### copy 和 mutableCopy 区别

主要就是深拷贝与浅拷贝的区别。所谓浅拷贝，在 Objective-C 中可以理解为引用计数加1，并没有申请新的内存区域，只是另外一个指针指向了该区域。深拷贝正好相反，深拷贝会申请新的内存区域，原内存区域的引用计数不变。

可变对象的 copy 和 mutableCopy 都是深拷贝。

不可变对象的 copy 是浅拷贝，mutableCopy 是深拷贝。




## 参考资料

Effective Objective-C 2.0

[https://juejin.im/post/5c105c7ce51d4562d138086f](https://juejin.im/post/5c105c7ce51d4562d138086f)



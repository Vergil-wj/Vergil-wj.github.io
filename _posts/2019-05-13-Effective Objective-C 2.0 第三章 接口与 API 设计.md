---
layout:     post
title:      Effective Objective-C 2.0 第三章 接口与 API 设计
subtitle:   借口与 API
date:       2019-05-13
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - Effective Objective-C 2.0
---

## 第 15 条：用前缀避免命名空间冲突

1、选择与你的公司、应用程序或二者皆有关联之名称作为类名的前缀，并在所有代码中使用这一前缀。

Apple 宣称其保留使用所有“两字母前缀”的权利，所以我们使用前缀时候最好是三个字母的。举个例子，假如开发者不遵守这条守则，使用 TW 这两个字母作为前缀，那么就会出现问题。iOS 5.0 SDK 发布时，包含 Twitter 框架，此框架就使用 TW 作前缀，其中有个类叫做 TWRequest，它可以发送 HTTP 请求以调用 Twitter API。如果你所在的公司叫做 Tiny Widgets，那么很有那可能把访问本公司 API 所用的那个类也命名为 TWRequest。

2、若自己所开发的程序中用到了第三方库，并准备将其发布为程序库供他人开发应用，则应为其中的名称加上前缀。

## 第 16 条：提供“全能初始化方法”

“designated initializer” 直译为 “指定初始化方法”，本书译为"全能初始化方法"。

就是提供必要信息的初始化方法。如果创建类实例的方式不止一种，那么这个类就会有多个初始化方法，我们需要选定一个作为全能初始化方法，令其它初始化方法都来调用他。

```
@interface YXRectangle : NSObject
@property (nonatomic,readonly) float width;
@property (nonatomic,readonly) float height;
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
@end
@implementation YXRectangle

- (instancetype)init{
    return [self initWithWidth:500 height:600];
}

- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height{
    self = [super init];
    if (self) {
        _width = width;
        _height = height;
    }
    return self;
}
@end
```

`-(instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height`这个方法就是全能初始化方法。这个类的所有初始化方法最终都会通过全能初始化方法创建。

### 子类的初始化方式：

比如我们创建一个正方形的子类 YXSquare

```
@interface YXSquare : YXRectangle
- (instancetype)initWithDimension:(CGFloat)dimension;
@end
@implementation YXSquare

- (instancetype)initWithDimension:(CGFloat)dimension{
   return  [super initWithWidth:dimension height:dimension];
}

//要阻止子类直接调用父类的全能初始化方式，这样有可能出现长宽不等的情况。
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height{
    @throw  [NSException exceptionWithName:NSInternalInconsistencyException reason:@"Must be use initWithDimension :instead" userInfo:nil];
}

//如果不重写的话，使用者可能会使用 init 创建实例，这样会调用父类的 init 方法，可能会出现长宽不等的情况，所以需要重写。
-(instancetype)init{
    return [self initWithDimension:500];
}
@end
```

### 存在两个全能初始化方法

对象实例有两种完全不同的创建方式，必须分开处理，就会出现这种情况。

```
- (id)initWithCoder:(NSCoder *)decoder;
```

我们在实现此方法时一般不调用平常所使用的那个全能初始化方法，因为该方法要通过"解码器"(decoder)将对象数据解压缩，所以和普通的初始化方法不同。因此，严格的说，在这种情况下出现了两个全能初始化方法。

具体到父类 YXRectangle 代码就是：

```
@interface YXRectangle : NSObject
@property (nonatomic,readonly) float width;
@property (nonatomic,readonly) float height;
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
@end
@implementation YXRectangle

- (instancetype)init{
    return [self initWithWidth:500 height:600];
}

- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height{
    self = [super init];
    if (self) {
        _width = width;
        _height = height;
    }
    return self;
}

-(id)initWithCoder:(NSCoder *)decoder{
	if(self = [super init]){
		_width = [decoder decodeFloatForKey:@"width"];
		_height = [decoder decodeFloatForKey:@"height"];
	}
	return self;
}

@end
```

子类 YXSquare：

```
@interface YXSquare : YXRectangle
- (instancetype)initWithDimension:(CGFloat)dimension;
@end
@implementation YXSquare

- (instancetype)initWithDimension:(CGFloat)dimension{
   return  [super initWithWidth:dimension height:dimension];
}

//要阻止子类直接调用父类的全能初始化方式，这样有可能出现长宽不等的情况。
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height{
    @throw  [NSException exceptionWithName:NSInternalInconsistencyException reason:@"Must be use initWithDimension :instead" userInfo:nil];
}

//如果不重写的话，使用者可能会使用 init 创建实例，这样会调用父类的 init 方法，可能会出现长宽不等的情况，所以需要重写。
-(instancetype)init{
    return [self initWithDimension:500];
}

-(id)initWithCoder:(NSCoder *)decoder{
	if((self = [super initWithCoder:decoder])){
		//YXSquare's specific initializer
	}
	return self;
}

@end
```

每个子类的全能初始化方法都应该调用其超类的对应方法，并逐层向上，实现`initWithCoder:`时也要这样，应该先调用超类的相关方法，然后再执行与本类有关的任务。这样编写出来的 YXSquare 类就完全遵守 NSCoding 协议了(fully NSCoding compliant)。如果编写`initWithCoder:`方法时没有调用超类的同名方法，而是调用了自制的初始化方法，或是超类的其他初始化方法，那么 YXSquare 类的`initWithCoder:`方法就没机会执行，于是，也就无法将 \_width 及 \_height 这两个实例变量解码了。

### 要点

- 在类中提供一个全能初始化方法，并于文档里指明。其他初始化方法均应调用此方法。
- 若全能初始化方法与超类不同，则需覆写超类中的对应方法。
- 如果超类的初始化方法不适用于子类，那么应该覆写这个超类方法，并在其中抛出异常。

## 第 17 条 实现 description 方法

我们自定义一个类 TestObject：

TestObject.h

```
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface TestObject : NSObject

@property(nonatomic,copy)NSString *name;

-(instancetype)initWithName:(NSString *)name;

@end

NS_ASSUME_NONNULL_END
```

TestObject.m

```
#import "TestObject.h"

@implementation TestObject

-(instancetype)initWithName:(NSString *)name{
    if (self = [super init]) {
        _name = [name copy];
    }
    return self;
}

@end
```

我们打印查看对象信息时：

```
TestObject *obj = [[TestObject alloc]initWithName:@"Bob"];
NSLog(@"obj = %@",obj);
```

输出：

```
obj = <TestObject: 0x283880010>
```

我们只能看到类名和对象地址，得不到我们想要的信息。这时候想要看到更详细的信息，需要重写`-(NSString *)description`方法。在 TestObject.m 中，改为：

```
#import "TestObject.h"

@implementation TestObject

-(instancetype)initWithName:(NSString *)name{
    if (self = [super init]) {
        _name = [name copy];
    }
    return self;
}

-(NSString *)description{
    return [NSString stringWithFormat:@"<%@: %p, %@>",
            [self class],
            self,
            @{@"name":_name}];
}

@end

```

此时再次运行，控制台输出：

```
obj = <TestObject: 0x2818ec610, {
    name = Bob;
}>
```

这样在打印自定义类时，就能得到我们想要的信息。

还有与`-(NSString *)description`方法功能相同的另一个方法`-(NSString *)debugDescription`,两者的区别就是后者是开发者在调试器中以控制台命令打印对象才调用的。在 NSObject 类默认实现中，debugDescription 直接调用了 description。我们可以重写 debugDescription 方法，以便在调试时打印出更详尽的信息。例如：

```
-(NSString *)description{
    return [NSString stringWithFormat:@"<%@>",
            @{@"name":_name}];
}

-(NSString *)debugDescription{
    return [NSString stringWithFormat:@"<%@: %p, %@>",
            [self class],
            self,
            @{@"name":_name}];
```

在 description 中，我们不需要看到类名与指针地址，而在调试时候想要看到更详尽的，就可以这么写。

测试：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    TestObject *obj = [[TestObject alloc]initWithName:@"Bob"];
    NSLog(@"obj = %@",obj);
    //此处打一个断点
}
```

打印：

```
obj = <{
    name = Bob;
}>
(lldb) po obj //此处使用 LLDB 的 “po” 命令可以完成对象打印
<TestObject: 0x2816ec010, {
    name = Bob;
}>
```

### 要点

- 实现 description 方法返回一个有意义的字符串，用以描述该实例。
- 若想在调试时打印出更详尽的对象描述信息，则应实现 debugDescription 方法。

## 第 18 条 尽量使用不可变对象

就是声明属性为 readonly。这样可以防止外人改动。

有时我们需要修改封装在对象内部的数据，可以在类扩展中将对象重新声明为 readwrite。例如：

EOCPointOfInterest.h:

```
#import <Foundation/Foundation.h>
 
@interface EOCPointOfInterest : NSObject
 
@property (nonatomic, copy, readonly) NSString *identifier;
 
@end

```

EOCPointOfInterest.m

```
#import "EOCPointOfInterest.h"
 
@interface EOCPointOfInterest()
 
@property (nonatomic, copy, readwrite) NSString *identifier;
 
@end
 
@implementation EOCPointOfInterest
 
@end
```

对象表示的各种集合属性，应该提供一个readonly属性供外界使用，该属性将返回不可变的集合，而该集合是内部那个可变集合的一份拷贝。即不要把可变的集合作为属性公开，而应该提供相关方法，以此修改对象中的可变集合。

```
// 头文件：
#import <Foundation/Foundation.h>
 
@interface EOCPerson : NSObject
 
@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;
@property (nonatomic, copy, readonly) NSSet *friends;
 
- (id)initWithFirstName:(NSString *)firstName andLastName:(NSString *)lastName;
- (void)addFriend:(EOCPerson *)person;
- (void)removeFriend:(EOCPerson *)person;
 
@end
 
// 实现文件：
#import "EOCPerson.h"
 
@interface EOCPerson()
 
@property (nonatomic, copy, readwrite) NSString *firstName;
@property (nonatomic, copy, readwrite) NSString *lastName;
 
@end
 
@implementation EOCPerson {
    NSMutableSet *_internalFriends;
}
 
- (NSSet *)friends
{
    return [_internalFriends copy];
}
 
 - (void)addFriend:(EOCPerson *)person
{
    [_internalFriends addObject:person];
}
 
- (void)removeFriend:(EOCPerson *)person
{
    [_internalFriends removeObject:person];
}
 
- (id)initWithFirstName:(NSString *)firstName andLastName:(NSString *)lastName
{
    if (self = [super init]) {
        _firstName = firstName;
        _lastName = lastName;
        _internalFriends = [NSMutableSet new];
    }
    return self;
}
 
@end
```

### 要点

- 尽量创建不可变的对象
- 若某属性仅可于对象内部修改，则在 “class-continuation 分类”中将其由 readonly 属性扩展为 readwrite 属性。
- 不要把可变的 collection 作为属性公开，而应提供相关方法，以此修改对象中可变的 collection。

## 第 19 条：使用清晰而协调的命名方式

有种代码即文档的感觉，读起来就像日常用语的句子。不要用缩略后的名称来命名，例如用 str 替代 string。

给方法命名时可以总结成下面几条规则：

- 如果方法的返回值时新创建的，那么方法名的首个词应是返回值的类型。例如`+stringWithString`。除非前面还有个修饰词例如`localizedString`,用来描述其逻辑含义。
- 应该吧表示参数类型的名词放在参数前面。例如`-(instancetype)initWithName:(NSString *)name`。
- 如果方法要在当前对象上执行操作，那么就应该包含动词；
- 不要使用 str 这种简称，应该使用 string 这样的全称。
- Bool 属性应加 is/has 前缀。
- 将get 这个前缀留给那些借由“输出参数”来保存返回值的方法。比如，把返回值填充到“C语言式数组”（C-style array）里的那种方法就可以使用这个词做前缀。例：NSString类里的`-(void)getCharacters:(unichar*)buffer range:(NSRange)aRange`。

## 第 20 条：为私有方法名加前缀

- 给私有方法的名称加上前缀，这样可以很容易地将其同公共方法分开。
- 不要单用一个下划线做私有方法的前缀，因为这种做法是预留给苹果公司用的。

## 第 21 条 理解 Objective-C 错误模型

### 1、极其严重的错误使用异常

例如某人使用了我们自己编写的某个抽象基类，可以考虑抛出异常。

```
- (void)mustOverrideMethod {
    @throw [NSException 
        exceptionWithName:NSInternalInconsistencyException
        reason:[NSString stringWithFormat:@"%@ must be overridden", _cmd]
        userInfo:nil];

```

### 2、一般错误可以令方法返回 nil 或使用 NSError

#### 使用 nil

```
- (id)initWithValue:(id)value {
    if ((self = [super init])) {
        if ( /* Value means instance can’t be created */ ) {
            self = nil;
        } else {
            // Initialise instance
        }
    }
    return self;
}
```

在这种情况下，如果 if 语句发现无法用传入的参数值来初始化当前实例（比如这个方法要求传入的 value 参数必须是 non-nil 的），那么就把 self 设置成 nil，这样的话，整个方法的返回值也就是 nil 了。调用者发现初始化方法并没有把实例创建好，于是便可确定其中发生了错误。

#### 使用 NSError

```
- (BOOL)doSomethingError:(NSError**)error {
    // Do something that may cause an error
    
    if (/* there was an error */) {
        if (error) {
            *error = [NSError errorWithDomain:NSURLErrorDomain
                                         code:110
                                     userInfo:nil];
        }
        return NO;///< Indicate failure
    } else {
        return YES; ///< Indicate success
    }
}
```

调用这段代码：

```
NSError *error = nil;
BOOL ret = [object doSomethingError:&error];
if (!ret) {
    // There was an error
    NSLog(@"err = %@",error);
}
```
可以获取到错误信息。

### 要点：

- 只有发生了可使整个应用程序崩溃的严重错误时，才使用异常。
- 在错误不那么严重的情况下，可以指派“委托方法”来处理错误，也可以把错误信息放在 NSError 对象里，经由“输出参数”返回给调用者。

## 第 22 条 理解 NSCopying 协议

NSCopying 协议下只有这一个方法：

```
- (id)copyWithZone:(nullable NSZone *)zone;
```

其中参数 zone 已经不再使用，我们无需关心。

当我们自定义对象想要具有拷贝功能时，我们就需要实现此协议。

例如：

TestObject.h

```
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface TestObject : NSObject<NSCopying>

@property(nonatomic,copy)NSString *name;

-(instancetype)initWithName:(NSString *)name;

@end

NS_ASSUME_NONNULL_END
```

TestObject.m

```
#import "TestObject.h"

@implementation TestObject

-(instancetype)initWithName:(NSString *)name{
    if (self = [super init]) {
        _name = name;
    }
    return self;
}

-(id)copyWithZone:(NSZone *)zone{
    
    TestObject *obj = [[[self class]allocWithZone:zone]initWithName:_name];
    return obj;
}

@end
```

当然了，这样实现的是浅拷贝。我们若想实现深拷贝，需要实现 NSMutableCopying 协议，其协议下也只有一个方法：

```
- (id)mutableCopyWithZone:(nullable NSZone *)zone;
```

其中参数 zone 没有用，不用考虑。返回一个可变对象就可以了。

### 要点：

- 若想令自己所写的对象具有拷贝功能，则需实现 NSCopying 协议。
- 如果自定义的对象分为可变版本和不可变版本，那么就要同时实现 NSCopying 与 NSMutableCopying 协议。
- 复制对象时需决定采用浅拷贝还是深拷贝，一般情况下应该尽量执行浅拷贝。
- 如果你写的对象需要深拷贝，那么可考虑新增一个专门执行深拷贝的方法。



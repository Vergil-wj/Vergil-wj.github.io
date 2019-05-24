---
layout:     post
title:      iOS NSCache 缓存
subtitle:   缓存
date:       2019-05-24
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
---

NSCache 是一个可变集合，即缓存。它存储key-value对，这一点类似于NSDictionary类。

- NSCache 在系统内存很低时，会自动释放一些对象

- NSCache 是线程安全的，在多线程操作中，不需要对 Cache 加锁

- NSCache 的 Key 只是做强引用，不需要实现 NSCopying 协议

### 属性

```
//缓存名
- @property (copy) NSString *name;

//缓存所能容纳最大总成本。
//默认为 0，没有限制。
//但需要注意的是，这不是一个严格的限制。如果缓存的数量超过这个数量，缓存中的一个对象可能会被立即丢弃、或者稍后、也可能永远不会，具体依赖于缓存的实现细节。
- @property NSUInteger totalCostLimit;	

//缓存对象最大数量。
//默认为 0，没有限制。
//被丢弃的对象的顺序无法保证，同totalCostLimit 限制策略一样。
- @property NSUInteger countLimit;	

//标示缓存是否回收废弃的内容。
//默认值是 YES，表示自动回收。
- @property BOOL evictsObjectsWithDiscardedContent;
```

### 方法

```
//返回键值关联对象
- (nullable ObjectType)objectForKey:(KeyType)key;

//在缓存个中设置指定键名对应的值
//与可变字典不同，缓存对象不会对键名做 copy 操作
- (void)setObject:(ObjectType)obj forKey:(KeyType)key; // 0 cost

//在缓存中设置指定键名对应的值，并且指定该键值对的成本
//成本 (cost) 用于计算记录在缓冲中的所有对象的总成本，成本可以自行指定
- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;

//移除指定键关联的对象
- (void)removeObjectForKey:(KeyType)key;

//移除所有缓存对象
- (void)removeAllObjects;
```

### 代理

```
@property(assign) id< NSCacheDelegate > delegate

//缓存将要删除对象时调用
//不能在此方法中修改缓存
- (void)cache:(NSCache *)cache willEvictObject:(id)obj;
```

### 成本的理解

在上述提到的属性和方法中：

```
- @property NSUInteger totalCostLimit;	

- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;
```

这两个是搭配使用的，其中`totalCostLimit`控制最大成本，限制缓存大小。
`setObject: forKey: cost:`其中 cost 表示当前对象的大小。

比如，我们希望用我们的缓存来存储图片，我们限制图片缓存的总成本为 10M。

那么我们我们可以这么设置：

```
_cache.totalCostLimit = 10; //假设我们的单位为 M
```

当我们添加图片到缓存中的时候，可以如下添加

```
[_cache setObject:image forKey:key cost:5];//假设我们的图片大小 5M
```

那么我们添加第一、二个图片的时候，就会存入到图片缓存中（假设图片大小都是5M）。现在我们来添加第三张图片，发现我们的缓存已经到了最大值，我们无法添加，
那么 NSCache,就会把第一张图片从缓存中删除，然后来检查是空间是否已经足够添加第三张图片了，如果不够，那么继续删除第二张，如果够了，那么把第三张图片添加到缓存中。

这是我们假设的一个例子，实际开发中我们并不知道图片的大小。通常，精确的 cost 应该是对象占用的字节数。如果它不可以直接读出来的话，你没必要费劲地去计算它，因为这么做的话会增加使用缓存的代价，导致性能变差。

所以

> 如果你没有有效的值传入，那就传入 0，或者用 setObject:forKey: 方法，它不需要传入 cost 值。

### 代码示例

声明 NSCache 变量：

```
@property (nonatomic, strong) NSCache *cache;
```

懒加载：

```
- (NSCache *)cache {
    if (_cache == nil) {
        _cache = [[NSCache alloc] init];
        // 设置数量限制,最大限制为3
        _cache.countLimit = 3;
        _cache.delegate = self;
    }
    return _cache;
}
```

测试方法：

```
//添加缓存
-(void)addCache{
    for (int i = 0 ; i < 5 ; i++) {
        NSString *str = [NSString stringWithFormat:@"在这里进行了存储数据：%d",i];
        [self.cache setObject:str forKey:@(i)];
        NSLog(@"存储数据----%@",@(i));
    }
}

// 检查缓存
- (void)checkCache {
    NSLog(@"---------------------------------------------");
    for (int i = 0; i < 5 ; i++) {
        NSString *str = [self.cache objectForKey:@(i)];
        if (str) {
            NSLog(@"取出缓存中存储的数据-----%@",@(i));
        }
    }
}

#pragma mark - NSCacheDelegate
// 即将回收对象的时候进行调用，实现代理方法之前要遵守NSCacheDelegate协议。
- (void)cache:(NSCache *)cache willEvictObject:(id)obj{
    NSLog(@"回收--------%@",obj);
}
```

运行，点击添加缓存按钮：

```
存储数据----0
存储数据----1
存储数据----2
回收--------在这里进行了存储数据：0
存储数据----3
回收--------在这里进行了存储数据：1
存储数据----4
```

因为我们设置 _cache.countLimit = 3，所以在存储第四条数据时，先释放了第一条数据，在添加第四条数据，第五条数据也是如此。

点击查看缓存按钮验证下：

```
取出缓存中存储的数据-----2
取出缓存中存储的数据-----3
取出缓存中存储的数据-----4
```

缓存中只有三条数据，与我们设置的 countLimi 为 3 一致。




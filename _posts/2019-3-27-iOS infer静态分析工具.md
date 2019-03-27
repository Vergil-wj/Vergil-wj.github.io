---
layout:     post
title:      iOS infer静态分析工具使用
subtitle:   Infer可以对 C、Java 和 Objective-C 代码进行静态分析.
date:       2019-03-27
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - 静态分析
---

## infer介绍

[infer官网](https://fbinfer.com/docs/getting-started.html)

[infer中文网站](https://infer.liaohuqiu.net/)

Infer 是 Facebook 开源的、使用 OCaml 语言编写的静态分析工具，可以对C、Java 和 Objective-C 代码进行静态分析，可以检查出空指针访问、资源泄露以及内存泄露。

## infer安装

mac 终端下输入：

`brew install infer`

infer 就安装好了。

## infer使用

我们可以先在 test 目录下 ViewController.m 写一段 Objective-C 代码：

```
#import "ViewController.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSString *str = @"1";
}

@end
```
首先用cd命令进入test目录，然后运行以下命令进行编译:

`infer -- xcodebuild -target test -configuration Debug -sdk iphonesimulator`

注意其中 test 是自己的项目名，编译结果:

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1h9vhh6b6j30bg04qgm0.jpg)

在项目所在目录下多出build和infer-out文件夹.

build文件夹：捕获阶段: Infer 捕获编译命令(上面介绍的编译器命令)，将文件翻译成 Infer 内部的中间语言。运行环境和设备信息也有所体现。

infer-out文件夹：分析阶段产生的文件，Infer将分析bugs结果输出到不同格式文件中，如csv、txt、json 方便对分析结果进行加工分析。

运行后在终端会看到日志信息(同infer-out文件，可以以多种形式查看log信息):

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1ha1vrykxj30ra0awgp7.jpg)

可以看出，我们的 ViewController.m 代码里有1个问题:

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1hae6uc86j30og022q3y.jpg)

将代码修改如下:

```
NSString *str = @"1";
NSLog(@"str = %@",str);
```

先手动删除build文件夹(注意事项第2点),再次执行上述命令:

`infer -- xcodebuild -target test -configuration Debug -sdk iphonesimulator`

结果:

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1hagvf3xmj30vq08w41g.jpg)

问题已经消除，以上就是infer用法。

## 注意事项

1、输入命令：

`infer -- xcodebuild -target test -configuration Debug -sdk iphonesimulator`

出现error:

```
xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance
```

这种情况是这种情况是xcodebuild的路径不正确。将路径切换到Xcode的目录下,如下:

`sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer/`

2、在俩次执行编译命令的过程中，发现在没有对代码做任何更改的时候(bug还在)，报出BUILD SUCCEEDED的提示：

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1hbpckqe1j30la05mdhi.jpg)

根据提示可以看到，此次build并没有分析任何文件。原因涉及到增量分析：

#### 增量模式和非增量模式

>在第一次运行的时候，两种模式是一样的，都会对工程的所有文件进行编译检查，产生检查结果:

>增量模式：当已经产生分析结果后（build和infer-out文件夹），再执行编译命令，即为增量模式。如有代码没有改动，则此次不会有编译结果产生，如果代码有新的改动，此次只产生新的编译结果。这种以增量为基准的原则叫做增量模式。

>非增量模式：在删除了俩个文件夹的情况下，运行文件，会输出所有的编译信息，即此时处于非增量模式。

##### 增量模式转化为非增量模式

第一种 直接删除文件夹(build和infer-out文件夹)

第二种 输入以下命令

`xcodebuild -target HelloWorldApp -configuration Debug -sdk iphonesimulator clean`

即在原来命令末尾加上 clean 。














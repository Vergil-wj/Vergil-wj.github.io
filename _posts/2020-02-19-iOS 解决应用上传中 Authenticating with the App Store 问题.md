---
layout:     post
title:      iOS 解决应用上传中 Authenticating with the App Store 问题
subtitle:   app store
date:       2020-2-19
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - app store
---

2020-2-19，今天在上传应用的时总是卡住提示 “**Authenticating with the App Store**” ，尝试了多次还是不行，使用其他方式  Transporter 提交，同样卡住不动，提示 “**正在验证 APP - 正在通过App Store进行认证...**”。

## 原因

1、在上传 ipa 文件时需要使用 java 程序的 iTMSTransporter 处理。

2、在第一次上传应用时，iTMSTransporter 需要从 Internet 下载一组 jar 文件并将其缓存在本地文件夹中。我们遇到的问题就是卡在了这一步，下载不下来！

## 解决方法

XCode11 及以上解决方法：

iTMSTransporter 文件位置：

```
XCode 中位置：
/Applications/Xcode.app/Contents/SharedFrameworks/ContentDeliveryServices.framework/itms/bin/iTMSTransporter

或者在 Transporter 中位置：
/Applications/Transporter.app/Contents/itms/bin/iTMSTransporter
```

缓存文件位置：

```
/Users/你的电脑用户名/Library/Caches/com.apple.amp.itmstransporter/
```

我们可以先看下缓存文件大小，也就 3M 左右，而实际全部缓存完成后多达 70M 左右。

接下来直接删除缓存文件 com.apple.amp.itmstransporter，在 Terminal 运行 iTMSTransporter，它会重新生成缓存文件:

输入命令

```
/Applications/Xcode.app/Contents/SharedFrameworks/ContentDeliveryServices.framework/itms/bin/iTMSTransporter
```

如果安装了 Transporter.app 也可以使用此命令：

```
/Applications/Transporter.app/Contents/itms/bin/iTMSTransporter
```

这个过程就是下载一堆缓存文件的过程，完成后就可以正常上传 app 了。

## 其它问题

网络好的同学上面步骤一气呵成，网络不好的同学继续往下看：

#### 1、在下载文件过程中命令行可能会报错：

> INFO: An error occurred downloading: https://contentdelivery.itunes.apple.com/transporter/repositories/j2se8/2.0.0/bundles/org.xerial.sqlite-jdbc-3.27.2.1.jar

jar 包没有下载下来，可以将这个链接复制到浏览器，通过浏览器下载下来，并手动放到`/Users/你的电脑用户名/Library/Caches/com.apple.amp.itmstransporter/obr/2.2.0/
`目录下。再次重新运行刚才的命令。

#### 2、终端始终不见完成。

打开缓存文件`
/Users/你的电脑用户名/Library/Caches/com.apple.amp.itmstransporter/
`,看下缓存文件的大小，差不多快 60M 了，强行关闭终端，使用 Transporter.app 尝试提交 app，成功！此时缓存文件大小 69.8M。

#### 3、关于 Application Loader

在 XCode11 中，Apple 移除了 Application Loader，并提供了新的 Transporter 代替 Application Loader。所以网上很多关于此问题的解决方法都是针对 XCode11 以下的。例如：

```
1. cd ~ 
2. mv .itmstransporter/ .old_itmstransporter/
3. /Applications/Xcode.app/Contents/Applications/Application Loader.app/Contents/itms/bin/iTMSTransporter
```

解决方法都是一样的，只是 iTMSTransporter 的文件位置不同。

## 参考资料

[https://stackoverflow.com/questions/22443425/application-loader-stuck-at-authenticating-with-the-itunes-store-when-uploadin/59261475#59261475](https://stackoverflow.com/questions/22443425/application-loader-stuck-at-authenticating-with-the-itunes-store-when-uploadin/59261475#59261475)
---
layout:     post
title:      iOS 13 NSBluetoothAlwaysUsageDescription
subtitle:   App 启动
date:       2019-09-18
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS 
    - iOS13
---

项目中用到了蓝牙，今天提交时报被拒了，问题如下：

> ITMS-90683: Missing Purpose String in Info.plist - Your app's code references one or more APIs that access sensitive user data. The app's Info.plist file should contain a NSBluetoothAlwaysUsageDescription key with a user-facing purpose string explaining clearly and completely why your app needs the data. 


原因是 iOS 13 之后新增 Info.plist 的 key NSBluetoothAlwaysUsageDescription

```
//name
Privacy - Bluetooth Always Usage Description

//type
String
```

iOS 13 之前：

```
//Name
Privacy - Bluetooth Peripheral Usage Description
//Type
String
```
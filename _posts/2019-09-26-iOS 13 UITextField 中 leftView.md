---
layout:     post
title:      iOS 13 UITextField 中 leftView
subtitle:   iOS 13 适配
date:       2019-09-25
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS 
    - iOS13
---

iOS 13 之后，发现我的 imageView 在 textfield.leftView 中宽高被 sizeToFit 了，imageView 中设置的宽高无效了。

![屏幕快照 2019-09-26 10.17.59.png](https://upload-images.jianshu.io/upload_images/1776587-963f092a3fa98d61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


解决办法：在 imageView 上嵌套一个 UIView，使 `textfield.leftView = uiview`;
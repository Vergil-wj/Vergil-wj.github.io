---
layout:     post
title:      iOS 去除 UITextView 内边距
subtitle:   UITextView
date:       2020-1-6
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
---

去除 textView 左右边距：

```
self.textView.textContainer.lineFragmentPadding = 0;
```

去除 textView 上下边距：

```
self.textView.textContainerInset = UIEdgeInsetsZero;
```
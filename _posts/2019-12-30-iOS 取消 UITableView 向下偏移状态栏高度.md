---
layout:     post
title:      iOS 取消 UITableView 向下偏移状态栏高度
subtitle:   iOS 隐藏导航栏后，取消 UITableView 向下偏移状态栏高度
date:       2019-12-30
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
---

iOS 隐藏导航栏后，取消 UITableView 向下偏移状态栏高度

```
if (@available(iOS 11.0, *)) {
    self.tableView.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;
} else {
    self.automaticallyAdjustsScrollViewInsets = NO;
}
```

---
layout:     post
title:      webView 添加头部与底部视图
subtitle:   缓存
date:       2019-06-12
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - UIWebView
---

有时我们用 webView 展示 html 是需要自定义头部视图与底部视图，原生没有像 tableView 那样提供相应的方法。我们只能自己写了。

## 添加头部视图

设置 contentInset，而不是 contentSize。

```
_webView.scrollView.contentInset = UIEdgeInsetsMake(headerHeight, 0, 0, 0);
```

配置区头：

```
//设置header
UIView *headerView = [[UIView alloc]initWithFrame:CGRectMake(0, -headerHeight, kDeviceWidth, headerHeight)];
[_webView.scrollView addSubview:headerView];
```

### 添加头部视图不使用 contentSize 的原因：

设置 contentSize 的时机是在`- (void)webViewDidFinishLoad:(UIWebView *)webView`网页加载完成之后，此时 html 网页是和头部视图重叠在一起，需要 1s 左右时间更新视图，体验不好。而使用 contentInset 不会发生这种现象。

## 添加底部视图

在 webView 加载完成后，获取 contentSize 并重新设置。

```
- (void)webViewDidFinishLoad:(UIWebView *)webView{
	CGSize contentSize = self.webView.scrollView.contentSize;
    //footer
    UIView *footerView = [[UIView alloc]initWithFrame:CGRectMake(0, contentSize.height, kDeviceWidth, footerHeight)];
    self.webView.scrollView.contentSize = CGSizeMake(contentSize.width, contentSize.height + footerHeight);
    [self.webView.scrollView addSubview:footerView];
}
```

## 小结

- 添加头部视图使用 contentInset。

- 添加底部视图在 webView 加载完成后重新设置 contentSize。
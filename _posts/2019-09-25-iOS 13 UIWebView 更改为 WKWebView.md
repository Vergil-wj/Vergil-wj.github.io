---
layout:     post
title:      iOS 13 UIWebView 更改为 WKWebView
subtitle:   iOS 13 适配
date:       2019-09-25
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS 
    - iOS13
---

项目中一直使用的是 UIWebView，虽然 UIWebVeiw 会造成 30M 左右的内存泄漏，但实际对自己的项目影响不大。WKWebView 的坑又很多，所以也没有考虑换过。这次升级 iOS 13 之后 UIWebView 被废弃了，项目也不得不全改为 WKWebView 了，顺便记录下遇到的问题。

## 字体变小

UIWebView 网页显示正常，切换到 WKWebView 后整体字体变小了。解决方法：给服务器返回的 html 字符串拼接一个 \<header>\</header> 就可以了。

```
NSString *headerString = @"<header><meta name='viewport' content='width=device-width, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0, user-scalable=no'></header>";

NSString *htmlStr = [headerString stringByAppendingString:textModel.artText];

[self.webView loadHTMLString:htmlStr baseURL:nil];
```

## 改变 scrollView 的 contentSize 失效

项目中是在网页下方放了一个 tableView 用来展示相关评论。使用 UIWebView 的时候，直接在 UIWebView 中的 scrollView 后面添加 tableView ，然后根据 tableView 高度改变 scrollView 的 contentSize。这种方法在 WKWebView 下失效了，系统会在下一次帧率刷新时，返回原来的。

解决办法：在网页加载完成之后，在网页下面拼接一个空白 div，然后再上面添加 tableView。

```
self.tableView.frame = CGRectMake(0, self.webView.scrollView.contentSize.height, tableViewWidth, tableViewHeight);
NSString *js = [NSString stringWithFormat:@"\
                            var appendDiv = document.getElementById(\"AppAppendDIV\");\
                            if (appendDiv) {\
                            appendDiv.style.height = %@+\"px\";\
                            } else {\
                            var appendDiv = document.createElement(\"div\");\
                            appendDiv.setAttribute(\"id\",\"AppAppendDIV\");\
                            appendDiv.style.width=%@+\"px\";\
                            appendDiv.style.height=%@+\"px\";\
                            document.body.appendChild(appendDiv);\
                            }\
                            ", @(tableViewHeight), @(tableViewWidth-10), @(tableViewHeight)];
[self.webView evaluateJavaScript:js completionHandler:nil];
    
[self.webView.scrollView addSubview:self.tableView];
[self.tableView reloadData];
```

## 水平滚动条

显示水平滚动条的问题是由上面的问题产生的，即增加了一个空白 div，其宽度和 WKWebView 宽度相等，但是会产生水平滚动条。解决办法也很简单，设置新增 div 的宽度小于 WKWebView 的宽度即可。因为我们需要的是这个空白 div 的高来增加网页的高度，从而在网页上添加控件，而 div 的宽是多少无所谓。
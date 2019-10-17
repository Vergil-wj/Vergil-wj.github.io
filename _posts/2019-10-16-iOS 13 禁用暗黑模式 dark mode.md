---
layout:     post
title:      iOS 13 禁用暗黑模式 dark mode
subtitle:   WKWebView
date:       2019-10-10
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - iOS 13
---

项目没有适配暗黑模式的需求，苹果审核也没有强制要求，所以很开心的以为项目这方面不用管了，但是手机系统调到暗黑模式后，项目还是受影响了，这里白一块，那里黑一块。看来还是需要稍微了解下暗黑模式。

### 全局禁用暗黑模式

info.plist 中，新增 User Interface Style 值 为 light。

### 固定某个页面禁用暗黑模式

```
- (UIUserInterfaceStyle)overrideUserInterfaceStyle {

    return UIUserInterfaceStyleLight;
}
```

此方法影响的是当前页面及其子页面，不会影响之前的或者后续推出的页面。

### 参考：

[Human Interface Guidelines
](https://developer.apple.com/design/human-interface-guidelines/ios/visual-design/dark-mode/)

[Is it possible to opt-out of dark mode on iOS 13?](https://stackoverflow.com/questions/56537855/is-it-possible-to-opt-out-of-dark-mode-on-ios-13)
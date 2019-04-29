---
layout:     post
title:      css 中 font-size:62.5% 与 px em rem
subtitle:   css
date:       2019-04-28
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - html
---

写 html 发现别人这么写：

```
html{font-size:62.5%;}
```

这是什么意思，怎么用呢？了解 CSS 中 px、em、rem 这三个长度单位就知道了。

## px

px，像素，绝对长度。

## em

em，相对长度，相对于应用在当前元素的字体尺寸。

一般浏览器字体大小默认为16px。那么我们就可以得出:

```
1em = 16px;
```

如果我们在 css 中声明：

```
html{font-size:62.5%;}
```

则浏览器字体大小更改为：

```
16px * 62.5% = 10px;
```

因为 em 是相对长度，那么此时相对于当前元素尺寸：

```
1em = 10px;
```

这样就方便多了，我们只需要将原来的 px 数值除以10，然后换上 em 作为单位就行了。

但是，em 的缺点也很明显，它会继承父级元素字体的大小，也就是说我们在#content 中声明了字体大小为1.2em（12px），那么在声明它的子元素 p 的字体大小时就需要注意是1em，而不是1.2em, 否则 p 字体大小将变为 

```
1.2 * 1.2 = 1.44em（14.4px）
```

## rem

rem（root em），相对长度，相对于 html 根元素的大小。

rem 很好的避免了 em 的缺点，通过它既可以做到只修改根元素就成比例地调整所有字体大小，又可以避免字体大小逐层复合的连锁反应。由此可见，rem 相对于 em 将是更好的选择。

## 小结

适配不同设备时候，使用相对长度更合适。所以我们在写 css 字体大小时， 使用 rem 就是非常棒的选择。在根元素中设置 `font-size:62.5%`,方便我们 px 与 rem 转换，只需要 px 数值除以10，就是 rem 的值了。


## 参考资料

[https://www.runoob.com/cssref/css-units.html](https://www.runoob.com/cssref/css-units.html)

[https://www.cnblogs.com/leejersey/p/3662612.html](https://www.cnblogs.com/leejersey/p/3662612.html)
---
layout:     post
title:      使用CocoaPods构建自己的公有库
subtitle:   公有库
date:       2019-04-03
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - CocoaPods
---

类似于 AFNetworking、SDWebImage，我们也可以利用 CocoaPods 建立一个自己的公开库。

## 第一步 注册 Trunk

> CocoaPods Trunk is an authentication and CocoaPods API service. To publish new or updated libraries to CocoaPods for public release you will need to be registered with Trunk and have a valid Trunk session on your current device.
> 
> CocoaPods Trunk is available starting with CocoaPods 0.33.

想要发布一个自己库，需要注册 Trunk，且 CocoaPods 版本在0.33及以上。

> We recommend including a description with your session to give some context when you list your sessions later. For example:
> 
> $ pod trunk register orta@cocoapods.org 'Orta Therox' --description='macbook air'

开始注册：

```
$ pod trunk register email(自己的邮箱) 'name（昵称）' --description='macbook pro（随便写点）'
```

按下回车，注册完成！此时你的邮箱会受到一封邮件,打开邮件里面有个链接,需要点击确认一下

接下来输入以下命令，查看注册信息：

`$ pod trunk me`

## 第二步 创建项目

> There are only a few differences between a CocoaPod and a generic open source library. The most important ones, aside from the actual source, are the .podspec and LICENSE.

使用CocoaPods构建公开库需要 **.podspec** 和 **LICENSE**两个文件。

1、github上创建一个项目 **VJUtils** ，并添加 **LICENSE**(通常选择MIT类型) 文件,然后 clone 到本地。

![](https://ws1.sinaimg.cn/large/006tKfTcly1g1ogg0967tj313o0u0n1p.jpg)

将我们准备开源的文件放入刚刚创建的 **VJUtils** 项目中，如图：

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1og7haf06j30my06kgmd.jpg)

其中 **VJUtils** 文件夹是准备开源的文件， **VJUtils.podspec** 文件下文会提到。

2、在工程根目录中初始化一个 *.podspec* 描述文件
输入命令：

`$ pod spec create VJUtils.podspec `

就会出现上图中 *VJUtils.podspec* 描述文件，打开文件并修改描述信息如下图：

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1p6jd8d98j319w0iktbw.jpg)

参数说明：

- s.name：库名，和.podspec名字保持一致。
- s.versin：版本号。
- s.ios.deployment_target：支持最低版本。
- s.summary：简介
- s.homepage：项目主页地址
- s.license：许可证
- s.author：作者
- s.source：项目的地址
- s.source_files：需要包含的源文件
- s.requires_arc：是否支持ARC

其它：

- s.social\_media\_url：社交网址，你的podspec发布成功后会@你

- s.dependency：依赖库，不能依赖未发布的库，例如：

	`s.dependency 'AFNetworking'`
	
未涉及到的请看[官方文档](https://guides.cocoapods.org/syntax/podspec.html)。

## 第三步 上传项目到 GitHub，并打上 tag

1、上传到 GitHub

```
git add .
git commit -m '1.0.0 release'
git push
```

2、打上 tag

```
git tag 1.0.0
git push origin 1.0.0
```

tag 的版本号要和 .podspec 中的版本号保持一致。

## 第四步 验证 .podspec 文件

两种方式验证：

第一种：`$ pod lib lint`

第二种：`$ pod spec lint VJUtils.podspec --verbose`

- verbose：输出详细信息

我使用的第二种，输出结果如下：

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1p7yr6d8uj30ck01sjrm.jpg)

验证成功！如果验证出错，请看文章末尾。

两种方式验证的不同之处：

> The difference between them is that pod lib lint does not access the network, whereas pod spec lint checks the external repo and associated tag.

pod lib lint 是只从本地验证你的 pod 能否通过验证.

pod spec lint 是从本地和远程验证你的 pod 能否通过验证.

## 第五步 发布

输入：

`$ pod trunk push VJUtils.podspec`

会输出一段很长的描述，然后会看到如图成功提示：

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1p8fcny8gj30w807676r.jpg)

## 最后一步！验证！

首先删除 `~/Library/Caches/CocoaPods`目录下的 search_index.json 文件。

然后执行搜索

 `$ pod search VJUtils`
 
 就会出现你要搜索的库了！公有库的创建到此结束。


### 以上步骤中可能出现的错误

1、出现
> ERROR | [iOS] unknown: Encountered an unknown error (Could not find a `ios` simulator (valid values: com.apple.coresimulator.simruntime.ios-12-2, com.apple.coresimulator.simruntime.tvos-12-2, com.apple.coresimulator.simruntime.watchos-5-2). Ensure that Xcode -> Window -> Devices has at least one `ios` simulator listed or otherwise add one.) during validation.

升级CocoaPods:

`$ sudo gem install cocoapods `

2、出现

> warning: inexact rename detection was skipped due to too many files.
> warning: you may want to set your diff.renameLimit variable to at least 29913 and retry the command.

输入以下命令：

```
$ git config merge.renameLimit 999999
$ git config --unset merge.renameLimit

```

3、出现

> [!] There was an error pushing a new version to trunk: execution expired

执行：

`$ pod trunk push VJUtils.podspec`

若还是报这条错误，那就多试几次；


###### 参考资料
> [CocoaPods官网 https://guides.cocoapods.org/making/index.html](https://guides.cocoapods.org/making/index.html)
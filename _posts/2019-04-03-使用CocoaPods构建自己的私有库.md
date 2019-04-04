---
layout:     post
title:      使用CocoaPods构建自己的私有库
subtitle:   私有库
date:       2019-04-04
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - CocoaPods
---
使用 CocoaPods 建立一个私有库，将项目中的公共组件交由它管理，方便于其它项目的引用，也是组件化项目开发的第一步。

整体说明一下创建私有库的步骤：

1. 创建并设置一个私有的 Spec Repo。
2. 创建Pod的所需要的项目工程文件，并且有可访问的项目版本控制地址。
3. 创建 Pod 所对应的 podspec 文件。
4. 上传项目到 GitHub,并打上 tag。
5. 本地测试配置好的 podspec文件是否可用。
6. 向私有的 Spec Repo 中提交 podspec。
7. 在个人项目中的 Podfile 中增加刚刚制作的好的 Pod 并使用。

## 创建私有的 Spec Repo

什么是Spec Repo？

他是所有的 Pods 的一个索引，就是一个容器，所有公开的 Pods 都在这个里面，他实际是一个 Git 仓库 remote 端
在 GitHub 上，但是当你使用了 Cocoapods 后他会被 clone 到本地的 `~/.cocoapods/repos` 目录下，可以进入到这个目录看到 master 文件夹就是这个官方的 Spec Repo 了。这个 master 目录的结构是这个样子的：

```
.
├── Specs
    └── [SPEC_NAME]
        └── [VERSION]
            └── [SPEC_NAME].podspec
```

因此我们需要创建一个类似于 master 的私有 Spec Repo，这里我们可以 fork  官方的 Repo，也可以自己创建，建议不 fork，因为你只是想添加自己的 Pods，没有必要把现有的公开 Pods 都 copy 一份，就如官方文档所说：

> 1.Create a Private Spec Repo
> 
> To work with your collection of private pods, we suggest creating your own Spec repo. This should be in a location that is accessible to all who will use the repo.
> 
> You do not need to fork the CocoaPods/Specs Master repo. Make sure that everyone on your team has access to this repo, but it does not need to be public.

在 github 创建完私有库 *VJRepos* 后，执行下面命令:

`$ pod repo add VJRepos https://github.com/Vergil-wj/VJRepos`

成功的话就会在`~/.cocoapods/repos`目录下看到 *VJRepos* 文件夹了,如下图：

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1pgwg1b23j30zk046t9k.jpg)

## 创建 Pod 的所需要的项目工程文件，并且有可访问的项目版本控制地址

如果是有现有的组件项目，并且在 Git 的版本管理下，那么这一步就算完成了，可以直接进行下一步了。

或者从 github 上新建一个空项目，勾选上 **LICENSE** 文件，并 clone 到本地，将项目中的公共组件拖进来。

## 创建 Pod 所对应的 podspec 文件

在项目根目录下，输入：

`$ pod spec create VJDviceInfo`

会看到项目中，多出一个 .podspec 文件

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1phth4b84j30me04saav.jpg)

打开 .podspec 文件，并修改信息如下：

```
Pod::Spec.new do |s|
  s.name         = "VJDeviceInfo"
  s.version      = "1.0.0"
  s.ios.deployment_target = '8.0'
  s.summary      = "ios device info"
  s.homepage     = "https://github.com/Vergil-wj/VJDeviceInfo"
  s.license      = "MIT"
  s.author       = {  "houweijia" => "houweijiav@163.com"  }
  s.source       = { :git => "https://github.com/Vergil-wj/VJDeviceInfo.git", :tag => s.version }
  s.source_files  = "VJDeviceInfo"
  s.requires_arc = true
end
```

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

## 上传项目到 GitHub,并打上 tag

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

## 本地测试配置好的 podspec 文件是否可用

两种方式验证：

第一种：`$ pod lib lint`

第二种：`$ pod spec lint VJDeviceInfo.podspec --verbose`

- verbose：输出详细信息

我使用的第二种，输出结果如下：

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1qg0x56wwj30ho03s0tj.jpg)

验证成功！如果验证出错，请看文章末尾。

两种方式验证的不同之处：

> The difference between them is that pod lib lint does not access the network, whereas pod spec lint checks the external repo and associated tag.

pod lib lint 是只从本地验证你的 pod 能否通过验证.

pod spec lint 是从本地和远程验证你的 pod 能否通过验证.


## 向私有的 Spec Repo 中提交 podspec

向我们的私有 Spec Repo 提交 podspec 只需要一个命令:

`$ pod repo push VJRepos VJDeviceInfo.podspec`

完成之后这个组件库就添加到我们的私有 Spec Repo 中了，可以进入到`~/.cocoapods/repos/VJRepos`目录下查看:

![](https://ws1.sinaimg.cn/large/006tKfTcly1g1qg6jm332j30ms03mjrw.jpg)

再去看我们的 Spec Repo 远端仓库，也有了一次提交，这个 podspec 也已经被Push上去了。

## 在个人项目中的 Podfile 中增加刚刚制作的好的 Pod 并使用


![](https://ws4.sinaimg.cn/large/006tKfTcly1g1qixbvy3qj30zq0bugpv.jpg)

第一行是官方的 Spec repo，第二行是自己的 Spec reop；

然后打开项目可以看到，我们自己的库文件已经出现在 Pods 子项目中的 Pods 子目录下了。

到此，使用 CocoaPods 创建私有库已经完成，我们可以方便的管理自己的组件了。 

### 以上步骤中可能出现的错误

1、
> ERROR | [iOS] unknown: Encountered an unknown error (Could not find a `ios` simulator (valid values: com.apple.coresimulator.simruntime.ios-12-2, com.apple.coresimulator.simruntime.tvos-12-2, com.apple.coresimulator.simruntime.watchos-5-2). Ensure that Xcode -> Window -> Devices has at least one `ios` simulator listed or otherwise add one.) during validation.

升级CocoaPods:

`$ sudo gem install cocoapods `

2、

> warning: inexact rename detection was skipped due to too many files.
> warning: you may want to set your diff.renameLimit variable to at least 29913 and retry the command.

输入以下命令：

```
$ git config merge.renameLimit 999999
$ git config --unset merge.renameLimit

```

3、

> [!] There was an error pushing a new version to trunk: execution expired

执行：

`$ pod trunk push VJUtils.podspec`

若还是报这条错误，那就多试几次；


###### 参考资料

> [CocoaPods官网 https://guides.cocoapods.org/making/private-cocoapods.html](https://guides.cocoapods.org/making/private-cocoapods.html)

> [http://blog.wtlucky.com/blog/2015/02/26/create-private-podspec/](http://blog.wtlucky.com/blog/2015/02/26/create-private-podspec/)
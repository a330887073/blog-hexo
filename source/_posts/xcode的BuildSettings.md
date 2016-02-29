---
layout: post
title: 关于xcode的buildSettings
date: 2016-2-29
categories: blog
tags: [xcode,buildSetting]
description: 一个关于BuildSetting的点~
---

之前在新项目的自动化打包中碰到的问题，发现在A电脑上可以使用，而在B电脑上总是报“error: Specified application doesn't exist or isn't a bundle directory : 'build/Products/Release-iphoneos/xxx.app”这样的error。很明显这个原因是因为在项目目录下的build文件下找不到相应的.app文件。
但是当时还是有点摸不着头脑的。因为两边的代码一样,打包脚本都是一样的，而且因为用的是cocoapod做的第三方托管，所以相对来说工程文件的差异我当时觉得也是没有的。但是不得不承认问题是出在工程文件上。之后用
```Objective-C xcodebuild -showBuildSettings ```分别查看了下当前项目下的build setting的配置。发现```Objective-C BUILD_DIR ```和 ```Objective-C BUILD_ROOT ```这两项在两台电脑是不一样的，一个是在项目下而另一个是在  ```Objective-C /Users/xxx/Library/Developer/Xcode/DerivedData ```下。
当时就郁闷了，不知道什么时候设置了这个路径。
看了关于BuildSetting的设置之后发现，在xcode->preference->locations里面有个advanced的设置build location 里面的说明也很明了，选择自定义custom 里面的 “relative to workspace”就可以了。其中下面还可以自定义文件路径。
具体如下图所示：

![](http://7xrcp9.com1.z0.glb.clouddn.com/blog1.png =500x)
![](http://7xrcp9.com1.z0.glb.clouddn.com/blog2.png =500x)

而其中的custom中有三类分别是:Relative to Derived Data、Relative to Workspace和Absolute。
其实光从字面意思上也不难理解，最后一个absolute的本身意思就是“不受任何限制[约束]的; 无条件的”。那个就是说是完全可以自定义的。而前两者Derived Data和Workspace我截了下面两张图来做说明，主要的区别在于红色框内，由于本身我就是在这两处地方的路径不能出现了打包的错误。所以这里会具体说明下。

![](http://7xrcp9.com1.z0.glb.clouddn.com/blog3.png =500x)
![](http://7xrcp9.com1.z0.glb.clouddn.com/blog4.png =500x)
看了图片之后其实已经是一目了然了。当选择Relative to Derived Data的时候，打包的build文件会在xcode应用的derivedData下，而选择Relative to Workspace的时候build的文件是在项目目录下的build下。
而之前我所使用的打包脚本，所取的文件地址就是项目目录下的。看完之后才恍然大悟，但是我现在还是没有想起来当时是什么时候修改的这个参数。这个东西平常的时候也不会去关注。
那么既然说到了这个打包时候的build文件，就顺带介绍一下build文件夹下到底有哪些东西。
一般情况下build下面主要的是下面两个文件夹
Intermediates ————主要是编译中产生的一些文件
Products ————编译最终产品的文件（如果是debug下的编译 那么是Debug-xxx，相对的如果是release下的是Release-xxx）。

关于xcode的buildSetting的东西还有很多。
大家可以进到项目文件下 用 ```Objective-C xcodebuild -showBuildSettings ``` 自己去看看 

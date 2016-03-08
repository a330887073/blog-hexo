---
layout: post
title: iOS中如何优雅的应对无限闪退
date: 2016-3-8
categories: blog
tags: [crash,防爆,闪退]
description: 告别无限闪退带来的烦恼
author: YYDD

---

 恩，这是背景。每次更新版本的时候，必须要做的一件事情就是兼容老版本。因为从老版本升级的时候也许会出现数据的兼容性问题。特别是对于数据库的兼容尤为重要。然而如果一不小心兼容性处理的不好，那么就有可能出现无限性的闪退。也许闪退这个动作对于我们开发来说，是很明了的一个现象（app出现问题，导致crash）。但是对于一般的用户而言他们并不知道这到底发生了什么事情。之前和用户接触，他们对于我们所说的闪退他们的描述有很多种：有的说我打开应用就退出来了；有的说我打开应用一下子就回到了桌面，还有的说我打不开应用。由此可以看出用户对于闪退这件事情并不是很理解。有的甚至会以为因为装了这个app导致手机出现了问题。好吧，说了这么多，我只想说明一点就是“无限闪退”是一件很恐怖的事情。
之前我安装的某应用，由于升级版本的时候的数据库升级问题，导致无限闪退，只能卸载app重装。在之前由于在一个项目中，我将错误的数据类型保存在本地，并且没有加类型判断，导致了app无限闪退，这种情况下也只能卸载app，重新安装才能正常运行。
通过这些惨痛的经历，我一不小心想到了“防爆系统”，这个霸气名字，呵呵~。是对于app出现无限闪退的时候，进行的一些操作。
主要的代码就是下面这个方法：

```
static void uncaughtExceptionHandler(NSException *exception) {
	//do something ....
}

```
在这个方法中你可以把闪退的堆栈，原因等记录下来，通过一定的渠道获取（发邮件，上报服务器等）。
下图主要是整个防爆系统的实现逻辑。
![实现逻辑](http://7xrcp9.com1.z0.glb.clouddn.com/blog%E9%98%B2%E7%88%86%E7%B3%BB%E7%BB%9F.png?imageView2/2/w/800/q/75)
其实这个逻辑很简单，主要是一种思路。之前使用手淘的时候，无意中闪退了多次，然后再次进入app的时候我的所有数据都被清除了。所以我在“是否达到了需要清除本地数据的条件”这里是这样设定的，每次闪退必定会记录下闪退的时间，每次进入app判断如果每次闪退的时间间隔为x（例如1分钟）则可以认为上一次以及上上次的闪退为有效记录，再判断如果闪退的有效记录已经达到n次（例如5次）则我认为有可能是本地的持久化数据导致了app出现问题，所以需要清除数据。
当然x和n这两个值，应该根据自己的app取舍。甚至可以在记录闪退数据的时候可以把闪退原因记录下来，对闪退的原因进行比较归类，这样的话就更加精确了。

下面是我的防爆系统中的判断逻辑：

```
//检查并处理闪退的数据
-(void)checkAndDealCrashData
{
    NSDictionary *lastCrashDict = [self.crashArr lastObject];
    long long lastCrashTms = [lastCrashDict[@"crashTms"]longLongValue];
    long long curTms = [self curSystemTms];
    if (curTms - lastCrashTms > maxCrashTimeInterval) {
        //说明 闪退间隔已经超出了 可以理解为不需要记录
        [self cleanCrashArr];
        [self addCrashToData];
    }else
    {
        if (self.crashArr.count >= (maxCrashTimes - 1)) {
            //需要清除数据
            [self cleanAppLocalData];
        }else
        {
            //还没有到达上限
            [self addCrashToData];
        }
    }
}
```

好了，防爆系统的思路就是这些。让我们优雅的面对闪退吧~

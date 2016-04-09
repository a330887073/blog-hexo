---
layout: post
title: ios的mock
date: 2016-04-09 16:01:39
categories: blog
tags: [ios,mock,运行时]
description: 关于ios中的mock的。。。
author: YYDD

---
因为单元测试的缘故，在测试项目中接入了“OCMock”,觉得它的实现很有意思。所以就看了部分“OCMock”的源码。在mock的时候大部分用到的还是运行时的一些特性。之后觉得蛮好玩的，就根据OCMock的大致思路，自己写了一个小小的mock的类。
那么先来介绍一下什么是mock吧。
mock可以理解为纸老虎。当我们写单元测试的时候，不可避免的要去尽可能少的实例化一些具体的组件来保持测试既短又快。而且保持单元的隔离。在现代的面向对象系统中，测试的组件很可能会有几个依赖的对象。我们用mock来替代实例化具体的依赖class。mock是在测试中的一个伪造的有预定义行为的具体对象的替身对象。被测试的组件不知道其中的差异！你的组件是在一个更大的系统中被设计的，你可以很有信心的用mock来测试你的组件。
通过上面的文字其实已经能够说明，mock需要做的事情。一就是吧原有的类被一个mock的类替换掉，再者就是对于该类的方法，进行伪造。
在参考了OCMock的源码之后，发现需要完成这些操作，需要借鉴的主要是运行时的以下方法：
`object_setClass(<#id obj#>, <#__unsafe_unretained Class cls#>)`
这个方法将一个对象设置为别的类,而返回值仍旧是返回原来的类别，具体看下下面的这段代码吧

```` objectivec
    NSString *str = @"hello";
    Class class1 = object_setClass(str, [UIButton class]);

````
在这段代码中实则的作用是将str从NSString类别转化为UIButton类别。而object_setClass的返回值class1还是NSString类别的。
这个方法看似很强大，实际上我认为没有多大的用处，我唯一能够想到的是在子类和父类的转化的时候估计可以用到一下。其他的倒还真是没有想到哪里可以用。
有了上面的这个方法，那么mock一个类可以轻轻松松的搞定了，当然mock一个类还远远不够，当然还需要mock这个类中的方法。对于mock一个类中的方法，主要用到的是消息转发的思想。在mock类中，执行之前没有的方法，那么重定向到一个已经存在的方法，通过一定的trick返回mock得到的值。这样就实现了整个mock的思路。
下面这段主要是在消息转发时候如何处理重定向到一个已经存在的方法：

```` objectivec
-(void)forwardInvocation:(NSInvocation *)anInvocation
{
    BOOL hasMoc = NO;
    SEL selector = anInvocation.selector;

    NSString *selectorName = NSStringFromSelector(selector);

    for (NSString *mocSelectorName in self.mocSelNameLists) {
        if ([mocSelectorName isEqualToString:selectorName]) {
            anInvocation.selector = @selector(getValueInMocMethod:);
            [anInvocation setArgument:&mocSelectorName atIndex:2];
            [anInvocation invokeWithTarget:self];

            hasMoc = YES;
            break;
        }
    }
    
    if (!hasMoc) {
        [super forwardInvocation:anInvocation];
    }

}

````

其中先对需要方法进行筛选，如果是mock的方法，那么对其进行重定向。因为是需要重定向到一个新的方法，那么所以需要改变`NSInvocation`类中的部分属性，主要是指向的方法和传递参数。这里需要注意的是`[anInvocation setArgument:&mocSelectorName atIndex:2];`这边的index必须从2开始，因为0和1分别被targe和selector占用了。
以上介绍的就是mock的主要的思路处的实现了。虽然自己完成了一个mock的小demo，但是mock这个好像除了在测试的时候能够派上用场，在其他地方还想也没什么用武之地了。或者说只是我现在暂时还没有碰到或者想到而已。
不过虽然这么说，但是在研究mock的过程中，对于oc在运行时能够做的事情，有了更深一步的了解。
下面是自己写的demo的git地址：
https://github.com/YYDD/MockDemo


---
title: iOS中一个可以获得参数个数的宏
date: 2016-03-03 16:03:23
tags: [iOS, Macros, ReactiveCocoa]
author: Kael
---

## [metamacro_argcount(...)](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/e16f47cf9cb568136ebd81430b24af274c3c27c7/ReactiveCocoa/Objective-C/extobjc/metamacros.h#L45)


metamacro_argcount一个可以获得传入参数个数的Macro,
```objc
metamacro_argcount(NSObject, version); 结果为2.
metamacro_argcount(NSObject, version, super); 结果为3
metamacro_argcount(NSObject, version, nil); 结果为3
```
这个宏在编译期(并非运行期)就获得参数个数,是不是很神奇?来看看里面有什么鬼
```c
//RACmetamacros.h中
#define metamacro_argcount(...) \
        metamacro_at(20, __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
```
需要展开下,以下就`metamacro_argcount(NSObject, version)`做展开
```c
metamacro_argcount(NSObject, version)
-->
metamacro_at(20, NSObject, version, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
```

<!-- more -->

`metamacro_at(N, ...)`定义:
     
```c
#define metamacro_at(N, ...) \
        metamacro_concat(metamacro_at, N)(__VA_ARGS__)
```
继续展开
```c
metamacro_at(20, NSObject, version, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
-->
 metamacro_concat(metamacro_at,20)(NSObject, version, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1);
```

`metamacro_concat`定义:
```c
#define metamacro_concat(A, B) \
        metamacro_concat_(A, B)
#define metamacro_concat_(A, B) A ## B
```
宏定义中`A ## B`,[`##`](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation)对A,B分割,再将A,B代入拼接,    
所以上文中`metamacro_concat(metamacro_at,20)就变成了metamacro_at20
```c
metamacro_concat(metamacro_at,20)(NSObject, version, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1);
-->
metamacro_at20(NSObject, version, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1);
```
metamacro_at20又是一个宏定义
`metamacro_at##Nth`定义:
```c
#define metamacro_at0...
...
#define metamacro_at19...
#define metamacro_at20(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, ...) metamacro_head(__VA_ARGS__)
```
继续转化 = =...
```c
NSObject对应_0
version对应_1
20对应_2
...
3对应_19

还剩(2,1)就是metamacro_head(__VA_ARGS__)的参数
所以

metamacro_at20(NSObject, version, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1);
-->
metamacro_at20(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, ...) metamacro_head(2,1)

```
`metamacro_head`定义:
```c
#define metamacro_head(...) \
        metamacro_head_(__VA_ARGS__, 0)
#define metamacro_head_(FIRST, ...) FIRST
```
再转化
```c
metamacro_head(2,1)
-->
metamacro_head_(2,1,0) 2
FIRST==2
```
经过各种展开
```c
metamacro_argcount(NSObject, version)
-->
metamacro_head(2,1)
-->
2
```
硬是获得了参数个数2......

大家都很熟悉数组,用数组来类比下metamacro_argcount的功能
```objc
NSArray *array20=@[@20, @19, @18, @17, @16, @15, @14,@13, @12, @11, @10, @9, @8, @7, @6, @5, @4, @3, @2, @1,@0];
array20就取第20个
array20[20]=0默认没有参数
```
当我有参数的时候,我就插入到数组前面,取值就只取第20个元素
插入NSObjcet
```objc
NSArray *array20=@[NSObjcet,@20, @19, @18, @17, @16, @15, @14,@13, @12, @11, @10, @9, @8, @7, @6, @5, @4, @3, @2, @1,@0];
array20[20]的值就变成了@1,就是1个参数
```
再插入version
```objc
NSArray *array20=@[NSObjcet,version,@20, @19, @18, @17, @16, @15, @14,@13, @12, @11, @10, @9, @8, @7, @6, @5, @4, @3, @2, @1,@0];
array20[20]的值就变成了@2,就是2个参数
```

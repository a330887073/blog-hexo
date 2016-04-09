---
title: ReactiveCocoa2-5修炼 (三)
date: 2016-03-02 10:37:18
tags: [iOS,ReactiveCocoa]
author: Kael
---

上一篇做了流转化源码的阅读,再看看流还有哪些用的到的操作,并阅读下源码,
1.concat
2.then
3.merge
4.flatten
<!-- more -->
## concat:
写了简单的例子测试下:
```objc
RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"ABC"];
    //[subscriber sendCompleted];
    return nil;
}];
RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"CBA"];
    return nil;
}];
[[signalA concat:signalB] subscribeNext:^(id x) {
    NSLog(@"x--%@",x);
}];
//输出:2016-03-02 11:15:43.704 RACFunTest[2366:155132] x--ABC
```
只输出了ABC

### 看看concat的源码
```objc
- (RACSignal *)concat:(RACSignal *)signal {
return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
    RACSerialDisposable *serialDisposable = [[RACSerialDisposable alloc] init];
    RACDisposable *sourceDisposable = [self subscribeNext:^(id x) {
        [subscriber sendNext:x];
    } error:^(NSError *error) {
        [subscriber sendError:error];
    } completed:^{
        RACDisposable *concattedDisposable = [signal subscribe:subscriber];
        serialDisposable.disposable = concattedDisposable;
    }];
    serialDisposable.disposable = sourceDisposable;
    return serialDisposable;
}] setNameWithFormat:@"[%@] -concat: %@", self.name, signal];
}
```
可以看到内部创建了一个新的RACSignal,继续看代码`[self subscribeNext:^(id x)`,因为是`[signalA concat:signalB]`,所以这里的self就是signalA,` [subscriber sendNext:x];`,符合输出的结果,在`completed`的block中调用了`[signal subscribe:subscriber]`,原来signalA发了`sendCompleted`signalB才会被`subscribe:`
```objc
//打开被注释掉的代码
[subscriber sendCompleted];
//2016-03-02 11:36:38.561 RACFunTest[2422:163940] x--ABC
//2016-03-02 11:36:38.562 RACFunTest[2422:163940] x--CBA
```
### concat能干嘛?
让信号能按顺序来给订阅者发消息.
A concat B ,A发完了B再发
B concat A ,B发完了A再发
另外结合源码的理解,可以知道concat之后可以继续concat,A concat B concat C
只要B能够发送sendCompleted那么C发送的值就能被订阅者收到.

## then
继续写个栗子:
```objc
[[signalA then:^RACSignal *{
    return signalB;
}] subscribeNext:^(id x) {
    NSLog(@"x--%@",x);
}];
输出:@"CBA"
```
### 看看then的源码
```objc
- (RACSignal *)then:(RACSignal * (^)(void))block {
    NSCParameterAssert(block != nil);
    return [[[self
        ignoreValues]
        concat:[RACSignal defer:block]]
        setNameWithFormat:@"[%@] -then:", self.name];
}
//[self ignoreValues]
- (RACSignal *)ignoreValues {
    return [[self filter:^(id _) {
        return NO;
    }] setNameWithFormat:@"[%@] -ignoreValues", self.name];
}
//[RACSignal defer:block]
+ (RACSignal *)defer:(RACSignal * (^)(void))block {
    NSCParameterAssert(block != NULL);
    return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        return [block() subscribe:subscriber];
    }] setNameWithFormat:@"+defer:"];
}
```
第一步方法的调用,使SignalA的值全过滤掉了,然后concat了signalB,所以跟concat的不同是,只会输出signalB的值,不会输出signalA,而且从concat源码中了解到,必须在signalA发送了sendCompleted,signalB才会起效,所以then操作也是这样,只有前一个结束了,后一个才有效

## merge 
简单栗子:
```objc
RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"A"];
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.01 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [subscriber sendNext:@"A2"];
    });
    return nil;
}];
RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"B"];
    return nil;
}];

[[signalA merge:signalB] subscribeNext:^(id x) {
    NSLog(@"x--%@",x);
}];
输出:
2016-03-02 14:29:57.832 RACFunTest[2935:218636] x--A
2016-03-02 14:29:57.833 RACFunTest[2935:218636] x--B
2016-03-02 14:29:57.860 RACFunTest[2935:218636] x--A2
```
### 看看merge源码
```objc
//在RACSignal+Operations中
- (RACSignal *)merge:(RACSignal *)signal {
    return [[RACSignal
        merge:@[ self, signal ]]
        setNameWithFormat:@"[%@] -merge: %@", self.name, signal];
}
+ (RACSignal *)merge:(id<NSFastEnumeration>)signals {
    NSMutableArray *copiedSignals = [[NSMutableArray alloc] init];
    for (RACSignal *signal in signals) {
        [copiedSignals addObject:signal];
    }
    return [[[RACSignal
        createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
            for (RACSignal *signal in copiedSignals) {
                [subscriber sendNext:signal];
            }
            [subscriber sendCompleted];
            return nil;
        }]
        flatten]
        setNameWithFormat:@"+merge: %@", copiedSignals];
}
```
merge也产生了新的流,当这个流被订阅的时候,他就遍历自己的copiedSignals数组,将signal发送,将signal当做数据发送,那么我们在订阅的时候,`id x`就是一个信号,还需要对x进行订阅才能触发didSubscribe?这里还涉及到另一个操作,`flatten`,

## flatten
```objc
//在RACStream.m中
- (instancetype)flatten {
    __weak RACStream *stream __attribute__((unused)) = self;
    return [[self flattenMap:^(id value) {
        return value;
    }] setNameWithFormat:@"[%@] -flatten", self.name];
}
```
这里的flattenMap逻辑可以按照修炼(二)中的过程走一遍,

从表面上看,flatten作用就是:如果一个SignalB的sendNext:发送了一个signalA,可以将这个signalB做flatten操作后,订阅者看起来就是订阅的signalA.

根据修炼二中的逻辑过一遍的时候,
```objc
id stream = block(value) ?: [class empty];
NSCAssert([stream isKindOfClass:RACStream.class], @"Value returned from -flattenMap: is not a stream: %@", stream);
return stream;
```
回调到这一段代码时,block(value)在flatten方法中就是`return value;`,即signalA
再回到`bind:`方法内,执行`addSignal(signal)`....

不得不感叹,RAC设计者的逻辑真是强大

搞明白了flatten,那么merge中`[subscriber sendNext:signal];`也就明白了,外部订阅者拿到的就是signal发送的

### flatten的作用,merge的作用
*flatten的作用:*SignalY发送signalX,flatten SignalY之后产生signalZ,
订阅signalZ的作用跟订阅signalX的作用一样,还是写个栗子吧

```objc
RACSignal *signalX = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"A"];
    [subscriber sendNext:@"B"];
    return nil;
}];
RACSignal *signalY = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:signalX];
    return nil;
}];

//*不用flatten:*
[signalY subscribeNext:^(id x) {
    RACSignal *signal=x;
    [signal subscribeNext:^(id x) {
        NSLog(@"x--%@",x);
    }];
}];
//*用flatten*
[[signalY flatten] subscribeNext:^(id x) {
        NSLog(@"x--%@",x);
}];

输出:A B
```
*merge的作用*将2个signal合并成一个,任一一个发出信息,订阅者都能接受,



## 总结下
操作:signalA 操作符 signalB
1.concat : 会先输出signalA的值,signalA发送了sendCompleted,signalB才会起效,接着输出signalB
2.then : 不会输出signalA的值,但需要等到signalA执行完了自己的sendNext之后sendCompleted,signalB才会起效,只输出signalB
3.merge : signalA和signalB,按sendNext的执行顺序依次输出
操作:[signalA sendNext:signalB]
4.flatten : 就相当于,解包,订阅flatten之后的Signal跟订阅signalB一样.

## 写了3篇的源码阅读的回顾,谈谈收获了啥.

1.对RACSignal相关的一系列类的父子类关系有了一点了解,RACEmptySignal,来干嘛的呀,
RACReturnSignal是干嘛的呀,这些等等.不然看别人写的RAC看的一脸懵逼
2.之前没想到过block来返回block,(别笑...),等于就是传递IMP,可以来统一接口
3.再看RACSignal的各种方法(`ignore:`,`startWith:`,...)轻松多了,看下源码,再结合源码文档的注释基本知道在干嘛.


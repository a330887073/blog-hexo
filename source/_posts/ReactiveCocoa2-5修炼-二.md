---
title: ReactiveCocoa2-5修炼 (二)
date: 2016-03-01 10:29:01
tags: [iOS,ReactiveCocoa]
author: Kael
---

上一篇阅读了`RACSignal`订阅过程的一些源码,这一篇来看看流的一系列操作Mapping,Filtering
本文主要内容:
1.map操作的源码阅读;(其实是flattenMap:的源码)
2.filter操作
3.空信号RACEmptySignal

## RACStream
`RACStream`是`RACSignal`和`RACSequence`的父类,所以很多方法定义在`RACStream`中,`RACStream`是一个抽象类.     
来看看`RACStream`中定义了些什么东西

```objc
@interface RACStream : NSObject
+ (instancetype)empty;
+ (instancetype)return:(id)value;
- (instancetype)bind:(RACStreamBindBlock (^)(void))block;
- (instancetype)concat:(RACStream *)stream;
- (instancetype)zipWith:(RACStream *)stream;
@end
@interface RACStream (Operations)
- (instancetype)flattenMap:(RACStream * (^)(id value))block;
- (instancetype)flatten;
- (instancetype)map:(id (^)(id value))block;
...........
@end

```


## Map操作
如果把`RACStream`比作是水流的话,Map操作可以是一个个加工工厂,比如一个可乐工厂,水流流进工厂,工厂处理过后,流出了可乐流,在可乐流上接了龙头,打开阀门就可以喝可乐了.

写个栗子:

```objc
RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"ABC"];
    return nil;
}];
RACSignal *mappedSignalA=[signal map:^id(id value) {
   NSString *text=value;
   return @(text.length);
}];
[mappedSignalA subscribeNext:^(id x) {
    NSLog(@"class-%@,value-%@",[x class],x);
}];

[signal subscribeNext:^(id x) {
	NSLog(@"class-%@,value-%@",[x class],x);
}];
输出:
2016-03-01 14:06:29.725 RACFunTest[2703:176158] class-__NSCFNumber,value-3
2016-03-01 14:06:29.725 RACFunTest[2703:176158] class-__NSCFConstantString,value-ABC
```
从栗子中可以看到,流进可乐工厂产生了新的可乐流,而不是将原先的水流全部转化成了可乐流,我还想喝水还能喝.

### 来看看map的实现

```objc     
//RACStream.m中
- (instancetype)map:(id (^)(id value))block {
    NSCParameterAssert(block != nil);
    Class class = self.class;
    return [[self flattenMap:^(id value) {
        return [class return:block(value)];
    }] setNameWithFormat:@"[%@] -map:", self.name];
}
//RACSignal和RACDynamicSignal没有重写map和flattenMap方法
//RACStream.m中
- (instancetype)flattenMap:(RACStream * (^)(id value))block {
    Class class = self.class;
    return [[self bind:^{
        return ^(id value, BOOL *stop) {
            id stream = block(value) ?: [class empty];
            return stream;
        };
    }] setNameWithFormat:@"[%@] -flattenMap:", self.name];
}

//RACSignal.m重写了bind方法
- (RACSignal *)bind:(RACStreamBindBlock (^)(void))block {
	//这个方法返回了一个新创建的RACSignal.
    return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        RACStreamBindBlock bindingBlock = block();
		...(太多不复制,有需要知道这里面的细节再看)...
			}] setNameWithFormat:@"[%@] -bind:", self.name];
}
```
为了看清楚些,转化下源码的代码
```objc
//map方法
- (instancetype)map:(id (^)(id value))block {
    NSCParameterAssert(block != nil);
    Class class = self.class;
    RACStream *(^innerBlock)(id value)=^(id value)
    {
        return [class return:block(value)];
    };
    RACStream *stream=[self flattenMap:innerBlock];
    //-(instancetype)flattenMap:(RACStream * (^)(id value))block;方法
    return [stream setNameWithFormat:@"[%@] -map:", self.name];
}
//flattenMap方法
- (instancetype)flattenMap:(RACStream * (^)(id value))block {
    Class class = self.class;
    RACStreamBindBlock bindBlock = ^(id value, BOOL *stop)
    {
        id stream = block(value) ?: [class empty];
        return stream;
    };
    RACStreamBindBlock (^A_Block_That_Return_A_RACStreamBindBlock)(void)=^RACStreamBindBlock(void)
    {
        return bindBlock;
    };
    RACStream *stream=[self bind:A_Block_That_Return_A_RACStreamBindBlock];
    //- (RACSignal *)bind:(RACStreamBindBlock (^)(void))block;方法
    //typedef RACStream * (^RACStreamBindBlock)(id value, BOOL *stop);
    return [stream setNameWithFormat:@"[%@] -flattenMap:", self.name];
}
```
从转化后的代码可以看到
1.`map:`方法内,调用`flattenMap:`需要一个`RACStream`类型的block  
2.那么把这个block写明白点,命名为innerBlock,然后将innerBlock传给`flattenMap:`方法
3.`flattenMap:`方法内,调用`bind:`方法需要一个`RACStreamBindBlock (^)(void)`类型的block
4.看到`RACStreamBindBlock`是一个返回RACStream类型,参数是value和stop指针的block
5.所以`RACStreamBindBlock (^)(void)`这个block的作用就是返回一个`RACStreamBindBlock`类型的block

恩,map的工作就是传了很多block,好了,来看看`RACSignal`中的`bind`做了什么
```objc
//bind方法
- (RACSignal *)bind:(RACStreamBindBlock (^)(void))block {
     RACSignal *mappedSignalA=[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        //(以下一大段代码就是被保存在了signal的`mappedSignalA`属性中)
        //(订阅逻辑修炼一有介绍)
        RACStreamBindBlock bindingBlock = block();                          
        NSMutableArray *signals = [NSMutableArray arrayWithObject:self];  
        void (^completeSignal)(RACSignal *, RACDisposable *)//一个completeSignal block
        ...
        void (^addSignal)(RACSignal *)//一个addSignal block
        ...
        RACDisposable *bindingDisposable = [self subscribeNext:^(id x) {
                BOOL stop = NO;
                id signal = bindingBlock(x, &stop);
                @autoreleasepool {
                    if (signal != nil) addSignal(signal);
                    if (signal == nil || stop) {
                        [selfDisposable dispose];
                        completeSignal(self, selfDisposable);
                    }
                }
            } error:^(NSError *error) {
                [compoundDisposable dispose];
                [subscriber sendError:error];
            } completed:^{
                @autoreleasepool {
                    completeSignal(self, selfDisposable);
                }
            }];
         selfDisposable.disposable = bindingDisposable;
        }
        return compoundDisposable;
     }
     return [mappedSignalA setNameWithFormat:@"[%@] -bind:", self.name];
}
```
bind就是把blocks埋起来
到这里map的工作完成了,产生了新的RACSignal--栗子中的`mappedSignalA`,
如果不订阅,这一切都是瞎忙
```objc
[signal subscribeNext:^(id x) {
    NSLog(@"class-%@,value-%@",[x class],x);
}];
//subscribeNext的逻辑在修炼(一)中有介绍,
```
map把blocks埋起来,subscribeNext开始挖
1.调用mappedSignalA中的didSubscribe保存的代码
2.第一行代码`RACStreamBindBlock bindingBlock = block()`;这个block就是`A_Block_That_Return_A_RACStreamBindBlock`,从上文的代码中可以看到`A_Block_That_Return_A_RACStreamBindBlock`执行返回`bindBlock`,将`bindBlock`复制给`bindingBlock`
3.运行到`RACDisposable *bindingDisposable = [self subscribeNext:^(id x) {}`,这里的self即栗子中的`signal`,x的值就是@"ABC"
4.执行`id signal = bindingBlock(x, &stop);`即回调bindBlock
```objc
RACStreamBindBlock bindBlock = ^(id value, BOOL *stop)
    {
        id stream = block(value) ?: [class empty];
        return stream;
    };
```
5.执行block(value),即回调innerBlock,value=@"ABC"
```objc
RACStream *(^innerBlock)(id value)=^(id value)
    {
        return [class return:block(value)];
    };
```
6.RACSignal的[class return:block(value)],value=@"ABC"
```objc
+ (RACSignal *)return:(id)value {
    return [RACReturnSignal return:value];
}
```
block(value)即
```objc
NSString *text=value;
return @(text.length);
```
返回@(3)
7.在RACReturnSignal中value=@(3)
```objc
@interface RACReturnSignal ()
@property (nonatomic, strong, readonly) id value;
@end
+ (RACSignal *)return:(id)value {
    RACReturnSignal *signal = [[self alloc] init];
    signal->_value = value;
    return signal;
}
```
返回一个`RACReturnSignal`的实例,value为3;
8.结束了一大串的回调,
回到id signal = bindingBlock(x, &stop);
signal就是一个`RACReturnSignal`的实例,value为3
9.进入判断if (signal != nil) addSignal(signal);
```objc
void (^addSignal)(RACSignal *) = ^(RACSignal *signal) {
    ...(省略)
    RACDisposable *disposable = [signal subscribeNext:^(id x) {
        [subscriber sendNext:x];
    } error:^(NSError *error) {
        [compoundDisposable dispose];
        [subscriber sendError:error];
    } completed:^{
        @autoreleasepool {
            completeSignal(signal, selfDisposable);
        }
    }];
    selfDisposable.disposable = disposable;
};
```
10.可以看到对`RACReturnSignal`的实例订阅,会执行`RACReturnSignal`重写的[subscribe o]方法
```objc
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
    NSCParameterAssert(subscriber != nil);
    return [RACScheduler.subscriptionScheduler schedule:^{
        [subscriber sendNext:self.value];
        [subscriber sendCompleted];
    }];
}
```
返回了self.value即@(3)
11.addSignal这个block中执行[subscriber sendNext:@(3)];
12.mappedSignalA的block被调用 输出:NSNumber,3
```objc
[mappedSignalA subscribeNext:^(id x) {
    NSLog(@"class-%@,value-%@",[x class],x);
}];
```

### 整理下过程
1.对一个`RACSignal`调用`map:`方法后,会产生新的数据流mappedSignal
2.订阅mappedSignal
3.mappedSignal内各种回调,回调到`map:`的处理过程,即栗子中
`NSString *text=value; return @(text.length);`
4.内部新产生一个`RACReturnSignal`类型的signal保存`mapd:`block回调的值
5.`bind:`内订阅`RACReturnSignal`,`RACReturnSignal`内的方法将value返回
6.[subscriber sendNext:x];mappedSignal的订阅者获得结果
其实看似简单的一个map,内部实现起来却不容易.

### 开发中可能遇到的场景:验证密码输入框的字数
密码长度大于6时候,登入按钮才可以点击
```objc
__weak typeof(self) weakSelf = self;
[[self.pwdTextField.rac_textSignal map:^id(id value) {
    NSString *pwd=value;
    return @(pwd.length);
}] subscribeNext:^(id x) {
    NSLog(@"x--%@",x);
    weakSelf.loginBtn.enabled=[x integerValue]>6;
} error:^(NSError *error) {
    NSLog(@"error");//正常情况下这个error是不会被调用的
}];
```

### flattenMap的一个应用场景
上一块代码中的error不会被调用,因为rac_textSignal正常下不会发error,那么我想要流有判断错误的功能怎么办呢,
1.订阅rac_textSignal,再外层重新创建一个会发送error的newSignal,然后订阅newSignal
```objc
__weak typeof(self) weakSelf = self;
RACSignal *newSignal=[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [weakSelf.pwdTextField.rac_textSignal subscribeNext:^(NSString *text) {
        text.length<6?[subscriber sendError:nil]:[subscriber sendNext:@(text.length)];
    }];
    return nil;
}];
[newSignal subscribeNext:^(id x) {
    weakSelf.loginBtn.enabled=YES;
} error:^(NSError *error) {
    weakSelf.loginBtn.enabled=NO;
}];
```
2.用flattenMap
```objc
__weak typeof(self) weakSelf = self;
[[self.pwdTextField.rac_textSignal flattenMap:^RACStream *(NSString *text) {
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        text.length<6?[subscriber sendError:nil]:[subscriber sendNext:@(text.length)];
        return nil;
    }];
}] subscribeNext:^(id x) {
    NSLog(@"x--%@",x);
    weakSelf.loginBtn.enabled=YES;
} error:^(NSError *error) {
    NSLog(@"error");
    weakSelf.loginBtn.enabled=NO;
}];
```

## Filtering
返回YES的值才能被订阅
简单例子:
```objc
//密码长度大于6时候,登入按钮才可以点击
 __weak typeof(self) weakSelf = self;
[[self.pwdTextField.rac_textSignal filter:^BOOL(id value) {
    NSString *text=value;
    return text.length>6;
}] subscribeNext:^(id x) {
    weakSelf.loginBtn.enabled=YES;
}];
```

### 看看filter:内部

```objc
- (instancetype)filter:(BOOL (^)(id value))block {
    Class class = self.class;
    return [[self flattenMap:^ id (id value) {
        if (block(value)) {
            return [class return:value];
        } else {
            return class.empty;
        }
    }] setNameWithFormat:@"[%@] -filter:", self.name];
}
```
有了map的理解基础,filter就很简单了,
关键在于
```objc
if (block(value)) {
    return [class return:value];
} else {
    return class.empty;
}
```
block(value)即例子中的return text.length>6;
如果判断为真,则返回值,
如果判断为假,则返回空信号量

### 空信号量

```objc
+ (RACSignal *)empty {
    return [RACEmptySignal empty];
}
//在RACEmptySignal.m中,subscribe的实现
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
    return [RACScheduler.subscriptionScheduler schedule:^{
        [subscriber sendCompleted];
    }];
}
```
在bind:内部就会调用`completeSignal(self, selfDisposable)`;
```objc
//completeSignal的block
void (^completeSignal)(RACSignal *, RACDisposable *) = ^(RACSignal *signal, RACDisposable *finishedDisposable) {
BOOL removeDisposable = NO;
@synchronized (signals) {
    [signals removeObject:signal];
    if (signals.count == 0) {
        [subscriber sendCompleted];
        [compoundDisposable dispose];
    } else {
        removeDisposable = YES;
    }
}
if (removeDisposable) [compoundDisposable removeDisposable:finishedDisposable];
};
```
从代码中可以看到[subscriber sendCompleted]就这样,类似发送了一个空信号,subscribeNext的block就不会被调用;

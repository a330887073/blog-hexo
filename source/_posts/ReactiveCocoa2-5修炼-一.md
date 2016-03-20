layout: post
title: ReactiveCocoa2.5修炼 (一)
date: 2016-02-28 15:26:02
tags: [iOS,ReactiveCocoa]
author: Kael
---

[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)是一个FRP的思想(函数式编程思想)在Objective-C中的实现框架,因此,在使用过程中,我们会发现RAC的参数都是一个block.到目前为此,我觉得RAC在做项目过程带来最大的便利是对状态能有很好的控制,自然block作为方法参数使代码变得高聚合,方便了阅读.本文主要对RACSignal类源码进行阅读,来弄明白开发过程中想要弄明白的东西.

本文主要内容:
1.RACSignal类的简单使用
2.subscribeNext就能处理数据,看看RAC源码是怎么做的,这个订阅过程是怎么样的

## RACSignal
RAC的核心,如果用RAC写项目,这个家伙应该是项目中使用次数最多的类.

简单使用:
{% codeblock lang:objc %}
	
//创建一个信号
RACSignal *signal=[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@(1)];
	return nil;
}];
//然后订阅
[signal subscribeNext:^(id x) {
	NSLog(@"x--%@",x);
}];

{% endcodeblock %}

可以看到createSignal:的参数就是一个block(一个返回RACDisposable,参数的是实现了RACSubscriber协议的subscriber的block)
subscribeNext:参数也是一个block(一个返回void ,参数是随意对象的block)

<!-- more -->

为了看的清楚些,可以这样写
{% codeblock lang:objc %}
//定义一个返回RACDisposable的block叫didSubscribe,作为createSignal参数
RACDisposable*(^didSubscribe)(id<RACSubscriber> subscriber)=^RACDisposable *(id<RACSubscriber> subscriber)
{
    [subscriber sendNext:@(1)];
    return nil;
};
RACSignal *signal=[RACSignal createSignal:didSubscribe];
//定义一个返回void的block叫nextBlock,作为subscribeNext的参数
void (^nextBlock)(id x)=^(id x)
{
    NSLog(@"x--%@",x);
};
[signal subscribeNext:nextBlock];
{% endcodeblock %}

## 看看源码是怎么写的
### 1.[RACSignal createSignal:]返回Signal
{% codeblock lang:objc %}
//RACSignal.m
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	return [RACDynamicSignal createSignal:didSubscribe];
}
//RACDynamicSignal.m
@interface RACDynamicSignal ()
@property (nonatomic, copy, readonly) RACDisposable * (^didSubscribe)(id<RACSubscriber> subscriber);
@end
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	RACDynamicSignal *signal = [[self alloc] init];
	signal->_didSubscribe = [didSubscribe copy];
	return [signal setNameWithFormat:@"+createSignal:"];
}
{% endcodeblock %}

**可以看到RACSignal实际是调用了子类RACDynamicSignal来创建Signal**

**子类RACDynamicSignal中有一个didSubscribe的block属性,RACDynamicSignal保存了didSubscribe(这段代码并没有起效)**
**所以返回的Signal内部携带了didSubscribe**


### 2.[signal subscribeNext:]做了什么
{% codeblock lang:objc %}
//RACSignal.m
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
	NSCParameterAssert(nextBlock != NULL);
	RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:NULL completed:NULL];
	return [self subscribe:o];
}
//RACSubscriber.m中,(RACSubscriber.h就是id<RACSubscriber> subscriber中RACSubscriber协议的定义者)
@interface RACSubscriber ()
@property (nonatomic, copy) void (^next)(id value);
@property (nonatomic, copy) void (^error)(NSError *error);
@property (nonatomic, copy) void (^completed)(void);
@property (nonatomic, strong, readonly) RACCompoundDisposable *disposable;
@end
+ (instancetype)subscriberWithNext:(void (^)(id x))next error:(void (^)(NSError *error))error completed:(void (^)(void))completed {
	RACSubscriber *subscriber = [[self alloc] init];
	subscriber->_next = [next copy];
	subscriber->_error = [error copy];
	subscriber->_completed = [completed copy];
	return subscriber;
}
//[self subscribe:o]在RACSignal.m
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCAssert(NO, @"This method must be overridden by subclasses");
	return nil;
}
//所以在RACDynamicSignal.m
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);
	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];
	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];
		[disposable addDisposable:schedulingDisposable];
	}
	return disposable;
}
{% endcodeblock %}

**可以看到subscribeNext方法内部,产生了一个新的对象RACSubscriber *o,*o的创建方法中,又将nextBlock调用subscriber->_next,保存在o的next属性中**

**目前为止createSignal在做准备工作,subscribeNext内部的第一步创建了RACSubscriber的实例也是准备工作**

**接来下就是[self subscribe:o]**

**RACSignal的[self subscribe:o]由子类RACDynamicSignal实现,在RACDynamicSignal.m中[self subscribe:o]方法可以看到这一行代码RACDisposable  *innerDisposable = self.didSubscribe(subscriber)--开始执行block了**

### 3.self.didSubscribe(subscriber)
这个didSubscribe被调用了,并且传入了一个subscriber,(这个subscriber内部有一个nextBlock)
{% codeblock lang:objc %}
//didSubscribe被调用
//上文的中didSubscribe定义:
RACDisposable*(^didSubscribe)(id<RACSubscriber> subscriber)=^RACDisposable *(id<RACSubscriber> subscriber)
{
    [subscriber sendNext:@(1)];
    return nil;
};

//RACSubscriber.m
- (void)sendNext:(id)value {
	@synchronized (self) {
		void (^nextBlock)(id) = [self.next copy];
		if (nextBlock == nil) return;
		nextBlock(value);
	}
}
//nextBlock被调用
//上文中的nextBlock被调用
void (^nextBlock)(id x)=^(id x)
{
    NSLog(@"x--%@",x);
};
{% endcodeblock %}

**didSubscribe被调用,会执行到[subscriber sendNext:]方法**
**在RACSubscriber.m的sendNext:方法将保存在自己身上的nextBlock取出(void (^nextBlock)(id) = [self.next copy]),然后执行nextBlock(value);**
**就是这样Signal可以被订阅产生数据**

**senderr和sendcomplete的执行过程和sendNext类似**

## 整理下过程
1.[RACSignal createSignal:didSubscribe]创建信号,signal内保存didSubscribe

2.[signal subscribeNext:nextBlock]信号被订阅,此方法内部产生一个RACSubscriber的实例subscriber,将nextBlock保存在subscriber中.(每次subscribeNext就会产生一个RACSubscriber的实例)

3.[signal subscribeNext:nextBlock]方法内接着调用[self subscribe:subscriber]方法,将保存了nextBlock的subscriber传递,

4.[self subscribe:subscriber]方法内部取出保存在signal中的self.didSubscribe,执行,

5.进入didSubscribe的block回调,即[subscriber sendNext:@(1)];return nil;

6.[subscriber sendNext]将subscriber中保存的nextBlock取出,执行,

7,进入nextBlock的block回调,即NSLog(@"x--%@",x);

## 从阅读源码过程中,我们可以知道的
**1.如果没有被订阅,那么didSubscribe是不会执行的,**

**2.didSubscribe如果不调用send的一系列方法,那么订阅也是没有用的**

## 做一些测试

{% codeblock lang:objc %}
__block unsigned subscriptions=0;
RACDisposable*(^didSubscribe)(id<RACSubscriber> subscriber)=^RACDisposable *(id<RACSubscriber> subscriber)
{
    subscriptions++;
    [subscriber sendNext:@(subscriptions)];
    return nil;
};
RACSignal *signal=[RACSignal createSignal:didSubscribe];

[signal subscribeNext:^(id x) {
   NSLog(@"x--%@",x);
}];
[signal subscribeNext:^(id x) {
    NSLog(@"x--%@",x);
}];
{% endcodeblock %}

输出:     
2016-02-29 15:53:59.239 RACFunTest[11936:604366] x--1    
2016-02-29 15:53:59.240 RACFunTest[11936:604366] x--2

-----------
下次作文内容:信号map,merge,concat,then操作的源码阅读

---
layout: post
title: Spring MVC HTTP请求参数劫持、修改
date: 2016-02-20
categories: blog
tags: [Java web,Spring,Spring MVC,HTTP]
description: 本篇博客主要介绍如何通过Spring MVC修改HTTP请求中的参数
---

由于历史原因，新项目里使用了一个国人写的PageHelper用做Mybatis的查询分页。PageHelper的使用起来是这样的:

```Java
PageHelper.startPage(page, PAGE_SIZE, ORDER_BY)

//后续查询会自动带上skip,limit等
```
乍一看这API设计的还挺简洁，然而问题就出现“简洁”上。简洁的代价是，这里使用了静态调用，如果完全按照作者的思路来当然没有问题，一旦出现订制的需求，这里就嗝屁了！静态方法不能重载！换成其他语言，函数地位高的也许还可以把函数给覆盖掉（例如JavaScript），在Java中，所有的重载都必须要基于对象，这也是为什么需要依赖注入框架和POJO，而不是一个个静态调用，因为要换方法，首先换对象，这是OO语言唯一的真理。

其实能正常使用也就算了，但是PageHelper的**页码居然是从1开始的**！？WTF！？这是哪个入门级程序员写的代码？这是跟全体程序员的直觉对着干啊卧槽。这下好了，静态调用的地方都是写死的，作者没有为页码地址预留配置空间，呵呵。

吐槽结束。

## Spring MVC救场 HandlerMethodArgumentResolver

坑已经遇见了，换方案暂时没空。现在能想到的就是如何AOP一下把这个参数给修正掉。在Controller层换是最直接的，因为客户端总要传page参数嘛，把这个参数+1就好了。

使用Spring他们家的框架，用起来的体验都是非常舒心加放心的，因为Spring的代码在任何有可能留钩子的地方都预留了钩子！轻轻松松，我们就可以找到HandlerMethodArgumentResolver，这个接口的定义如下:


```Java

public interface HandlerMethodArgumentResolver {

	/**
	 * 这个参数要不要解析
	 */
	boolean supportsParameter(MethodParameter parameter);

	/**
	 * 把这个参数给解析了
	 */
	Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception;

}
```

优 雅 。

实现一个Resolver并不难，无非是判断参数名是page，就解析成page+1即可。

## 拦截参数处理 RequestMappingHandlerAdapter

接下来，就是让这个Resolver起作用。网上有一些XML的配置，可以在<context:annotation-config />里面把这个Resolver给配置进去。但是在这个case里，并不能起作用。原因就在于，这个接口本身是为了给那些Spring解析不了的参数预留的，比如一些自定义的对象，所以优先级最低。像page这种基本类型参数，早就被默认的解析器解析完了。

通过Type Hierachy我们可以清楚的看到都有哪些解析器：
![HandlerMethodArgumentResolver的实现类](https://img.alicdn.com/imgextra/i4/56380417/TB2r2C7kpXXXXaxXXXXXXXXXXXX_!!56380417.png)


这么多，还有分页相关的，可以生成Spring的Pageable对象。不过这玩意页码肯定是从0开始的。找一个看着顺眼的Resolver进去，打个断点进去，调试一下，通过调用栈和源码，可以找到Spring设置这些Resolver的位置。

```Java
//RequestMappingHandlerAdapter.java

ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
		invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		invocableMethod.setDataBinderFactory(binderFactory);
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
		
```

也就是说，最终使用的argumentResolvers，是RequestMappingHandlerAdapter设置进去的，我们只需要替换掉这个对象的argumentResolvers即可。为什么不是其他的呢，可以看到这里ServletInvocableHandlerMethod已经是new出来的了，所以我们没法通过注入的方式找到这个对象。如果RequestMappingHandlerAdapter也是new出来的，那我们就还得向上层的调用者继续找。如果这中间，悲催的框架设计者把这里写死了一个new，让我们无法获得这个对象进行定制，那他估计会收到一个issue。

剩下的工作就简单了，在Spring Context加载之后，修改一下这个argumentResolvers就行：

```Java

//某bean，比如说上面的自定义Resolver
//为了防止这个类被lazy load，可以打上@Lazy(false)注解

    @Autowired RequestMappingHandlerAdapter adapter;
    
    @PostConstruct
    public void post() {
    
        List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<>(adapter.getArgumentResolvers());

        List<HandlerMethodArgumentResolver> customResolvers = adapter.getCustomArgumentResolvers();

        if (customResolvers != null) {
            argumentResolvers.removeAll(customResolvers);
            argumentResolvers.addAll(0, customResolvers);
        }

        argumentResolvers.add(0, this);

        adapter.setArgumentResolvers(argumentResolvers);
    }

```

写一个MVC测试，搞定。完整源码见：https://gist.github.com/rightgenius/8699bb4525df1185af90



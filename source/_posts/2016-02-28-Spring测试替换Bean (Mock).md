--
layout: post
title: Spring测试替换注入的Bean (Mock)
date: 2016-02-28
categories: blog
tags: [Spring, Mock bean, Injection]
description: 如何在Spring的测试中，动态的替换自动注入的bean，以方便测试
--

依赖注入框架在诞生之初的初衷，就是为了方便通过替换Bean，替换对象以是的程序有更好的可扩展性。运行时的Bean是通过Xml配置的，只要符合特定的接口，可以随意配置具体的实现类，让程序可以自由装配、修改、拓展、测试。

例如，在测试用例中需要针对一个业务逻辑进行测试，但又不想运行具体的查询，就可以通过替换到对应的查询Bean，某某Repository的实现类，以达到单纯测试业务逻辑的目的。

但Xml一致被人诟病，无法通过编译器进行检查、编写复杂，项目大了之后，很难维护。为了测试一个bean，要专门编写一套替换掉这个Bean的Xml，感觉并没有达到预期的方便。时至今日，Spring已经基本摆脱了Xml，通过注解和动态发现，现在项目里的Xml基本上是为了让老代码正常工作。通过注解解决测试中动态替换Bean的问题，也变的没有那么困难。

## @Configuration，@Bean和@Primary

这两个注解是现代Spring进行对象构建的主要工具，以替代Xml。@Configuration标记在类上，告诉Spring这个类是一个配置类；@Bean方法标记在配置类的方法上，这个方法返回的对象，就是需要进入注入框架的Bean。因此，我们可以在测试用例中，动态的编写一个Mock配置类，以及我们想要Mock的Bean。

```Java
public class SpringMockTest{

    @Configuration
    public static class Mock{
        @Primary
        @Bean
        public UserRepository userRepository(){
            return new UserRepository(){
                //mock things
            };
        }
    }


}
```
通过上面的代码，我们已经可以让我们Mock对象进入了注入系统，但是同一个接口有两个Bean，Spring在真正注入时无从选择，就会报错。因此我们可以在Mock的Bean上打上@Primary注解，从而告诉Spring选择这个Bean。

但此时还没有真正解决问题，因为Spring启动时自动检测到这个配置文件后，___会对所有的Test都产生影响___，可是其他的Test我们不希望使用这个Mock的Bean。

## @Profile和@ActiveProfiles

@Profile就可以解决这个问题，@Profile有点类似于定义配置类的使用场景，只有特定的Profile被启用时，才使用这个@Profile注解的配置类里面提供的Bean。而@ActiveProfiles则是告诉Spring要启动哪些Profile。于是我们可以在测试类中定义特定的profile，例如这个测试类的类名。

```Java
@Runwith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(...) //或者ContextConfiguration
@ActiveProfiles("SpringMockTest")
public class SpringMockTest{

    @Autowired UserService mUserServ;

    @Profile("SpringMockTest")
    @Configuration
    public static class Mock{
        @Primary
        @Bean
        public UserRepository userRepository(){
            return new UserRepository(){
                //mock things
            };
        }
    }
}
```
这样，在这个测试里，UserService里面所使用的UserRepository就是我们动态替换的Mock Bean，我们甚至可以在这个bean的具体方法里进行assert，帮助我们更好的测试。
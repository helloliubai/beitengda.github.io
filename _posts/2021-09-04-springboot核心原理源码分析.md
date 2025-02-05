---
title: springboot核心原理源码分析
date: 2021-09-04
comments: true
categories:
- java
- springboot
tags:
- springboot
- java
---

## 前言

本文旨在从启动流程切入，解释一些通俗常见的问题:
1. spring启动时做了什么，怎么扫描包加载类的?
2. 我们常用的@Configuration @Component @Autowired注解的是在哪一步生效的，如何生效的?
3. 自动配置是如何生效的，为什么业务工程引入一个starter包或者一个类似@EnableXXX这样的注解就可以开启某个功能了?
4. springb的aop动态代理是怎么实现的

## 启动流程概览

spring的启动流程关键步骤都会发送对应的事件，大家看这个事件发送的接口源码，其实从上到下串起来就是大概的流程。下面对流程的详细剖析会体现出哪些节点会触发对应的事件。

```java
public interface SpringApplicationRunListener {
    void starting();

    void environmentPrepared(ConfigurableEnvironment environment);

    void contextPrepared(ConfigurableApplicationContext context);

    void contextLoaded(ConfigurableApplicationContext context);

    void started(ConfigurableApplicationContext context);

    void running(ConfigurableApplicationContext context);

    void failed(ConfigurableApplicationContext context, Throwable exception);
}
```
<!-- more -->

## 启动流程代码

```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        this.configureHeadlessProperty();
        //获取监听器列表,注意这里的监听器叫SpringApplicationRunListeners监听器，有别于业务经常自定义的ApplicationListener监听器
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        //发布事件：starting
        listeners.starting();

        Collection exceptionReporters;
        try {
        
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            
            //读取配置文件到environment对象中
            //发布事件：environmentPrepared
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            
            //打印Banner
            Banner printedBanner = this.printBanner(environment);
            
            //创建应用上下文对象
            context = this.createApplicationContext();
            
            //异常处理器,下文捕获异常时触发
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            
            //准备上下文，把前面创建好的环境变量对象等组装到context对象中,
            //发布事件contextPrepared 和 contextLoaded，这步把启动类的定义加入ben定义列表里
            //过程中会打印下面这两行日志，大家会一定很眼熟
            //Starting RpcDemoServiceApplication on qianshan with PID 16015
            //The following profiles are active: local*/ 
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            
            //最核心的流程（扫描加载出BeanDefinetion定义，实例化Bean都在这里完成）
            this.refreshContext(context);
          
            //无操作
            this.afterRefresh(context, applicationArguments);
            
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }
            
            //发布事件：started
            listeners.started(context);
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            //
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            // 发布事件：running
            listeners.running(context);
            return context;
        } catch (Throwable var9) {
          // 捕获异常，打印错误日志，释放资源 
          // 发布事件：fail
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }
```

## 核心流程 refreshContext

上下文刷新流程主要分为三个阶段
1. 上下文初始化阶段
2. BeanDefinetion获取和处理阶段
3. Bean的实例化阶段

### 核心流程概览

**this.refreshContext(context);** 的实现里，下面看refresh的代码

```java
public void refresh() {
        synchronized(this.startupShutdownMonitor) {
            //启动计时开始，设置相关标志位
            this.prepareRefresh();
            
            //获取工厂对象
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            
            //7788的准备工作
            this.prepareBeanFactory(beanFactory);
          
            try {
                
                this.postProcessBeanFactory(beanFactory);
                
                //执行BeanFactoryPostProcessor扩展点
                /**
                这里最关键的一个类是ConfigurationClassPostProcessor，是在创建容器上下文时生成的，这个类会完成自动配置的实现。大概流程
                1. 扫描BeanDefinetion,第一轮只扫描到启动类，然后根据启动类的@ComponentScan扫描对应的包下的类(如果类上有Component注解则把Bean定义维护在列表里)
                2. 如果有@Configuration修饰的，则进一步处理
                 2.1 判断@Conditional注解的条件是否满足，不满足则跳过
                 2.2 如果类上有@ComponentScans注解，扫描递归处理
                 2.3 如果类上有@Import注解，解析引入的对象
                 2.4 如果类里有@Bean方法，先维护在ConfigurationClass类的成员列表里
                 2.5 执行资源路径下WEB-INF/spring.factories指定的类
                 2.6 根据上一步解析出的ConfigurationClass集合生成对应的BeanDefinition
                **/
                this.invokeBeanFactoryPostProcessors(beanFactory);
                //扫描实现了BeanPostProcessor的Bean
                this.registerBeanPostProcessors(beanFactory);
                
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                
                this.onRefresh();
                
                //扫描注册实现了ApplicationListener接口的监听器
                this.registerListeners();
                
                //完成Bean的实例化
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }
                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }

```

### Bean的创建和初始化(bean的生命周期)

前面的流程准备好了Bean的定义(BeanDefinetion结构)，接下来Bean的生成主要是三个阶段: 创建、属性赋值、初始化

**Bean的创建**

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) {
        BeanWrapper instanceWrapper = null;
        
        // 实例化对象
		if (instanceWrapper == null) {
            // 调用InstantiationAwareBeanPostProcessor接口postProcessBeforeInstantiation方法
            // 如果调用InstantiationAwareBeanPostProcessor接口postProcessAfterInstantiation方法
            //如果上面没有生成代理的实例对象，则走下面的流程正常实例化
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		
		// 单例且正在创建中
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
		  
            //三级缓存，延迟暴露最终对象
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// 初始化
		Object exposedObject = bean;
		try {

            //属性填充时会触发一类特殊的BeanPostProcessor：InstantiationAwareBeanPostProcessor,
            //如AutowiredAnnotationBeanPostProcessor负责扫描@Autowired注解的成员将其注入
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
```

**Bean初始化阶段**

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {

    //回调Aware接口 BeanNameAware BeanFactoryAware
    invokeAwareMethods(beanName, bean);

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
    //执行BeanPostProcessor初始化前接口
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    //执行初始化代码 回调InitializingBean、指定的init_method方法
    invokeInitMethods(beanName, wrappedBean, mbd);
    
    if (mbd == null || !mbd.isSynthetic()) {
    //执行BeanPostProcessor初始化后接口
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

## 常见问题

### sringboot的自动配置是怎么实现的？

底层依赖BeanFactoryProcessor回调实现，核心的实现类是ConfigurationClassPostProcessor，这个类会从启动类开始，扫描指定的包然后递归地解析每个类的定义。

而其中@Import注解很重要，我们经常用类似@EnableXXX的注解来实现某一个功能组件的开启和注入，而@EnableXXX注解通常里面就会@Import对应的外部类或者ImportSelector接口返回的类列表。ConfigurationClassPostProcessor会扫描这些外部类。

我们常用的一些starter包，之所以引用进来就可以直接启用对应的功能，其原理是Springboot启动类注解里默认嵌套引用了这个注解，而里面的
**AutoConfigurationImportSelector**
这个东东，会去加载spring资源目录下spring.factories里定义的启动类,而每个starter包里都会在spring.factories文件里定义配置入口类。
```java

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

### springboot的aop是怎么实现的？

简单来说分两步

1. 先生成切面类和对应的Advice对象
2. 在Bean实例化之后，调用BeanPostProcessor初始化后置接口，会执行代理相关的初始化后置逻辑，返回代理对象。（如果对应的Bean有适用的Advice说明可以被代理）

参考文章：https://blog.csdn.net/woshilijiuyi/article/details/83934407

### 三级缓存和循环依赖

所谓三级缓存指的是下面
```java
    /** 1级缓存 Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** 2级缓存 Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

	/** 3级缓存 Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

二级缓存可以解决循环依赖，但是解决不了动态代理的问题。 因为实例化是提前曝光，
还没有走到动态代理的逻辑，所以获取到的知识原生对象而不是代理对象，Spring选择先暴露一个对象工厂而不是实际对象，在需要注入的时候才触发工厂的生成代理对象逻辑。


### FactoryBean的作用

关于FactoryBean和BeanFactory的区别。一个是Bean一个是Bean容器，共同点是二者都是能生产其他Bean的“factory”。
```java
public interface FactoryBean<T> {
    //生产对象
    T getObject() throws Exception;
    //返回对象类型
    Class<?> getObjectType();
    //是否单例
    boolean isSingleton();
}
```
spring关于Bean的创建初始化的很多地方对bean是否为FactoryBean做判断处理，如果是FactoryBean则创建的Bean其实是FactoryBean的getObject方法返回的对象。这样设计的应用主要有两种。

1. 对接口进行动态代理。比如Mybatis的Mapper，我们只是在工程中定义了一个个接口，但是接口怎么能直接实例化出Bean呢，所以就用到FactoryBean的getObjectType方法，在这里动态代理出实际的对象来。
```java
@Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```
2. 对一些复杂的Bean流程，在创建时依赖很多前置资源或条件的准备，在getObjectType方法里实现所依赖的逻辑。

![](https://gitee.com/geqiandebei/picture/raw/master/2020-12-6/1607270117025-QQ20201205-1.png)
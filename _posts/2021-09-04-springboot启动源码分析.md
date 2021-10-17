---
title: springboot启动源码分析
date: 2021-09-04
comments: true
categories:
- java
- springboot
tags:
- springboot
- java
excerpt: 'springboot关键源码分析'
---

## 全局流程 

Springboot的启动流程中的关键步骤都会发送对应的事件，大家看这个事件发送的接口源码，其实从上到下串起来就是大概的流程。 

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

启动流程代码。

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

## 详细流程

详细流程主要分为三个阶段
1. 上下文初始化阶段
2. BeanDefinetion获取和处理阶段
3. Bean的实例化阶段

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
                1. 扫描BeanDefinetion,第一轮只扫描到启动类，然后根据启动类的@ComponentScan扫描对应的包下的类
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

### Bean创建流程

Bean的生成主要是三个阶段: 创建、属性赋值、初始化

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) {

		
		BeanWrapper instanceWrapper = null;
		
    // 实例化对象
		if (instanceWrapper == null) {
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
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
```



### Bean初始化阶段

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

1. sringboot的自动配置是怎么实现的？

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

2. springboot的动态代理是怎么实现的？

背题的话很多人都能答上来动态代理是通过cglib和jdk动态代理实现的。然而具体过程Juin说不清了。比如说如果一个类方法有多个拦截器(切面逻辑),会有多个代理对象么，如何同时生效呢？

简单来说分两步

1. 先生成切面类和对一个的Advice对象
2. 在Bean实例化之后，调用BeanPostProcessor初始化后置接口，会执行代理相关的初始化后置逻辑，返回代理对象。（如果对应的Bean有适用的Advice说明可以被代理）

参考文章：https://blog.csdn.net/woshilijiuyi/article/details/83934407

3. 三级缓存和循环依赖

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


4. FactoryBean的作用


![](https://gitee.com/geqiandebei/picture/raw/master/2020-12-6/1607270117025-QQ20201205-1.png)
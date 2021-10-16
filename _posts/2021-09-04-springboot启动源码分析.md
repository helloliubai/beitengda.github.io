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
excerpt: '摘要'
---

## 启动源码

### 全局流程 

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
            // //发布事件：running
            listeners.running(context);
            return context;
        } catch (Throwable var9) {
          // 捕获异常，打印错误日志，释放资源，发布应用启动失败事件
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }
```

### 详细流程

看了上面的大概流程，一定还有很多困惑没有答案，先抛出几个问题

1. springboot怎么找到 @Service、@Component @Configuration注解修饰的类并实例化对象的？

2. sringboot的自动配置是怎么实现的？

3. springboot的动态代理是怎么实现的？

答案在核心流程

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
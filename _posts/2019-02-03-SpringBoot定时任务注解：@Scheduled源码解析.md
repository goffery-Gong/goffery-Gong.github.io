---
layout:     post
title:      SpringBoot定时任务注解：@Scheduled源码解析
subtitle:   
date:       2019-02-03
author:     Gong
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - spring
---
### 1. @Scheduled

先看一下@Scheduled源码

```java
package org.springframework.scheduling.annotation;

@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(Schedules.class)
public @interface Scheduled {

	String cron() default "";

	String zone() default "";

	long fixedDelay() default -1;

	String fixedDelayString() default "";

	long fixedRate() default -1;

	String fixedRateString() default "";

	long initialDelay() default -1;

	String initialDelayString() default "";
}
```

支持cron表达式（"cron" expression）、固定频率（fixedRate）、固定延时（fixedDelay） 3种调度方式。

initial-delay : 表示第一次运行前需要延迟的时间，单位是毫秒  
fixed-delay : 表示从上一个任务完成到下一个任务开始的间隔, 单位是毫秒。  
fixed-rate : 表示从上一个任务开始到下一个任务开始的间隔, 单位是毫秒。(如果上一个任务执行超时，则可能是上一个任务执行完成后立即启动下一个任务)
cron : cron 表达式。(定时执行，如果上一次任务执行超时而导致某个定时间隔不能执行，则会顺延下一个定时间隔时间。下一个任务和上一个任务的间隔时间不固定)
区别见图

![img](https://images2015.cnblogs.com/blog/285763/201707/285763-20170717113617206-969853356.png)

### 2. ScheduledAnnotationBeanPostProcessor

`ScheduledAnnotationBeanPostProcesso`r是[@scheduled](https://github.com/scheduled)注解处理类，实现`BeanPostProcessor`接口（`postProcessAfterInitialization`方法实现注解扫描和类实例创建）、`ApplicationContextAware`接口（`setApplicationContext`方法设置当前`ApplicationContext`）、`org.springframework.context. ApplicationListener`（观察者模式，`onApplicationEvent`方法会被回调）,`DisposableBean`接口（destroy方法中进行资源销毁操作）。

`ScheduledAnnotationBeanPostProcessor`中 `postProcessAfterInitialization()`扫描所有[@scheduled](https://github.com/scheduled)注解，区分`cronTasks`、`fixedDelayTasks`、`fixedRateTasks`。

```java
package org.springframework.scheduling.annotation;

public class ScheduledAnnotationBeanPostProcessor
		implements MergedBeanDefinitionPostProcessor, DestructionAwareBeanPostProcessor,
		Ordered, EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, ApplicationContextAware,
		SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {

}
```

### 3. 总体步骤

1. 通过`@EnableScheduling`注解，创建`ScheduledAnnotationBeanPostProcessor`类实例；

   `@EnableScheduling` 注解定义，如下：

   ```java
   package org.springframework.scheduling.annotation;
   
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   @Import(SchedulingConfiguration.class)
   @Documented
   public @interface EnableScheduling {
   
   }
   ```
   Spring Boot项目中 `@EnableScheduling`的魔法就在于 `@Import(SchedulingConfiguration.class)`，看一下 `SchedulingConfiguration`源码：创建了`ScheduledAnnotationBeanPostProcessor`实例

   ```java
   @Configuration
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public class SchedulingConfiguration {
   
   	@Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
   	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   	public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
   		return new ScheduledAnnotationBeanPostProcessor();
   	}
   
   }
   ```

2. `ScheduledAnnotationBeanPostProcessor` 实现`BeanPostProcessor`接口（`postProcessAfterInitialization`方法实现@Scheduled注解扫描和类实例创建）;`postProcessAfterInitialization()`调用`processScheduled(Scheduled scheduled, Method method, Object bean)` 实现了对注解`@Scheduled` 的内容的解析，并将对应的调度任务类型添加到`ScheduledTaskRegistrar` 实例中；最后再集中将这些task放入一个regTasks中。【这个regtask在哪里被消费了？】

   ```java
   ...
   tasks.add(this.registrar.scheduleFixedRateTask(
       new FixedRateTask(runnable, fixedRate, initialDelay)));
   ...
   // Finally register the scheduled tasks
   synchronized (this.scheduledTasks) {
      Set<ScheduledTask> regTasks = this.scheduledTasks
          .computeIfAbsent(bean, key -> new LinkedHashSet<>(4));
      regTasks.addAll(tasks);
   }
   ```

3. spring 启动时，`AbstractApplicationContext`中的`finishRefresh`方法触发所有监视者方法回调：`ScheduledAnnotationBeanPostProcessor`类所实现的`ApplicationListener`类中的`onApplicationEvent()`方法。`onApplicationEvent()`调用==`finishRegistration()`==方法完成==`TaskScheduler`==的初始化（最终调用的是`ConcurrentTaskScheduler`）；

4. 默认的 `ConcurrentTaskScheduler` 计划执行器采用`Executors.newSingleThreadScheduledExecutor()` 实现单线程的执行器。因此，对同一个调度任务的执行总是同一个线程。**如果任务的执行时间超过该任务的下一次执行时间，则会出现任务丢失，**跳过该段时间的任务。上述问题有以下解决办法：

   - 采用异步的方式执行调度任务，配置 Spring 的 `@EnableAsync`，在任务执行的方法上标注 `@Async`
   - 配置任务执行池，采用 `ThreadPoolTaskScheduler.setPoolSize(n)`。 `n` 的数量为 `单个任务执行所需时间 / 任务执行的间隔时间`

问题：

创建了`ConcurrentTaskScheduler` 来执行tasks。但是如何将前面的regTasks和这里的executor联系起来的呢？

![img](https://images0.cnblogs.com/i/580631/201405/181453414212066.png)


### 参考文献

[Spring @EnableScheduling 注解解析](http://tramp.cincout.cn/2017/08/18/spring-task-2017-08-18-spring-boot-enablescheduling-analysis/#ScheduledAnnotationBeanPostProcessor)

[Spring @Scheduled执行原理解析](https://github.com/TFdream/blog/issues/27)
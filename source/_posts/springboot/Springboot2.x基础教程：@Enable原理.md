
---
title: Springboot2.x基础教程：@Enable原理
date: 2023-09-24 13:54:08
tags: springboot
categories: 
- java
---
>上一篇springboot2.x基础教程：@Async开启异步任务我们使用了@EnableAsync注解来启用异步执行。
>SpringBoot框架中@Enable*注解有很多例如：@EnableAspectJAutoProxy、@EnableCaching、@EnableAutoConfiguration、@EnableSwagger2这一章讲讲它背后的原理。
## 几个典型的@Enable*注解
下面贴出@EnableSwagger2、@EnableAsync、@EnableAspectJAutoProxy这三个注解，不难看出这三个注解都使用了@Import注解。
@Import注解支持导入普通的java类,并将其声明成一个bean。
实际@Enable注解就是通过@Import注解的能力实现Bean的注入springboot ioc容器，从而使某些配置生效。

### @EnableScheduling
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({SchedulingConfiguration.class})
@Documented
public @interface EnableScheduling {
}
```
### @EnableAsync
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AsyncConfigurationSelector.class})
public @interface EnableAsync {
    Class<? extends Annotation> annotation() default Annotation.class;
    boolean proxyTargetClass() default false;
    AdviceMode mode() default AdviceMode.PROXY;
    int order() default 2147483647;
}
```
### @EnableAspectJAutoProxy
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AspectJAutoProxyRegistrar.class})
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;

    boolean exposeProxy() default false;
}
```
## @Import注解使用方式
### 允许使用@Configuration注解的类
@EnableScheduling注解的@Import导入的是SchedulingConfiguration类，我们看下该类的实现。
我们可以看到该类使用了@Configuration、@Bean注解。
@Import注解允许使用@Configuration注解的类注入容器中。
```
@Configuration
@Role(2)
public class SchedulingConfiguration {
    public SchedulingConfiguration() {
    }
    @Bean(
        name = {"org.springframework.context.annotation.internalScheduledAnnotationProcessor"}
    )
    @Role(2)
    public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
        return new ScheduledAnnotationBeanPostProcessor();
    }
}
```
### 允许使用实现ImportSelectorj接口的类
@EnableAsync注解的@Import导入的是AsyncConfigurationSelector类，我们看下该类实现。
AsyncConfigurationSelector类实现了ImportSelector接口，重写了selectImports方法。
@Import注解这里会把selectImports返回的全路径的类注入容器中。
```
public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {
    private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME = "org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";

    public AsyncConfigurationSelector() {
    }

    @Nullable
    public String[] selectImports(AdviceMode adviceMode) {
        switch(adviceMode) {
        case PROXY:
            return new String[]{ProxyAsyncConfiguration.class.getName()};
        case ASPECTJ:
            return new String[]{"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration"};
        default:
            return null;
        }
    }
}
```

### 允许是实现了ImportBeanDefinitionRegistrar接口的类
@EnableAspectJAutoProxy注解的@Import导入的是AspectJAutoProxyRegistrar类，我们再看看该类实现。
AspectJAutoProxyRegistrar实现了ImportBeanDefinitionRegistrar接口，ImportBeanDefinitionRegistrar的作用是在运行时自动添加Bean到已有的配置类，并在Spring容器启动时解析生成bean，通过重写方法registerBeanDefinitions。
```
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    AspectJAutoProxyRegistrar() {
    }

    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        AnnotationAttributes enableAspectJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }

            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }
    }
}
```

## @Import注解SpringBoot处理流程源码分析
1. Spring IOC容器初始化的时候会调用AbstractApplicationContext的refresh方法
![spring-refresh](/img/spring-refresh.png)
2. 在refresh里会调用各种BeanFactoryPostProcessor。ConfigurationClassPostProcessor的processConfigBeanDefinitions方法有对@Configuration、@Import等对注解的处理。
![spring-configuration-processor](/img/spring-configuration-processor.png)
3. ConfigurationClassPostProcessor实际内部通过ConfigurationClassParser处理。
4. 我们重点看看ConfigurationClassParser这个类对@Import处理。

## ConfigurationClassParser类源码分析
**核心处理代码在ConfigurationClassParser的processImports方法中，实现了对于本文对于@Import三种使用方式的处理过程。**

```java
  private void processImports(ConfigurationClass configClass, ConfigurationClassParser.SourceClass currentSourceClass, Collection<ConfigurationClassParser.SourceClass> importCandidates, Predicate<String> exclusionFilter, boolean checkForCircularImports) {
	//....省略部分代码
                        ConfigurationClassParser.SourceClass candidate = (ConfigurationClassParser.SourceClass)var6.next();
                        Class candidateClass;
                        if (candidate.isAssignable(ImportSelector.class)) {
							//对于处理ImportSelector注解处理
                            candidateClass = candidate.loadClass();
                            ImportSelector selector = (ImportSelector)ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class, this.environment, this.resourceLoader, this.registry);
                            Predicate<String> selectorFilter = selector.getExclusionFilter();
                            if (selectorFilter != null) {
                                exclusionFilter = exclusionFilter.or(selectorFilter);
                            }
                            if (selector instanceof DeferredImportSelector) {
                                this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector)selector);
                            } else {
                                String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                                Collection<ConfigurationClassParser.SourceClass> importSourceClasses = this.asSourceClasses(importClassNames, exclusionFilter);
                                this.processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
                            }
                        } else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
							//对于ImportBeanDefinitionRegistrar接口处理
                            candidateClass = candidate.loadClass();
                            ImportBeanDefinitionRegistrar registrar = (ImportBeanDefinitionRegistrar)ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class, this.environment, this.resourceLoader, this.registry);
                            configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                        } else {
                            this.importStack.registerImport(currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
							//对于@Configuration注解处理
                            this.processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
                        }
	//....省略部分代码
    }
```
**千里之行，始于足下。这里是SpringBoot教程系列第十一篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**


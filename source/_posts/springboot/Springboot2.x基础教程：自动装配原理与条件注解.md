---
title: Springboot2.x基础教程：自动装配原理与条件注解
date: 2023-09-25 02:01:08
tags: springboot
categories: 
- java
---
> spring Boot采用约定优于配置的方式，大量的减少了配置文件的使用。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。
> 当springboot启动的时候，默认在容器中注入许多AutoCongfigution类。在我们加入spring-boot-stareter-xx时，XXXAutoConfiguration类根据对应的条件，自动选择装配对应的Bean实例注入IOC容器中。

## 先说结论

1. SpringBoot启动的时候加载主配置类，开启了自动配置功能@EnableAutoConfiguration
2. @EnableAutoConfiguration Import的AutoConfigurationImportSelector中代码最终调用SpringFactoriesLoader.loadSpringFactories扫描了Jar包的META-INF/spring.factories文件加载了大量的XXAutoConfiguration类
3. AutoConfiguration类配合Conditonal注解与ConfigurationProperties配置类在特定条件下自动装配我们需要的Bean到IOC容器中。


## 注入AutoConfiguration类核心源码分析

SpringBoot的主启动类上需要加入@SpringBootApplication注解，我们看看该注解的源码。

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
	//...省略
}
```

实际是@EnableAutoConfigurtaion注解起作用。

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
	Class<?>[] exclude() default {};
	String[] excludeName() default {};

}
```

看到@Import注解是不是很熟悉，该注解作用见教程@Enable原理，主要能够导入下面3种情况中的Bean。

1. 允许注入使用@Configuration注解的类
2. 允许使用实现ImportSelectorj接口的类,注入selectImports返回的类
3. 允许是实现了ImportBeanDefinitionRegistrar接口的类

AutoConfigurationImportSelector中selectImports源码

```java
	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
```

其中**AutoConfigurationImportSelector.getAutoConfigurationEntry**调用**AutoConfigurationImportSelector.getCandidateConfigurations**调用**SpringFactoriesLoader.loadFactoryNames**调用**SpringFactoriesLoader.loadSpringFactories**。

其中**SpringFactoriesLoader.loadSpringFactories**从指定的配置文件META-INF/spring.factories加载配置。

```java
    private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            try {
                Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
                LinkedMultiValueMap result = new LinkedMultiValueMap();

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        String factoryTypeName = ((String)entry.getKey()).trim();
                        String[] var9 = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                        int var10 = var9.length;

                        for(int var11 = 0; var11 < var10; ++var11) {
                            String factoryImplementationName = var9[var11];
                            result.add(factoryTypeName, factoryImplementationName.trim());
                        }
                    }
                }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var13) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var13);
            }
        }
    }
```

总结：
1. @SpringBootApplication中@EnableAutoConfiguration注解最终调用SpringFactoriesLoader.loadSpringFactories，从classpath中搜寻所有的META-INF/spring.factories配置文件
2. 并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项通过反射实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。

## 条件注解

在spring-boot-autoconfigure包在META-INF/spring.factories文件中autoconfiguration配置项一览。

![image-20200909232847766](https://pan.codehome.vip/images/iuCdkPvFtxMyO2B.png)

我们拿DataSourceAutoConfiguration源码分析，看看autoconfiguration类生效的条件。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

	@Configuration(proxyBeanMethods = false)
	@Conditional(EmbeddedDatabaseCondition.class)
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	@Import(EmbeddedDataSourceConfiguration.class)
	protected static class EmbeddedDatabaseConfiguration {

	}

	@Configuration(proxyBeanMethods = false)
	@Conditional(PooledDataSourceCondition.class)
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
			DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
			DataSourceJmxConfiguration.class })
	protected static class PooledDataSourceConfiguration {

	}
	//省略....

}

```

在DataSourceAutoConfiguration类中，存在大量的@ConditionalXX条件注解，常见条件注解作用：

1. @ConditionalOnBean：当SpringIoc容器内存在指定Bean的条件
2. @ConditionalOnClass：当SpringIoc容器内存在指定Class的条件
3. @ConditionalOnExpression：基于SpEL表达式作为判断条件
4. @ConditionalOnJava：基于JVM版本作为判断条件
5. @ConditionalOnJndi：在JNDI存在时查找指定的位置
6. @ConditionalOnMissingBean：当SpringIoc容器内不存在指定Bean的条件
7. @ConditionalOnMissingClass：当SpringIoc容器内不存在指定Class的条件
8. @ConditionalOnNotWebApplication：当前项目不是Web项目的条件
9. @ConditionalOnProperty：指定的属性是否有指定的值
10. @ConditionalOnResource：类路径是否有指定的值
11. @ConditionalOnSingleCandidate：当指定Bean在SpringIoc容器内只有一个，或者虽然有多个但是指定首选的Bean
12. @ConditionalOnWebApplication：当前项目是Web项目的条件
13. @EnableConfigurationProperties注解：使使用 @ConfigurationProperties 注解的类生效。
我们可以看在DataSourceAutoConfiguration激活了DataSourceProperties配置，并且根据条件注解在不同情况下加载不同的数据源到IOC容器中。

**千里之行，始于足下。这里是SpringBoot教程系列第十七篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。觉得不错可以评论、点赞、转发3连。**